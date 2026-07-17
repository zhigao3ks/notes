---
title: "秒、毫秒与时区之间：用 Intl 实现可靠的时间戳转换器"
date: 2026-07-17
updated: 2026-07-17
status: draft
category: projects
tags:
  - JavaScript Date
  - Intl
  - Time Zone
  - Preact
  - Vitest
summary: "以 ZGLab Tools 的时间戳转换器为例，讲解单位识别、日期合法性校验、IANA 时区换算、夏令时边界、实时钟和测试设计。"
---

# 秒、毫秒与时区之间：用 Intl 实现可靠的时间戳转换器

## 为什么时间转换比看起来更难

时间戳转日期似乎只需要：

```ts
new Date(Number(input));
```

这行代码没有回答：

- 输入是秒还是毫秒？
- `NaN`、`Infinity` 和超大数字如何处理？
- 负时间戳能否表示 1970 年以前？
- 用户输入的日期属于浏览器本地时区、UTC，还是其他 IANA 时区？
- 夏令时切换时根本不存在的本地时间应该怎样处理？
- 如何避免浏览器自动把 `2023-02-29` 修正成三月？
- 时区列表是否需要在代码中维护几百个字符串？

ZGLab Tools 把功能拆成当前时间、时间戳转日期和日期转时间戳三个区域，并把所有换算规则放在纯逻辑模块中。

## 模块结构

```text
src/
├── components/tools/TimestampConverter.tsx
├── pages/timestamp-converter.astro
└── tools/timestamp/
    ├── logic.ts
    ├── logic.test.ts
    └── types.ts
```

`TimestampConverter.tsx` 负责计时器、表单和结果展示；`logic.ts` 只接收字符串、单位、时区和可注入的当前时间，返回成功结果或明确错误。

## 第一步：明确秒和毫秒规则

当前支持三种单位：

```ts
type TimestampUnit = "auto" | "seconds" | "milliseconds";
```

自动识别使用绝对值阈值：

```ts
const threshold = 100_000_000_000;
const unit = Math.abs(value) >= threshold ? "milliseconds" : "seconds";
```

使用绝对值是为了让负时间戳也遵守同一规则。这个阈值是工程启发式，不是 Unix 时间戳标准的一部分。非常早期或非常遥远的日期可能存在歧义，因此 UI 必须同时允许用户手动指定单位，并公开显示识别规则。

## 第二步：拒绝不可表示的数字

输入先经过四道检查：

1. 去除首尾空白后不能为空；
2. `Number(input)` 必须是有限数字；
3. 秒级输入乘以 1000 后仍须为有限数字；
4. 绝对毫秒值不能超过 JavaScript `Date` 可表示范围。

当前范围上限是：

```ts
const MAX_DATE_MILLISECONDS = 8_640_000_000_000_000;
```

负数不会被拒绝，因为它可以合法表示 Unix 纪元之前的时间。例如 `-1` 秒对应 `1969-12-31T23:59:59.000Z`。

这种校验避免了两个常见错误：

- 把 `Infinity` 交给 `Date` 后才发现结果无效；
- 用 `value < 0` 粗暴拒绝所有 1970 年以前的日期。

## 第三步：一次转换生成多种表达

成功解析后，同一个 `Date` 会生成：

- Unix 秒；
- Unix 毫秒；
- ISO 8601；
- UTC 字符串；
- 浏览器本地时间；
- 指定时区时间；
- 相对当前时间；
- 实际识别的输入单位。

指定时区格式化由 `Intl.DateTimeFormat` 完成：

```ts
new Intl.DateTimeFormat("zh-CN", {
  timeZone,
  year: "numeric",
  month: "2-digit",
  day: "2-digit",
  hour: "2-digit",
  minute: "2-digit",
  second: "2-digit",
  weekday: "long",
  hourCycle: "h23",
}).format(date);
```

不手写“UTC+8”偏移，意味着夏令时和历史偏移规则由浏览器内置时区数据处理。

相对时间先选择最合适的年、月、日、小时、分钟或秒，再交给：

```ts
new Intl.RelativeTimeFormat("zh-CN", { numeric: "auto" });
```

这样可以得到本地化的“2 小时前”“3 天后”等结果。

## 第四步：从浏览器获取时区列表

现代浏览器可以提供它所支持的 IANA 时区：

```ts
Intl.supportedValuesOf?.("timeZone");
```

当前实现先做能力检测，并用 `try/catch` 防御异常。若不可用，就回退到：

- UTC；
- Asia/Shanghai；
- Asia/Tokyo；
- America/New_York；
- Europe/London。

