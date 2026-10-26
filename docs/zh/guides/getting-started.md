---
title: 快速开始
date: 2026-10-26
description: 5 分钟搭建你的第一个中文文档站点
tags: [入门, 指南]
---

# 快速开始 🚀

在本指南中，你将学会如何使用 docmd 搭建一个专业的中文技术文档站。

---

## 前置条件

- Node.js ≥ 22
- GitHub 账号（用于托管和部署）

## 安装依赖

```bash
npm install
```

## 本地开发

```bash
npm run dev
```

启动后访问 `http://localhost:3000` 即可预览。

## 构建发布

```bash
npm run build
```

构建产物输出至 `site/` 目录，GitHub Actions 会自动将其部署到 GitHub Pages。

---

## 接下来

- [写作规范](./writing-docs) — Markdown 语法与 frontmatter 指南
- [配置参考](../reference/configuration) — 完整配置项说明
- [第一个文档](../../tutorial/first-post) — 动手写你的第一篇文档
