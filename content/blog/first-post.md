---
title: "我的博客上线了：用 Hugo + PaperMod 搭建个人博客"
date: 2026-09-24
categories: ["学习随笔"]
tags: ["Hugo", "GitHub Pages", "博客建站"]
---

这是我的第一篇博客。目标很简单：**用零成本的方式，搭一个属于自己的个人网站**，既能写进 CV，也能记录学习过程。

## 为什么选 Hugo

- 生成的是纯静态页面，加载快，**不需要买服务器**
- 写作只用 Markdown，专注内容
- 配合 GitHub Pages 免费托管，域名都是免费的

技术栈如下：

![个人博客技术栈](/images/tech-stack.svg)

## 搭建过程

核心就三步：

```bash
# 1. 创建站点
hugo new site blog

# 2. 安装主题
git clone https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

# 3. 本地预览
hugo server
```

之后配置 `hugo.toml`、写页面、推送到 GitHub，配上 Actions 自动部署，就完成了。

## 为什么要有自己的博客

> 招生官和面试官看到的，不只是简历上的几句话，而是你能持续产出、持续学习的能力。

写博客逼着我把学过的知识讲清楚——**能写明白，才是真学会**。

后续我会在这里记录：项目日志、学习随笔、课程报告。第一篇就到这里，欢迎常来。
