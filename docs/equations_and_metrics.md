# 公式、指标与图表目标

本文档提取论文关键公式，并为每个公式指定未来复现模块。公式符号尽量沿用论文；代码实现时应使用清晰的 ASCII 变量名。

## 公式归属

| 公式 | 含义 | 复现模块 | 输入 | 输出 | 测试优先级 |
|---|---|---|---|---|---|
| (1) | 每个 snapshot 的全局拓扑信息 `Theta_u`：node degree/capacity 加 bandwidth、distance matrices | `reproduction/topology/local_features.py` | snapshot graph `Gu`、satellite capacities | `GlobalTopologyFeatures` | 高 |
| (2) | 节点 `ni` 周围跨 snapshots 的本地拓扑信息 `Theta^ni` | `reproduction/topology/local_features.py` | snapshot sequence、node ID | per-node local topology history | 高 |
| (3) | 语义本地拓扑特征 `vartheta(Theta_u^ni) = psi(textual topology; temporal change)` | `reproduction/tstcn/feature_encoder.py` | textual topology、temporal diff | embedding vector | 中 |
| (4) | topology compression ratio `gamma` | `reproduction/metrics/topology.py` | semantic topology node/edge counts、original counts、overlaps | scalar `gamma` | 高 |
| (5) | SDE `xi_K(ns,nd)` | `reproduction/metrics/sde.py` | 可靠投递语义数据量、投递时间跨度、topology scale | scalar SDE | 高 |
| (6) | 优化目标：在 C1-C5 约束下最大化 SDE | 实验目标，不单独成模块 | metrics 和 constraints | objective value | 中 |
| (7) | C1：topology semantic features 属于 `ST` 且 `gamma <= 1` | `reproduction/tstcn/compressor.py` | topology embeddings、gamma | constraint pass/fail | 中 |
| (8) | C2：task semantic features 属于 `SF` | `reproduction/semantic_coding/interface.py` | task data features | constraint pass/fail | 中 |
| (9) | 期望单跳投递时延 `Gamma` | `reproduction/metrics/delivery.py` 和 DQNSCR reward | link distance、bandwidth、file size、feedback size、packet loss | expected latency | 高 |
| (10) | conventional packet loss rate | `reproduction/semantic_coding/channel.py` | uplink/downlink BER、file size、feedback size | loss probability | 高 |
| (11) | 使用 semantic transmission accuracy 的 semantic packet loss rate | `reproduction/semantic_coding/channel.py` | semantic accuracy、downlink BER、feedback size | loss probability | 高 |
| (12) | C3：semantic transmission accuracy 应优于 conventional bit-level success | `reproduction/semantic_coding/accuracy_model.py` | semantic accuracy、BER、file size | constraint pass/fail | 中 |
| (13) | C4：node storage capacity 必须大于等于 file size | `reproduction/metrics/constraints.py` | file size、node capacity | constraint pass/fail | 高 |
| (14) | C5：link bandwidth 必须能在 delivery latency 内承载 file | `reproduction/metrics/constraints.py` | file size、latency、bandwidth | constraint pass/fail | 高 |
| (15) | 每个 snapshot 的 local information：`Theta_u^ni = {d, I, A(ni,eta_i), Phi(ni,eta_i)}` | `reproduction/topology/local_features.py` | snapshot graph、node | local feature object | 高 |
| (16) | textual topology 加 temporal change：`v(Theta_u^ni)` | `reproduction/tstcn/textual_topology.py` | numeric local features、previous features | text tokens | 中 |
| (17) | 将 textual sequence 拆为 blocks | `reproduction/tstcn/textual_topology.py` | text tokens | token blocks | 低 |
| (18) | one-head self-attention | `reproduction/tstcn/feature_encoder.py` | Q/K/V matrices | attention output | Stage 1 低，完整版本高 |
| (19) | multihead concatenation 和 projection | `reproduction/tstcn/feature_encoder.py` | per-head outputs | multihead output | Stage 1 低，完整版本高 |
| (20) | layer norm 加 feed-forward block | `reproduction/tstcn/feature_encoder.py` | attention output | final hidden state | Stage 1 低，完整版本高 |
| (21) | topology feature 与 task target 的 cosine similarity | `reproduction/tstcn/compressor.py` | topology embedding、task target embedding | similarity score | 高 |
| (22) | semantic nodes 不连通时选择 relay node | `reproduction/tstcn/semantic_topology.py` | original graph、selected nodes、similarity scores | relay nodes | 高 |
| (23) | selected nodes 加 relay nodes/links 组成 semantic topology | `reproduction/tstcn/semantic_topology.py` | selected graph、relay paths | connected semantic graph | 高 |
| (24) | TSTCN：跨 snapshots 的 task-specific semantic topologies 集合 | `reproduction/tstcn/semantic_topology.py` | per-task semantic graphs | TSTCN artifact | 中 |
| (25) | semantic encoder | `reproduction/semantic_coding/transformer_codec.py` | task text | encoded semantic bits/features | 延后 |
| (26) | semantic decoder | `reproduction/semantic_coding/transformer_codec.py` | channel output | recovered text | 延后 |
| (27) | 通过最小化 cross entropy 学得 semantic feature | `reproduction/semantic_coding/transformer_codec.py` | original/recovered text | semantic feature vector | 延后 |
| (28) | DQNSCR immediate reward：`1/Gamma + reliability_term` | `reproduction/routing/rewards.py` | expected latency、delivery outcome | reward | 高 |
| (29) | reliability term：delivered 为正，lost 为负 | `reproduction/routing/rewards.py` | delivery/loss event、latency | bonus/penalty | 高 |
| (30) | discounted cumulative reward | `reproduction/routing/dqnscr.py` | reward sequence、discount | return | 中 |
| (31) | policy 下的 long-term Q function | `reproduction/routing/dqnscr.py` | state/action/reward | Q estimate | 中 |
| (32) | temporal-difference target | `reproduction/routing/dqnscr.py` | reward、next Q | TD target | 中 |
| (33) | Q update rule | `reproduction/routing/dqnscr.py` | current Q、target、learning rate | updated Q | 中 |
| (34) | target-network DQN target | `reproduction/routing/dqnscr.py` | target network、next state | target value | 中 |
| (35) | replay batch 上的 MSE loss | `reproduction/routing/dqnscr.py` | replay batch、Q predictions | loss | 中 |
| (36) | 根据 Q value 做 greedy action selection | `reproduction/routing/dqnscr.py` | state、action mask、Q network | next-hop action | 高 |

