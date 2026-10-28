---
title: "用 JSP Model2 从零手写一个 B2C 蛋糕商城：Servlet + JDBC 的一次完整闭环"
date: 2026-10-28
categories: ["项目日志"]
tags: ["JavaWeb", "JSP", "Servlet", "MySQL", "Tomcat"]
---

> This article walks through building a B2C online cake store from scratch on the classic JavaWeb stack — JSP Model2, raw Servlet, JDBC and a C3P0 connection pool on MySQL. No frameworks, no scaffolding: the whole request flow, from a browser buying a cake to the admin backend managing orders, is implemented by hand.

## 一、前言：电商看起来简单，链路其实不短

电商网站是个"麻雀虽小、五脏俱全"的练手项目：前端要注册登录、商品分类、模糊搜索、购物车、下单；后台要管商品、管订单、管客户、管分类，还要一个推荐条幅。它不像一个 Hello World 那样两三行跑通，但又没有微服务、消息队列那些重型组件——**刚好能把一条完整的 Web 请求链路走通**。

我做这个项目的目标很明确：不引入 Spring 那一套，就用最原始的 Servlet + JSP + JDBC，把"浏览器发请求 → 后端处理 → 数据库读写 → 页面渲染"这条链路在脑子里彻底打通。下面按我实际做的顺序复盘一遍。

## 二、环境与架构：一张图看懂分层

技术栈非常朴素，全是 JavaWeb 的"老配方"：

| 组件 | 用途 | 说明 |
|---|---|---|
| JSP / HTML / JSTL | 视图层 | 页面布局 + 标签库渲染数据 |
| Servlet | 控制层 | 接收请求、调用 Service、跳转视图 |
| Java Bean (Model) | 模型层 | 封装 user/goods/order 等实体 |
| JDBC + C3P0 | 持久层 | DAO 直连 MySQL，连接池复用连接 |
| Tomcat | 容器 | 运行 Servlet，端口 8080 |
| MySQL | 数据库 | 库名 `cookieshop`，6 张核心表 |

设计上对齐经典的 **JSP Model2（MVC）**：浏览器把请求交给 Servlet，Servlet 实例化 Model、调 Service/DAO 操作数据库，再选一个 JSP 视图把结果渲染回去。

![JSP Model2 请求处理流程](/images/cakeshop-mvc-arch.png)

*浏览器 → Servlet（控制）→ Model + DAO/Service（业务与数据）→ JSP（视图）→ 数据库*

整个系统分前台用户侧和后台管理侧，两边共用同一套数据库：

![蛋糕商城前后台功能架构](/images/cakeshop-system-arch.png)

*用户侧管购物，管理侧管商品/订单/客户/分类，最终都落到数据库*

## 三、核心实现：从建库到下单

**Step 1：先把数据库和连接池搭好**

在 MySQL 里建 `cookieshop` 库，一共 6 张表：

| 表 | 作用 |
|---|---|
| `user` | 前台/后台用户信息（含 `isadmin`、`isvalidate` 两个 bit 位） |
| `goods` | 商品（名称、封面、详情图、价格、库存、所属分类） |
| `order` | 订单主表（总额、数量、支付状态、收货地址、下单时间） |
| `orderitem` | 订单项明细（某订单里买了哪个商品、多少件、单价） |
| `type` | 商品分类 |
| `recommend` | 首页推荐条幅里挂的商品 |

商品表大致长这样：

```sql
CREATE TABLE goods (
  id    INT(12) PRIMARY KEY,
  name  VARCHAR(45),
  price FLOAT,
  intro VARCHAR(300),
  stock INT(1),
  type_id INT(11)
);
```

数据库连接不写裸 `DriverManager`，而是用 C3P0 连接池。参数集中放在 `c3p0-config.xml`：

```xml
<c3p0-config>
  <named-config name="cookieshop">
    <property name="driverClass">com.mysql.cj.jdbc.Driver</property>
    <property name="jdbcUrl">jdbc:mysql://localhost:3306/cookieshop</property>
    <property name="user">root</property>
    <property name="password">123456</property>
    <property name="initialPoolSize">5</property>
  </named-config>
</c3p0-config>
```

再封一个 `DataSourceUtils` 统一对外拿连接，业务代码就不用每次关心连接从哪来。

**Step 2：用 Filter 统一全站编码**

中文乱码是 JavaWeb 入门第一个必踩的坑——POST 表单和响应头的编码得一起管。直接上一个全站 Filter：

