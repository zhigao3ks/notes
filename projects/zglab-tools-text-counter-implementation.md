---
title: "Emoji 到底算几个字符：实现 Unicode 感知的文本统计器"
date: 2026-07-17
updated: 2026-07-17
status: draft
category: projects
tags:
  - Unicode
  - Intl Segmenter
  - Text Processing
  - Preact
  - Vitest
summary: "以 ZGLab Tools 文本统计器为例，说明可见字符、中文、英文单词、段落、UTF-8 字节和阅读时间的统计规则与实现。"
---

# Emoji 到底算几个字符：实现 Unicode 感知的文本统计器

## 统计之前必须先定义规则

文本统计最容易犯的错误，是直接展示一组数字，却不说明这些数字代表什么。

例如 JavaScript 中：

```ts
"😀".length === 2;
```

这是因为 `length` 统计 UTF-16 码元，不等于用户看到的字符。家庭 Emoji `👨‍👩‍👧‍👦` 由多个码点和连接符组成，即使用 `Array.from` 也不能保证把它当作一个可见字符。

因此，ZGLab Tools 在写算法前先确定规则：

- 总字符优先按 Unicode 字素簇统计；
- 中文字符按 Unicode `Script=Han` 统计；
- 英文字母实际按拉丁文字脚本统计；
- 英文单词按拉丁字母及合理连接符识别；
- 段落按一个或多个空行分隔；
- 空文本为 0 行；
- CRLF 和 LF 先统一；
- 字节数按 UTF-8 编码计算；
- 阅读时间是可配置速度下的估算，不是假装精确的结论。

只有先固定口径，UI、测试和下载报告才可能一致。

## 模块结构

```text
src/
├── components/tools/TextCounter.tsx
├── pages/text-counter.astro
├── tools/text-counter/
│   ├── logic.ts
│   ├── logic.test.ts
│   └── types.ts
└── utils/text.ts
```

纯函数 `countText` 接收文本和阅读速度，返回完整统计对象。组件只负责防抖、设置项和展示。

## 用 Intl.Segmenter 统计可见字符

现代浏览器提供 `Intl.Segmenter`，可以按字素、单词或句子切分文本。当前实现选择 `grapheme`：

```ts
const segmenter = new Intl.Segmenter("zh-CN", {
  granularity: "grapheme",
});

const characters = [...segmenter.segment(input)].map(
  (segment) => segment.segment,
);
```

这样普通 Emoji、代理对以及常见的零宽连接 Emoji 序列可以按一个用户可见字符统计。

旧浏览器若没有 `Intl.Segmenter`，会回退到：

```ts
Array.from(input);
```

该降级至少能按 Unicode 码点拆分，而不是按 UTF-16 码元拆分，但复杂组合 Emoji 可能仍被统计为多个字符。教程和页面都不应把降级结果描述成绝对准确。

不含空白字符数是在字素数组上过滤：

```ts
characters.filter((character) => !/^\s+$/u.test(character)).length;
```

这让总字符和非空白字符使用同一基本单位。

## 使用 Unicode 属性转义分类字符

与其维护一份汉字区间，不如使用 Unicode 属性转义：

```ts
const chinese = input.match(/\p{Script=Han}/gu)?.length ?? 0;
const latin = input.match(/\p{Script=Latin}/gu)?.length ?? 0;
const digits = input.match(/\p{Number}/gu)?.length ?? 0;
const punctuation = input.match(/\p{Punctuation}/gu)?.length ?? 0;
```

需要注意名称和规则之间的差异：

- 页面上的“中文字符”实际统计 Han 脚本字符，因此日文汉字等也可能计入；
- 页面上的“英文字母”实际统计 Latin 脚本字符，不仅限于 ASCII `A-Z`；
- `\p{Number}` 比 `[0-9]` 范围更广；
- `\p{Punctuation}` 根据 Unicode 标点类别判断。

如果产品未来需要语言学意义上的“中文”和“英文”，就不能只依靠脚本类别，需要重新定义口径。

空格数量当前统计：

```ts
/[ \t\u3000]/gu;
```

即普通空格、制表符和全角空格，不把换行算作“空格”。

## 英文单词为什么不直接按空格切

下面的写法会错误处理标点和多个空格：

```ts
input.split(" ");
```

当前规则使用：

```ts
/[A-Za-z]+(?:['’-][A-Za-z]+)*/g;
```

它能把以下内容作为合理单词：

- `ZGLab`；
- `don't`；
- `user-facing`；
- 使用弯引号的英文缩写。

这个规则有意只统计 ASCII 英文字母组成的单词。带重音字符或其他语言的拉丁文字不会完整进入“英文单词”统计，这是当前版本的已知口径，而不是算法遗漏后仍宣称支持所有语言。

