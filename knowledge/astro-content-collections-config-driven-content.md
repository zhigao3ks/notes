---
title: "用 Astro Content Collections、YAML 与 Zod 构建配置驱动网站"
date: 2026-07-17
updated: 2026-07-17
status: draft
category: knowledge
tags:
  - Astro
  - Content Collections
  - Zod
  - YAML
  - TypeScript
summary: "以个人网站项目档案为例，解释如何划分 Markdown Collection 与集中式 YAML 数据、使用 Zod 提前校验内容，并自动生成项目列表和静态详情路由。"
---

# 用 Astro Content Collections、YAML 与 Zod 构建配置驱动网站

## 一句话理解

配置驱动网站的核心不是“把文字搬到 YAML”，而是建立一条明确的数据链路：**内容文件负责事实，Schema 负责边界，查询函数负责筛选排序，组件只负责展示。**

本文以 ZGLab 的项目档案为例，说明 Content Collections、普通 YAML 和 Zod 各自应该解决什么问题。

## 为什么不能把内容直接写进组件

在小型页面中，下面的写法很常见：

```astro
<article>
  <h2>实时会议理解与智能纪要 Agent</h2>
  <p>面向企业会议场景……</p>
</article>
```

当项目只有一个时，这种写法简单直接。但随着内容增长，会出现：

- 列表页和详情页重复维护标题与摘要；
- 新增项目必须修改页面组件；
- 不同项目拥有不同字段，缺少统一校验；
- 排序、隐藏和筛选逻辑混入展示模板；
- 空链接和空图片容易被直接渲染。

更稳妥的数据流如下：

```mermaid
flowchart LR
    FILE[Markdown / YAML] --> SCHEMA[Zod Schema]
    SCHEMA --> QUERY[查询与排序]
    QUERY --> PAGE[Astro 页面]
    PAGE --> COMPONENT[展示组件]
    COMPONENT --> HTML[静态 HTML]
```

任何内容错误都应尽量在 `SCHEMA` 阶段暴露，而不是等访问者打开页面才发现。

## Content Collection 和普通 YAML 如何分工

ZGLab 没有把全部数据强行放进同一种文件。

### 适合 Content Collections 的内容

项目和实验室日志具有以下特征：

- 同类内容会不断增加；
- 每条记录字段结构相同；
- 每条内容需要独立 Markdown 正文；
- 需要列表查询、过滤和动态路由；
- 需要在构建时进行统一类型校验。

因此它们分别位于：

```text
src/content/projects/*.md
src/content/logs/*.md
```

### 适合集中 YAML 的内容

个人资料、技能、论文、竞赛、当前重点和时间线当前具有以下特征：

- 数量较少；
- 经常需要整体查看和编辑；
- 不需要为每条记录生成独立正文页；
- 在多个页面共享。

因此它们位于：

```text
src/data/profile.yaml
src/data/skills.yaml
src/data/publications.yaml
src/data/awards.yaml
src/data/ideas.yaml
src/data/now.yaml
src/data/timeline.yaml
```

这种划分不是 Astro 的硬性要求，而是当前内容规模下的维护决策。未来论文需要独立页面时，可以再迁移为 Collection。

## 定义项目 Collection

Astro 当前使用项目根下的 `src/content.config.ts` 定义构建时 Collection。

下面保留了 ZGLab 项目 Schema 的关键部分：

```ts
import { defineCollection } from "astro:content";
import { glob } from "astro/loaders";
import { z } from "astro/zod";

const optionalUrl = z.url().optional();

const projects = defineCollection({
  loader: glob({
    base: "./src/content/projects",
    pattern: "**/*.md",
  }),
  schema: z.object({
    title: z.string(),
    slug: z.string().regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/),
    summary: z.string(),
    status: z
      .enum([
        "exploring",
        "planned",
        "building",
        "completed",
        "online",
        "paused",
      ])
      .optional(),
    progress: z.number().min(0).max(100).optional(),
    featured: z.boolean().default(false),
    visible: z.boolean().default(true),
    order: z.number().int().default(0),
    categories: z.array(z.string()).default([]),
    tags: z.array(z.string()).default([]),
    stack: z.array(z.string()).default([]),
    cover: z.string().optional(),
    gallery: z.array(z.string()).default([]),
    highlights: z.array(z.string()).default([]),
    links: z
      .object({
        demo: optionalUrl,
        repository: optionalUrl,
        documentation: optionalUrl,
      })
      .default({}),
  }),
});

export const collections = { projects };
```

