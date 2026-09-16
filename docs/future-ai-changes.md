# future_ai 文章设计实现：文件变更说明

本文记录上一轮实现中写入、修改的文件及各次实质性调整的作用，供人工检查。
按“文件 → 修改内容 → 作用 → 后续修正”归纳；纯格式化操作统一列出，不逐条复述工具调用。

**本文档本身位于 `docs/`，不会被当作博客文章加载。** 本轮只新增这份说明，没有继续修改文章或组件。

## 1. 总览

上一轮共涉及 **15 个源文件／说明文件：修改 4 个、新增 11 个**。本轮再新增本说明文档。

| 类型     | 文件                                     | 主要作用                         |
| -------- | ---------------------------------------- | -------------------------------- |
| 修改     | `src/content/blog/future_ai/index.mdx`   | 重排文章、启用随笔布局、接入组件 |
| 修改     | `src/content.config.ts`                  | 增加布局开关，排除设计文档       |
| 修改     | `src/layouts/BaseLayout.astro`           | 给随笔页面添加独立样式标记       |
| 修改     | `src/layouts/BlogPost.astro`             | 按文章配置选择新旧布局           |
| 新增     | `src/layouts/EssayPost.astro`            | 随笔页布局、目录、元信息及页尾   |
| 新增     | `src/assets/styles/essay.css`            | 全部随笔排版、主题及响应式样式   |
| 新增     | `src/components/essay/EssayLead.astro`   | 开篇标题与引言                   |
| 新增     | `src/components/essay/Section.astro`     | 章节编号与推理阶段标记           |
| 新增     | `src/components/essay/Concept.astro`     | 核心概念定义                     |
| 新增     | `src/components/essay/Aside.astro`       | 核心句摘记                       |
| 新增     | `src/components/essay/Equation.astro`    | 公式及术语解释                   |
| 新增     | `src/components/essay/Progression.astro` | 三个时代的对比图                 |
| 新增     | `src/components/essay/Scenario.astro`    | 情景分析笔记                     |
| 新增     | `src/components/essay/Original.astro`    | 默认折叠的英文原文               |
| 新增     | `src/components/essay/README.md`         | 组件使用与复用说明               |
| 本轮新增 | `docs/future-ai-changes.md`              | 当前检查说明                     |

注意：开始工作时，整个 `src/content/blog/future_ai/` 已处于 Git 未跟踪状态。里面的 `index.mdx`、`design.md`、`future.png` 都是原先已经存在的文件，不能因为 `git status` 显示 `??` 就认为是我新建的。

## 2. 实现路径与设计取舍

设计稿以 React / Next.js / Tailwind 为假设，但实际项目使用 Astro / UnoCSS。因此：

- 使用原生 Astro 组件，没有引入 React 或 Tailwind。
- 组件在构建阶段渲染，不新增客户端 hydration。
- 英文折叠采用浏览器原生 `details/summary`。
- 公式使用项目已有的 KaTeX。
- 新布局通过 `essay: true` 主动开启，不替换全部博客的样式。

页面调用关系：

```text
src/pages/blog/[...id].astro（未修改）
  └─ BlogPost.astro
       ├─ essay: false → 原有 ContentLayout / Hero / 侧栏目录
       └─ essay: true  → EssayPost.astro
                          └─ BaseLayout appearance="essay"
                               ├─ 原有站点 Header / Footer
                               ├─ 元信息与文章目录
                               ├─ index.mdx → 8 个随笔组件
                               └─ 版权、推荐文章、评论
```

## 3. 逐文件说明

### 3.1 `src/content/blog/future_ai/index.mdx`

**操作：整体重写排版结构；保留原有论证主体和英文段落。**

#### 第一次：文章元信息与内容结构重排

1. 标题从 `When Future doesn't need us` 改为 `当人类不再是社会运转所必需的`。
   - 作用：与设计稿的中文开篇一致。
   - 影响：文章列表、页面标题和分享标题也使用新标题；URL 仍为 `/blog/future_ai`。
2. 补全原本为空的 `description`。
   - 作用：给文章列表和页面元信息提供摘要。
3. 增加 `essay: true`。
   - 作用：仅这篇文章启用新布局。
