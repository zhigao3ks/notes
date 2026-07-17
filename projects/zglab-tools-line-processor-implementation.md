---
title: "去重、自然排序与随机打乱：实现可预测的逐行文本处理流水线"
date: 2026-07-17
updated: 2026-07-17
status: draft
category: projects
tags:
  - Text Processing
  - Unicode Normalization
  - Sorting
  - TypeScript
  - Vitest
summary: "以 ZGLab Tools 文本去重与排序工具为例，构建换行标准化、Trim、空行处理、去重、排序和统计组成的确定性流水线。"
---

# 去重、自然排序与随机打乱：实现可预测的逐行文本处理流水线

## 先固定处理顺序

逐行文本工具通常提供许多复选框：Trim、删除空行、忽略大小写、去重、排序、反转。真正困难的不是实现每个操作，而是定义它们组合后的顺序。

假设输入是：

```text
 ZGLab
zglab

```

如果先去重再 Trim，与先 Trim 再去重可能得到不同结果。功能越多，这类歧义越容易变成难以复现的 Bug。

ZGLab Tools 使用一条固定流水线：

```text
换行标准化
  → 按行拆分
  → Trim
  → 空行处理
  → 去重
  → 排序、打乱或反转
  → 合并输出
```

这条顺序同时写在逻辑代码、页面说明和测试中。用户改变的是每一步的选项，而不是步骤本身的相对位置。

## 模块结构

```text
src/
├── components/tools/TextLineProcessor.tsx
├── pages/text-line-processor.astro
└── tools/line-processor/
    ├── logic.ts
    ├── logic.test.ts
    └── types.ts
```

核心函数签名是：

```ts
processLines(input, options, random?)
```

第三个参数允许测试注入可预测的随机数函数，生产环境默认使用 `Math.random`。

## 用类型把组合限制在合法范围

每组选项使用字符串联合类型：

```ts
type EmptyLineMode = "preserve" | "remove" | "merge";
type DedupeMode = "none" | "first" | "last";
type LineOrder =
  "keep" | "asc" | "desc" | "natural" | "numeric" | "random" | "reverse";
```

完整配置包括：

- 是否 Trim；
- 空行处理方式；
- 重复行保留策略；
- 是否大小写敏感；
- 是否使用 NFKC 标准化值判重；
- 输出顺序。

默认值集中在 `DEFAULT_LINE_OPTIONS`，组件、逻辑和测试复用同一个对象，避免三处默认规则逐渐不一致。

## 第一步：统一换行符

输入可能来自 Windows、Linux、网页或表格软件。先统一：

```ts
input.replace(/\r\n?/g, "\n");
```

它同时处理：

- Windows 的 `\r\n`；
- 旧式单独 `\r`；
- 已经是 `\n` 的文本。

空字符串单独转为空数组，否则 `''.split('\n')` 会被误判成一行。

## 第二步：Trim 与空行处理

启用 Trim 时，对每一行调用 `line.trim()`。这一步发生在空行识别之前，因此只含空格的行会变成真正的空字符串。

空行有三种模式。

### 保留

直接返回原数组，不删除任何空行。

### 删除

只保留 `line !== ''` 的行，并用前后数组长度差计算删除数量。

### 合并

顺序扫描并记录上一行是否为空：

```ts
if (isEmpty && previousWasEmpty) {
  removed += 1;
  continue;
}
```

这样连续多个空行只保留一个，非连续空行仍然存在。

## 第三步：判重键与展示文本分离

“忽略大小写”和“Unicode 标准化”只应该影响是否重复，不应悄悄改写用户最终看到的文本。

当前实现先生成判重键：

```ts
const normalized = normalizeForDedupe ? line.normalize("NFKC") : line;

const key = caseSensitive ? normalized : normalized.toLocaleLowerCase();
```

`NFKC` 可以让部分兼容字符使用一致形式比较。保存到输出数组中的仍然是该行当前文本；如果开启 Trim，则是 Trim 后的文本，否则保留原始空白。

这就是“按标准化后的值判重，但输出保留代表行文本”的实现方式。

## 保留首次和保留末次不是同一个算法

### 保留首次

使用 `Set` 顺序扫描：

```ts
if (seen.has(key)) return false;
seen.add(key);
return true;
```

第一次出现的行进入结果，后续相同键被删除。

### 保留末次

先用 `Map` 记录每个键最后出现的位置：

```ts
lines.forEach((line, index) => {
  lastIndexes.set(createKey(line), index);
});
```

