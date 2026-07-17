---
title: "不上传内容也能生成二维码：从 Payload 封装到 PNG、SVG 下载"
date: 2026-07-17
updated: 2026-07-17
status: draft
category: projects
tags:
  - QR Code
  - Browser API
  - Privacy
  - Preact
  - TypeScript
summary: "以 ZGLab Tools 二维码生成器为例，逐步实现文本、URL、邮箱、电话和 Wi-Fi Payload、本地预览、容量校验、颜色提示与文件下载。"
---

# 不上传内容也能生成二维码：从 Payload 封装到 PNG、SVG 下载

## 设计目标

二维码生成器经常被做成一个远程接口：前端提交文本，服务器返回图片。对于可能包含 Wi-Fi 密码、内部地址或临时文本的工具，这条链路没有必要。

ZGLab Tools 把 `qrcode` 库打包进静态资源，所有步骤都在当前浏览器完成：

```text
用户输入
  → 生成标准 Payload
  → UTF-8 容量检查
  → 本地编码
  → Data URL 预览
  → PNG Blob 或 SVG 文本下载
```

页面不会自动打开二维码中的 URL，不执行输入内容，也不把正文写入 localStorage。

## 模块结构

```text
src/
├── components/tools/QrCodeGenerator.tsx
├── pages/qr-code-generator.astro
├── tools/qrcode/
│   ├── logic.ts
│   ├── logic.test.ts
│   └── types.ts
└── utils/download.ts
```

第三方库只在 `logic.ts` 中导入：

```ts
import QRCode from "qrcode";
```

Preact 组件调用项目自己的 `generateQrPngDataUrl` 和 `generateQrSvg`，不会直接散落大量库参数。以后替换库时，UI 层可以保持稳定。

## 先建立内容类型

当前支持：

```ts
type QrContentType = "text" | "url" | "email" | "phone" | "wifi";
```

普通文本和 URL 保持输入值；邮箱和电话分别增加：

```text
mailto:
tel:
```

所有非 Wi-Fi 内容先 `trim`，空值直接抛出面向用户的错误。生成二维码只编码字符串，不验证或访问 URL。

## Wi-Fi Payload 的转义

Wi-Fi 二维码使用形如：

```text
WIFI:T:WPA;S:Network;P:password;H:false;;
```

SSID 和密码可能包含反斜杠、分号、逗号、冒号或引号。这些字符在 Payload 中有语法含义，必须转义：

```ts
value.replace(/([\\;,:"])/g, "\\$1");
```

构建时根据加密类型决定是否加入密码：

```ts
const password =
  type === "nopass" ? "" : `P:${escapeWifiValue(wifi.password)};`;
```

无密码网络不会把输入框中的残留值写进 Payload。SSID 不能为空，并在构建前去除首尾空白。

这里的隐私边界要说清楚：Wi-Fi 密码虽然不上传、不持久化，但为了编码二维码，仍会短暂存在于组件状态、Payload 字符串和生成结果中。浏览器扩展、恶意软件和本地设备安全不属于网站能完全控制的范围。

## 封装 qrcode 库参数

UI 设置统一为：

```ts
interface QrSettings {
  size: number;
  margin: number;
  errorCorrectionLevel: "L" | "M" | "Q" | "H";
  foreground: string;
  background: string;
  transparentBackground: boolean;
}
```

逻辑层把它转换成库选项：

```ts
{
  width: settings.size,
  margin: settings.margin,
  errorCorrectionLevel: settings.errorCorrectionLevel,
  color: {
    dark: settings.foreground,
    light: settings.transparentBackground
      ? '#00000000'
      : settings.background,
  },
}
```

默认值集中在 `DEFAULT_QR_SETTINGS`。重置按钮直接恢复该对象，不在组件中重复写一套颜色和尺寸。

PNG 入口返回 Data URL：

```ts
QRCode.toDataURL(content, {
  ...options,
  type: "image/png",
});
```

SVG 入口返回字符串：

```ts
QRCode.toString(content, {
  ...options,
  type: "svg",
});
```

## 为什么按 UTF-8 字节检查容量

二维码容量不只取决于 JavaScript 字符数，还受编码模式、版本和纠错等级影响。当前项目以字节模式上限作为保守的前置检查：

| 等级 | 当前字节上限 |
| ---- | ------------ |
| L    | 2953         |
| M    | 2331         |
| Q    | 1663         |
| H    | 1273         |

字节数由：

```ts
new TextEncoder().encode(content).byteLength;
```

计算。中文和 Emoji 不会被错误地按一个 ASCII 字节处理。

