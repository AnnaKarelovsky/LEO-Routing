# LISDM 论文复现规划

本文档只用于规划。在负责人确认范围之前，不开始实现代码。

论文：《Task-Oriented Semantic Delivery in Large-Scale Heterogeneous Satellite Networks: A Local-Topological-Information-Dependable Deep Learning Approach》，IEEE Internet of Things Journal，2025。

## 1. 复现目标

论文提出 LISDM，即面向大规模异构卫星网络的本地信息依赖语义投递机制。本次复现不是从零编写仿真器，而是在当前 MA-DRL 卫星路由仿真框架上新增适配层、路由策略、语义指标和可重复实验流程。

需要复现的主要科学结论：

- 本地拓扑信息可以通过 TSTCN 压缩为面向任务的语义拓扑。
- semantic coding 能在噪声信道下提升单跳可靠性，并降低重传相关时延。
- DQNSCR 使用本地信息在语义拓扑中路由语义编码任务数据，相比 OSPF、DQN-IR、RTHop，在包投递率、时延、吞吐和 SDE 上表现更好。
- SDE 能同时刻画可靠投递的语义数据量，以及投递所消耗的拓扑规模和时间成本。

## 2. 论文拆解

### 核心目标

在 LEO/MEO 异构卫星网络中，从源节点 `ns` 到目的节点 `nd` 进行端到端文件投递。目标是在避免依赖全局拓扑的前提下，通过压缩参与路由的拓扑规模，最大化面向任务的 semantic delivery efficiency。

### 问题定义

物理卫星网络被建模为空间时间图：

- 一系列 snapshot：`Gu = (Vu, Eu)`。
- 节点是卫星。
- 边是该 snapshot 中可用的 ISL。
- snapshot 根据拓扑或接触窗口变化生成。

每个 snapshot 中，论文围绕每颗卫星提取本地拓扑信息：

- 节点 degree。
- 节点 capacity/storage。
- 邻居链路 bandwidth。
- 邻居链路 distance。
- 相邻 snapshot 之间的时间变化特征。

优化目标是 SDE：可靠投递的语义任务数据量，除以投递时间与语义拓扑规模的乘积。

### 系统模型

论文将系统分为两部分：

- 拓扑模型：物理 LSHSN -> STG snapshots -> 本地拓扑信息 -> 通过 TSTCN 生成语义拓扑。
- 投递模型：任务数据 -> semantic coding -> 在语义拓扑上执行 DQNSCR 路由 -> 统计可靠投递指标。

当前仿真器已有事件驱动的 SimPy 执行模型，但论文中的 STG 是 snapshot 抽象。复现时应把当前仿真器中每次图创建或重链路事件视为 snapshot 边界。

### 必须复现的核心组件

- LISDM 总体机制：
  - TSTCN + semantic coding + DQNSCR + SDE 统计。
- TSTCN：
  - 本地拓扑提取。
  - textual topology feature 或可控近似。
  - multihead self-attention 或轻量替代。
  - task target cosine matching。
  - semantic topology 构造。
  - topology compression ratio `gamma`。
  - semantic topology 不连通时的中继节点/链路补全。
- DQNSCR：
  - 在语义拓扑上的 DQN route policy。
  - 与论文公式一致的 state/action/reward/transition/terminal 定义。
  - replay buffer、target network、checkpoint、日志输出。
- SDE：
  - 可靠投递的语义数据量除以投递时间和语义拓扑规模。
- semantic coding：
  - 完整目标是 Transformer semantic encoder/decoder。
  - 第一阶段建议使用可控 semantic accuracy model。
- baselines：
  - OSPF + conventional coding + STG。
  - DQN-IR + DL-JSCC + global STG。
  - RTHop + DL-JSCC + global STG。

## 3. 当前仓库能力判断

当前仓库适合作为复现底座，已经具备：

- `Earth.moveConstellation()` 中的动态卫星位置和拓扑更新逻辑。
- 带有 `slant_range`、`dataRate`、`dataRateOG`、`hop`、`latency` 等边属性的 NetworkX 图构建。
- 基于 SimPy 的事件仿真，包括数据生成、队列、传播时延、传输时延和投递。
- `getShortestPath()` 中的 Dijkstra/shortest-path 路由。
- `SimulationRL.py` 中的 Q-Learning 和 Deep Q-Learning 路由选择。
- 延迟、吞吐、拥塞、reward、epsilon、loss 和路径图等既有统计/绘图逻辑。
- 既有 Q 表和神经网络 artifact 可作为工程参考，但除非通过新脚本重新生成，否则不能作为论文复现证据。