4. 保留 `future.png` 引用，把 `heroImage.color` 从 `#9698C1` 改为 `#8C2F2B`。
   - 作用：元数据强调色与设计稿一致；新文章页不显示大幅 Hero 图，也不传入背景渐变色。
5. 导入八个随笔组件，建立“开篇 → 观点 → 反对意见 → 精确论证”的结构。
6. 将中英文交替排列改为中文连续阅读、英文按节集中折叠。
   - 共四个英文折叠块：开篇、观点、反对者们、更加精确的论点。
   - 英文仍在生成的 HTML 中，不是删除，也不是点击后才下载。
7. 将三个原章节标题改成叙事式标题：
   - `观点` → `失去工作并不是最严重的问题`
   - `反对者们` → `如果这一切根本不会发生呢？`
   - `更加精确的论点` → `到底什么才叫拥有力量？`
8. 把已有英文中的参考链接补到相应中文位置，避免必须打开英文才能访问参考资料。
9. 把五项反对情景改为五个 `Scenario`，其中“经典毁灭结局”拆成两段短句。
10. 把中文里的两个裸公式改为 `Equation`，添加“第一次近似／修正版”和术语解释。
11. 英文参考块中保留相应公式，便于单独阅读原文。

#### 新增的编辑性内容：请重点检查

这些内容来自设计稿的提炼或为图表补充，并非全部逐字来自原稿：

- `Concept` 中对“物质杠杆”的定义。
- `Progression` 的三个时代、能力高低与结论摘要。
- 图表注释：高低是论证示意，不是历史数据；监控可能进一步削弱协调能力。
- 第一条摘记：“真正有力量的‘同意’，必须包含撤回同意的能力。”
- 结尾摘记：“有杠杆却无法协调，力量无法形成；能够协调却没有杠杆，力量无处落地。”
- 公式前的“权力 = 暴力能力 + 协调能力”改为“拥有暴力能力，并且能够协调行动”，使文字与逻辑合取符号一致。
- 少量空格、引号和标点规范化。

没有为文章新增独立的论证章节，也没有对原文中的历史、经济或 AI 预测做事实核查。

#### 第二次：日期与标题修正

- 日期从 `'September 16, 2026'` 改为 `'2026-09-16'`，发布与更新日期一起修改。
  - 原因：浏览器检查发现，英文日期字符串按构建机器本地时区解析后，再按 UTC 显示，会出现前一天。
  - 作用：保持原本要表达的 2026 年 9 月 16 日，并避免本次页面上的日期偏移。
- `EssayLead.title` 改为两行数组：`['当人类不再是', '社会运转所必需的']`。
  - 原因：自动换行把“社会”拆到了两行。
  - 作用：按语义断行；手机端配合缩小标题字号。

### 3.2 `src/content.config.ts`

**操作一：博客 loader 增加排除项。**

```ts
pattern: ['**/*.{md,mdx}', '!**/design.md']
```

- 原因：原配置会把文章目录里的 `design.md` 也当作博客条目，设计稿又不具备正常文章的 frontmatter。
- 作用：设计稿不进入博客内容集合。
- 范围：排除的是 **所有博客子目录中名为 `design.md` 的文件**，不只 `future_ai`。
- 注意：没有排除其他名字的说明文档。因此本检查文档放在根目录下的 `docs/`，而不是文章目录。

**操作二：schema 增加开关。**

```ts
essay: z.boolean().default(false)
```

- 作用：允许文章声明专用随笔布局，未声明的文章默认继续使用原布局。

另有 Prettier 自动调整 import 顺序，无业务逻辑变化。

### 3.3 `src/layouts/BaseLayout.astro`

- Props 新增可选 `appearance?: 'essay'`。
- 从页面属性中取出 `appearance`。
- 在 `<body>` 上输出 `data-appearance={appearance}`。

作用：让随笔页通过 `body[data-appearance='essay']` 获得纸张底色、文字色和字体变量；普通页面没有此标记。

没有修改站点 Header、Footer、ThemeProvider 或原有渐变实现。随笔布局不传入 `highlightColor`，因此不产生该页面的背景渐变。

### 3.4 `src/layouts/BlogPost.astro`

- 导入新增的 `EssayPost`。
- 根据 `data.essay` 分支渲染：新随笔布局或原有布局。
- 把文章、推荐文章列表和 Markdown 标题传给随笔布局。
- 将 `MediumZoom` 限定在非随笔页面启用。

