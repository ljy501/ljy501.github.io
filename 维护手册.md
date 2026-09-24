# 博客维护手册

> 项目位置：`D:\blog`（2026-09-24 从 C 盘迁移，线上网站不受影响）

## 1. 写一篇新文章

1. 在 `D:\blog\content\blog\` 新建一个 `.md` 文件，文件名建议用英文（如 `second-post.md`）
2. 文件开头写模板（front matter）：

```markdown
---
title: "文章标题"
date: 2026-09-24
categories: ["学习随笔"]        # 可选：项目日志 / 学习随笔 / 课程报告
tags: ["标签1", "标签2"]
---

正文内容，用 Markdown 写...
```

3. 图片：把图片放进 `D:\blog\static\images\`，正文里写 `![图片说明](/images/文件名.png)`

## 2. 本地预览

```
cd D:\blog
hugo server
```

浏览器打开 http://localhost:1313 查看。改文件会自动刷新。

## 3. 发布上线

```
cd D:\blog
git add -A
git commit -m "写本次改动的说明"
git push
```

**重要**：本机直连 github.com 不通，推送前先开代理：

```
set HTTPS_PROXY=http://127.0.0.1:7897
set HTTP_PROXY=http://127.0.0.1:7897
```

推送后等 1-2 分钟，GitHub Actions 自动构建部署，线上自动更新。

## 4. 改网站设置

| 想改什么 | 编辑文件 |
|---|---|
| 标题/导航/主题开关 | `D:\blog\hugo.toml` |
| 首页简介 | `D:\blog\content\_index.md` |
| About 内容 | `D:\blog\content\about.md` |
| Contact 邮箱/GitHub | `D:\blog\content\contact.md` |

改完按第 2、3 步预览 + 推送即可。

## 5. 其他

- 预览服务器关闭：在跑 `hugo server` 的窗口按 `Ctrl+C`
- 电脑重启后预览需重新运行第 2 步命令
- 文章删除：直接删掉对应 `.md` 文件再推送
- 域名：以后想绑独立域名（提速国内访问），可加 Cloudflare CDN，到时再更新此手册