```java
public class EncodingFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        req.setCharacterEncoding("UTF-8");
        resp.setContentType("text/html;charset=UTF-8");
        chain.doFilter(req, resp);   // 放行到下一个 Filter 或目标 Servlet
    }
}
```

在 `web.xml` 里把它映射到所有请求，后面每个 Servlet 就不用再重复写编码设置。

**Step 3：一个请求的完整闭环——注册下单**

以后台最常见的注册为例，一个请求要走 Servlet → Service → DAO 三层：

```java
public class UserRegisterServlet extends HttpServlet {
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        User user = new User();
        user.setUsername(req.getParameter("username"));
        user.setPassword(req.getParameter("password"));
        user.setEmail(req.getParameter("email"));
        new UserService().register(user);   // Service 再调 UserDao 写库
        resp.sendRedirect("/user_login.jsp");
    }
}
```

购物车则是另一套逻辑：把商品塞进当前订单，加购走 `GoodsBuyServlet`、删除走 `GoodsDeleteServlet`，最后在 `goods_cart.jsp` 里用 JSTL 遍历购物清单、统计金额。商品搜索按蛋糕名做模糊查询，分类导航按 `type_id` 过滤。

为了不重复写首页那个推荐条幅，页面里用 JSP 静态包含（`<%@ include %>`）把它抽成公共片段，其他页面直接引入，改一处全站生效。

## 四、踩坑实录

**坑 1：Tomcat 启动卡 45 秒后报连接池超时**

现象：项目一启动就卡住，半分钟多后抛出 C3P0 初始化超时异常。原因很简单——**启动 Tomcat 时 MySQL 进程没在跑**，连接池拿不到任何连接，干等到超时才报错。解决：先确认 `mysqld` 已经起来，再启动 Tomcat。这类"环境顺序"问题最耗调试时间，记住一句话：连接池报错先看数据库活着没。

**坑 2：后台传过来的到底是字符串还是文件？**

商品添加时，表单里既有名称、价格这种普通字段，也有商品图片这种文件上传。Servlet 里得分别处理：普通字段直接 `getParameter` 写库；文件字段要用 Java I/O 把上传流读出来落盘，再把图片路径存进 `goods` 表。混在一起处理就会出现"文字存进去了、图却丢了"。

**坑 3：删改类操作要先定位主键**

不管是商品修改、删除，还是订单状态变更，核心都是先拿到那条记录的主键 `id`，再按 id 去表里更新或删除，最后重定向回列表页刷新状态。绕开主键直接按名字删，很容易误伤同名记录。

## 五、功能与展示

前台 + 后台一共 19 项功能，全部跑通：

| 模块 | 实现的功能 |
|---|---|
| 用户 | 注册、登录、个人信息查看 |
| 商品 | 分类浏览、按名模糊搜索、热销/新品推荐条幅、加入购物车 |
| 交易 | 购物车增删、在线下单、查看我的订单 |
| 后台-商品 | 增删改、加入/移出推荐栏 |
| 后台-订单 | 修改订单状态、删除订单 |
| 后台-客户/分类 | 客户增删改、重置密码、分类增删改 |

首页长这样，顶部是分类导航和搜索框，输入"芝士"还会带出模糊提示：

![商城首页与搜索模糊提示](/images/cakeshop-home-search.png)

*首页轮播 + 导航栏 + 搜索框，输入关键字即时提示匹配商品*

点进"热销"页，就是一整屏蛋糕商品，每个都能一键加入购物车：

![热销商品列表页](/images/cakeshop-hot-goods.png)

*localhost:8080 下的热销商品列表，价格、库存、加购按钮齐全*

## 六、总结与思考

这次手写下来，最大的收获不是某个功能，而是**把 MVC 三层在脑子里串成了一条线**：请求进来永远是 Servlet 接、Service 编排、DAO 落库、JSP 出页。C3P0 连接池、Filter 编码、JSP 静态包含这些"基础设施"，在框架里被封装得看不见，自己手写一遍才知道它们到底在替你做什么。

当然它还很初级：密码是明文存的、没有登录态拦截之外的权限模型、也没有打折和优惠券。要往生产走，下一步该补的是密码加密、登录拦截器、分页，以及把现在到处手写的 JDBC 换成更安全的参数化查询——这也是下一个阶段可以继续打磨的方向。

---

*本文整理自一次 JavaWeb 课程设计实践，功能与界面均为本地 Tomcat + MySQL 实测截图。*