作用：复用现有文章路由，不为 `future_ai` 硬编码新路由。

需注意的差异：随笔分支没有沿用旧 Hero、悬浮侧栏目录、旧布局的回顶按钮和 `bottom-sidebar` 插槽；它采用顶部目录、文字开篇和页尾回顶链接。随笔页目前也没有图片点击放大功能。

文件中较多行差异来自新增条件分支后的缩进，以及 Prettier 的 import 排序。

### 3.5 `src/layouts/EssayPost.astro`

**新增完整随笔布局。**

- 复用 `BaseLayout`，启用 `appearance='essay'`。
- 引入随笔样式与 KaTeX 样式。
- 保留标题、摘要、分享图片及文章日期元信息。
- 输出返回博客链接、发布日期和更新日期。
- 从 Astro 提供的 `headings` 中取二级标题，生成“阅读路径”。
- 使用原生锚点跳转，不增加目录滚动脚本。
- 正文容器标记为 `lang='zh-CN'`。
- 保留原有版权、推荐文章与评论组件；评论仍受 `draft`、`comment` 控制。
- 增加页尾“回到开篇”链接。

当前定位是中文随笔模板，日期格式和“阅读路径”等文案为中文，尚未做多语言模板化。

### 3.6 `src/assets/styles/essay.css`

**第一次写入：整套随笔视觉与排版。**

- 暖白纸张背景、深灰正文、克制的暗红强调色。
- `.dark` 模式下改用深灰背景与暖灰文字。
- 正文最大 720px，宽视觉组件最大 960px。
- 宋体正文、无衬线界面标签、等宽概念标签。
- 正文桌面字号 18px、行高 1.95；小屏正文 17px。
- 各组件的线条、间距、字号及布局。
- 普通链接使用低对比下划线，悬停时强调。
- 提供键盘焦点轮廓。
- 手机端时代对比纵向排列，长公式在自身容器内滚动。
- 添加打印样式，隐藏部分导航和页尾辅助内容。

**第二次：浏览器检查后的修正。**

1. 增补中文字体回退，包括 Linux 上可用的宋体和无衬线字体。
   - 原因：测试环境的默认中文 serif 回退成了偏楷体的外观。
   - 作用：更接近设计稿的宋体气质；不下载新的站点字体资源。
2. 增加 `.essay-title-line` 和小屏标题字号规则。
   - 作用：支持指定语义分行，保证 320px 屏幕下不溢出。
3. 调整宽组件的宽度计算。
   - 原因：初版在 768px 宽度下，图表反而比正文更窄。
   - 作用：组件至少与正文等宽，同时不超出页面可用空间。

**第三次：对比图对齐修正。**

- 增加 `.essay-progression li + li { margin-top: 0; }`。
- 原因：正文通用 `li + li` 的间距规则让第二、第三栏向下偏移。
- 作用：消除图表列上的正文列表间距。

样式使用 `.essay-*` 类及专用 body 属性，而不是全站重写 `a`、`h1`、`p` 等元素样式。

### 3.7 `src/components/essay/EssayLead.astro`

- 第一次新增：接收 `eyebrow` 和 `title`，输出开篇标签、唯一 `h1` 和引言插槽。
- 后续修改：`title` 从只接受字符串扩展为 `string | string[]`；数组逐行输出 `span`。
- 作用：支持本篇标题按语义断行，同时保留普通字符串用法。

### 3.8 `src/components/essay/Section.astro`

- 接收 `index` 和 `kicker`，输出编号与英文阶段标签。
- 章节标题留在插槽内，以 Markdown `##` 编写。
- 作用：兼顾设计稿的阶段感，以及 Astro 对目录、标题 ID 的静态提取。

与设计稿示例的差异：没有使用 `title` 属性直接生成标题，这是为了让现有 Markdown 标题提取流程正常工作。

### 3.9 `src/components/essay/Concept.astro`

- 接收英文 `term` 和中文 `zh`。
- 使用 `aside`、`dfn` 和内容插槽组织定义。
- 作用：呈现书籍边栏式概念说明，而不是彩色提示卡片。

### 3.10 `src/components/essay/Aside.astro`

