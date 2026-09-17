# MDX Essay System

为思想类长文提供的原生 Astro 排版组件，无 React、无客户端 hydration。
参考设计：`src/content/blog/future_ai/design.md`。
完整界面介绍、原布局对比与启用示例见 [Essay 随笔界面与使用指南](../../../docs/essay-layout.md)。

## 启用

在文章 frontmatter 中添加 `essay: true`，即可使用 `src/layouts/EssayPost.astro`。
默认文章布局不变。`heroImage` 仍用于文章列表与社交分享，不在随笔页展示大图。
日期推荐使用带引号的 ISO 日期（如 `'2026-09-16'`），避免构建机器时区导致日期偏移。

排版由 `src/assets/styles/essay.css` 控制：正文 720px、视觉组件最大 960px，
继承站点的 `.dark` 模式。字体优先使用本地宋体，不额外下载中文字体；
如需跨设备完全一致的字体，可另行配置自托管的中文字体子集。

组件标签、图表说明、公式术语和英文折叠标题沿用正文的衬线字体，
辅助文字以 13–14px 为主，不强制等宽字体、全大写或加宽字距。
章节编号采用同一衬线字体的数字特性；数学公式仍使用 KaTeX 专用字体。
无衬线字体仅保留在页面导航、日期与页尾等功能性区域。

## 基本用法

```mdx
import EssayLead from '@/components/essay/EssayLead.astro'
import Original from '@/components/essay/Original.astro'
import Section from '@/components/essay/Section.astro'

<EssayLead eyebrow="AI · POWER · POLITICAL ORDER" title="文章标题">

引言。

</EssayLead>

<Section index="01" kicker="The argument">

## 第一阶段

</Section>

正文。

<Original label="本节英文原文">

Original English text.

</Original>
```

- 每篇只放一个 `EssayLead`，它输出唯一的 `h1`。`title` 可传字符串数组，指定分行。
- `Section` 内保留 Markdown `##`，不要改为组件的 `title` 属性，Astro 依靠它生成目录与锚点。
- Markdown 与组件标签之间保留空行，保证段落、链接、引用能正常解析。
- `Original` 使用原生 `details/summary`，默认折叠，支持键盘，无需 JavaScript。

## 组件

| 组件          | 属性                                             | 用途                                                        |
| ------------- | ------------------------------------------------ | ----------------------------------------------------------- |
| `EssayLead`   | `eyebrow`, `title: string \| string[]`           | 开场标题与引言                                              |
| `Section`     | `index`, `kicker`                                | 推理阶段；插槽中使用 Markdown 二级标题                      |
| `Concept`     | `term`, `zh`                                     | 核心术语定义                                                |
| `Aside`       | 无                                               | 少量核心句摘记                                              |
| `Equation`    | `label`, `formula`, `terms?: [string, string][]` | 构建时 KaTeX 渲染，附 MathML；窄屏公式内部滚动              |
| `Progression` | `items`                                          | 时代、群体、杠杆与协调能力对比；高低为定性示意，不是统计值  |
| `Scenario`    | `index`, `title`, `en`                           | 情景笔记；多个实例可放进 `className="essay-scenarios"` 容器 |
| `Original`    | `label?`                                         | 整节英文参考材料                                            |

`Equation` 的 TeX 在 MDX 字符串中需要双反斜杠，例如：

```mdx
<Equation
  label='第一次近似'
  formula={'\\text{Power} \\iff \\text{Violence} \\land \\text{Coordination}'}
  terms={[
    ['Violence', '暴力能力'],
    ['Coordination', '协调能力']
  ]}
/>
```

完整 `Progression` 数据与其他示例见 `src/content/blog/future_ai/index.mdx`。
强视觉节点保持在每篇 3–5 处，不把文章变成组件展示页。

## 检查

- `npm run check`：Astro / TypeScript 检查。
- `npm run build`：MDX、内容集合与静态页面生成检查。
- 浏览 `/blog/future_ai`：检查 320px–1440px、浅深色、英文折叠键盘操作、目录锚点和公式滚动。
- `design.md` 已从博客 glob 排除，不会进入文章列表、RSS 或搜索内容集合。
