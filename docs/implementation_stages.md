# 实施阶段规划

本文档描述在范围确认后的分阶段实现路线。它不是开始写代码的批准。

## Stage 0 - 规划与确认

状态：当前阶段。

交付物：

- `docs/reproduction_plan.md`
- `docs/paper_to_framework_mapping.md`
- `docs/equations_and_metrics.md`
- `docs/assumptions_and_unknowns.md`
- `docs/experiment_matrix.md`
- `docs/implementation_stages.md`

退出标准：

- 负责人确认第一阶段目标范围。
- 负责人确认是否允许 proxy。
- 负责人确认 ML framework 选择。
- 负责人确认是否必须先实现论文 LEO/MEO topology。

## Stage 1 - 指标与 Snapshot 基础

目标：

增加不侵入当前 SimPy routing engine 的基础设施，并可独立测试。

计划变更：

- 增加 `reproduction/metrics/`：
  - topology compression ratio。
  - packet loss/success formulas。
  - expected one-hop latency。
  - SDE。
  - semantic quality proxy。
- 增加 `reproduction/topology/`：
  - simple snapshot schema。
  - 从 NetworkX graph 提取 local topology features。
  - synthetic graph fixtures。
- 增加 seed management utility。
- 为所有公式增加测试。

不深度集成 `SimulationRL.py`。

验收标准：

- 所有公式模块的 unit tests 通过。
- synthetic topology fixture 能产生 `Theta_u`、`Theta_u^ni` 和 `gamma`。
- SDE 测试与手工计算一致。
- 不改变现有仿真输出。

风险：

- 工程风险低。
- 价值高，因为先锁定公式，再触碰仿真行为。

## Stage 2 - STG Adapter 与当前仿真器 Snapshot Export

目标：

把当前仿真器的 graph states 捕获为论文式 snapshots。

计划变更：

- 在 `createGraph()` 输出外侧增加只读 adapter。
- 导出 satellite-only snapshot graph 和可选 gateway access graph。
- 记录：
  - `snapshot_id`。
  - `sim_time`。
  - node layer/capacity/degree/occupation。
  - edge bandwidth/distance。
- 增加 snapshot-only extraction 的小型 run script。

验收标准：

- existing Kepler 或 small constellation run 能输出 snapshot artifacts。
- 固定 seed/config 下 node/edge counts 稳定。
- feature extraction 能处理导出的 snapshots。

风险：

- 中等，因为现有 graph 包含 gateway nodes，需要干净拆分。

## Stage 3 - TSTCN Stage 1

目标：

在训练 Transformer 前，实现 deterministic/controlled TSTCN pipeline。

计划变更：

- textual topology template。
- temporal change features。
- deterministic feature encoder。
- task target config。
- cosine similarity selection。
- relay insertion。
- `gamma` export。

验收标准：

- 在当前 constellation snapshots 上生成 Fig. 6 风格数据。
- controlled fixture 中 `gamma` 随更严格 `lambda` 降低。
- 原始图有连接路径时，semantic topology 保持连通。
- 所有选择都写入 run report。

风险：

- 中高，主要来自 target-vector 的论文缺失细节。

## Stage 4 - Semantic Coding Proxy

目标：

先提供稳定 semantic reliability 接口，不直接实现完整 Transformer coding。

计划变更：

- `SemanticCodingModel` interface。
- conventional coding proxy。
- DL-JSCC proxy。
- semantic coding proxy。
- SNR/BER/semantic dimension 配置。

验收标准：

- 能生成 Fig. 8 风格 proxy curve，且随 SNR 单调提升。
- 可检查 semantic accuracy constraint C3。
- delivery metrics 能一致地请求 semantic success/loss。

风险：

- 科学风险中等，因为 proxy 不能验证 full semantic coding claim。

## Stage 5 - DQNSCR Strategy 集成

目标：

以最小耦合方式把论文式 DQN route policy 接入仿真。

