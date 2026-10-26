---
title: 写作规范
date: 2026-10-26
description: Markdown 语法、frontmatter 字段与内容组织规范
tags: [指南, 规范]
---

# 写作规范 ✍️

本文档约定了本项目中文档的写作规范，请遵循这些规则保持一致性。

---

## 文件命名

| 规则 | 示例 |
|------|------|
| 使用英文小写 + 连字符 | `getting-started.md` |
| 禁止中文、空格、特殊字符 | ❌ `我的文档.md` |
| 层级不超过 3 层 | ✅ `guides/quick-start.md` |

---

## Frontmatter 字段

每篇文档顶部必须包含 YAML frontmatter：

```yaml
---
title: 文档标题
date: 2026-10-26
description: 简短描述，用于 SEO（50 字以内）
tags: [标签1, 标签2]
sidebar: 导航组名称
---
```

### 必填字段

| 字段 | 说明 | 示例 |
|------|------|------|
| `title` | 页面标题 | `"快速开始"` |

### 可选字段

| 字段 | 说明 | 示例 |
|------|------|------|
| `date` | 发布日期 | `"2026-10-26"` |
| `description` | SEO 描述 | `"5 分钟搭建文档站"` |
| `tags` | 标签数组 | `[入门, 指南]` |
| `sidebar` | 侧边栏分组名 | `"指南"` |

---

## 标题层级

- 一级标题 `#` — 页面主标题（唯一）
- 二级标题 `##` — 章节
- 三级标题 `###` — 子章节
- 最多使用到四级标题 `####`

---

## 代码块

始终指定语言标识：

```python
# ✅ 正确
print("Hello, world!")

# ❌ 错误（无语言标识）
print("Hello, world!")
```

---

## 内部链接

使用相对路径，不带 `.md` 后缀：

```markdown
<!-- ✅ 正确 -->
[快速开始](../guides/getting-started)

<!-- ❌ 错误 -->
[快速开始](../guides/getting-started.md)
```

---

## 图片引用

使用绝对路径或 GitHub raw 链接：

```markdown
![描述](https://raw.githubusercontent.com/Kalaser/kalaser_docmd/main/assets/screenshot.png)
```

---

## 调用提示框

docmd 支持丰富的容器组件：

````markdown
::: callout info "提示"
这是一条信息提示。
:::

::: callout warning "警告"
请注意安全风险。
:::

::: callout danger "危险"
高危操作，请谨慎执行。
:::
:::
````

---

## 按钮

````markdown
::: button "点击这里" https://example.com icon:external
:::
````

---

## 检查清单

发布前请确认：

- [ ] 标题准确且唯一
- [ ] 无拼写错误
- [ ] 代码块指定了语言
- [ ] 内部链接可正常跳转
- [ ] 图片可以正常加载