- 使用 `aside` 包裹内容插槽，并添加“论点摘记”的可访问标签。
- 作用：把少量核心句从普通正文中提出来。
- 角线和放大字号由 `essay.css` 实现，没有新增交互。

### 3.11 `src/components/essay/Equation.astro`

- 接收 `label`、TeX 字符串 `formula` 和可选 `terms`。
- 构建时调用 `katex.renderToString`。
- 同时生成可视 HTML 与 MathML。
- `trust: false`，不启用 KaTeX 的受信任命令能力；`throwOnError: true`，公式写错时直接报错。
- 公式容器可获取键盘焦点，并允许横向滚动。
- 使用 `figure/figcaption` 和 `dl` 显示公式标题及术语表。

作用：把两次模型修正做成清晰的思想节点，不依赖浏览器再执行公式渲染脚本。

### 3.12 `src/components/essay/Progression.astro`

- 接收时代、群体、物质杠杆、协调能力和结论数组。
- 使用列表、定义列表和条形示意呈现比较。
- 桌面三栏，小屏单列；条形仅用于定性高低比较。
- 装饰条形标记为 `aria-hidden`，文本仍明确写出“高／低”。

作用：直观比较“有杠杆但难协调”“两者兼备”“能协调却无杠杆”。

需要检查：底部说明文字直接写在组件内，包含本篇“后 AI 社会”的假设说明；复用到其他题材时，可能需要将该注释改为属性或插槽。

### 3.13 `src/components/essay/Scenario.astro`

- 接收 `index`、中文 `title`、英文 `en`。
- 输出 `section`、编号、英文标签、三级标题及正文插槽。
- 作用：把五种情景排成研究笔记，不用五种颜色的卡片。

### 3.14 `src/components/essay/Original.astro`

- 使用原生 `details/summary`，默认不设置 `open`。
- 支持自定义折叠标题 `label`。
- 英文内容容器设置 `lang='en'`。
- 作用：减少中英文交替对阅读节奏的干扰，保留原文参考和键盘操作。

注意：视觉折叠不等于从 HTML、RSS 或搜索索引中删除；本次没有额外编写这些系统的英文过滤规则。

### 3.15 `src/components/essay/README.md`

- 说明 `essay: true` 的启用方式。
- 给出 MDX 引用方式、各组件属性和公式转义示例。
- 解释为什么 `Section` 必须保留 Markdown 二级标题。
- 记录字体、日期、检查命令和视觉节点数量建议。

作用：供后续文章复用，而当前这份文档用于审查本次修改；两者用途不同。

## 4. 格式化与未改动范围

### 格式化

对本次涉及的代码、样式和说明执行了定向 Prettier，而不是全项目格式化。其作用包括 import 排序、缩进、引号和 JSX/MDX 换行统一，不额外改变业务逻辑。

### 明确未改动

- `src/content/blog/future_ai/design.md`：只读取，未改写。
- `src/content/blog/future_ai/future.png`：未修改图片内容。
- 其他文章正文：未改写。
- `src/pages/blog/[...id].astro`：未修改文章路由。
- `src/layouts/ContentLayout.astro`：保留旧布局实现。
- `astro.config.ts`、`package.json`、`package-lock.json`、`bun.lock`：未改动。
- 没有提交 Git commit，没有执行部署。

## 5. 验证结果及边界

| 检查                                  | 结果与说明                                                                                                        |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `npm run check` / 构建中的 Astro 检查 | 0 errors、0 warnings；有 1 条首页 `ProjectCard` 未使用的 hint，不在本次修改范围                                   |
| `npm run build`                       | 页面构建完成，生成 `/blog/future_ai/index.html`                                                                   |
| 定向 ESLint                           | 修改涉及的 Astro 文件检查无错误；一次把 TS 配置文件传给 ESLint 时提示该文件没有匹配配置，不代表它经过 ESLint 检查 |
| Prettier                              | 本次文件格式检查通过                                                                                              |
| `git diff --check`                    | 已跟踪文件的差异检查通过；它不覆盖未跟踪文件，后者另由格式检查覆盖                                                |
| HTML 检查                             | 一个 `h1`、目录锚点有效、四个英文块默认折叠、无设计稿博客页面                                                     |
| 英文保留检查                          | 对原稿与新稿规范化空白及 Markdown 链接写法后，原有英文块均保留                                                    |
| 浏览器宽度                            | 检查 320、390、768、1024、1440px，无整页横向溢出                                                                  |
| 深色模式                              | 检查 390、1440px，页面背景及文字切换正常                                                                          |
| 折叠操作                              | 键盘 Enter 可展开／关闭英文                                                                                       |
| 公式滚动                              | 320px 下公式自身横向滚动，不撑宽页面                                                                              |
| 旧布局回归                            | 抽查 `smart_pointer`，仍使用旧正文布局和背景渐变；不是对所有旧文章做完整视觉回归                                  |