## 行和段落的处理

Windows 常用 `\r\n`，Unix 常用 `\n`。统计前统一为：

```ts
const normalized = input.replace(/\r\n?/g, "\n");
```

空文本单独返回空行数组：

```ts
const lines = input === "" ? [] : normalized.split("\n");
```

如果直接执行 `''.split('\n')`，结果会是包含一个空字符串的数组，从而错误显示为 1 行。

非空行使用 `line.trim() !== ''` 判断。段落先对全文 `trim`，再按一个或多个空行切分：

```ts
normalized
  .split(/\n[ \t\u3000]*\n+/)
  .filter((paragraph) => paragraph.trim() !== "");
```

因此只有空格或全角空格的分隔行也能形成段落边界。

## UTF-8 字节数

JavaScript 字符串内部表示不能直接告诉我们 UTF-8 文件大小。当前实现使用：

```ts
new TextEncoder().encode(input).byteLength;
```

测试中的典型结果：

| 文本 | UTF-8 字节 |
| ---- | ---------- |
| `a`  | 1          |
| `中` | 3          |
| `😀` | 4          |

这项指标适合估算网络负载、文件大小或接口字段限制，但不能与可见字符数混为一谈。

## 阅读时间如何估算

默认速度是：

```ts
const speeds = {
  chineseCharactersPerMinute: 400,
  englishWordsPerMinute: 225,
};
```

分别计算：

```ts
chineseMinutes = chineseCharacters / chineseSpeed;
englishMinutes = englishWords / englishSpeed;
combinedMinutes = chineseMinutes + englishMinutes;
```

用户可以在 1 到 2000 的范围内调整速度。组件通过 `clampNumber` 限制输入，逻辑层也对非正速度做防御，避免除以零。

综合时间采用两部分相加，是一个清晰、可复算的工程估算。它不考虑：

- 内容难度；
- 代码块；
- 图表；
- 重读；
- 中英文混合时的上下文切换。

因此 UI 使用“估算”措辞，不把结果包装成精确阅读时长。

## 为什么实时统计仍然需要防抖

每次键盘输入都会产生新字符串。字素切分、多个正则匹配和 UTF-8 编码都需要遍历文本。大文本下，每个按键立即重复完整计算没有必要。

组件把原始输入和待统计输入分开：

```ts
const [input, setInput] = useState("");
const [debouncedInput, setDebouncedInput] = useState("");
```

每次输入后等待 140 毫秒：

```ts
useEffect(() => {
  const timer = window.setTimeout(() => setDebouncedInput(input), 140);
  return () => window.clearTimeout(timer);
}, [input]);
```

如果用户继续输入，上一轮定时器会被清理。统计结果再通过 `useMemo` 依赖防抖文本和阅读速度：

```ts
useMemo(() => countText(debouncedInput, speeds), [debouncedInput, speeds]);
```

这不是把任务移出主线程，而是减少无意义的调用频率。若未来支持数十 MB 文本，仍应考虑 Worker 或增量统计。

## 统计摘要和本地下载

展示层使用仪表板式 `dl` 列表，下载层则把同一份统计对象转换为纯文本报告：

```text
ZGLab Tools · 文本统计报告

总字符数：...
中文字符：...
英文单词：...
UTF-8 字节：...
```

复制和下载使用同一个 `report` 字符串，避免页面看到的口径与文件口径不一致。TXT 由浏览器本地创建 `Blob`，不会把正文发给服务器。

## 测试设计

当前单元测试覆盖：

- 空文本的字符、行和段落都为 0；
- 纯中文和中文标点；
- 英文字母和带撇号单词；
- 中英文、数字、标点混合；
- 单个 Emoji；
- 零宽连接的家庭 Emoji；
- LF 与 CRLF 一致；
- 多个连续空行；
- UTF-8 字节；
- 自定义阅读速度；
- 可下载报告内容。

单独执行：

```bash
npx vitest run src/tools/text-counter/logic.test.ts
```

## 可以继续改进什么

后续可以根据产品目标增加：

- 使用 `Intl.Segmenter` 的单词模式扩展多语言词数；
- 代码行、句子数和最长行；
- Markdown 去除语法后的正文统计；
- Worker 处理超大文本；
- 允许用户导入本地文件但不上传；
- 将统计口径导出为机器可读 JSON。

无论增加什么指标，都应先定义规则、再实现、最后用反例测试，不能只追求页面上的数字数量。

## 参考资料

- [MDN：Intl.Segmenter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter)
- [MDN：TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder)
- [MDN：JavaScript 国际化 API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization)
- [浏览器本地处理的数据与隐私边界](../knowledge/browser-local-processing-privacy-boundaries.md)
