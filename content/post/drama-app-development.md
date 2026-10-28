---
title: "从零手写一个 Android 视频 App：登录态、JSON API、VideoView 播放全记录"
date: 2026-10-28
categories: ["项目日志"]
tags: ["Android", "Java", "SQLite", "VideoView", "移动开发"]
---

> This is a full retrospective of Drama, an online video streaming Android app I built from scratch: Java on Android Studio, SQLite for local accounts, a third-party JSON API for searchable video lists, VideoView for playback, and a foreground Service for playing notifications. It covers the MVP layering, the async network thread, login-state persistence, and the bugs that actually cost me nights.

## 一、前言：看起来是个"小视频 App"，坑全在细节里

学完 Android 课程后，我给自己定的课设目标是做一个**能真跑起来的在线视频 App**：用户注册登录、搜索视频、点列表就播、播放不被切后台打断。需求听着简单——不就是 CRUD + 一个播放器吗？真做起来才发现，每个模块都藏着 Android 的老规矩：**主线程不能碰网络、数据库要封 DAO、登录态要持久化、播放要配 Service 保命**。

这篇复盘我把 Drama 这个 App 从架构到实现完整过一遍，代码都是当时写的原版。

## 二、环境与架构：MVP 三层怎么落到包结构

开发栈很朴素：Android Studio + JDK 1.8 + Java，本地存储用 SQLite，网络请求原生 `HttpURLConnection` + JSON 解析，图片加载用 Glide。

| 层 | 职责 | 落地 |
|---|---|---|
| 渲染层 | 界面展示 | LoginActivity / MainActivity / PlayerActivity / ProfileActivity + xml 布局 |
| 逻辑层 | 业务交互 | 各 Activity 内处理点击、搜索、跳转 |
| 数据层 | 存储与网络 | DatabaseHelper / UserDao（SQLite）+ NetworkUtil（API 请求） |

![Drama App 的 MVP 三层架构：渲染层 → 逻辑层 → 数据层 → 网络](/images/drama-mvp-architecture.png)

*渲染层是四个 Activity；数据层管 SQLite 存储、Service 和网络请求，网络请求走远程 JSON API*

包结构也按这个思路拆：`activity / adapter / db / model / service / util` 六个包，数据相关全归 `db`（DatabaseHelper + UserDao）和 `util`（NetworkUtil），不跟 Activity 混在一起。这么分的好处是**换页面不动数据层**：以后把 SQLite 换成网络登录，只需要改 db 包，四个 Activity 一行都不用动。

功能上收敛成四大块：注册登录、视频列表（搜索 + 下拉刷新）、视频播放（全屏 + 后台保活）、"我的"（信息展示 + 退出）。首页默认搜索词写死"爱情"，用户可以随时改关键词重新搜——所有请求都打同一个远程 JSON API，只是关键词参数不同。

## 三、Step 1：账号体系——SQLite 建表 + DAO 封装

用户信息存在 App 自带的 SQLite 里。建表语句很简单，用户名加唯一约束防止重复注册：

```sql
CREATE TABLE users(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL
);
```

DAO 层把数据库操作封死在 `UserDao` 里——插入用户返回新 ID，按用户名查用户，外部只面对 `User` 这个模型类，不直接写 SQL：

```java
public User getUser(String username) {
    SQLiteDatabase db = dbHelper.getReadableDatabase();
    Cursor cursor = db.query(DatabaseHelper.TABLE_USERS,
        new String[]{COLUMN_ID, COLUMN_USERNAME, COLUMN_PASSWORD},
        COLUMN_USERNAME + "=?", new String[]{username},
        null, null, null);
    User user = null;
    if (cursor != null && cursor.moveToFirst()) {
        user = new User(cursor.getString(...), cursor.getString(...));
        cursor.close();
    }
    db.close();
    return user;
}
```

登录流程：用户名密码非空校验 → DAO 查库 → 密码比对成功就把用户名写进 SharedPreferences（`user_info`），下次启动直接跳进主页，不用重复登录；失败弹 Toast。退出登录时清掉 SharedPreferences 并清空返回栈，防返回键偷溜回去。

## 四、Step 2：视频列表——子线程请求 API，主线程刷 UI

这是全 App 最关键的一条链路。Android 4.0 之后**主线程禁止访问网络**，所以网络请求必须开子线程，结果再通过 Handler 抛回主线程更新列表：

