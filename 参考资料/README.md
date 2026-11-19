# 参考资料

本目录只放公开来源的链接与摘要。第三方文章全文和 PDF 因版权原因不纳入本仓库。

## Super-agent 设计

- 官方文章（DeepSeek Harness 团队）：https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg
- 官方原型仓库：https://github.com/deepseek-ai/dsh-superagent-prototype
- 第三方解读（外部作者）：https://mp.weixin.qq.com/s/K_cafXoicZ8Y6dN6mvmAew

摘要（自写）：

- Router + Worker 双层：Router 常驻、负责跨任务分派和对外交互；Worker 隔离上下文、执行具体任务。
- 事件驱动：两级过滤（确定性 Fast Path + 模型 Slow Path）、订阅直通、上下文 Inline & Spill、链式召回、投机执行。
- 运行：异步 ReAct、计算/状态/环境分离、外部 Watchdog。
- 实验：合成消息流，主要衡量调度与协作，不代表真实代码质量。

## Octop 团队模式

- 项目：https://github.com/TencentCloud/Octop
- 文章：https://mp.weixin.qq.com/s/5r_EhRzdgRgzg8kWmZDyRQ

摘要（自写）：

- 主持人 + 专家 Agent：主持人梳理任务、分派子任务，专家在协作房间内以自身身份回复。
- 与 Super-agent 的 Router 收敛不同，Octop 更强调过程透明和成员可见。

## 与数字人项目的对照

- 产品层更像 Octop：群聊里的数字分身是带身份和显示名的成员，可以直接回包。
- 执行层保留收敛：内部子 Agent 向父执行 Agent 提交，由父级验收。
- 详细对照见 [数字分身系统对照卡](数字分身系统对照卡_2026-11-16.md)。
