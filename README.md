# Aurelio的个人博客（sheng-blog）

一个基于 [Astro](https://astro.build/) 与 [astro-theme-pure](https://github.com/cworld1/astro-theme-pure) 主题二次开发、部署在 [Vercel](https://vercel.com/) 上的个人博客。

站点默认语言为简体中文，同时支持中英双语内容。

> 本仓库为 `astro-theme-pure` 的 fork/二次开发版本，在保留主题原有能力的基础上，按个人需求改造了首页、关于页、项目页、友链页与评论系统等内容。

博客地址：Aurelio-blog[https://sheng-blog-pure.vercel.app/]
## ✨ 特性

- 🚀 基于 Astro 5，构建快速、性能优秀
- 🎨 简洁干净的设计，支持响应式布局与明暗主题切换
- 📝 支持 Markdown / MDX 编写文章，自动生成文章预览、分页、标签与归档
- 🏷️ 标签云与按标签筛选文章
- 🗂️ 按年份归档（Archives）页面
- 📚 内置主题文档站（`/docs`）
- 🔗 友链页面（`/links`），基于 `public/links.json` 配置
- 💼 项目展示页（`/projects`），含项目列表、打赏与赞助墙
- 👤 关于页（`/about`），含工具集、社交统计（Substats）、站点历史时间线
- 🔍 基于 [Pagefind](https://pagefind.app/) 的全站搜索
- 📡 博客与文档双 RSS 订阅（`/rss.xml`、`/docs/rss.xml`）
- 🗺️ 站点地图（sitemap）与 `robots.txt`，友好的 SEO（Open Graph / Twitter Card）
- 📑 文章目录（TOC）、MediumZoom 图片灯箱
- 💻 Shiki 代码高亮：代码标题、语言标识、一键复制、代码折叠、diff / highlight 标注
- 🧮 支持 KaTeX 数学公式（`remark-math` + `rehype-katex`）
- 💬 集成 [Waline](https://waline.js.org/) 评论系统与浏览量统计
- 💬 随机语录（Quote）组件
- 🖼️ 动态社交分享图、GitHub 活动热力图

## 🧰 技术栈

| 类别 | 技术 |
| --- | --- |
| 框架 | [Astro](https://astro.build/) 5 |
| 主题 | [astro-pure](https://www.npmjs.com/package/astro-pure)（v1.4.1） |
| 语言 | TypeScript |
| 样式 | [UnoCSS](https://unocss.dev/) + `@unocss/preset-typography` |
| 评论 / 统计 | [Waline](https://waline.js.org/) |
| 搜索 | [Pagefind](https://pagefind.app/) |
| 公式 | [KaTeX](https://katex.org/) |
| 代码高亮 | [Shiki](https://shiki.style/) |
| 包管理器 | [Bun](https://bun.sh/)（同时保留 `package-lock.json`，可用 npm） |
| 部署 | [Vercel](https://vercel.com/) |

## 📁 目录结构

```text
sheng-blog/
├── astro.config.ts          # Astro 配置（adapter、Markdown、Shiki、集成等）
├── uno.config.ts            # UnoCSS 预设与排版样式
├── tsconfig.json            # TypeScript 配置（含路径别名）
├── public/                  # 静态资源（favicon、links.json、RSS 样式等）
├── preset/                  # 主题预设组件 / 图标 / 脚本
├── packages/pure/           # astro-pure 主题包源码（NPM 包模式）
├── src/
│   ├── site.config.ts       # 站点与集成配置（主题、友链、Waline、排版等）
│   ├── content.config.ts    # 内容集合 schema（blog）
│   ├── content/
│   │   ├── blog/            # 博客文章（Markdown / MDX）
│   │   └── docs/            # 主题文档
│   ├── pages/               # 页面路由（首页、博客、文档、友链、项目、关于等）
│   ├── layouts/             # 页面布局
│   ├── components/          # 自定义组件（首页、关于、友链、项目、Waline）
│   ├── plugins/             # 本地 rehype / Shiki 插件
│   ├── assets/              # 图片、图标、样式等资源
│   └── type.d.ts            # 虚拟模块类型声明
├── .github/                 # GitHub 模板与资助配置
└── package.json             # 依赖与脚本
```

## 🚀 快速开始

环境要求：

- [Node.js](https://nodejs.org/) 18.0.0+
- [Bun](https://bun.sh/)（推荐，也可使用 npm）

克隆并安装依赖：

```shell
git clone git@github.com:jackism336434/sheng-blog.git
cd sheng-blog
bun install
```

常用命令：

| 命令 | 说明 |
| --- | --- |
| `bun run dev` | 启动本地开发服务器 |
| `bun run dev:check` | 同时运行 `astro check` 与开发服务器 |
| `bun run build` | 执行 `astro-pure check`、`astro check` 并构建生产版本 |
| `bun run preview` | 本地预览构建产物 |
| `bun run sync` | 生成 Astro 类型 |
| `bun run check` | 运行 `astro check` 类型检查 |
| `bun run lint` | ESLint 检查并自动修复 `src/**` 代码 |
| `bun run format` | Prettier 格式化代码 |
| `bun run yijiansilian` | 一键四连：`lint` + `sync` + `check` + `format` |
| `bun run clean` | 清理 `.astro`、`.vercel`、`dist` 构建产物 |
| `bun run pure` | 调用 astro-pure CLI（如 `bun run pure new` 新建文章） |
| `bun run cache:avatars` | 缓存友链头像到 `public/avatars/` |





### 主题文档

在 `src/content/docs/` 下按分类（`setup`、`integrations`、`advanced`）组织文档，frontmatter 需包含 `title`、`description` 与 `order`（排序）。

## ⚙️ 配置说明

主要配置集中在 `src/site.config.ts`：

- **基础信息**：站点标题、作者、描述、favicon、社交分享图、语言（`locale`）、logo
- **页头菜单**：`header.menu`（Blog / Projects / Links / About）
- **页脚**：`footer`（版权年份、链接、社交账号、是否显示主题署名）
- **内容**：`content`（外链标记、博客分页大小 `blogPageSize`、分享按钮）
- **集成**：`integ`
  - `links`：友链日志、申请信息、是否缓存头像
  - `pagefind`：是否启用全站搜索
  - `quote`：随机语录接口
  - `typography`：排版样式
  - `mediumZoom`：图片灯箱
  - `waline`：评论系统（服务地址、emoji、浏览量等）

Astro 自身配置（部署 adapter、站点 `site`、Markdown / Shiki 插件、图片服务等）位于 `astro.config.ts`。

## 🎨 自定义

- **主题色 / 深色模式**：编辑 `src/assets/styles/app.css` 中的 CSS 变量（`--primary`、`--foreground`、`--background` 等）
- **排版样式**：编辑 `uno.config.ts` 中的 `typographyConfig`
- **全局样式**：`src/assets/styles/global.css`（动画、KaTeX、Shiki、滚动条等）
- **组件**：`astro-pure` 提供基础/高级组件（`Aside`、`Tabs`、`Steps`、`GithubCard`、`Quote` 等），可在 `.astro` 与 `.mdx` 中使用
- **路径别名**：`@/assets`、`@/components`、`@/layouts`、`@/plugins`、`@/site-config` 等（见 `tsconfig.json`）

## 🔌 集成功能

- **评论 / 浏览量**：`src/components/waline/`（`Comment`、`PageInfo`、`Pageview`），服务地址在 `site.config.ts` 的 `integ.waline.server` 配置
- **友链**：`public/links.json` 维护友链数据，页面在 `src/pages/links/index.astro`
- **社交统计**：`src/components/about/Substats.astro`，基于 [Substats](https://github.com/spencerwooo/substats)
- **RSS**：`src/pages/rss.xml.ts`（博客）与 `src/pages/docs/rss.xml.ts`（文档）
- **robots.txt**：`src/pages/robots.txt.ts`



如需修改站点域名，请将 `astro.config.ts` 中的 `site` 改为你自己的域名。

## ⚠️ 首次使用前需要修改的占位配置

模板中仍保留了一些示例/占位内容，建议按需替换：

- `astro.config.ts`：`site` 仍为 `https://astro-pure.js.org`
- `src/site.config.ts`：
  - `footer.links` 中的注册链接（示例为 `2026 - Heng` / `114514`）
  - `footer.social.github`、`links.applyTip` 中的 `Link` / `Avatar`
  - `header.menu` 与 `logo`
- `public/links.json`：友链数据
- `src/pages/index.astro`、`src/pages/about/index.astro`、`src/pages/projects/index.astro` 中的示例文案、项目与赞助信息
- `.github/FUNDING.yml`：资助信息

## 🤝 贡献

代码风格由 Prettier（含 import 排序插件）与 ESLint 约束，提交前请运行：

```shell
bun run yijiansilian
```

## 🙏 鸣谢

- [Astro](https://astro.build/)
- [astro-theme-pure](https://github.com/cworld1/astro-theme-pure)
- [Waline](https://waline.js.org/)
- [Pagefind](https://pagefind.app/)
- [Substats](https://github.com/spencerwooo/substats)
- 以及所有开源库的贡献者

## 📄 许可证

本项目沿用主题的 [Apache 2.0](LICENSE) 许可证。
