---
title: "从 JSON.parse 到 Web Worker：实现一个不会吞掉错误上下文的 JSON 格式化器"
date: 2026-07-17
updated: 2026-07-17
status: draft
category: projects
tags:
  - JSON
  - Web Worker
  - Preact
  - TypeScript
  - Vitest
summary: "以 ZGLab Tools 的 JSON 格式化器为例，逐步实现格式化、压缩、递归键排序、错误定位、后台处理、复制下载和边界测试。"
---

# 从 JSON.parse 到 Web Worker：实现一个不会吞掉错误上下文的 JSON 格式化器

## 要解决的不是一个按钮

最简单的 JSON 格式化器只需要两行代码：

```ts
const value = JSON.parse(input);
const output = JSON.stringify(value, null, 2);
```

真正做成可长期使用的网页工具，还需要回答更多问题：

- 非法 JSON 应该显示什么，而不是只说“解析失败”？
- 如何在不修改数组顺序的前提下递归排序对象键？
- 深层数据如何避免递归调用造成调用栈溢出？
- 大文本处理时怎样减少主线程卡顿？
- 格式化、压缩和校验是否应该共用同一套逻辑？
- 下载文件怎样在本地生成并及时释放资源？
- UI 如何保留原始输入、阻止重复操作并提供键盘快捷键？

ZGLab Tools 把这些问题拆成纯逻辑、Worker 通信和 Preact 交互三层。这样既能独立测试算法，也不会让第三方或浏览器 API 散落在组件里。

## 模块划分

当前实现涉及以下文件：

```text
src/
├── components/tools/JsonFormatter.tsx
├── pages/json-formatter.astro
├── tools/json/
│   ├── logic.ts
│   ├── logic.test.ts
│   ├── types.ts
│   ├── worker-client.ts
│   └── worker.ts
└── utils/download.ts
```

各层职责如下：

| 层次          | 职责                                                   |
| ------------- | ------------------------------------------------------ |
| `logic.ts`    | 解析、格式化、压缩、键排序、元数据和错误上下文         |
| `worker.ts`   | 在独立线程调用纯逻辑                                   |
| Worker 客户端 | 创建 Worker、发送请求、接收结果并终止线程              |
| Preact 组件   | 管理输入、选项、按钮状态、提示、错误和展示             |
| Astro 页面    | 提供静态页面外壳、SEO、使用说明和按页加载的交互 island |

## 先用联合类型固定结果边界

解析操作只有成功和失败两种结果。与其让函数返回一堆可能为空的字段，不如使用可辨识联合类型：

```ts
type JsonProcessResult =
  | {
      ok: true;
      output: string;
      value: JsonValue;
      metadata: JsonMetadata;
    }
  | {
      ok: false;
      issue: JsonParseIssue;
    };
```

组件只要判断 `result.ok`，TypeScript 就能自动收窄后续字段。错误对象保留：

- 浏览器原始错误消息；
- 字符位置；
- 行号和列号；
- 错误附近的一段文本；
- 指针在上下文中的偏移。

这比抛出一个字符串更适合 UI 展示，也方便单元测试逐项验证。

## 统一格式化、压缩和校验流程

三个操作都从 `JSON.parse` 开始，区别只在输出缩进和成功提示，因此可以使用同一个入口：

```ts
const parsed = JSON.parse(input) as JsonValue;
const value = options.sortKeys ? sortJsonKeys(parsed) : parsed;
const indentation = options.mode === "minify" ? 0 : options.indent;
const output = JSON.stringify(value, null, indentation);
```

当前“校验”模式也会生成规范化输出。这样校验成功后，用户仍然可以复制或下载结果，不需要再点一次格式化。

成功时同时计算：

- 顶层 JSON 类型；
- 顶层对象键数量或数组元素数量；
- 输入字符数；
- 输出字符数。

`null` 需要先于 `typeof` 判断，因为 JavaScript 中 `typeof null` 的结果是 `"object"`。数组也需要先使用 `Array.isArray` 分离，否则会和普通对象混在一起。

## 如何递归排序对象键而不改变数组顺序

排序规则是：

1. 普通对象的键按 `localeCompare` 排序；
2. 嵌套对象继续排序；
3. 数组元素位置保持不变；
4. 数组中的对象仍然排序键；
5. 字符串、数字、布尔值和 `null` 原样保留。

实现没有直接递归调用自身，而是显式维护待处理栈：

```ts
const stack = [{ source: value, target: root }];

while (stack.length > 0) {
  const current = stack.pop();
  // 为子对象创建目标容器，再把子任务放入 stack
}
```

显式栈不会把嵌套深度等量转化为 JavaScript 函数调用深度。对于极深但仍可被 `JSON.parse` 接受的数据，这种写法比朴素递归更稳妥。

实现还使用 `WeakSet<object>` 记录已经访问的容器：

```ts
if (seen.has(item)) {
  throw new TypeError("检测到循环引用，无法排序");
}
```

标准 JSON 文本本身不可能包含循环引用，但 `sortJsonKeys` 是导出的通用函数，未来也可能被其他代码直接传入对象。防御检查让函数边界更加完整。

## 从浏览器错误消息中恢复位置

不同 JavaScript 引擎的 `JSON.parse` 错误消息并不完全一致。有的提供：