再保留“当前位置等于最后位置”的行。这样输入：

```text
a
b
a
c
```

得到：

```text
b
a
c
```

输出保留的是各键最后一次出现的内容，并按这些最后出现位置的先后排列。

## 字典序、自然排序和数字排序

三个名称必须对应三套明确规则。

### 字典序

使用中文区域规则的 `Intl.Collator`：

```ts
new Intl.Collator("zh-CN", {
  sensitivity: caseSensitive ? "variant" : "base",
});
```

升序直接比较，降序在升序结果上反转。它比直接使用 `<` 比较 Unicode 码点更适合面向用户的中文和混合文本。

### 自然排序

自然排序使用同一个 Collator，但打开：

```ts
{
  numeric: true;
}
```

因此：

```text
item1
item2
item10
```

不会变成常见字典序下的 `item1、item10、item2`。

### 数字排序

数字排序只把“整行可被识别为有限数字”的内容转成数字。正则支持：

- 正负号；
- 整数；
- 小数；
- 科学计数法。

通过正则后仍会检查 `Number.isFinite`。

排序规则是：

1. 两行都是数字时按数值比较；
2. 只有一行是数字时，数字排在前面；
3. 两行都不是数字时，使用 Collator 比较。

混合文本不会因为 `Number()` 失败而被过滤或变成 `NaN` 后丢失。

## 随机打乱为什么不能用随机 sort

下面的写法很常见：

```ts
lines.sort(() => Math.random() - 0.5);
```

比较器不满足稳定的传递关系，不会产生可靠的均匀洗牌。

当前实现使用 Fisher–Yates：

```ts
for (let index = output.length - 1; index > 0; index -= 1) {
  const target = Math.floor(random() * (index + 1));
  [output[index], output[target]] = [output[target], output[index]];
}
```

算法操作输入副本，不修改传入数组。把 `random` 设计成参数后，测试可以注入固定序列，验证元素既不增加也不丢失。

## 统计信息如何产生

处理结果不仅返回字符串，还返回行数组与统计：

- 输入行数；
- 输出行数；
- 删除重复行数；
- 删除空行数；
- 处理前字符数；
- 处理后字符数。

统计必须在对应步骤完成时产生。不能简单用“输入行数减输出行数”表示重复行，因为空行删除也会改变总数。

## Preact 交互层

桌面端输入和输出左右排列，手机端由 CSS 切换为上下布局。组件提供：

- 开始处理；
- 交换输入输出；
- 复制；
- 下载 TXT；
- 加载示例；
- 清空；
- 完整选项区域。

点击处理后先设置 `busy`，再通过零延迟定时器让浏览器有机会绘制“处理中”状态：

```ts
await new Promise<void>((resolve) => {
  window.setTimeout(resolve, 0);
});
```

需要注意，这只是在计算前让出一次事件循环，真正的 `processLines` 仍在主线程执行。当前文本规模下足够简单；若未来支持超大文件，应像 JSON 工具一样移入 Worker。

交换按钮会把当前输出放回输入，便于用户进行第二轮不同规则处理。下载时补结尾换行，生成本地 TXT 文件。

## 测试组合而不是只测单个函数

当前测试覆盖：

- 默认保留首次重复行；
- 保留末次及末次出现顺序；
- 忽略大小写去重；
- Trim 后去重；
- CRLF 标准化；
- 删除空行；
- 合并连续空行；
- `item2` 排在 `item10` 前；
- 数字和混合文本共同排序；
- 中文内容不丢失；
- 反转顺序；
- Fisher–Yates 不增加、不删除元素，也不修改原数组。

运行：

```bash
npx vitest run src/tools/line-processor/logic.test.ts
```

## 当前限制和扩展方向

当前版本没有实现：

- 撤销历史；
- 多列 CSV 排序；
- 自定义正则提取排序键；
- 超大文件 Worker；
- 具有种子的可复现随机排序；
- 单独的数字降序模式。

如果继续扩展，应该保留“固定流水线”原则。新增步骤时，需要同时更新：

1. 类型定义；
2. 默认配置；
3. 纯逻辑；
4. 页面说明；
5. 统计口径；
6. 组合测试。

## 参考资料

- [MDN：JavaScript 国际化与 Collator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization)
- [ZGLab Tools 平台架构笔记](building-local-first-astro-tool-platform.md)
- [浏览器本地处理的数据与隐私边界](../knowledge/browser-local-processing-privacy-boundaries.md)
