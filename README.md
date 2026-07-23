# Notes

这个仓库用于沉淀能够长期复用和对外分享的学习笔记、问题复盘与项目实践。

## 当前笔记

### 项目实践

- [从五个小工具到可扩展平台：Astro + Preact 构建本地优先工具箱](projects/building-local-first-astro-tool-platform.md)
- [从 JSON.parse 到 Web Worker：实现一个不会吞掉错误上下文的 JSON 格式化器](projects/zglab-tools-json-formatter-implementation.md)
- [秒、毫秒与时区之间：用 Intl 实现可靠的时间戳转换器](projects/zglab-tools-timestamp-converter-implementation.md)
- [Emoji 到底算几个字符：实现 Unicode 感知的文本统计器](projects/zglab-tools-text-counter-implementation.md)
- [去重、自然排序与随机打乱：实现可预测的逐行文本处理流水线](projects/zglab-tools-line-processor-implementation.md)
- [不上传内容也能生成二维码：从 Payload 封装到 PNG、SVG 下载](projects/zglab-tools-qrcode-generator-implementation.md)
- [不靠免密 sudo 的静态站发布：目录授权、严格健康检查与重复覆盖](projects/sudo-free-static-deployment-and-strict-health-check.md)
- [从一张静态页面到可维护的个人数字档案：Astro 个人网站从零搭建实践](projects/from-zero-to-maintainable-astro-personal-site.md)
- [从构建到上线：Ubuntu 24.04 + Nginx 部署 Astro 静态网站](projects/deploy-astro-static-site-with-nginx.md)
- [从多入口免登到统一身份：从属应用接入 CAS 的设计实践](projects/subordinate-app-cas-unified-auth.md)
- [两个长期分叉仓库如何安全同步：以测试仓与生产仓联调为例](projects/dual-repository-sync-with-environment-isolation.md)
- [断线不等于结束：实时会议录音的可恢复会话设计](projects/recoverable-realtime-recording-session.md)
- [从“录音正常”到证据链完整：移动会议录音的可观测性改造](projects/observable-mobile-recording-pipeline.md)
- [从岗位 JD 到可追溯 PDF：ResumeTailor Agent 的可控工作流设计](projects/resume-tailor-agent-controlled-workflow.md)

### 问题复盘

- [应用已经启动，却被发布脚本自动回滚：一次过期健康检查的排查复盘](problems/stale-health-check-causes-false-rollback.md)
- [匹配不少，为什么最终简历仍然是空的：一次多节点生成链路复盘](problems/resume-matches-but-generated-document-empty.md)
- [最后音频停了两小时，会议为什么还在进行中：一次僵尸录音会议复盘](problems/meeting-stuck-recording-after-recovery-expiry.md)
- [1000 / Bye 明明是正常关闭，为什么却被记成录音失败](problems/normal-websocket-close-misclassified-during-recovery.md)

### 知识与方案

- [浏览器本地处理不等于天然安全：工具网站的数据与隐私边界](knowledge/browser-local-processing-privacy-boundaries.md)
- [用 Astro Content Collections、YAML 与 Zod 构建配置驱动网站](knowledge/astro-content-collections-config-driven-content.md)
- [静态网站也需要工程化：备份发布、功能开关与动态接口边界](knowledge/static-site-release-and-runtime-boundaries.md)
- [不要把临时 OSS 地址直接交给前端：录音安全中转与签名链接设计](knowledge/secure-recording-proxy-and-signed-links.md)
- [内嵌 H5 为什么拿不到麦克风：Cordova 权限桥接与能力探测](knowledge/hybrid-app-microphone-permission-bridge.md)
- [长会议 H5 如何避免越用越卡：有界状态、音频自愈与大文件下载](knowledge/long-running-h5-memory-and-download-optimization.md)
- [结束按钮不是最后一步：移动 H5 录音分片持久化与后台续传](knowledge/durable-h5-audio-chunk-upload.md)
- [把大模型生成限制在事实边界内：Evidence 驱动的内容生成与校验](knowledge/evidence-grounded-resume-generation.md)
- [结构化大模型调用如何可靠落地：Provider、宽松边界与确定性降级](knowledge/reliable-structured-llm-calls.md)
- [页面没有线程锁，为什么仍会发生竞态：用 Generation Fencing 管理浏览器实时生命周期](knowledge/generation-fencing-for-browser-realtime-lifecycles.md)
- [同一句话为什么出现两遍：实时转写与最终转写的时间线对齐](knowledge/reconciling-realtime-and-final-transcripts-by-timeline.md)
- [短会议不是不能生成纪要：如何同时控制幻觉与业务副作用](knowledge/grounding-short-meeting-minutes-and-side-effects.md)

### 日常记录

- [2026-07-16 AIMEETING 项目日报](daily/2026-07-16.md)

## 目录说明

- `inbox/`：尚未完整整理的素材和草稿；
- `knowledge/`：概念、原理和通用方案；
- `problems/`：故障排查、修复与复盘；
- `projects/`：项目设计、技术决策和阶段总结；
- `conversations/`：交流记录与认知修正；
- `daily/`：日常学习和工作记录；
- `archive/`：已经过时但需要保留的内容；
- `figures/`：图片、截图和其他可视化资源。