页面提示使用“最多建议”措辞，因为二维码库可能根据内容选择不同编码段，实际容量还与版本和混合模式有关。前置校验负责尽早给出可理解错误，最终生成是否成功仍由库决定。

纠错等级越高，允许恢复的破损越多，但可用容量越小。默认使用 M，在容量和容错之间取中间值。

## 实时预览如何避免频繁生成

尺寸、颜色、类型和输入任何一项变化都会影响结果。若每个按键都立即编码，输入过程中会产生大量无意义任务。

组件使用 220 毫秒防抖：

```ts
const timer = window.setTimeout(async () => {
  // 构建 Payload、校验并生成 PNG
}, 220);
```

effect 清理时：

```ts
cancelled = true;
window.clearTimeout(timer);
```

异步生成已经开始后无法仅靠 `clearTimeout` 取消，因此还需要 `cancelled` 标志，阻止旧任务完成后覆盖新输入的预览。

每轮开始前先清空旧预览、Payload 和错误。空输入不显示红色错误，只展示空状态；用户已经输入但格式或容量有问题时才显示错误。

预览使用普通 `<img src={preview}>`，说明文本通过 Preact 文本节点渲染。没有使用 `innerHTML` 注入用户内容，也不会把生成的 SVG 直接插入 DOM。

## 颜色对比提示是扫码启发式

二维码通常需要深色前景和浅色背景。当前实现把十六进制颜色转换为 sRGB 相对亮度，再计算：

```ts
(lighter + 0.05) / (darker + 0.05);
```

当比值低于 `3:1` 时提示可能影响扫码。这个阈值是实用警告，不等于对所有相机、屏幕和二维码尺寸的成功保证。

透明背景无法计算固定对比度，因此单独提示：放到复杂底图上可能降低识别率。

## PNG 与 SVG 下载

### PNG

预览已经是 Base64 Data URL。下载前先解析 MIME 类型和内容，再转换成 `Blob`：

```ts
const blob = await dataUrlToBlob(preview);
downloadBlob(blob, createQrFilename("png"));
```

### SVG

下载时使用相同 Payload 和设置重新生成 SVG 字符串，再创建文本 Blob：

```ts
const svg = await generateQrSvg(payload, settings);
downloadText(svg, filename, "image/svg+xml;charset=utf-8");
```

文件名使用浏览器本地时间：

```text
zglab-qr-YYYYMMDD-HHmmss.png
zglab-qr-YYYYMMDD-HHmmss.svg
```

Object URL 在点击后释放。所有文件均在浏览器内生成，不存在上传后再下载的中间步骤。

## Preact 交互细节

组件根据内容类型切换字段：

- 文本、URL、邮箱和电话使用文本区域；
- Wi-Fi 使用 SSID、密码、加密类型和隐藏网络；
- 无密码模式禁用密码输入；
- 空内容时下载和复制按钮禁用；
- 生成期间展示 `busy` 状态；
- 可以加载示例、清空、复制实际 Payload；
- 可以调整尺寸、边距、纠错等级、前景色、背景色和透明背景；
- 重置样式不修改正文。

密码字段使用 `type="password"` 和 `autocomplete="new-password"`，降低浏览器把它当作网站登录密码自动填充的可能性。

## 单元测试

当前测试覆盖：

- 普通文本；
- URL；
- 中文和 Emoji；
- `mailto:` 与 `tel:`；
- Wi-Fi 特殊字符转义；
- 开放 Wi-Fi 不包含密码字段；
- 空输入；
- SVG 生成入口；
- PNG Data URL 生成入口；
- 不同纠错等级的容量校验；
- 文件名；
- 黑白和同色对比度。

运行：

```bash
npx vitest run src/tools/qrcode/logic.test.ts
```

二维码测试不需要真的调用摄像头扫描每张图。纯逻辑测试验证 Payload 和生成入口；扫码兼容性仍适合在真实手机、不同颜色和不同尺寸下做人工验收。

## 当前限制

当前实现没有：

- Logo 嵌入；
- 二维码扫描；
- vCard；
- 批量生成；
- 自定义二维码版本；
- 扫码设备兼容性自动测试；
- 对 URL、邮箱或电话做真实性验证。

其中最后一点是有意设计：工具负责编码用户提供的字符串，不替用户访问目标或判断地址可信度。

## 参考资料

- [node-qrcode 官方 README](https://github.com/soldair/node-qrcode/blob/master/README.md)
- [MDN：TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder)
- [浏览器本地处理的数据与隐私边界](../knowledge/browser-local-processing-privacy-boundaries.md)
- [ZGLab Tools 平台架构笔记](building-local-first-astro-tool-platform.md)
