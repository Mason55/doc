# verl-recipe 仓库分析

> 位置：`/home/lmy/RL/agentic/verl-recipe`
> 目标：概览仓库结构、主要 recipe、关键脚本与配置入口，便于快速定位与复现。

## 1. 仓库定位与定位结论

- `verl-recipe` 是围绕 `verl` 框架的社区 recipe 集合，包含大量 RL/对齐相关算法的训练脚本、配置与示例实现。
- 组织结构以 **算法/研究方向** 为目录维度（如 `dapo/`, `flowrl/`, `gkd/`, `spo/` 等），每个 recipe 目录通常配套：
  - README（复现指南与参数说明）
  - 训练脚本（`run_*.sh` / `train*.sh`）
  - 配置文件（`config/*.yaml`）
  - 关键实现（`*.py`）

## 2. 顶层结构概览

- `README.md`：仓库入口，说明 `verl-recipe` 可作为 `verl` 子模块使用，并列出部分 recipes。
- `pyproject.toml`：项目元信息与 ruff/mypy 规则（用于 lint/typecheck）。
- 主要 recipe 目录：
  - 强化学习/对齐：`dapo/`, `fapo/`, `gvpo/`, `spo/`, `sppo/`, `spin/`, `prime/`, `rep_exp/`
  - 多模态/工具调用：`retool/`, `deepeyes/`, `langgraph_agent/`
  - 任务类示例：`char_count/`, `open_math_reasoning/`
  - 系统/工程支持：`fault_recover/`, `genrm_remote/`, `r1_ascend/`, `r1/`
  - 其他：`flowrl/`, `entropy/`, `specRL/`

## 3. 主要 recipe 概览与入口

### 3.1 DAPO（Decoupled Clip and Dynamic Sampling Policy Optimization）
- 目录：`dapo/`
- 入口脚本：`run_dapo_qwen2.5_32b.sh`, `run_dapo_wo_ds_qwen2.5_32b.sh`, `run_dapo_early_qwen2.5_32b.sh`
- 配置：`config/dapo_trainer.yaml`, `config/dapo_megatron_trainer.yaml`
- 特点：分离 clip、动态采样（group filtering）、token-level loss 聚合、overlong reward shaping 等。

### 3.2 FAPO（Flawed-Aware Policy Optimization）
- 目录：`fapo/`
- 入口脚本：`run_fapo_genrm_train.sh`, `run_fapo_{7b,32b}.sh`, `run_fapo_{7b,32b}_remote.sh`
- 依赖：外部/内置 GRM 服务（见 `genrm_remote/`）

### 3.3 FlowRL（Flow Balance 方向）
- 目录：`flowrl/`
- 入口脚本：`run_flowrl_qwen2.5_7b.sh`, `prepare/prepare_data.sh`, `prepare/prepare_model.sh`
- 特点：匹配 reward 分布的 flow-based RL 方案，并提供自研实现指导文档 `FLOWRL_SIMPLE_GUIDE.md`。

### 3.4 GVPO（Group Variance Policy Optimization）
- 目录：`gvpo/`
- 入口脚本：`run_qwen2-7b_math_gvpo.sh`
- 关键实现：`gvpo_core_algos.py`, `gvpo_ray_trainer.py`, `gvpo_dp_actor.py`

### 3.5 SPO / SPPO / SPIN（偏好/自博弈类）
- `spo/`：单流策略优化，包含离线 value 估计流水线与训练脚本 `train.sh`。
- `sppo/`：Self-Play Preference Optimization，入口 `run_qwen2.5-7b_rm.sh`。
- `spin/`：Self-Play Fine-Tuning（在线 DPO 变体），入口 `run_spin.sh`。

### 3.6 Retool（工具调用与 RL）
- 目录：`retool/`
- 入口脚本：`run_qwen2-32b_sft.sh`, `run_qwen2-32b_ppo.sh`, `run_qwen2-32b_dapo.sh`
- 核心流程：SFT 冷启动 + RL 中工具调用策略优化。

### 3.7 CollabLLM（多轮协作训练）
- 目录：`collabllm/`
- 入口脚本：`train_sft_collabllm.sh`, `train_rl_collabllm.sh`
- 指标：`metrics/` 中提供多轮交互度量（accuracy / interactivity / token_amount / bleu_score）。

### 3.8 GKD（异步 On-Policy 蒸馏）
- 目录：`gkd/megatron/`
- 入口脚本：`run_moonlight_dsv3_training.sh`
- 关键点：teacher 服务（`teacher/`）、异步调度（one_step_off / two_step_off）与 KL loss 注入。

### 3.9 Fault Recover（训练容错与 rollouts）
- 目录：`fault_recover/`
- 入口：`run_qwen2_5_0.5b_megatron.sh`
- 功能：训练阶段与 rollout 阶段的自动恢复与 token 保存。

### 3.10 Open Math Reasoning（SFT 样例）
- 目录：`open_math_reasoning/`
- 入口脚本：`run_sft_qwen3_8b.sh`, `run_generation.sh`, `run_eval.sh`

### 3.11 Char Count（极简 RLVR 教学样例）
- 目录：`char_count/`
- 入口脚本：`create_dataset.py`, `train_sft.sh`, `train_grpo.sh`

### 3.12 DeepEyes（多模态工具调用）
- 目录：`deepeyes/`
- 入口脚本：`run_deepeyes_grpo.sh`
- 注意点：图表/低分辨率数据的 OOM 风险与预处理策略。

### 3.13 R1 / R1 Ascend（DeepSeek-R1 方向）
- 目录：`r1/`, `r1_ascend/`
- 内容：评估结果记录、NPU 上训练适配与补丁说明（`# NPU-ADAPTATION`）。

## 4. 关键配置与脚本约定

- 多数训练脚本遵循 `bash recipe/<name>/run_*.sh` 形式。
- 典型配置位于 `recipe/<name>/config/*.yaml`。
- 常见流程：
  1) 数据准备（`prepare_*.py/sh`）
  2) 训练（`run_*.sh` / `train*.sh`）
  3) 合并/评估（`merge` + `eval`）

## 5. 工程与依赖提示

- 依赖 `verl` 框架：不同 recipe 有不同版本/commit 要求（见各自 README）。
- GPU/集群：大量脚本默认 Ray/多机/多卡环境，部分需要外部服务（如 GenRM、teacher server）。
- 统一风格：多使用 shell 脚本驱动 + YAML 配置 + Python 核心实现。

## 6. 建议的使用路径（最短路径）

1. 先读 `README.md` 和目标 recipe 的 README。
2. 明确依赖版本与数据路径。
3. 根据脚本改路径参数（模型、数据、输出目录）。
4. 先用小模型/少步数验证流程，再放大规模。

---

如需更深入的代码级别分析（模块调用链、关键类/函数职责、训练 loop 细节等），请指定具体 recipe 或目标模块，我可以进一步拆解。