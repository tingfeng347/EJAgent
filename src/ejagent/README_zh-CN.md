# EJAgent 模块指南

[English](README.md) · [项目概览](../../README_zh-CN.md)

EJAgent 是 agent harness，负责准备任务、编排执行与验收、组装模型上下文，以及提交会话状态。
接入 EJAgent 的应用负责环境、凭据、工具和验收能力。

## 模块与配置入口

| 模块 | 配置与应用接入责任 |
| --- | --- |
| [Harness](harness/README_zh-CN.md) | 组件装配、完成策略、Run 限制、控制与生命周期 |
| [评估](evaluation/README_zh-CN.md) | 证据源、验证函数注册、语义 Judge、预算 |
| [规划](planning/README_zh-CN.md) | 能力目录、任务生成、计划更新、输出纠错 |
| [Provider](providers/README_zh-CN.md) | 模型连接、环境变量、请求选项 |
| [工具](tools/README_zh-CN.md) | 工具 schema 与函数、MCP、执行控制 |
| [上下文](context/README_zh-CN.md) | 投影、压缩、自定义 compactor、pipeline 组合 |
| [Skills](skills/README_zh-CN.md) | 本地文件、发现、显式指令选择 |
| [存储](storage/README_zh-CN.md) | 持久化、锁超时、自定义存储与审计读取 |
| [协议](contracts/README_zh-CN.md) | 扩展接口、不可变输入输出类型 |
| [内核](kernel/README_zh-CN.md) | 直接执行接入、检查点边界 |
| [内部轨迹](_trajectory/README_zh-CN.md) | 分析、上下文投影、内部调参边界 |

## 最小应用配置

`AgentHarness` 必须接收 `agent_id`、`ModelPort` 和 `ToolExecutor`。
可直接使用内置 Provider 和 `FunctionToolExecutor()`；空执行器可用于纯文本流程。
持久化、规划、评估、Skills 和压缩均需显式接入。

启用任务验收时，先注册证据源与验证函数，再通过 `EvaluationPlan` 引用注册名称。
动态规划还需要 `VerificationCapability` 能力目录。预先配置的是应用能力；证据值在执行中采集，
面向具体任务的计划可以由 Planner 生成。用户 query 不会自动生成新的证据源或验证函数实现。

## 配置边界

各模块分别说明构造默认值、作用范围、前置条件和失败行为。Actor Run 限制、Planner/Judge
预算与完成验收重试是独立控制。`RunStatus` 等结果枚举描述运行结果，不是功能开关。
内部常量不属于公共配置项。

控制台日志通过 [logger.py](logger.py) 的
`setup_logger(name="ejagent", level="INFO", fmt=...)` 配置，默认格式为
`%(asctime)s | %(levelname)-8s | %(name)s | %(message)s`。
已有 handler 会被保留，因此后续调用不会替换其格式。

完整接入示例见[使用指南](../../docs/usage-guide.md)，JSON 校验见
[结构化输出指南](../../docs/structured-output.md)。