```java
public static void getDramaList(String keyword, final NetworkCallback<List<Drama>> callback) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                String kw = URLEncoder.encode(keyword, "UTF-8");
                URL url = new URL("https://api.kuleu.com/api/action?text=" + kw);
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                BufferedReader reader = new BufferedReader(
                    new InputStreamReader(conn.getInputStream()));
                // 读响应...
                JSONObject json = new JSONObject(response.toString());
                if (json.getInt("code") == 200) {
                    JSONArray arr = json.getJSONArray("data");
                    List<Drama> list = new ArrayList<>();
                    for (int i = 0; i < arr.length(); i++) {
                        JSONObject item = arr.getJSONObject(i);
                        list.add(new Drama(item.getString("name"),
                                item.getString("viewlink"), item.getString("addtime")));
                    }
                    // 切回主线程回调
                    new Handler(Looper.getMainLooper()).post(() -> callback.onSuccess(list));
                }
            } catch (Exception e) {
                new Handler(Looper.getMainLooper()).post(() -> callback.onError(e.getMessage()));
            }
        }
    }).start();
}
```

主界面用 RecyclerView 卡片列表展示结果，搜索栏回车或点按钮触发查询，默认关键词是"爱情"，SwipeRefreshLayout 下拉刷新重新拉数据。Glide 负责把封面图异步加载进每个 item：

![视频列表页：搜索"旅行"后卡片展示剧名、时间与播放入口](/images/drama-video-list.png)

*列表 item = 封面图 + 剧名 + 添加时间 + 播放入口；点击 item 随机取一个视频链接跳进播放页*

## 五、Step 3：播放页——VideoView + 前台 Service 保命

播放用 Android 原生的 VideoView，内置 MediaController（暂停/快进/进度条），加载期间显示 ProgressBar，`onPrepared` 就绪后自动起播：

```java
videoView.setVideoURI(Uri.parse(dramaLink));
videoView.setMediaController(new MediaController(this));
videoView.setOnPreparedListener(mp -> {
    progressBar.setVisibility(View.GONE);
    videoView.start();
});
```

光有 VideoView 还不够——用户切到别的 App，播放界面退到后台，系统随时可能回收资源。所以播放时同时拉起一个**前台 Service**（DramaService）：发一条"正在播放"的常驻通知，点通知能跳回 App，通知栏带暂停按钮。Service 里还挂了个每 5 分钟跑一次的定时任务，后台刷新视频列表，数据更新后发广播通知主界面。

![视频播放界面：全屏播放游轮视频，底部 MediaController 控制条](/images/drama-player-screen.png)

*播放器默认全屏，支持暂停、快进快退、进度拖动；后台由前台 Service 维持，切应用不中断*

## 六、"我的"页与测试

"我的"页从 SharedPreferences 读当前用户名展示，配头像、观看历史、我的收藏、关于我们四个入口和退出登录按钮（带二次确认对话框）：

![我的页面：头像、用户名、功能入口与退出登录](/images/drama-profile-page.png)

测试环境搭在模拟器上：JDK 1.8 + Android SDK（API 35 模拟器），AndroidManifest 里注册权限和 Service。功能逐项过——注册后登录正常、搜索可用、下拉刷新有效、视频能播能暂停能快退、退出登录生效；为了结果普遍，换不同模拟器反复跑了多轮。

## 七、踩坑实录

**坑 1：NetworkOnMainThreadException 直接闪退**
图省事把网络请求写在主线程，一点搜索 App 就崩。这不是 bug 是 Android 的硬规矩——4.0 后主线程联网直接抛异常。解法就是上面那套：子线程请求 + Handler 回主线程刷 UI，回调接口把成功/失败分出去。

**坑 2：数据库没关游标，查一次卡一次**
早期 `getUser` 查完忘了 `cursor.close()` 和 `db.close()`，频繁登录操作后数据库连接堆积、界面变卡。Cursor 用完必须关，SQLiteOpenHelper 单例管理连接，别到处 new。

**坑 3：切后台播放无声无息被杀**
只做了 VideoView，没做 Service——用户切出 App，播放很快被系统回收。加了前台 Service 和常驻通知后，播放状态才算真正"活着"。Android 8.0 以上还得走 `startForegroundService()` 并建通知渠道，否则前台服务起不来。

## 八、总结与思考

Drama 最终交付了完整闭环：注册登录 → 搜索列表 → 在线播放 → 个人中心，代码在模拟器上多轮测试通过。回头看，这个小 App 麻雀虽小五脏俱全——**网络异步、本地存储、界面解耦、后台服务、登录态**，Android 开发的核心组件基本都摸到了。

当然短板也很实在：视频封面写死成同一张图、用户头像不能个性化、观看历史和收藏还没做、UI 偏粗糙。真要往下走，下一步就是把 Glide 换成真实封面 URL、用 Room 替代裸 SQLite、把原生 HttpURLConnection 换成成熟网络库。

---

*本文整理自课程设计《Drama 系统的设计与实现》，代码片段为项目原版代码，接口与测试结果均为文档记录。*
