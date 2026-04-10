# Agents 使用指南

本文档记录了本项目中 AI 代理（Agent）的使用规范和指南。

## 项目概述

这是一个基于 [Astro](https://astro.build) 构建的博客项目，支持中英文双语内容。

## 技术栈

- **框架**: Astro
- **样式**: UnoCSS
- **语言**: TypeScript
- **包管理器**: bun

## 开发命令

| 命令              | 说明           |
| ----------------- | -------------- |
| `bun run dev`     | 启动开发服务器 |
| `bun run build`   | 构建生产版本   |
| `bun run preview` | 预览生产构建   |
| `bun run lint`    | 运行代码检查   |
| `bun run format`  | 格式化代码     |

## 代码规范

1. 使用 TypeScript 进行类型安全开发
2. 遵循 ESLint 配置的代码风格
3. 使用 Prettier 统一代码格式
4. 组件使用 Astro 的最佳实践

## 添加新内容

### 添加博客文章

在 `src/content/blog/` 目录下创建新的 Markdown 文件，frontmatter 需包含：

```yaml
---
title: 文章标题
description: 文章描述
pubDate: 发布日期
heroImage: /images/hero.jpg
---
```

### 添加文档

在 `src/content/docs/` 目录下创建新的 Markdown 文件。

## 注意事项

- 不要提交包含敏感信息的文件（如 `.env`、凭证文件等）
- 修改代码前确保了解相关模块的功能
- 提交代码前运行 lint 和 format 命令