这个 Schema 不只是生成 TypeScript 提示，还表达内容规则：

- `slug` 只能由小写字母、数字和连字符组成；
- `status` 只能使用已知生命周期值；
- `progress` 必须在 0 到 100 之间；
- 数组缺省时自动变为空数组；
- URL 存在时必须是有效 URL；
- `visible` 和 `featured` 有明确默认值。

## 为什么可选字段要谨慎设计

“字段支持”不代表“所有项目必须填写”。

项目开始时间、进度、封面和仓库地址如果没有可靠来源，应该允许缺省。将它们强制设为必填，维护者往往会为了通过构建而补造数据。

合理做法是区分：

- **内容识别必需字段**：`title`、`slug`、`summary`；
- **列表管理字段**：`visible`、`order`、`featured`；
- **可靠时才填写的元数据**：日期、进度、外部链接；
- **展示增强字段**：封面、图集、技术栈和亮点。

Schema 应保护事实边界，而不是逼迫内容看起来完整。

## 创建项目 Markdown

每个项目是一份独立文件：

```markdown
---
title: 实时会议理解与智能纪要 Agent
slug: meeting-agent
summary: 面向企业会议场景，构建实时转写、摘要和待办提取流程。
status: building
featured: false
visible: true
order: 2
categories: [Agent, Real-time AI]
tags: [会议理解, 实时转写, 状态机]
stack: [Spring Boot, MySQL, WebSocket, Docker, Nginx]
highlights:
  - 浏览器录音和实时音频推流
  - 实时转写和临时结果展示
  - 会议任务状态机
---

## 目标

将会议从实时音频输入到结构化纪要输出串联为可恢复、可追踪的任务流程。
```

Frontmatter 进入 `project.data`，Markdown 正文通过 Astro 的 `render()` 渲染。

新增项目不需要修改组件，只需要：

1. 新建一份 `.md`；
2. 填写必需 Frontmatter；
3. 编写正文；
4. 执行类型检查和构建。

## 查询、过滤和排序

查询逻辑集中在 `src/lib/content.ts`：

```ts
import { getCollection } from "astro:content";

export const getVisibleProjects = async () => {
  const projects = await getCollection("projects", ({ data }) => data.visible);

  return projects.sort((a, b) => a.data.order - b.data.order);
};
```

这样首页和项目列表共享同一套“公开且按顺序排列”的规则。

如果每个页面都自己写：

```ts
projects.filter(...).sort(...)
```

后续修改隐藏规则时就可能出现页面之间不一致。

## 自动生成静态详情页

动态文件名为：

```text
src/pages/projects/[slug].astro
```

构建阶段使用 `getStaticPaths()` 为每个公开项目创建路径：

```astro
---
import { getCollection, render } from 'astro:content';

export async function getStaticPaths() {
  const projects = await getCollection(
    'projects',
    ({ data }) => data.visible,
  );

  return projects.map((project) => ({
    params: { slug: project.data.slug },
    props: { project },
  }));
}

const { project } = Astro.props;
const { Content } = await render(project);
---

<h1>{project.data.title}</h1>
<Content />
```

这里的“动态路由”只描述文件命名和参数来源。因为项目使用静态输出，最终仍然是构建好的 HTML，不需要服务器收到请求后再查询 Markdown。

## 普通 YAML 如何获得同样的校验

集中 YAML 使用 Vite 的 `?raw` 导入，再通过 `yaml` 包解析，最后交给 Zod：

```ts
import { z } from "astro/zod";
import { parse } from "yaml";
import profileSource from "../data/profile.yaml?raw";

const profileSchema = z.object({
  name: z.string(),
  role: z.string(),
  school: z.string().optional(),
  status: z.string(),
  email: z.email().optional(),
  avatar: z.string().optional(),
});

const load = <T>(source: string, schema: z.ZodType<T>): T =>
  schema.parse(parse(source));

export const profile = load(profileSource, profileSchema);
```

这种方式的关键点是：解析成功不等于数据有效。YAML 只负责把文本变成 JavaScript 值，Zod 才负责检查邮箱、数组、枚举和必填字段。

## 空链接和空图片怎样自动隐藏

配置驱动网站必须把“缺省”作为正常状态。