## 复现指标定义

### Topology Compression Ratio

论文概念：

- `rho_u = sum_k (N(V_hat_u^k) + N(E_hat_u^k)) - P_u`
- `gamma = sum_u rho_u / (U*M + sum_u N(E_u))`

实现说明：

- `P_u` 表示跨 task semantic topologies 的 overlap。实现时建议把重复出现的 node 和 edge 只计一次，并在 run report 中说明具体处理。
- Stage 1 同时计算：
  - `gamma_paper`：尽量贴近论文公式。
  - `gamma_union`：semantic topology union size / original topology size，用于 sanity check。

### Semantic Delivery Efficiency

论文概念：

- 分子：可靠投递的 semantic task data sizes 之和。
- 分母：delivery time span 乘以 semantic topology scale。

实现说明：

- 使用 `DataBlock.creationTime` 和 `DataBlock.totLatency` 推导 delivery timestamps。
- 第一阶段建议一个 block 等于一个 file，减少 task file 与 packet grouping 的歧义。
- 是否 semantically reliable 由 semantic coding model 输出决定。

### Packet Delivery Ratio

指标：

- `PDR = delivered_count / created_count`

实现说明：

- 当前全局变量 `createdBlocks` 和 `receivedDataBlocks` 提供原始数量。
- 新增 lost by cause：
  - packet loss model。
  - no semantic route。
  - TTL/hop budget。
  - movement/re-link invalidation。

