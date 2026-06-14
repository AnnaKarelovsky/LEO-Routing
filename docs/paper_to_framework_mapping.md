# 论文到当前框架的映射

本文档用于把论文概念映射到当前仓库，并给出后续推荐文件路径。本文档只做规划，不表示可以开始实现。

## 当前框架流程参考

- `Simulation.py`
  - 非 RL 仿真入口是 `main()`。
  - 读取 `input.csv`，按网关数量循环，调用 `initialize()`，运行 `simpy.Environment`。
  - 支持 `dataRate`、`slant_range`、`hop`、`latency` 等 shortest path metric。
- `SimulationRL.py`
  - RL 仿真入口是 `RunSimulation(GTs, inputPath, outputPath, populationData, radioKM)`。
  - 读取 `inputRL.csv`，调用 `initialize()`，运行 SimPy，并保存 latency、throughput、reward、loss 等输出。
  - 包含 `QLearning`、`DDQNAgent`、DQN state helpers、reward helpers 和绘图/报告辅助函数。
- `Earth`
  - 读取 population map 和 gateway CSV。
  - 创建星座。
  - 启动数据块生成和星座移动进程。
- `createGraph()`
  - 根据 satellites、gateways、GSL、ISL 生成 NetworkX 图。
  - 边属性包含 `slant_range`、`dataRate`、`dataRateOG`、`hop`、`latency`。
- `moveConstellation()`
  - 周期性旋转卫星、重建 GSL/ISL、重建 graph、更新 path 和发送进程。
- `Gateway` 与 `Satellite`
  - 持有 SimPy send buffer 和 receive logic。
  - 非 RL 模式在 block 创建时决定路径；RL 模式在 satellite receive 时决定下一跳。
- `DataBlock` / `BlocksForPickle`
  - 存储 path、checkpoints、queue latency、transmission latency、propagation latency、total latency。

## 映射表

