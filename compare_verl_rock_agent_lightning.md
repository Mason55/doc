# VERL vs ROCK/ROLL vs Agent Lightning 对比

> 目标：对比三者的定位、闭环结构、接入方式、扩展点与适用场景，给出工程视角的选型依据。

## 1. 定位与目标

| 项目 | 核心定位 | 主要目标 | 典型使用者 |
|---|---|---|---|
| VERL | RL 训练框架 + tool agent loop | 强化学习训练、工具调用与多轮 rollout | 研究/训练工程团队 |
| ROCK/ROLL | 基础设施 + 大规模训练管线 | ROCK 提供环境隔离与 agent 执行；ROLL 提供 agentic 训练闭环与分布式调度 | 平台/基础设施 + 训练工程团队 |
| Agent Lightning | 训练闭环与追踪平台 | “零改动”接入 agent，形成训练闭环 | 训练/算法平台团队 |

## 2. 闭环结构对比

### VERL（训练闭环 + 工具调用）
- 入口：`ToolAgentLoop` 在 rollout 中解析 tool call 并执行工具。
- 核心链路：模型生成 → 解析 tool call → 执行工具 → tool response 注入 → 继续生成。
- Reward 回流：工具奖励写入 `AgentLoopOutput.extra_fields`，供训练统计。

### ROCK/ROLL（环境隔离 + 训练管线）
- ROCK 入口：`Sandbox` + `RockAgent`。
- ROCK 链路：启动 sandbox → 部署工作目录 → 初始化 runtime/model service → agent 执行命令。
- ROLL 入口：`AgenticPipeline` / `AgenticRolloutPipeline`。
- ROLL 链路：rollout 调度 → env manager 执行 → reward/advantage → train step。
- 重点：ROCK 负责执行与隔离，ROLL 负责训练与调度。

### Agent Lightning（观测/训练闭环）
- 入口：`Trainer` 连接 `Runner/Tracer/Store/Algorithm`。
- 核心链路：Runner 执行 → Tracer 产出 span → Store 汇聚 → Algorithm 训练更新 → Trainer 协调。
- 重点：统一的“可观测轨迹 → 可训练资源”闭环。

## 3. Agent 接入方式对比（重点）

| 维度 | VERL | ROCK/ROLL | Agent Lightning |
|---|---|---|---|
| 接入入口 | `ToolAgentLoop` 读取 `tool_config_path` | ROCK: `Sandbox` + `RockAgent.install/run`；ROLL: `AgenticPipeline`/`EnvManager` | `Trainer` + `Runner/LitAgent` |
| 接入形式 | 模型输出解析为 tool call，工具为一等公民 | ROCK 以“命令/脚本”方式运行 agent；ROLL 以“环境 + rollout 调度”接入 agentic 任务 | 以 `LitAgent.rollout()` 接入，Runner 驱动 |
| 代码改动 | 需要适配 tool schema 与 parser | ROCK 需定义安装/运行命令；ROLL 需定义 env_manager 与工具/奖励逻辑 | 最少改动：包装 agent 为 `LitAgent` |
| 观测接入 | rollout_trace + metrics | ROCK 以日志为主；ROLL 以 metrics + 轨迹 dump | Tracer/Span/Store 统一观测 |
| 多轮工具 | 内建（ToolParser + BaseTool） | ROLL 内建 MCP/Code tools；ROCK 依赖 agent 自身 | 依赖 agent 框架自身与 tracer 采集 |

## 4. Agent 组织方式对比（重点）

| 维度 | VERL | ROCK/ROLL | Agent Lightning |
|---|---|---|---|
| Agent 组织结构 | `AgentLoop` 管理多轮状态机 | ROCK: `RockAgent`/`SweAgent` 执行体；ROLL: `EnvManager` + `EnvironmentWorker` | `LitAgent` 只负责 rollout，Runner 管调度 |
| 生命周期 | 生成/工具执行/交互的状态机循环 | ROCK: 安装 → 运行；ROLL: rollout → reward → train | init → rollout → trace → store |
| 资源依赖 | tool config + tokenizer/processor | ROCK: Deploy + RuntimeEnv + ModelService；ROLL: Cluster/Env/Tools | Store 的资源快照/Adapter 注入 |
| 适配成本 | 中等（需编排 tool schema/format） | ROCK 中等；ROLL 中等偏高（需要 env/奖励/调度配置） | 低（包装 agent 接口即可） |
| 适配弹性 | 强：支持多模态/多工具 | 强：环境隔离 + 分布式训练 | 强：与多种 agent 框架兼容 |

## 5. 扩展点对比

### VERL
- 新工具：实现 `BaseTool` 并配置 tool schema。
- 新解析器：扩展 `ToolParser`。
- 新 trainer：`trainer/` 下扩展 PPO/GRPO 等。

### ROCK/ROLL
- ROCK 新 agent：继承 `RockAgent` 实现 install/run。
- ROCK 新运行时：继承 `RuntimeEnv`。
- ROCK 新沙箱/部署策略：扩展 sandbox/Deploy 侧能力。
- ROLL 新环境：实现 `BaseEnvManager` 与 env 配置。
- ROLL 新工具：扩展 `roll/pipeline/agentic/tools`（MCP/Code）。
- ROLL 新 pipeline：扩展 `roll/pipeline/agentic` 或新增策略/奖励 worker。

### Agent Lightning
- 新算法：实现 `Algorithm.run()`。
- 新 runner：继承 `Runner` 定义执行逻辑。
- 新 tracer：实现 `Tracer` 接入 OTEL/AgentOps。
- Store：替换存储（memory/mongo 等）。

## 6. 适用场景建议

- **VERL**：需要把“工具调用 + 多轮对话 + RL 训练”打通的训练系统。
- **ROCK/ROLL**：ROCK 负责稳定可复用的 sandbox 环境管理；ROLL 负责 agentic 训练闭环与分布式调度。
- **Agent Lightning**：希望低成本接入已有 agent 并形成训练闭环、统一观测与奖励采集。

## 7. 组合方式建议（工程实践）

- VERL + ROCK/ROLL：
  - VERL 做工具调用与多轮 rollout，ROCK 负责环境执行隔离，ROLL 负责大规模训练调度。
- Agent Lightning + ROCK/ROLL：
  - ROCK 管环境，ROLL 管训练闭环，Agent Lightning 负责统一观测与评估。
- VERL + Agent Lightning：
  - VERL 负责策略更新，Agent Lightning 负责跨 agent 的统一观测与评估。

## 8. 结论（工程选型）

- **训练闭环优先** → VERL / Agent Lightning。
- **环境稳定与隔离优先** → ROCK。
- **大规模 agentic 训练调度优先** → ROLL。
- **已有 agent 框架快速接入** → Agent Lightning。
- **多轮工具调用训练** → VERL。