```text
Unexpected token ... at position 12
```

有的会直接给出：

```text
line 3 column 1
```

当前实现分别匹配字符位置和行列信息：

```ts
const positionMatch = message.match(/position\s+(\d+)/i);
const lineColumnMatch = message.match(/line\s+(\d+)\s+column\s+(\d+)/i);
```

如果只拿到字符位置，就统计该位置之前的换行符，换算成行列；如果只拿到行列，就累加前面各行长度，换算成字符位置。

最后截取错误前后约 32 个字符，并把换行替换为可见的 `↵`：

```text
{"name":"ZGLab",↵"active":,}
                         ^
```

这里有一个需要诚实说明的限制：位置精度依赖浏览器提供的错误消息。如果引擎只返回笼统描述，工具可以保留原始消息，但不能凭空推断准确位置。

## 为什么把处理放进 Web Worker

超过 1 MB 的 JSON 在格式化和排序时可能产生多份字符串与对象副本。如果这些工作都发生在主线程，输入框、按钮和页面绘制会一起等待。

ZGLab Tools 使用构建工具支持的 Worker 导入：

```ts
import JsonProcessingWorker from "./worker?worker";
```

每次操作的生命周期是：

1. 创建专用 Worker；
2. 使用 `postMessage` 发送输入和选项；
3. Worker 调用相同的 `processJson` 纯函数；
4. 返回可结构化克隆的输出和元数据；
5. 收到 `message` 或 `error` 后立即 `terminate`。

Worker 不能直接操作 DOM，但可以运行 `JSON.parse`、`JSON.stringify` 和普通 TypeScript 逻辑。这正好符合“逻辑与 UI 分离”的设计。

当前实现为每次点击创建一个 Worker，结构简单且能保证任务结束后释放线程。若以后增加连续自动格式化，可以改成复用单个 Worker，并为请求增加 ID 和取消策略。

## 大输入提示按字节而不是字符判断

“1 MB”描述的是字节，不是 JavaScript 字符串的 `length`。中文和 Emoji 的 UTF-8 占用与 ASCII 不同，因此使用：

```ts
new TextEncoder().encode(input).byteLength;
```

超过 `1024 * 1024` 字节时只显示性能警告，不强制拦截。是否继续处理由用户决定。

## Preact 组件如何接入

组件维护以下状态：

- 原始输入；
- 处理输出；
- 2 或 4 空格缩进；
- 是否排序键；
- 成功元数据；
- 解析错误；
- 操作提示；
- `busy` 状态。

运行时先检查空输入和重复触发：

```ts
if (busy || input.trim() === "") return;
```

处理失败时只更新错误状态，不清空 `input`。这是错误定位工具最重要的交互原则之一：错误发生后，用户最需要的正是原始文本。

快捷键在工具根容器监听：

- `Ctrl/Cmd + Enter`：格式化；
- `Ctrl/Cmd + Shift + M`：压缩。

命中后调用 `preventDefault()`，避免浏览器或输入控件执行冲突行为。没有实现 `Ctrl/Cmd + K` 清空，因为它常与浏览器地址栏搜索等功能冲突。

Astro 页面使用：

```astro
<JsonFormatter client:load />
```

只有进入 JSON 工具页时才加载这一交互组件。首页和其他工具页不会加载 JSON Worker 客户端。

## 本地下载的实现

下载内容会补一个结尾换行，便于命令行工具读取：

```ts
const content = output.endsWith("\n") ? output : `${output}\n`;
```

随后在浏览器中创建 `Blob` 和临时 Object URL：

```ts
const url = URL.createObjectURL(blob);
anchor.href = url;
anchor.download = filename;
anchor.click();
URL.revokeObjectURL(url);
```

实际项目把释放操作放进零延迟定时器，避免某些浏览器在点击动作完成前过早撤销 URL。整个过程不需要上传文件，也不需要后端生成下载链接。

## 应该怎样测试

当前测试覆盖的重点不是按钮是否存在，而是纯逻辑是否满足约定：

- 合法对象和数组；
- 嵌套键递归排序；
- 数组顺序不变；
- `null`、空对象、空数组和原始值类型；
- 中文与 Emoji；
- 压缩输出；
- 尾逗号和未加引号键名；
- 字符位置到行列的换算；
- 下载内容结尾换行；
- 循环引用防御。

可以单独运行该测试文件：

```bash
npx vitest run src/tools/json/logic.test.ts
```

项目级验证使用：

```bash
npm run format:check
npm run lint
npm run check
npm run test
npm run build
```

## 当前限制与可继续改进的方向

当前版本有意保持为严格 JSON 工具，因此不支持：

- JSON5；
- 注释；
- 尾逗号；
- 未加引号的对象键；
- 流式解析；
- 超大文件分块处理。

如果以后需要处理几十 MB 以上的文件，应优先增加文件输入、流式解析器和任务取消能力，而不是继续扩大 `textarea` 能承受的数据量。

## 参考资料

- [MDN：Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [MDN：TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder)
- [ZGLab Tools 平台架构笔记](building-local-first-astro-tool-platform.md)
- [浏览器本地处理的数据与隐私边界](../knowledge/browser-local-processing-privacy-boundaries.md)
