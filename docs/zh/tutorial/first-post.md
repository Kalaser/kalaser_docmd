---
title: 第一个文档
date: 2026-10-26
description: 从零开始创建你的第一篇文档，手把手教程
tags: [教程, 入门]
---

# 第一个文档 🎯

本教程将带你从零开始，完成第一篇文档的创建与发布。

---

## 第一步：创建文件

在 `docs/zh/` 目录下新建文件 `my-first-post.md`：

```bash
touch docs/zh/my-first-post.md
```

---

## 第二步：编写内容

编辑文件，添加以下内容：

```markdown
---
title: 这是我的第一篇文档
date: 2026-10-26
description: 第一次使用 docmd 写文档的体验
tags: [教程, 入门]
---

# 这是我的第一篇文档 🎉

欢迎！这是我在 [我的文档站](/) 上发布的第一篇文章。

## 内容示例

docmd 支持标准的 Markdown 语法：

- **加粗** 和 *斜体*
- [链接](https://docmd.io)
- 代码 `inline` 或块状：

```python
print("Hello, docmd!")
```

## 下一步

探索更多页面，创建属于你的文档体系！
```

---

## 第三步：本地预览

启动开发服务器：

```bash
npm run dev
```

访问 `http://localhost:3000/my-first-post` 查看效果。

---

## 第四步：添加到导航

编辑 `docmd.config.json`，在 `navigation` 中添加：

```json
{
  "title": "我的文档",
  "path": "/my-first-post"
}
```

---

## 第五步：提交发布

```bash
git add docs/zh/my-first-post.md
git commit -m "新增文档：这是我的第一篇文档"
git push origin main
```

推送后 GitHub Actions 自动构建并部署。

---

## 常见问题

**Q: 为什么页面没有出现在导航中？**

A: docmd 的自动导航基于文件目录结构。如需手动控制，请在 `docmd.config.json` 的 `navigation` 中显式添加。

**Q: 如何修改文章顺序？**

A: 在文件名前添加数字前缀，如 `01-getting-started.md`、`02-writing-docs.md`。

**Q: 支持哪些 Markdown 扩展？**

A: docmd 支持 GFM（GitHub Flavored Markdown）、表格、任务列表、Mermaid 图表等。

---

## 接下来

- [写作规范](../guides/writing-docs) — 了解完整的写作规范
- [配置参考](../reference/configuration) — 深入学习配置项
- [快速开始](../guides/getting-started) — 回顾整体流程
