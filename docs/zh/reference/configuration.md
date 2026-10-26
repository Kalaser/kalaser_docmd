---
title: 配置参考
date: 2026-10-26
description: docmd.config.json 完整配置项说明
tags: [参考, 配置]
---

# 配置参考 ⚙️

本文档详细说明 `docmd.config.json` 中所有可用的配置项。

---

## 核心配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `title` | string | — | 站点标题 |
| `url` | string | — | 站点部署地址 |
| `src` | string | `"docs"` | 文档源码目录 |
| `out` | string | `"site"` | 构建输出目录 |

---

## 主题配置

```json
{
  "theme": {
    "name": "default"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 主题名称，当前支持 `"default"` |

---

## 国际化 (i18n)

```json
{
  "i18n": {
    "default": "zh",
    "locales": [
      { "id": "zh", "label": "中文" },
      { "id": "en", "label": "English" }
    ]
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `default` | string | ✅ | 默认语言 ID |
| `locales` | array | ✅ | 支持的语言列表 |
| `locales[].id` | string | ✅ | 语言标识符（如 `"zh"`） |
| `locales[].label` | string | ✅ | 显示名称（如 `"中文"`） |
| `locales[].dir` | string | ❌ | 文本方向，默认 `"ltr"` |

### 多语言目录结构

```
docs/
├── zh/          # 中文内容
│   ├── index.md
│   └── guide.md
└── en/          # 英文内容
    ├── index.md
    └── guide.md
```

---

## 导航栏

```json
{
  "navigation": [
    {
      "title": "首页",
      "path": "/",
      "icon": "home"
    },
    {
      "title": "指南",
      "icon": "book",
      "collapsible": true,
      "children": [
        { "title": "快速开始", "path": "/guides/getting-started" },
        { "title": "写作规范", "path": "/guides/writing-docs" }
      ]
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `title` | string | 导航项标题 |
| `path` | string | 页面路径（内部页）或 `external:` 前缀（外部链接） |
| `icon` | string | 图标名称 |
| `collapsible` | boolean | 是否可折叠（仅父级） |
| `children` | array | 子导航项 |

### 支持的图标

`home` · `book` · `code` · `cpu` · `brain` · `rocket` · `feather` · `bot` · `search` · `settings`

---

## 插件配置

### SEO 插件

```json
{
  "plugins": {
    "seo": {
      "defaultDescription": "基于 docmd 构建的中文技术文档站。"
    }
  }
}
```

---

## 完整配置示例

```json
{
  "title": "我的文档",
  "url": "https://yourname.github.io/your-repo",
  "src": "docs",
  "out": "site",
  "theme": {
    "name": "default"
  },
  "i18n": {
    "default": "zh",
    "locales": [
      { "id": "zh", "label": "中文" }
    ]
  },
  "plugins": {
    "seo": {
      "defaultDescription": "中文技术文档站。"
    }
  },
  "navigation": [
    { "title": "首页", "path": "/", "icon": "home" }
  ]
}
```