这种方式避免在项目中复制一份很快会过时的完整时区表，同时保证旧浏览器仍然能完成常见转换。

## 最难的一步：指定时区的日期转时间戳

用户输入：

```text
2024-01-01 08:00:00 Asia/Shanghai
```

这里的 `08:00:00` 是目标时区的墙上时间，不是 UTC。原生 `Date` 构造函数不能直接接收任意 IANA 时区，因此当前实现采用“迭代校正 + 反向验证”。

### 先严格解析字段

日期和时间分别用正则解析，时间秒数可以省略：

```ts
/^(\d{4,6})-(\d{2})-(\d{2})$/
/^(\d{2}):(\d{2})(?::(\d{2}))?$/
```

月份、日期、小时、分钟和秒先做范围检查。这里只能排除明显错误，像 2 月 30 日仍需要后续反向校验。

### 把输入暂时当作 UTC 候选

先计算：

```ts
const targetUtc = Date.UTC(year, month - 1, day, hour, minute, second);
```

然后用目标时区格式化候选时间，观察它在该时区实际显示成什么日期字段。把显示字段再次视作 UTC 数值，与目标字段的差值就是下一轮需要修正的偏差。

当前进行三轮校正：

```ts
candidate += targetUtc - representedUtc;
```

### 最后必须反向验证

只做偏移校正还不够。以 `America/New_York` 为例，春季切换夏令时时，某一天的 `02:30` 可能根本不存在。浏览器或简单算法可能把它自动滚到其他时间。

因此得到最终 `Date` 后，再用目标时区格式化回年月日时分秒，并逐项比较：

```ts
return sameParts(partsFromDate(date, timeZone), inputParts) ? date : null;
```

如果无法完整还原用户输入，就返回：

```text
该日期无效，或处于时区的不存在时间段。
```

同样的反向验证也用于本地时区和 UTC，因此 `2023-02-29` 不会被静默修正成有效日期。

## 实时钟如何避免构建时与客户端不一致

Astro 会先生成静态 HTML，而“当前时间”只能在访问者浏览器中确定。组件初始状态设为 `null`，等挂载后再读取：

```ts
useEffect(() => {
  setNow(new Date());
}, []);
```

随后每秒更新一次，并在 effect 清理函数中取消定时器：

```ts
const timer = window.setInterval(() => setNow(new Date()), 1000);
return () => window.clearInterval(timer);
```

暂停时不再创建计时器，恢复后重新订阅。页面显示的是浏览器系统时钟，不是服务器时间，也没有暗示它经过网络校时。

## UI 层的几个细节

- 时间戳输入使用文本框和 `inputMode="decimal"`，避免数字输入控件对大整数做不透明处理；
- 自动识别结果会明确显示“秒”或“毫秒”；
- 日期、时间和时区是三个独立输入，不把时区藏在描述中；
- 所有结果都可以单独复制；
- 时区不受支持、日期无效或数值超界时清空旧结果并展示错误；
- 初始日期和时间只在浏览器挂载后填充，避免构建机器时区污染页面。

## 测试哪些边界才有价值

当前测试覆盖：

- `0` 时间戳；
- 秒级和毫秒级识别；
- 同一时刻的秒、毫秒结果一致；
- 负时间戳；
- 空值、`NaN`、`Infinity` 和超范围数字；
- Asia/Shanghai 格式化；
- 闰年 2 月 29 日；
- 非法日期；
- 指定时区到 UTC 的转换；
- 夏令时跳变前的合法时间和跳变中的不存在时间；
- 相对过去和未来时间。

时区测试不应手写所有地区的固定偏移。地区规则会受日期影响，应让 `Intl` 处理规则，并验证转换关系或可逆性。

单独执行：

```bash
npx vitest run src/tools/timestamp/logic.test.ts
```

## 当前限制

当前方案适合轻量浏览器工具，但仍有边界：

- 自动单位识别只是公开的启发式规则；
- 秋季夏令时回拨会出现两个同名本地时刻，当前实现没有让用户选择较早或较晚实例；
- 精度到秒，未提供毫秒输入字段；
- 相对时间中的“月”和“年”使用固定秒数近似；
- 准确性依赖浏览器系统时钟和内置时区数据。

如果未来引入原生 `Temporal` 或成熟的时区库，应继续保留“输入字段反向验证”和“歧义时间显式选择”这两个原则。

## 参考资料

- [MDN：Intl.supportedValuesOf](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/supportedValuesOf)
- [MDN：Intl.RelativeTimeFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/RelativeTimeFormat)
- [MDN：JavaScript 国际化 API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Internationalization)
- [ZGLab Tools 平台架构笔记](building-local-first-astro-tool-platform.md)