计划变更：

- routing strategy interface。
- DQNSCR state builder。
- semantic topology neighbors 上的 action mask。
- 使用 `Gamma` 和 reliability term 的 reward formula。
- replay buffer 和 checkpoint/log output。
- tiny deterministic training fixture。

集成点：

- 优先通过 `Satellite.receiveBlock()` 调用窄 adapter。
- 避免重写 queue mechanics。

验收标准：

- DQNSCR 能在 synthetic semantic topology 上路由。
- invalid actions 被 mask。
- reward 与 Eq. (28)-(29) 测试一致。
- checkpoints、rewards、losses 被导出。
- 现有 baseline route modes 仍能运行。

风险：

- 高，因为这会接触 route decision timing 和当前 movement/re-link 行为。

## Stage 6 - Baselines

目标：

使用同一套 metrics 和 traffic 增加可比 baselines。

计划变更：

- global STG 上的 OSPF baseline。
- global STG 上的 DQN-IR 近似。
- global STG/local neighbor state 上的 RTHop 近似。
- 每个方法共享 coding-model interface：
  - OSPF with conventional coding。
  - DQN-IR 和 RTHop with DL-JSCC proxy。
  - LISDM with semantic coding proxy。

验收标准：

- 同一 config 能运行所有 mechanisms。
- metrics CSV 中字段可比较。
- loss/coding assumptions 在 report 中明确。

风险：

- 中等；但 exact RTHop/DQN-IR fidelity 仍然是科学未知项。

## Stage 7 - 论文 LEO/MEO Topology

目标：

匹配论文的异构 topology profile。

计划变更：

- LEO/MEO constellation profile。
- 232 satellite node generation。
- orbital layer metadata。
- inter-layer 与 intra-layer ISL policy。
- snapshot validation。

验收标准：

- node count：200 LEO + 32 MEO。
- altitude/inclination/planes/sat-per-plane 匹配 Table II。
- snapshot graph 导出 LEO/MEO layer labels。
- 若 inter-layer link policy 是假设而来，必须文档化。

风险：

- 高，因为当前 constellation model 是单 profile，且论文 contact-table generation 不完整。

## Stage 8 - 论文图表实验复现

目标：

使用 proxy semantic coding 跑优先图表复现实验。

实验：

- Fig. 6 到 Fig. 12。
- ablations。
- multiple seeds。

验收标准：

- 每次 run 都有 config、seed、logs、metrics、plots、report。
- 趋势判断必须诚实分类：
  - reproduced。
  - partially reproduced。
  - not reproduced。
- 只有在数值目标和完整模块都具备时，才能声称 exact reproduction。

风险：

- 计算和时间风险高。

## Stage 9 - 完整 Semantic Coding

目标：

用 Transformer semantic codec 替换 proxy。

计划变更：

- Europarl preprocessing。
- Transformer encoder/decoder。
- AWGN channel。
- quantization。
- training loop。
- semantic quality calculation。

验收标准：

- 在 held-out set 上恢复文本。
- semantic quality curves 来自真实 recovered text。
- Fig. 8 可从 proxy trend 改为 semantic coding reproduction。

风险：

- 科学和工程风险最高。

## 最小第一 PR/Commit 方案

推荐第一 PR 标题：

`docs: add LISDM reproduction design and mapping`

范围：

- 只增加 `docs/` 下文档。
- 不增加 Python 实现。
- 不修改仿真器。

验收标准：

- 文档经过 review 并批准。
- open questions 得到回答。
- 选定下一 PR 范围。

确认后推荐第二 PR 标题：

`reproduction: add metric formulas and synthetic topology sanity checks`

范围：

- 只增加 metric functions 和 tests。
- 增加 synthetic graph fixtures。
- 不接入 SimPy。

验收标准：

- tests 覆盖公式 (4)、(5)、(9)-(14)、(21)、(28)-(29)。
- 不改变当前仿真行为。

