---
title: "共享组件不是万能组件：工具平台中的受控复用设计"
date: 2026-07-29
updated: 2026-07-29
status: draft
category: knowledge
tags:
  - Frontend Architecture
  - Component Design
  - Preact
  - TypeScript
summary: "总结 ZGLab Tools 中 ImageTool、TextEnhancer、DesignTool 等共享组件模式，以及为什么显式 mode 映射优于万能组件。"
---

# 共享组件不是万能组件：工具平台中的受控复用设计

## 问题背景

当工具数量增加时，最直观的做法是复制组件：

```text
ImageCompressor.tsx
ImageResize.tsx
ImageConverter.tsx
```

短期简单，但长期会产生重复状态、重复样式和重复逻辑。

另一种极端是设计一个万能组件：

```ts
<Tool type="image" mode="compress" config={hugeObject}/>
```

虽然减少文件数量，但组件内部会出现大量条件判断，最终难以维护。

## 更合理的方法：受控复用

工具平台更适合采用：

```text
多个工具
    ↓
共享领域组件
    ↓
明确 mode
    ↓
独立逻辑函数
```

例如：

```tsx
<ImageTool mode="compress" />
<ImageTool mode="resize" />
<ImageTool mode="convert" />
```

组件共享结构，但每个能力仍然具有明确身份。

## 判断是否应该合并组件

适合共享：

- 输入方式类似；
- 输出形式类似；
- 生命周期类似；
- 通用操作一致。

例如图片处理工具通常都有：

- 文件读取；
- Canvas处理；
- Blob导出；
- 下载。

不适合共享：

- 核心交互完全不同；
- 状态模型不同；
- 错误处理不同；
- 未来扩展方向不同。

## 显式映射优于动态魔法

工具平台中推荐：

```ts
switch(toolId){
 case 'image-compressor':
   return <ImageTool mode="compress" />
}
```

虽然代码略多，但优点明显：

- 所有支持能力可搜索；
- 类型检查更可靠；
- 新开发者容易理解；
- 不会因为配置错误加载未知能力。

## 逻辑与组件继续分离

共享组件只负责：

- 状态；
- 用户操作；
- 展示结果。

纯逻辑负责：

- 转换算法；
- 输入校验；
- 错误处理；
- 单元测试。

这样一个组件可以变化，而算法仍然独立可靠。

## 总结

优秀的组件设计不是最大程度减少文件数量，而是在变化频率不同的部分建立边界。

工具平台中的最佳实践通常是：

- 小能力独立；
- 同领域能力共享组件；
- 通过显式模式控制变化；
- 核心逻辑保持纯函数。

复用的目标不是少写代码，而是降低未来修改成本。
