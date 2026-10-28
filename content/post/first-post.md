---
title: 从 0 到 1 上线个人博客：Hugo + GitHub Pages 部署踩坑全记录
date: 2026-07-24T00:00:00.000+08:00
categories:
  - 学习随笔
tags:
  - Hugo
  - GitHub Pages
  - GitHub Actions
  - 博客建站
  - Decap CMS
---
\### 前言



\> \*\*Abstract\*\*: This post is a real-world record of building a personal blog from scratch with Hugo and GitHub Pages at zero cost. It covers the full pipeline — local site generation, theme setup, GitHub Actions auto-deploy — and, more importantly, the pitfalls I actually hit: GitHub connection timeouts, rejected pushes caused by concurrent CMS commits, and CMS login pain. If you plan to build your own static site, the troubleshooting section alone is worth your time.



这是我的第一篇博客的完全重写版。最早那版只有三行命令，今天把整个部署过程重新讲一遍，\*\*包括我实际踩过的所有坑\*\*。目标很简单：用零成本的方式，搭一个属于自己的个人网站，既能写学习笔记，也能当作品集。



\## 为什么选 Hugo



选型时对比过 WordPress、Hexo、Hugo，最终选了 Hugo：



\* 生成纯静态页面，加载快，\*\*不需要买服务器\**，托管和域名都免费

\* 写作只用 Markdown，专注内容本身

\* 构建速度快，一套配置 + GitHub Actions 就能全自动部署



!\[个人博客技术栈](/images/tech-stack.svg)



完整技术栈如下：



\| 层 | 工具 | 用途 |

\|---|---|---|

\| 静态站点 | Hugo | 站点生成器，Markdown → HTML |

\| 主题 | PaperMod → Stack v4.0.3 | 极简风格（后期全站风格统一时迁移） |

\| 托管 | GitHub Pages | 免费静态托管，\`ljy501.github.io\` |

\| CI/CD | GitHub Actions | push 后自动构建部署 |

\| 内容管理 | Decap CMS | 网页后台写文章、传图 |

\| 前端 | Tailwind CSS（CDN） | 首页自定义与全站导航样式 |



\## 搭建过程



\### 1. 安装 Hugo 并创建站点



Windows 下直接下载官方 exe 放到 \`bin\` 目录，或走包管理器：



\`\``bash

\# 创建站点（Windows 下用 hugo.exe）

hugo new site blog

cd blog



\# 安装 PaperMod 主题

git clone https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

\`\``



\### 2. 配置 hugo.toml



设置站点标题、语言、导航菜单、主题：



\`\``toml

baseURL = "https://ljy501.github.io/"

languageCode = "zh-cn"

title = "Liu Junyu's Blog"

theme = "PaperMod"



\[menu.main]

  \[[menu.main]]

\    name = "首页"

\    url = "/"

\    weight = 1

  \[[menu.main]]

\    name = "关于"

\    url = "/about/"

\    weight = 2

\`\``



\### 3. 本地预览与构建



\`\``bash

hugo server -D        # 本地预览，浏览器开 http://localhost:1313

hugo --minify         # 正式构建，输出到 public/

\`\``



\### 4. 推送 GitHub + 自动部署



初始化仓库，推送后配置 Actions 工作流（\`hugo.yml\`）：



\`\``yaml

name: Deploy Hugo site to Pages

on:

  push:

\    branches: \[main]

permissions:

  contents: read

  pages: write

jobs:

  build:

\    runs-on: ubuntu-latest

\    steps:

\- uses: actions/checkout@v4

\- uses: peaceiris/actions-hugo@v3

\    with:

\    hugo-version: 'latest'

\    extended: true

\- run: hugo --minify

\- uses: actions/upload-pages-artifact@v3

\    with:

\    path: ./public

  deploy:

\    needs: build

\    runs-on: ubuntu-latest

\    environment:

\    name: github-pages

\    steps:

\- id: deployment

\    uses: actions/deploy-pages@v4

\`\``



