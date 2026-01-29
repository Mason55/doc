# 在 VERL 中接入一个 Agent（参考与开发清单）

> 目标：说明在 VERL 中接入 agent 的参考路径、最小开发工作与可用示例。

## 1. 可参考的现有示例

### 1.1 Multi-turn Tool Agent 示例（GSM8K）
**示例路径**
- 数据预处理：`verl/examples/data_preprocess/gsm8k_tool_agent_loop.py`
- 工具配置：`verl/examples/sglang_multiturn/config/tool_config/gsm8k_tool_config.yaml`
- 工具实现：`verl/verl/tools/gsm8k_tool.py`
- 多轮 rollout 示例说明：`verl/examples/sglang_multiturn/README.md`

**示例说明**
- 通过 `tool_config_path` 注入工具 schema。
- 通过数据的 `extra_info.tools_kwargs` 注入工具实例参数（见 `gsm8k_tool_agent_loop.py` 第 91-104 行）。
- ToolAgentLoop 解析模型输出的 tool_call 并执行工具（参见 `verl/verl/experimental/agent_loop/tool_agent_loop.py`）。

## 2. 接入 Agent 的核心路径

VERL 的“agent 接入”更偏 **多轮工具调用 + 交互式 rollout**，核心入口在：
- `verl/verl/experimental/agent_loop/tool_agent_loop.py`
- `verl/verl/tools/base_tool.py`
- `verl/verl/tools/utils/tool_registry.py`

### 2.1 工具（Tool）是最小接入单元
- 实现 `BaseTool` 子类：定义 `create/execute/calc_reward/release`。
- 在 YAML 中配置 `tool_schema` 与 `class_name`。

### 2.2 数据与交互输入
- 数据集样本需要包含 `prompt`（消息列表），可选 `extra_info.tools_kwargs` 与 `interaction_kwargs`。
- 参考：`verl/examples/data_preprocess/gsm8k_tool_agent_loop.py`。

### 2.3 配置入口（tool_config_path）
- 配置示例：`verl/examples/sglang_multiturn/config/tool_config/gsm8k_tool_config.yaml`。
- 在训练/rollout config 中指定 `actor_rollout_ref.rollout.multi_turn.tool_config_path`。

## 3. 最小开发清单（Checklist）

**必须做**
1) 新增 Tool：继承 `BaseTool`，定义 schema 与执行逻辑。
2) 配置 Tool：新增 tool_config YAML（绑定 class_name + schema）。
3) 数据准备：prompt 中明确 tool 使用指令；`tools_kwargs` 填充实例参数。
4) 训练/rollout 配置：设置 `tool_config_path`，使用 `ToolAgentLoop`。

**可选增强**
- 交互式环境：实现 `BaseInteraction` 并在配置中启用。
- 多模态：在 ToolResponse 中返回图像/视频并确保 processor 可处理。
- 奖励：在 tool 中返回 step reward，并在 `calc_reward` 中评估结果。

## 4. 接入步骤建议（工程流程）

1) **定义 Tool**
   - 参考 `verl/verl/tools/gsm8k_tool.py`。
   - 关注 `execute()` 返回 `ToolResponse` + `tool_reward`。

2) **定义 Tool Schema**
   - 参考 `verl/examples/sglang_multiturn/config/tool_config/gsm8k_tool_config.yaml`。

3) **准备数据与提示词**
   - 在 `prompt` 中明确 tool usage 规则。
   - 配置 `extra_info.tools_kwargs`。

4) **配置 Rollout**
   - 设置 `tool_config_path`。
   - 指定 multi-turn 格式与 parser 类型。

## 5. 适配边界与注意事项

- VERL 更强调“工具调用”范式的 agent；如果你的 agent 是独立可执行实体（类似 ROCK/ROLL），需要额外封装为 Tool 或 Interaction。
- Tool schema 与解析格式需要与模型输出一致（`ToolParser`）。
- 多轮环境下，建议先在 `examples/sglang_multiturn` 跑通流程，再接入自定义 agent。

