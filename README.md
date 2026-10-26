# 我的文档站 📚

基于 [docmd](https://github.com/docmd-io/docmd) 构建的中文技术文档站点，专为 AI 代理优化。

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](LICENSE)
[![docmd](https://img.shields.io/badge/built%20with-docmd-6365d6.svg?style=flat-square)](https://docmd.io)
[![GitHub Pages](https://img.shields.io/badge/deploy-github%20pages-blue?style=flat-square&logo=github)](https://github.com/Kalaser/kalaser_docmd)
[![Node.js](https://img.shields.io/badge/node-%3E%3D22-green?style=flat-square&logo=node.js)](https://nodejs.org)

---

## 🌐 访问地址

> 文档站将部署至：`https://kalaser.github.io/kalaser_docmd/`

---

## ✨ 特性

- **AI 优先** — 自动生成 `llms.txt` 上下文，无缝对接 AI 代理与 MCP 服务器
- **零配置起步** — 只需 Markdown，开箱即用
- **多语言支持** — 内置 i18n，当前默认中文界面
- **快速构建** — esbuild 驱动，毫秒级生成静态站点
- **免费托管** — GitHub Actions 自动部署至 GitHub Pages

---

## 🚀 快速开始

### 本地开发

```bash
# 安装依赖
npm install

# 启动开发服务器（热更新）
npm run dev

# 构建生产版本
npm run build
```

构建产物输出至 `site/` 目录，可直接预览或部署。

### 添加新页面

在 `docs/zh/` 目录下创建新的 `.md` 文件，docmd 会自动将其纳入站点导航。

```
docs/zh/
├── index.md      # 首页
└── your-page.md  # 新页面
```

---

## 📂 项目结构

```
kalaser_docmd/
├── .github/workflows/docs.yml  # GitHub Actions 自动部署
├── .gitignore
├── .gitattributes
├── docmd.config.json           # 站点配置（标题、URL、i18n、导航）
├── package.json                # 依赖与脚本
├── README.md                   # 本文件
└── docs/
    └── zh/
        └── index.md            # 中文首页
```

---

## ⚙️ 配置说明

编辑 `docmd.config.json` 即可定制站点：

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `title` | 站点标题 | `"我的文档"` |
| `url` | 站点 URL | `"https://kalaser.github.io/kalaser_docmd"` |
| `src` | 文档源码目录 | `"docs"` |
| `out` | 构建输出目录 | `"site"` |
| `i18n.default` | 默认语言 | `"zh"` |
| `i18n.locales` | 支持的语言列表 | `[{"id": "zh", "label": "中文"}]` |
| `navigation` | 侧边栏导航 | `[{"title": "首页", "path": "/"}]` |

---

## 🔄 自动部署

本项目通过 GitHub Actions 实现自动部署：

1. 推送到 `main` 分支 → 触发 `docs.yml` 工作流
2. Actions 安装依赖并构建站点
3. 自动部署至 GitHub Pages

> 首次使用需前往 **Settings → Pages → Source** 选择 **GitHub Actions**。

---

## 🛠 技术栈

- **[docmd](https://docmd.io)** — 零配置文档引擎
- **[Node.js](https://nodejs.org)** ≥ 22
- **[GitHub Pages](https://pages.github.com/)** — 静态托管
- **[GitHub Actions](https://github.com/features/actions)** — CI/CD

---

## 📄 License

[MIT](LICENSE) © Kalaser