主要缺口：

- 论文需要 LEO/MEO 异构星座。当前 `create_Constellation()` 支持多个单层 LEO/Walker 星座，不能直接匹配论文的 LEO+MEO 设置。
- 当前图接近 snapshot，但没有显式 STG snapshot 归档，也没有 snapshot ID、timestamp、本地拓扑张量和稳定实验序列化。
- 没有 TSTCN 模块。
- 没有 semantic coding。
- 没有 SDE。
- 当前 DDQN 是路由导向，但 state/reward 与 DQNSCR 不一致。
- README 明确指出 `updateSatelliteProcessesRL()` 在星座移动和重链路后有已知风险；这与论文的动态拓扑场景高度相关。

## 4. 关键技术路线

### A. 动态拓扑与 STG 适配

复用当前图创建和移动流程：

- `Earth.__init__()` 创建 `self.LEO`、gateways 和 SimPy 移动进程。
- `Earth.moveConstellation()` 旋转卫星、重建 GSL/ISL、执行 `graph = createGraph(...)`、更新 gateway graph 引用、更新 satellite processes 和 GT paths。
- `createGraph(earth, matching=...)` 生成包含卫星、网关、GSL、ISL 的 NetworkX 图。

复现设计：

- 后续在图创建点外侧增加 `SnapshotRecorder` adapter，不直接塞进核心路由逻辑。
- 将 `initialize()` 和 `moveConstellation()` 产生的每个图记为 `Gu`。
- 存储 snapshot 元数据：
  - `snapshot_id`。
  - `sim_time`。
  - `nodes`。
  - `edges`。
  - 每条边的 `bandwidth`、capacity proxy、distance、latency、hop。
  - 每个节点的 degree、capacity、queue occupation、orbital layer。
- 论文 STG 只保留卫星节点/ISL；仿真路由需要 gateway 时，把 gateway 作为单独 access-layer view。

本地拓扑信息：

- `degree`：satellite-only graph 上的 NetworkX degree。
- `capacity I`：第一阶段映射为 `Satellite.quota` 或配置中的 storage capacity；后续替换为真实 buffer/storage accounting。
- `bandwidth A`：edge `dataRateOG`。
- `link distance Phi`：edge `slant_range`。
- `temporal change omega`：当前 snapshot 与上一个 snapshot 的本地特征差分。

建议未来文件：

- `reproduction/topology/snapshot_recorder.py`
- `reproduction/topology/local_features.py`
- `reproduction/topology/stg_export.py`

### B. TSTCN 拓扑压缩

node-centric local topology 提取：

- 对每颗卫星 `ni`，从 satellite-only `Gu` 读取邻居。
- 构造 `Theta_u_ni = {degree, capacity, A(ni, eta_i), Phi(ni, eta_i)}`。
- 加入来自上一 snapshot 的 temporal change。

textual topology feature：

第一阶段不依赖大语言模型，使用确定性的文本模板和数值 token：

```text
node=<id>; layer=<LEO/MEO>; degree=<bucket>; capacity=<bucket>;
neighbors=<count>; bandwidth_profile=<bucket-list>; distance_profile=<bucket-list>;
delta_degree=<bucket>; delta_bandwidth=<bucket>; delta_distance=<bucket>
```

该模板保留论文“拓扑可文本化表达”的思想，同时保证实验可重复。

语义特征提取：

- 第一阶段：使用确定性或轻量可训练 embedding，例如 TF-IDF/hash-vector + projection，或在依赖允许时使用小型 Transformer encoder。
- 第二阶段：实现更接近论文的 Transformer：word embedding、positional encoding、4 heads、3 layers。

task target vectors：

- 定义论文中的三类任务：
  - fault tolerance：偏好 high degree 和冗余链路。
  - large data：偏好 high capacity 和 high bandwidth。
  - high reliability：偏好 stable、high-bandwidth、low-distance、少 hop 的链路。
