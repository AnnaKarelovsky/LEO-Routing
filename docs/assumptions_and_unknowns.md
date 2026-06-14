# 假设与未知项

本文档列出论文中缺失的细节，以及复现阶段允许的可控假设。实现时不能静默硬编码论文没有给出的参数。

## 已从论文确认的信息

- 网络场景是 LEO/MEO 异构卫星网络。
- LEO：高度 1600 km，10 个轨道面，每面 20 颗卫星，45 degree inclination。
- MEO：高度 8000 km，4 个轨道面，每面 8 颗卫星，0 degree inclination。
- 卫星存储空间：10 MB。
- 初始 occupation：[0, 10] MB。
- transmitted files：R in [100, 500]。
- similarity threshold：lambda in [0.5, 0.9]。
- DQN discount factor：0.99。
- DQN learning rate：0.001。
- replay pool：10000。
- mini-batch：32。
- semantic dimension：64。
- semantic Transformer：3 layers，4 heads。
- quantization：8 bits。
- dataset：Europarl，max sequence length 20。

## 主要未知项

### Topology 与 Contact Generation

论文没有给出精确 contact table 生成算法、time horizon、snapshot 数量 `U`，也没有给出 event-driven slotting 的实现细节。

论文没有充分说明 MEO/LEO inter-layer ISL 构造规则。

论文没有说明 gateway/access-node 如何建模。系统模型主要围绕 satellite nodes `ns` 和 `nd`；当前框架则是 gateway 之间通过 linked satellites 路由。

规划假设：

- 使用当前仿真器的 graph rebuild 作为 snapshot boundary。
- 论文 STG 使用 satellite nodes/ISLs。gateways 作为 traffic injection/collection 的 access-layer wrapper。
- 后续显式实现 LEO/MEO constellation profile，并验证 node counts 和 layer labels。

### Node Capacity 与 Occupation

论文给出了 storage size 和 occupation range，但没有说明 occupation dynamics、admission policy 或 per-file storage reservation。

当前仿真器有 `Satellite.quota`，但没有论文式 storage occupation model。

规划假设：

- Stage 1 使用 config 中的 capacity，以及由 seed 控制的 deterministic/random occupation。
- Stage 2 根据需要把 occupation 接入真实 queue/storage state。

### Textual Topology Feature 构造

论文描述了 textual local topology 和语义类别，但没有给出精确 text template、vocabulary、bucket 规则或训练语料。

三种 task type 的 target vectors 只做了语义描述，没有给出数值。

规划假设：

- Stage 1 使用确定性 text templates，包含数值和 bucket labels。
- task targets 由 config 声明，并记录版本。
- 每次实验报告都必须写明 target vector 的选择。

### TSTCN 训练

论文给出了结构形状，但没有给出 topology feature extraction 的 training objective、dataset generation procedure、tokenization、optimizer、epochs 或 loss。

规划假设：

- Stage 1 使用 deterministic encoder 或小型未训练/配置式 projection，先测试 Fig. 6 和 routing integration。
- Stage 2 在 snapshot/text pipeline 稳定后再引入 trainable Transformer。

### Semantic Coding

论文给出了高层 Transformer encoder/decoder 和 semantic quality metric，但缺少很多训练细节：

- Europarl preprocessing。
- vocabulary size。
- optimizer。
- training epochs。
- SNR grid。
- AWGN 之外的 channel details。
- baseline DL-JSCC/LSTM 的精确配置。

规划假设：

- Stage 1 使用 controlled semantic transmission accuracy function。
- 该 proxy 能复现 routing/SDE 对 semantic reliability 的敏感性，但不能证明 semantic codec 的 representation quality。
- Fig. 8 在 full semantic coding 训练完成前应标注为 proxy trend。

### DQNSCR State 与 Action

论文说每个 task file 在节点 `ni` 处是 agent，并观察 TSTCN 中的 local environment information，但没有完整指定 state vector。

action space 可能随相邻节点数量变化；当前仿真器使用 directional `U/D/R/L` actions。

规划假设：

- Stage 1 使用 current semantic topology 上的 masked neighbor actions。
- 若为了最小集成需要兼容方向动作，则尽量把 selected neighbors 映射到 U/D/R/L；无法映射时在新 adapter 中使用 ordered-neighbor action set。

### Reward 细节

论文定义 reward 包含 `1/Gamma` 和可靠性项：可靠接收为正，丢失为负。

论文没有说明 scaling、clipping、terminal handling、TTL，或 file 跨多个 packet 时如何累计 reward。

规划假设：

- 先实现 formula-faithful unscaled reward。
- normalization 只能通过 config 开启，并在报告中说明。
- Stage 1 定义一个 block 等于一个 file。

### Baseline 保真度

OSPF 足够清晰，因为论文指定了 STG 上的 Dijkstra。

DQN-IR 和 RTHop 是引用方法，但论文没有给出完整实现细节。

规划假设：

- Stage 1 实现可比 baseline，并明确标注为近似：
  - OSPF：global graph shortest path + conventional coding proxy。
  - DQN-IR：不使用 TSTCN 的 global graph DQN + DL-JSCC proxy。
  - RTHop：不使用 TSTCN 的 local hop-by-hop policy + DL-JSCC proxy。
- 除非完整复现原论文算法，否则数值比较必须标注为 implemented approximations。

## 可重复性规则

- 所有随机选择必须从 config 读取 seed。
- seed 必须作用于 Python `random`、NumPy 和所选 ML framework。
- run artifacts 必须包含：
  - effective config。
  - git commit hash。
  - dependency snapshot。
  - seed。
  - input topology profile。
  - output metrics 和 raw logs。
- 不能把现有 `Post-Processing` artifacts 当作 LISDM 复现证据。
- 在脚本实际运行并产生新结果文件前，不能声称已经复现成功。

## 需要负责人确认的问题

1. 第一阶段是否接受定性趋势复现？
2. Stage 1 是否允许使用 semantic coding 和 TSTCN feature proxies？
3. 新增 ML 模块使用论文的 PyTorch，还是沿用当前仓库 TensorFlow/Keras？
4. 是否必须先实现论文 LEO/MEO，再做 TSTCN proof-of-concept？
5. gateway-to-satellite access 是否计入 SDE latency，还是视为论文模型之外？
6. DQN-IR 和 RTHop 是否必须完整复现其原论文算法，还是第一轮可用可比近似？

