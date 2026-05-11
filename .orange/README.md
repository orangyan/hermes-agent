# Hermes Agent 项目分析文档

本目录包含 Hermes Agent 项目的详细分析文档。

## 文档目录

| 文件 | 内容 |
|------|------|
| [01-产品介绍.md](./01-产品介绍.md) | 项目背景、核心定位、目标用户 |
| [02-功能结构.md](./02-功能结构.md) | 40+ 工具、多平台网关、自学习闭环 |
| [03-技术架构.md](./03-技术架构.md) | 技术栈、整体架构、20+ Provider 支持 |
| [04-代码结构.md](./04-代码结构.md) | 目录组织、核心文件、依赖关系 |
| [05-数据流.md](./05-数据流.md) | 完整请求链路、工具执行流、记忆更新流 |
| [06-Agent设计.md](./06-Agent设计.md) | 循环设计、工具系统、记忆管理、技能系统 |
| [07-部署运行.md](./07-部署运行.md) | 安装配置、Docker、网关配置 |
| [08-记忆架构.md](./08-记忆架构.md) | 三项目记忆形态对比：Facts JSON vs MEMORY.md vs Dream 两阶段 |
| [09-多轮会话管理.md](./09-多轮会话管理.md) | LangGraph Checkpointer vs SQLite WAL vs JSON 文件会话 |
| [10-Lead-Sub-Agent.md](./10-Lead-Sub-Agent.md) | task 工具 / delegate_task / spawn 三种子 Agent 派生方式 |
| [11-AgentLoop.md](./11-AgentLoop.md) | 中间件链 vs 手写 while 循环 vs 异步事件总线 |
| [12-Thinking模式.md](./12-Thinking模式.md) | Extended/Adaptive Thinking 在三项目中的实现差异 |
| [13-SKILL管理.md](./13-SKILL管理.md) | SKILL.md 格式、加载策略、进化机制横向对比 |
| [14-如何评测.md](./14-如何评测.md) | Terminal-Bench 2 / 组件测试 / 基础测试三级评测体系 |

## 项目一句话

**Hermes Agent** 是 Nous Research 开源的自我提升型 AI Agent，以 SQLite+FTS5 为核心存储，Skills（程序性记忆）+ Memory（事实记忆）+ Session Search（历史召回）三层递进，支持 20+ 平台接入和 20+ LLM Provider。