- target vectors 必须由配置生成并记录。论文缺失的细节写入 `docs/assumptions_and_unknowns.md`。

semantic topology 构造：

- 计算每个 local topology feature 与 task target 的 cosine similarity。
- 选择 score >= `lambda` 的节点。
- 保留被选节点之间满足 `A(ni,nj) != 0` 的 induced edges。
- 若被选节点不连通，则使用确定性规则插入 relay nodes：
  - 优先取原始 snapshot graph 中 disconnected components 之间的 shortest path。
  - tie-break 按 task-target cosine、bandwidth、node ID。
- 按论文公式计算 `gamma`，扣除跨任务重叠。

建议未来文件：

- `reproduction/tstcn/textual_topology.py`
- `reproduction/tstcn/feature_encoder.py`
- `reproduction/tstcn/compressor.py`
- `reproduction/tstcn/semantic_topology.py`

### C. Semantic Coding

论文使用 Transformer-based semantic encoder/decoder、AWGN channel、semantic dimension `N = 64`、3-layer 4-head self-attention、bit quantization `B = 8`，数据集为 Europarl，最大序列长度 20。

第一阶段不建议直接训练完整 semantic coding 网络，而使用可控 semantic transmission accuracy model：

- 输入：task type、semantic dimension、SNR、BER、file size。
- 输出：semantic transmission accuracy `psi(vartheta(f); b)` 和 recovered semantic quality。
- 保证单调趋势：
  - accuracy 随 SNR 增大。
  - accuracy 随 semantic dimension 增大。
  - 低信道质量下 semantic coding accuracy 优于 conventional bit-level success，匹配 Fig. 8 的定性趋势。

简化方案影响：

- 可复现 routing 和 SDE 对 semantic reliability 的趋势。
- 不能证明 Transformer semantic coding 本身的质量。
- Fig. 8 在完整 codec 前只能标注为 proxy trend。

后续替换路径：

- 保持统一 `SemanticCodingModel` interface。
- 后续用完整 semantic coding 模块替换 proxy，不影响 DQNSCR 和 metrics。

建议未来文件：

- `reproduction/semantic_coding/interface.py`
- `reproduction/semantic_coding/accuracy_model.py`
- `reproduction/semantic_coding/transformer_codec.py`

### D. DQNSCR 路由

复用当前 RL 基础设施，但隔离论文专用 route policy：

- 当前 `SimulationRL.py` 已有 `DDQNAgent`、`makeDeepAction()`、`getDeepLinkedSats()`、replay buffer、target network、training、reward logging、loss 和 checkpoint。
- DQNSCR 不应通过大量改写现有函数硬塞进去，而应作为 strategy adapter，由 satellite receive/routing 的小集成点调用。

state：

- 当前节点本地拓扑特征。
- semantic topology 内的 neighbor features。
- 到候选邻居的 link bandwidth/distance。
- 当前 queue occupancy 或 queue delay proxy。
- destination 与 semantic/task target 的关系。
- task type 和 semantic coding 参数。

action：

- 从当前 task-specific semantic topology 的相邻节点中选择一个 next hop。
- 对 unavailable neighbors 做 action mask。

reward：

- 论文中的 `1/Gamma` immediate term。
- semantic packet success/loss 对应的 reliability bonus/penalty。
- 可选使用当前 topology scale `rho` 做 SDE-shaped normalization。
- terminal success 给正向可靠性项；terminal loss 给负向项。

transition：

- 当一个 block/file 从 satellite `ni` 移动到 `nj`，或被 drop/lost 时产生 transition。
- 存储 `(state, action, reward, next_state, done)` 到 replay buffer。

terminal：

- 成功投递到 destination access satellite/gateway。
- 因 packet loss 或无 feasible semantic path 丢失。
- TTL/hop budget 超限。
- snapshot 变化导致当前 queued packet 不再有有效 semantic path。该情况需要单独统计。

降低耦合：

- 定义 `RoutingStrategy` 和 `SemanticRoutingContext` adapters。
- 保留现有 SimPy queue 和 propagation mechanics。
- strategy 只返回 next hop 和 route metadata。
- training logs/checkpoints 由 strategy 管理，不放进 `Earth` 或 `Satellite`。

建议未来文件：

