# Margin

> A Personal Content Garden

Margin 是一个基于 GitHub 的个人内容花园（Personal Content Garden）。

用户拥有自己的内容仓库（Repository），Margin 负责阅读、编辑、组织与展示内容。

它不是博客系统。

它不是知识管理工具。

它也不是 CMS。

Margin 希望成为一个长期陪伴用户成长的内容空间，用于记录那些值得留下来的内容，以及这些内容在用户脑海中留下的痕迹。

------

# 产品愿景

现代人记录了大量内容：

- 书籍摘录
- 阅读笔记
- 博客文章
- 灵感碎片
- 项目思考
- 学习记录
- 人生感悟

这些内容通常散落在：

- Notion
- Obsidian
- Flomo
- GitHub
- 博客
- 微信收藏

最终形成信息孤岛。

Margin 希望提供一个统一的内容空间：

- 所有内容归用户所有
- 所有内容存储在用户自己的 GitHub Repo
- 网站只负责展示和编辑
- 数据可以随时迁移
- 内容随着时间自然沉淀

------

# 核心理念

## Content First

内容优先。

Margin 不强调复杂的知识管理系统。

用户应该专注于记录内容，而不是管理内容。

------

## Data Ownership

数据归用户所有。

用户的内容存储在自己的 GitHub Repository 中。

Margin 不保存用户内容。

即使 Margin 停止运营：

- Repo 仍然存在
- Markdown 文件仍然存在
- 用户仍然拥有全部数据

------

## Timeline First

时间流优先。

Margin 不以分类浏览为核心。

首页永远是时间流。

用户看到的是：

自己的成长轨迹。

而不是：

复杂的信息架构。

------

## Simple Organization

简单组织。

Margin 不使用：

- 多层文件夹
- 多层分类
- 知识图谱
- 强制双向链接

仅使用：

- Area
- Tags
- Search

完成内容组织。

------

# 产品定位

Margin

A Personal Content Garden

用于记录：

- 阅读
- 思考
- 创造
- 实践
- 生活

所有值得留下来的内容。

------

# 内容模型

Margin 只有一种内容类型：

Content

不区分：

- Quote
- Note
- Article
- Review

因为这些边界会随着时间变化。

一句摘录：

↓

增加思考

↓

形成笔记

↓

扩展成长文

这是自然演化过程。

------

# Content Schema

```yaml
id:

title:

content:

area:

tags:

source:

createdAt:

updatedAt:
```

------

# Example

```yaml
id: c_20260614_001

title: 关于自由

area: reading

tags:
  - 自由
  - 哲学

source:
  type: book
  title: 论自由
  author: John Stuart Mill

createdAt: 2026-06-14
自由意味着承担后果。

我的思考：

很多时候我们想要自由，
实际上想要的是没有代价。
```

------

# Area System

Margin 借鉴 Johnny Decimal 的组织思想。

但不照搬其数字管理体系。

目标：

让内容自然归属。

而不是让用户维护分类系统。

------

## Areas

### 00 Inbox

收件箱。

所有新内容默认进入这里。

适用于：

- 灵感
- 临时记录
- 待整理内容

------

### 10 Reading

阅读与输入。

包括：

- 书籍摘录
- 文章摘录
- 播客笔记
- 视频记录

------

### 20 Thinking

思考与观点。

包括：

- 想法
- 观点
- 长期思考
- 哲学问题

------

### 30 Building

项目与实践。

包括：

- 产品设计
- 编程项目
- Side Project
- 创业记录

------

### 40 Living

生活与经历。

包括：

- 旅行
- 健康
- 家庭
- 人生感悟

------

### 90 Archive

归档区。

用于存放：

- 已过时内容
- 已完成项目
- 历史记录

------

# Repository Structure

```text
content/

├── 00-inbox/
│
├── 10-reading/
│
├── 20-thinking/
│
├── 30-building/
│
├── 40-living/
│
└── 90-archive/
```

------

# Content File

```text
content/
└── 10-reading/
    └── 2026/
        └── 2026-06-14-freedom.md
```

------

# Markdown Format

```markdown
---
id: c_20260614_001

title: 关于自由

area: reading

tags:
  - 自由
  - 哲学

source:
  type: book
  title: 论自由

createdAt: 2026-06-14
---

自由意味着承担后果。

我的思考：

很多时候我们想要自由，
实际上想要的是没有代价。
```

------

# GitHub Integration

Margin 是 GitHub Native 产品。

------

## 登录

GitHub OAuth

------

## 初始化

首次登录：

Create Your Garden

系统自动：

- Fork Template Repo
- 初始化结构
- 完成绑定

用户无需理解 GitHub Fork。

------

## 编辑

用户编辑内容：

↓

Margin UI

↓

生成 Markdown

↓

Commit

↓

Push 到用户 Repo

------

# 浏览模式

## Timeline

默认首页。

展示：

```text
2026-06-14

关于自由

2026-06-13

人类简史摘录

2026-06-10

Agent 架构思考
```

------

## Area

按 Area 浏览。

- Reading
- Thinking
- Building
- Living

------

## Tags

按标签浏览。

------

## Search

全文搜索。

支持：

- 标题
- 内容
- 标签
- 来源

------

# 为什么没有双向连接

Margin 不采用 Roam/Logseq 风格的强制双向链接。

原因：

维护成本高。

用户容易从：

记录内容

变成：

维护系统。

------

未来版本可以支持：

Related Content

自动关联推荐。

而不是：

手工维护知识图谱。

------

# 设计原则

参考：

- Kami
- Obsidian
- GitHub

------

# Visual Style

关键词：

- 克制
- 安静
- 长期阅读
- 内容优先

------

# Layout

内容宽度：

680px - 760px

------

# Typography

正文：

Source Han Serif（思源宋体）

引用与摘录：

LXGW WenKai（霞鹜文楷）

界面：

Inter

------

# Core Visual Element

左侧引用线。

```text
│ 人生最大的危险，
│ 不是死亡，
│ 而是从未真正活过。
```

成为整个产品的核心视觉语言。

------

# Theme System

仅保留两套主题。

------

## Paper

灵感来源：

Kami

特点：

- 暖白背景
- 纸张质感
- 适合长时间阅读

------

## Midnight

灵感来源：

Obsidian

特点：

- 深色背景
- 降低夜间阅读疲劳

------

# MVP Scope

Version 1.0

------

## 必须实现

- GitHub OAuth
- Template Repo
- Content CRUD
- Timeline
- Area
- Tags
- Search
- Paper Theme
- Midnight Theme

------

## 不实现

- 双向链接
- Graph View
- 多级分类
- AI 总结
- AI 标签
- 评论系统
- 社交系统

------

# Long-Term Vision

Margin 希望成为：

用户十年后仍然愿意保存的内容仓库。

不是因为功能很多。

而是因为：

这里保存着他的成长、思考与演化。

内容随着时间沉淀。

思想随着时间生长。

最终形成属于自己的 Garden。