外部链接先过滤：

```ts
const projectLinks = [
  { label: "在线演示", href: data.links.demo },
  { label: "代码仓库", href: data.links.repository },
  { label: "项目文档", href: data.links.documentation },
].filter((link): link is { label: string; href: string } => Boolean(link.href));
```

模板再判断：

```astro
{projectLinks.length > 0 && (
  <nav>
    {projectLinks.map((link) => <a href={link.href}>{link.label}</a>)}
  </nav>
)}
```

头像同样使用条件渲染：

```astro
{profile.avatar && (
  <img src={profile.avatar} alt={profile.avatarAlt ?? ''} />
)}
```

不要把空值改成 `#`。`#` 仍然是一个可点击链接，只是行为错误。

## 功能开关与导航

尚未实现的页面不应只靠注释隐藏。ZGLab 使用显式功能开关：

```ts
export const features = {
  notes: false,
  tools: false,
  runtimeStatus: false,
  githubActivity: false,
  downloadResume: false,
};
```

导航项可以绑定 feature：

```ts
const navigation = [
  { label: "首页", href: "/" },
  { label: "笔记", href: "/notes", feature: "notes" },
];

export const visibleNavigation = navigation.filter(
  (item) => !item.feature || features[item.feature],
);
```

这避免了“页面还不存在，但入口已经上线”的问题。开启开关前仍需确保对应页面已经实现并通过构建。

## Schema 演进策略

内容模型会持续变化。新增字段时建议：

1. 先判断字段是否真的属于所有条目；
2. 对历史内容无法补齐的字段使用 `optional()` 或合理默认值；
3. 更新内容维护文档；
4. 在组件中实现空值策略；
5. 执行 `astro check` 和完整构建；
6. 检查静态产物中是否出现空链接或错误路径。

删除或重命名字段时，不要只改组件。必须同步搜索：

```bash
rg -n "oldFieldName" src
```

并检查所有 Markdown 和 YAML。

## 最小验证方法

```bash
npm run format:check
npm run check
npm run build
```

还可以增加针对构建产物的检查：

```bash
rg -n 'href=(""|"#")|src=""' dist --glob '*.html'
```

如果命令没有输出，说明当前构建产物中没有这些常见空值形式。但它不能代替浏览器交互、可访问性和内容事实审查。

## 常见误区

### 把所有内容都做成一个巨大 YAML

集中编辑方便，但大量项目正文塞进一个 YAML 后会难以审查、合并和维护。需要独立正文与路由的内容更适合一条一文件。

### 只写 TypeScript interface，不做运行时校验

TypeScript 类型不会自动检查 YAML 中的实际值。外部文件进入程序时仍需要 Zod 这类运行时校验。

### 为了复用而过度抽象 Schema

项目、论文和想法可能都有 `status`，但它们的状态集合和语义不同。强行共用一个宽泛枚举会降低约束价值。

### 把默认值当成真实事实

`visible: true` 是展示行为默认值；“项目开始时间为当前日期”却是事实声明。只有前者适合无条件默认。

### 认为构建成功就代表内容正确

Zod 能检查格式，不能判断论文是否真的发表、竞赛名次是否准确。事实仍需由内容维护者负责。

## 适用边界

这种架构适合个人网站、项目档案、文档站和中小型内容门户。如果内容由多人高频编辑、需要后台审核、实时预览或权限管理，可能需要接入 CMS 或数据库。

即便未来换成 CMS，Schema、空值策略和组件边界仍然可以保留。

## 总结

Content Collections 解决“一组独立内容如何被加载、查询和生成页面”，普通 YAML 解决“少量共享资料如何集中维护”，Zod 负责把两者都限制在可预测的结构内。

真正的配置驱动不是让模板读取任意对象，而是确保：

- 内容有明确来源；
- 数据有可执行约束；
- 查询有统一规则；
- 缺省有稳定表现；
- 新增内容不需要修改组件；
- 错误在构建阶段尽早暴露。

相关项目实践见：[从一张静态页面到可维护的个人数字档案](../projects/from-zero-to-maintainable-astro-personal-site.md)。

## 参考资料

- [Astro Content Collections 指南](https://docs.astro.build/en/guides/content-collections/)，访问于 2026-07-17。
- [Astro 安装与 TypeScript 配置](https://docs.astro.build/en/install-and-setup/)，访问于 2026-07-17。