- `reproduction/routing/strategy.py`
- `reproduction/routing/dqnscr.py`
- `reproduction/routing/baselines.py`
- `reproduction/training/replay_buffer.py`
- `reproduction/training/checkpoints.py`

### E. Baselines

OSPF：

- 已可由 `getShortestPath()` 近似实现。
- 需要纸面一致的 STG/global graph wrapper 和 conventional coding model。
- 在全局 snapshot graph 上执行 Dijkstra。

DQN-IR：

- 可复用 DDQN 框架结构，但当前 state/reward 不是 DQN-IR。
- 实现为不使用 TSTCN 的 global-STG DQN route policy。
- 第一阶段使用同一 semantic/coding interface 里的 DL-JSCC proxy。

RTHop：

- 当前 online per-satellite DDQN 类似 decentralized hop-by-hop policy，但不是严格的论文 RTHop。
- 第一阶段可实现最小可比版本：
  - local neighbor state。
  - 不使用 semantic topology compression。
  - 根据 queue 和 destination proximity 的 learned 或 heuristic score 选择 next hop。
  - 使用和 DQN-IR 相同的 DL-JSCC proxy。

所有 baseline 必须共享：

- 同一 topology snapshots。
- 同一 traffic generation。
- 同一 seed。
- 同一 semantic/conventional coding interface。
- 同一 metric collector。

## 5. 优先复现实验

建议按以下顺序复现：

1. Fig. 6：topology compression ratio vs similarity threshold `lambda`。
2. Fig. 7：semantic topology 与 global topology 下 DQNSCR 收敛对比。
3. Fig. 8：semantic quality vs SNR。
4. Fig. 9：packet delivery ratio vs transmitted files。
5. Fig. 10：average end-to-end delivery latency vs transmitted files。
6. Fig. 11：throughput vs transmitted files。
7. Fig. 12：SDE vs transmitted files。

前两个最适合作为首轮实现目标，因为它们先验证 topology extraction/compression 和 routing-loop 集成，再进入完整 semantic coding。

## 6. 复现难点排序

1. 完整 semantic coding network：难度最高，因为论文缺少很多训练、数据预处理和信道细节。
2. DQNSCR 的 reward 和 state 精确对齐：当前 DDQN 结构接近，但语义和公式不一致。
3. LEO/MEO 异构星座：当前代码主要是单层星座逻辑。
4. 动态 topology under movement：README 已记录 RL process 在星座移动后的风险。
5. TSTCN textual topology 和 task-target vectors：论文描述不够细。
6. SDE 和 metric collection：公式清晰，但需要精确绑定事件和拓扑规模。
7. OSPF baseline：当前 NetworkX Dijkstra 已基本具备。

## 7. 最小第一阶段 PR

第一个实现 PR 不应深度改 SimPy core。建议先增加可测试的指标和拓扑基础设施：

- 增加 `reproduction/` package 的接口/基础结构：
  - snapshot data structures。
  - metric definitions。
  - config loading。
  - seed utility。
- 增加公式层测试：
  - `gamma`。
  - SDE。
  - packet success/loss formulas。
  - semantic quality proxy monotonicity。
- 增加实验目录模板：
  - `experiments/lisdm/configs/`。
  - `experiments/lisdm/runs/`。
  - `experiments/lisdm/reports/`。
- 不接入 `SimulationRL.py`；最多在后续 PR 增加只读 exporter。

验收标准：

- 不改变现有仿真输出。
- 指标公式测试通过。
- 小规模 synthetic topology 能产生 local features、semantic topology 和 `gamma`。
- `assumptions_and_unknowns.md` 记录所有未解决的论文细节。

## 8. 需要确认的问题

1. 第一阶段是否接受定性趋势复现，还是必须接近论文曲线数值？
2. Stage 1 是否允许使用 semantic coding proxy，完整 Transformer semantic coding 后置？
3. 是先补论文 LEO/MEO 拓扑，还是先在当前 Kepler/Starlink-style 星座上验证 TSTCN/DQNSCR proof-of-concept？
4. 新增 ML 模块使用当前仓库的 TensorFlow/Keras，还是按论文使用 PyTorch？
5. gateway 节点是否视为论文 STG 外的 access layer，只把卫星放进 `Gu`？
6. 是否允许创建干净的 `reproduction/` package，并尽量不改 `Simulation.py` / `SimulationRL.py`？