| 论文概念 | 论文来源 | 当前框架对应模块 | 是否已有 | 需要新增或修改 | 推荐路径 | 风险 | 验收方式 |
|---|---|---|---|---|---|---|---|
| LEO/MEO 异构 LSHSN | Sec. III，Sec. VI Table II | `SimulationRL.py:create_Constellation()` 支持单层 LEO 星座 | 部分 | 增加 1600/8000 km、10/4 planes、20/8 sats per plane 的异构星座 adapter | 先写 `reproduction/topology/heterogeneous_constellation.md`，后续写代码 adapter | 高 | snapshot 中有 232 颗卫星且 layer label 正确 |
| STG snapshots `Gu=(Vu,Eu)` | Definition 1 | `createGraph()` 和 `moveConstellation()` 产生图状态 | 部分 | 增加显式 snapshot recorder、satellite-only STG view、timestamp/snapshot ID 序列化 | `reproduction/topology/snapshot_recorder.py` | 高 | 每次 graph rebuild 都产生确定性 snapshot artifact |
| 全局拓扑信息 `Theta_u` | Eq. (1) | NetworkX graph + satellite fields | 部分 | 提取 degree、capacity、bandwidth matrix、distance matrix | `reproduction/topology/local_features.py` | 中 | synthetic graph 和真实 graph 都能生成预期矩阵 |
| 本地拓扑信息 `Theta_u^ni` | Eq. (2)、Eq. (15) | `getDeepLinkedSats()`、`getQueues()`、edge attrs | 部分 | 构造 node-centric local feature object，包含邻居边数组和 temporal diff | `reproduction/topology/local_features.py` | 中 | 每个节点 feature 含 degree/capacity/bandwidth/distance |
| temporal change `omega_delta` | Eq. (3)、Eq. (16) | `moveConstellation()` 暴露 graph 更新 | 部分 | 比较连续 snapshot 的 local features | `reproduction/topology/snapshot_diff.py` | 中 | 两个已知 snapshot fixture 的 delta 正确 |
| textual local topology | Sec. IV-A，Fig. 3 | 无 | 否 | 确定性模板，保留数值和 bucket 化语义描述 | `reproduction/tstcn/textual_topology.py` | 中 | 相同输入/seed 生成 byte-stable text |
| topology semantic dictionary `DT` 和 knowledge base `ST` | Definition 3 | 无 | 否 | 按 snapshot/node 持久化 text tokens 和 embeddings | `experiments/lisdm/runs/<run_id>/artifacts/topology_semantics/` | 中 | artifact 包含 text、vector、node ID、snapshot ID |
| TSTCN feature encoder | Eq. (17)-(20)，Fig. 4 | 仓库有 TensorFlow/Keras DQN，但没有 topology encoder | 否 | Stage 1 使用确定性 encoder；Stage 2 实现 4-head 3-layer Transformer | `reproduction/tstcn/feature_encoder.py` | 高 | embedding 可重复；完整版本有训练日志 |
| task requirement target vectors | Sec. IV-B | 无 | 否 | 配置 fault tolerance、large data、high reliability 的 target vectors | `experiments/lisdm/configs/task_profiles.yaml` | 高 | target config 被记录并可重复 |
| cosine similarity selection | Eq. (21)，Algorithm 1 | 无 | 否 | 计算每个 node/task/snapshot 的 score，并按 `lambda` threshold 选择 | `reproduction/tstcn/compressor.py` | 中 | 可生成 Fig. 6 所需 score table |
| relay insertion | Eq. (22)-(23)，Algorithm 1 | NetworkX shortest path 已有 | 部分 | 基于原始 snapshot graph 和 cosine tie-break 的确定性 relay selection | `reproduction/tstcn/semantic_topology.py` | 高 | 物理路径存在时 semantic graph 连通 |
| TSTCN 集合 `G_hat` | Eq. (24) | 无 | 否 | 存储每个 snapshot、每个 task 的 semantic subgraph | `experiments/lisdm/runs/<run_id>/artifacts/semantic_topologies/` | 中 | 每类任务都有 graph artifact |
| compression ratio `gamma` | Eq. (4)，Algorithm 1 | 无 | 否 | 按选中 node/edge 数和跨任务 overlap 实现 metric | `reproduction/metrics/topology.py` | 低 | hand-counted graph 单元测试通过 |
| task semantic features `vartheta(f)` | Definition 5，Eq. (27) | 无 | 否 | Stage 1 使用 proxy semantic feature/accuracy model；后续替换 Transformer codec | `reproduction/semantic_coding/` | 高 | 可生成 Fig. 8 趋势，assumption 已记录 |
| semantic encoder/decoder | Eq. (25)-(27)，Fig. 5 | 无 | 否 | 完整 Transformer codec 在 proxy 验证后实现 | `reproduction/semantic_coding/transformer_codec.py` | 高 | Europarl preprocessing 和重构质量测试通过 |
| AWGN/channel/BER model | Eq. (10)-(12)，Fig. 8 | RF link rate model 已有，但没有 semantic channel model | 部分 | 增加 coding/channel model interface，输入 BER/SNR | `reproduction/semantic_coding/channel.py` | 中 | success probability 随 SNR 单调增加 |
| one-hop latency `Gamma` | Eq. (9) | `timeToSend()`，`DataBlock` 中的 prop/tx latency | 部分 | 使用 distance、file size、feedback size、packet loss 计算公式期望时延 | `reproduction/metrics/delivery.py` | 中 | 无 loss sanity case 与仿真事件时间一致 |
| SDE `xi` | Eq. (5)-(6) | 已有 throughput/latency plots | 否 | 基于可靠语义数据量、时间跨度、topology scale 的 SDE collector | `reproduction/metrics/sde.py` | 中 | synthetic run 得到预期 SDE |
| DQNSCR state | Sec. V-B | `getDeepStateDiff()`、`getDeepLinkedSats()` | 部分 | 替换为 local topology + semantic topology + task/coding features | `reproduction/routing/dqnscr.py` | 高 | state shape 被记录且可重复 |
| DQNSCR action | Sec. V-B | 当前 actions 为 `U/D/R/L` | 部分 | 对 semantic topology 邻居做 action mask；支持 variable neighbors 或方向映射 | `reproduction/routing/dqnscr.py` | 高 | invalid semantic actions 不会被选择 |
| DQNSCR reward | Eq. (28)-(29) | 当前 reward 包含 distance、queue、arrival、penalty | 部分 | 对齐 `1/Gamma` 和可靠投递 bonus/penalty，可选 SDE normalization | `reproduction/routing/rewards.py` | 高 | delivered/lost transition reward 测试通过 |
| DQN target/loss/action | Eq. (30)-(36) | `DDQNAgent.createModel()`、`train()`、target net | 部分 | 复用模式，但隔离出论文 DQN class、logs、checkpoints | `reproduction/routing/dqnscr.py` | 中 | 小型确定性 fixture 中 loss 下降 |
| OSPF baseline | Sec. VI-A | `getShortestPath()` 使用 NetworkX shortest path | 是/部分 | 在 global STG 上包装为 baseline，并接 conventional coding | `reproduction/routing/baselines.py` | 低 | Dijkstra path 等于 NetworkX reference |
| DQN-IR baseline | Sec. VI-A | 现有 DDQN infrastructure | 部分 | 实现不使用 TSTCN 的 global-STG DQN，并接 DL-JSCC proxy | `reproduction/routing/baselines.py` | 中 | 与 LISDM 共享 metrics |
| RTHop baseline | Sec. VI-A | online per-satellite DDQN 类似 hop-by-hop | 部分 | 增加最小可比 local hop-by-hop baseline，并标注近似 | `reproduction/routing/baselines.py` | 中 | 同 traffic 下输出 PDR/latency/throughput |
| packet delivery ratio | Fig. 9 | 全局 tracking created/delivered blocks | 是/部分 | 按 run、task、baseline、file count 标准化 | `reproduction/metrics/collector.py` | 低 | created/delivered/lost 数量守恒 |
| average E2E latency | Fig. 10 | `block.totLatency`、`getBlockTransmissionStats()` | 是 | 标准化导出 CSV/JSON | `reproduction/metrics/collector.py` | 低 | 与当前 stats 一致 |
| throughput | Fig. 11 | 已有 throughput plotting functions | 部分 | 标准化 bins 和 units，导出 raw values | `reproduction/metrics/collector.py` | 中 | raw blocks 计算结果与 plot input 一致 |
| SDE curves | Fig. 12 | 无 | 否 | 增加 SDE CSV/plot pipeline | `experiments/lisdm/plots/` | 中 | 可由 metrics CSV 生成 SDE 曲线 |
| seed 管理 | 工程约束 | 当前有 random usage，但没有中心 seed manager | 部分 | 对 Python、NumPy、TensorFlow/PyTorch 统一设 seed | `reproduction/utils/seeding.py` | 中 | synthetic run 重复结果一致 |
| experiment reports | 用户要求 | 现有 Results/Post-Processing 较临时 | 部分 | 建立结构化 run directory 和 reproduction report | `experiments/lisdm/` | 低 | 每次 run 都有 config、logs、metrics、plots、report |

## 当前文件职责说明

- `SimulationRL.py:RunSimulation()` 是 LISDM 实验最接近的 orchestration point，因为它已经负责 RL、metrics、plots、checkpoints 和 gateway count loop。
- `SimulationRL.py:initialize()` 是 topology、graph、paths、satellite send buffers 和 RL agents 的创建点。后续集成应轻量，并尽量委托给 adapter。
- `SimulationRL.py:Satellite.receiveBlock()` 是当前 RL next-hop decision point。后续应调用 strategy interface，而不是继续扩大硬编码的 Q-Learning/DDQN 分支。
- `SimulationRL.py:Earth.moveConstellation()` 是天然 snapshot boundary，但存在 README 已记录的 RL movement 风险。可先增加 snapshot recording，再修复 movement routing。
- `SimulationRL.py:createGraph()` 是拓扑提取源，但论文 STG 应使用 satellite-only view；gateway nodes 作为 access-layer nodes，不放入论文 `Gu`。
- `SimulationRL.py:getDeepStateDiff()` 已经收集本地邻居队列/位置，是有价值的参考。但它没有论文需要的 capacity、bandwidth vector、distance vector、semantic topology membership 和 task target similarity。