### Average End-to-End Latency

指标：

- 对 semantically reliable delivered blocks 的 `block.totLatency` 求均值。

实现说明：

- 同时保留 queue、transmission、propagation 分量；当前 `DataBlock` 已有这些字段。

### Throughput

指标：

- 单位时间内可靠投递的 semantic data size，通常以 Mbps 表示。

实现说明：

- 当前 plotting code 已根据 block creation/arrival time 做 binning 计算 throughput。
- 新 metric collector 应在绘图前导出 raw bins 和单位。

### Semantic Quality

论文 Fig. 8 概念：

- 基于正确恢复词数，计算 accuracy-like ratio 和 completeness-like ratio 的乘积。

Stage 1 proxy：

- 使用由 SNR、semantic dimension、BER、coding type 决定的 deterministic semantic quality function。
- 必须随 SNR 和 semantic dimension 单调增加。
- 不能用 proxy 宣称已经完成 full semantic coding reproduction。

完整版本：

- 使用 Europarl sentences，最大序列长度 20。
- 按论文公式计算 recovered-word overlap。

## 图表目标

| 图 | 展示内容 | X 轴 | Y 轴 | 所需输入 | 指标 | 优先级 |
|---|---|---|---|---|---|---|
| Fig. 1 | system model | 不适用 | 不适用 | 规划图 | 不适用 | 不做实验复现 |
| Fig. 2 | topology definitions relationship | 不适用 | 不适用 | 规划图 | 不适用 | 不做实验复现 |
| Fig. 3 | textual local topology construction | snapshot local features | textual tokens | STG snapshots | token/text artifacts | 中 |
| Fig. 4 | TSTCN architecture | 不适用 | 不适用 | encoder architecture | 不适用 | Stage 1 只文档化 |
| Fig. 5 | semantic coding architecture | 不适用 | 不适用 | codec architecture | 不适用 | 延后 |
| Fig. 6 | similarity thresholds 下 compression ratio | `lambda` in [0.5, 0.9] | `gamma` | TSTCN outputs for 3 task types | `gamma` | 最高 |
| Fig. 7 | 不同 topology 下 convergence | episode | average reward | DQNSCR training logs | reward | 高 |
| Fig. 8 | channel condition 下 semantic quality | SNR | semantic quality | semantic coding proxy/full codec | semantic quality | 中 |
| Fig. 9 | 不同机制下 packet delivery ratio | transmitted files R，100-500 | PDR | LISDM 和 baselines | PDR | 高 |
| Fig. 10 | 不同机制下 average E2E latency | R，100-500 | latency | LISDM 和 baselines | mean latency | 高 |
| Fig. 11 | 不同机制下 throughput | R，100-500 | throughput | LISDM 和 baselines | throughput | 高 |
| Fig. 12 | 不同机制下 SDE | R，100-500 | SDE | LISDM 和 baselines | SDE | 高 |

## 从论文中确认的参数

来自 Table II 和仿真设置：

- Constellation：LEO/MEO。
- LEO altitude：1600 km。
- MEO altitude：8000 km。
- Orbital inclination：45 degrees / 0 degrees。
- Orbital planes：10 / 4。
- Satellites per plane：20 / 8。
- 每颗 satellite storage space：10 MB。
- satellite occupation：[0, 10] MB。
- transmitted files `R`：[100, 500]。
- similarity threshold `lambda`：[0.5, 0.9]。
- DQN discount factor `delta`：0.99。
- DQN learning rate `kappa`：0.001。
- replay pool capacity：10000。
- mini-batch size：32。
- semantic dimension：`N = 64`。
- semantic encoder：3-layer、4-head self-attention。
- bit quantization：`B = 8`。
- dataset：Europarl，max sequence length 20。
- 论文 DQN 实现环境：PyTorch 1.10.2，Python 3.9.7。