之后每次 \`git push\`，GitHub Actions 自动构建并部署，一分钟后新内容上线。



\## 踩坑实录（这篇文章的灵魂）



\### 坑 1：GitHub 直连超时，push 永远失败



第一个大坑：\`git push\` 直连 github.com \*\*稳定超时\*\*（21 秒连接不上），偶尔还 Connection reset。国内网络环境直连 GitHub 基本不可用。



\*\*解决\*\*：走本地代理。用 Clash 的混合端口（127.0.0.1:7897）作为 git 代理：



\`\``bash

\# PowerShell 设置环境变量后 push

$env:HTTPS_PROXY='http://127.0.0.1:7897'

$env:HTTP_PROXY='http://127.0.0.1:7897'

git push origin main

\`\``



\> 注意：Clash 没启动时端口不存在，\`git push\` 会报 \`Could not connect to server\`；启动后要等核心进程（mihomo）真正监听 7897 再操作。另外节点失效时会报 \`SSL connection failed\`，换一个可用节点即可。



\### 坑 2：push 被拒绝（rejected），因为后台也在提交



用 Decap CMS 后台写了一篇文章后，本地 \`git push\` 直接报：



\`\``

! \[rejected] main -> main (fetch first)

\`\``



原因：\*\*后台的提交已经领先本地\*\*，远程有我本地没有的提交。



\*\*解决\*\*：先拉取重放再推：



\`\``bash

git pull --rebase origin main

git push origin main

\`\``



\`--rebase\` 会把本地提交重放到远程最新提交之上，避免产生多余的合并节点。这是我后来每次推送的固定动作，先 pull --rebase 再 push。



\### 坑 3：Decap CMS 后台登录折腾



后台用的是 Netlify 的 git-gateway + GitHub OAuth。初期登录遇到：



\- 邮箱密码登录提示 \`invalid_grant: No user found with that email, or password invalid\`（密码不对）

\- 触发密码重置，登录又跳到博客首页而不是后台

\- 反复几次才用对的密码进到后台



\*\*关键结论\*\*：后台写文章是\*\*浏览器端直连 GitHub API\*\*，走的是 Netlify 的服务，\*\*不需要本地代理\*\*。也就是说，日常写文章完全可以在后台完成，只有动图片、布局、样式这类后台管不到的文件时才需要本地 push（这时才需要代理）。



\### 坑 4：主题从 PaperMod 迁移到 Stack



初期用 PaperMod，后来想要更现代的全站风格（顶部导航、米白背景、深蓝主色、无侧边栏），迁移到 Stack v4.0.3 并做了定制：



\* 用 \`layouts/baseof.html\` 覆盖主题默认布局：去掉左右侧边栏，非首页统一渲染顶部导航

\* 配色变量 accent 覆盖为深蓝 \`#1e3a8a\`，浅色背景改米白 \`#faf9f7\`

\* 首页用 Tailwind 从零重构：像素风头像 + 呼吸灯效果 + 胶囊标签 + 双语简介 + CTA 按钮

\* 一个小坑：用空 block 定义覆盖侧边栏对列表页不生效，得在 body 加 class 再用 CSS 隐藏



\## 现在的样子



截至这篇文章重写，博客已经上线了：



\* 全站统一风格：顶部导航 + 米白深蓝配色，暗色模式保留深蓝渐变

\* 首页双语个人介绍（像素头像 + 呼吸灯效果）

\* 14+ 篇文章：项目日志、学习随笔、课程报告，配图 50+ 张

\* 发布链路双通道：\*\*后台写文章（免代理）\*\* / \*\*本地推文件（需代理）\**



\## 总结与思考



几个可以复用的经验：



1. \*\*静态站 + 托管 = 零成本博客\*\*：Hugo + GitHub Pages + Actions，全链路免费，构建部署全自动

2. \*\*网络问题不可怕，找对通道就行\*\*：GitHub 直连不行就走代理；后台写文章压根不需要本地网络

3. \*\*双通道发布最舒服\*\*：日常内容更新用网页后台，涉及代码/样式/资源文件才走 git，分工清楚，互不阻塞

4. \*\*踩坑要记录\*\*：这次踩的每个坑（代理、rebase、CMS 登录）都变成了这篇文章的内容，写博客最大的受益者其实是自己



博客是我的个人作品集，项目记录、学习笔记和课程报告，中文为主、配英文摘要。后续会持续更新：\*\*我做过什么、能做什么、怎么思考问题\*\*。欢迎常来。