### 尚未验证成功／未覆盖

1. **Pagefind 搜索索引**：构建日志报告安装 Pagefind 平台包失败。文章 HTML 已生成，但不能据此声称搜索索引生成成功；本次未修复该工具问题。
2. **外部集成**：浏览器布局测试主动阻断外部网络，因此存在 `QRCode is not defined`、`Failed to fetch`；抽查旧文章还有 `mediumZoom is not defined`。这些测试没有验证二维码、远端评论、统计和图片放大等外部依赖的完整可用性。
3. **字体一致性**：使用本地字体回退，不保证所有操作系统呈现完全相同的字形。
4. **无障碍／打印**：做了语义、焦点和键盘支持，但没有执行完整屏幕阅读器审计或打印视觉测试。
5. **文章事实核查**：没有验证预测、历史概括、比例数据或外链内容的真实性和时效性。

## 6. 临时文件、测试依赖与构建产物

这些不属于上述 15 个源文件，但为完整说明操作范围一并列出。

### 临时文件

上一轮创建在 `/tmp/`，不属于仓库交付内容：

- `/tmp/future-ai-original.mdx`：改写前文章备份，可用于对照。
- `/tmp/future-ai-browser-test.mjs`：页面宽度、主题、目录与折叠行为测试。
- `/tmp/future-ai-components-test.mjs`：图表／公式截图和内部滚动测试。
- `/tmp/future-ai-*.png`：桌面、手机、主题及组件截图。
- `/tmp/future-ai-*.log`：检查、构建、浏览器安装、临时服务器日志。
- `/tmp/future-ai-server.pid`：临时静态服务器进程记录；测试结束后已停止该服务器。
- `/tmp/future-ai-browser-libs/`：浏览器运行所需的临时库文件。

临时文件可能被系统清理，不能当作长期版本备份。

### 浏览器测试依赖

- 通过 `npm exec` 尝试新版 Playwright，因当前 Ubuntu 版本不支持，改用 Playwright 1.51.1。
- 下载 Chromium 到 `/root/.cache/ms-playwright/`，npm 工具缓存位于 `/root/.npm/_npx/`。
- 将 `libgbm1` 和 `libwayland-server0` 的 deb 包下载并解压到上述 `/tmp/` 目录，通过 `LD_LIBRARY_PATH` 提供给测试浏览器。
- 没有把 Playwright 写入项目依赖，也没有安装这些 deb 包到系统目录。

### 构建产物

Astro 检查和构建更新了 `.astro/`、`dist/`、`.vercel/output/` 中的生成内容。这些是工具输出，不是手工编辑的源文件，也不代表已向 Vercel 部署。

## 7. 建议你的检查顺序

1. **先检查文章内容**：`src/content/blog/future_ai/index.mdx`，重点看标题、三节标题、两条摘记、概念定义和图表注释是否符合你的表达。
2. **再检查页面效果**：运行 `npm run dev`，访问 `/blog/future_ai`，切换手机宽度和深色模式。
3. **检查站点影响**：审阅 `src/content.config.ts` 和三个布局相关文件，特别是全局排除 `design.md` 的规则与随笔分支的功能差异。
4. **检查复用方式**：查看 `src/components/essay/README.md`；新文章需自己放置唯一的 `EssayLead`，并在 `Section` 插槽中使用 `##`。
5. **检查旧链接兼容性**：本篇三个章节标题修改后，原先指向 `#观点`、`#反对者们`、`#更加精确的论点` 的外部锚点链接不再对应新标题；本次未添加旧锚点别名。
6. **最后处理环境问题**：如果上线依赖站内搜索，需要单独解决 Pagefind 安装／索引问题，并验证真实网络下的评论与其他外部组件。
