# EthanKIEZ.github.io

Ethan 的个人博客，使用 [Jekyll](https://jekyllrb.com/) + [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) 主题搭建，托管于 [GitHub Pages](https://pages.github.com/)。

## 本地运行

```bash
bundle install
bundle exec jekyll serve
```

访问 <http://localhost:4000> 预览。

## 发布新文章

在 `_posts/` 目录下新建 Markdown 文件，文件名格式为 `YYYY-MM-DD-title.md`，并在文件头部添加 YAML Front Matter：

```markdown
---
title: "文章标题"
date: 2026-06-16 12:00:00 +0800
categories:
  - 分类名
tags:
  - 标签1
  - 标签2
---

正文内容...
```

## 目录结构

```
.
├── _config.yml          # 站点配置
├── _data/
│   ├── navigation.yml   # 顶部导航
│   └── ui-text.yml      # 界面文案（中文）
├── _pages/              # 独立页面
├── _posts/              # 博客文章
├── assets/              # 图片、样式等资源
├── Gemfile              # Ruby 依赖
└── index.html           # 首页
```

## 自定义配置

- 站点信息：修改 `_config.yml`
- 导航菜单：修改 `_data/navigation.yml`
- 关于页面：修改 `_pages/about.md`
