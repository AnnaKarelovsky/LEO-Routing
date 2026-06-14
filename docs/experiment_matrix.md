# 实验矩阵

本文档定义建议的可重复实验目录结构，并将实验映射到论文图表。

## 推荐目录结构

仅在实现获批后创建：

```text
experiments/
  lisdm/
    configs/
      base.yaml
      topology_leo_meo.yaml
      topology_existing_kepler.yaml
      tasks_three_types.yaml
      coding_proxy.yaml
      coding_transformer.yaml
      routing_lisdm.yaml
      routing_ospf.yaml
      routing_dqn_ir.yaml
      routing_rthop.yaml
      sweep_lambda.yaml
      sweep_transmitted_files.yaml
      sweep_snr.yaml
    scripts/
      run_experiment.py
      run_sweep.py
      plot_figures.py
      make_report.py
    runs/
      <run_id>/
        effective_config.yaml
        seed.txt
        git_commit.txt
        raw_logs/
        artifacts/
          snapshots/
          local_topology/
          topology_text/
          topology_embeddings/
          semantic_topologies/
          routing_checkpoints/
        metrics/
          topology_metrics.csv
          delivery_metrics.csv
          semantic_quality.csv
          sde.csv
          rewards.csv
          losses.csv
        plots/
        report.md
    reports/
      reproduction_summary.md
```

## 通用 Config 字段

每个实验都必须回答：

- 使用了什么拓扑？
- 使用了哪种 snapshot policy？
- 使用了哪个 routing module？
- 使用了哪个 coding module？
- 使用了哪个 topology compression module？
- 使用了什么 seed？
- 输出了哪些 raw metrics？
- 对应论文哪张图？

推荐 config sections：

- `run`
  - `seed`
  - `run_id`
  - `output_dir`
  - `commit_required`
- `topology`
  - `profile`：`existing_kepler`、`paper_leo_meo` 或 synthetic fixture。
  - `snapshot_policy`：`on_graph_rebuild`、`fixed_interval` 或 `event_driven`。
  - `movement_time`
  - `duration`
- `traffic`
  - `num_files`
  - `file_size_bytes`
  - `feedback_size_bytes`
  - `source`
  - `destination`
  - `task_mix`
- `semantic_coding`
  - `mode`：`proxy`、`transformer`、`conventional`、`dl_jscc_proxy`。
  - `snr_db`
  - `semantic_dimension`
  - `ber_model`
- `tstcn`
  - `enabled`
  - `lambda`
  - `encoder`：`deterministic`、`light_transformer`、`paper_transformer`。
  - `task_targets`
  - `relay_policy`
- `routing`
  - `algorithm`：`lisdm_dqnscr`、`ospf`、`dqn_ir`、`rthop`。
  - `state_version`
  - `reward_version`
  - `checkpoint_interval`
- `metrics`
  - `emit_raw_blocks`
  - `emit_snapshot_metrics`
  - `emit_sde`
  - `plot`

## 实验表

| 实验 ID | 对应论文图 | 要回答的问题 | 拓扑 | 压缩 | 编码 | 路由 | sweep | 主要输出 | 成功标准 |
|---|---|---|---|---|---|---|---|---|---|
| E0 | 无 | 整条 pipeline 能否在 tiny deterministic graph 上跑通？ | synthetic 6-10 nodes | deterministic TSTCN | proxy | OSPF 和 LISDM smoke route | seed only | metrics CSV、snapshot artifacts | 重复运行得到相同 metrics |
| E1 | Fig. 6 | 更严格 similarity threshold 是否降低 semantic topology scale？ | 先 existing Kepler，后 paper LEO/MEO | TSTCN | 无 | 无 | lambda 0.5-0.9 | `gamma`、selected node/edge counts | `gamma` 随 lambda 增大而降低 |
| E2 | Fig. 7 | semantic topology 上的 DQNSCR 是否比 global topology 更易收敛？ | 两者共享 snapshots | TSTCN vs no compression | proxy | DQNSCR | episodes | rewards、losses | reward 提升并稳定；semantic topology 趋势优于 global |
| E3 | Fig. 8 | semantic coding quality 是否随 SNR 和 dimension 提升？ | 不依赖拓扑 | 不适用 | proxy first，Transformer later | 不适用 | SNR、dimension | semantic quality | quality 随 SNR/dimension 单调提升 |
| E4 | Fig. 9 | LISDM 是否在负载下提升 packet delivery ratio？ | shared STG | 仅 LISDM 使用 TSTCN | 各方法对应 proxy/conventional | LISDM、OSPF、DQN-IR、RTHop | R 100-500 | PDR | LISDM 最高或趋势一致 |
| E5 | Fig. 10 | LISDM 是否降低 average E2E latency？ | shared STG | 仅 LISDM 使用 TSTCN | 各方法对应 proxy/conventional | LISDM、OSPF、DQN-IR、RTHop | R 100-500 | mean latency | 中高负载下 LISDM 更低 |
| E6 | Fig. 11 | LISDM 是否提升 throughput？ | shared STG | 仅 LISDM 使用 TSTCN | 各方法对应 proxy/conventional | LISDM、OSPF、DQN-IR、RTHop | R 100-500 | throughput | 中高负载下 LISDM 更高 |
| E7 | Fig. 12 | LISDM 是否最大化 SDE？ | shared STG | 仅 LISDM 使用 TSTCN | 各方法对应 proxy/conventional | LISDM、OSPF、DQN-IR、RTHop | R 100-500 | SDE | LISDM 最高；SDE 先升后降 |
| E8 | Ablation | 收益分别来自 TSTCN、semantic coding 还是 DQNSCR？ | shared STG | on/off | proxy on/off | DQNSCR/OSPF | selected R | all metrics | ablation 能解释趋势 |

## 图表复现说明

### Fig. 6 - Topology Compression Rates

输入：

- snapshot set。
- 三种 task profiles。
- TSTCN encoder。
- lambda thresholds 0.5 到 0.9。

指标：

- `gamma`。
- 每个 task 和 snapshot 的 node count、edge count。
- overlap count `P_u`。

趋势目标：

- `gamma` 随 lambda 增大而降低。
- 低 lambda 时，过多拓扑被保留，`gamma` 可能接近 1。

### Fig. 7 - Convergence Analysis

输入：

- 相同 traffic 和 seed。
- global topology 上的 DQNSCR。
- 各 semantic topology 上的 DQNSCR。
- 论文文字中使用 lambda 0.8。

指标：

- 每 episode average reward。
- loss。
- training 期间 delivery success。

趋势目标：

- 早期 reward 因投递失败接近 0。
- reward 上升并稳定；论文中约 200 episodes 后稳定。
- semantic topologies 比 global topology 更快收敛。

### Fig. 8 - Semantic Quality

输入：

- SNR sweep。
- semantic dimensions，包括论文 `N=64`。
- semantic coding proxy 或 full codec。
- DL-JSCC baseline proxy 或完整实现。

指标：

- semantic quality。
- semantic transmission accuracy。

趋势目标：

- quality 随 SNR 增加。
- 更高 semantic dimension 提升 quality。
- 完整 codec 实现后，Transformer semantic coding 应优于 DL-JSCC。

### Fig. 9 - Packet Delivery Ratio

输入：

- 共享 topology snapshots。
- R in [100, 500]。
- mechanisms：LISDM、OSPF、DQN-IR、RTHop。

指标：

- created files。
- delivered files。
- semantically reliable delivered files。
- loss cause breakdown。

趋势目标：

- OSPF 在 congestion 下退化最快。
- DQN-IR/RTHop 优于 OSPF。
- LISDM 最好。

### Fig. 10 - Average E2E Delivery Latency

输入：

- 与 Fig. 9 相同。

指标：

- mean latency。
- queue、transmission、propagation components。
- 用于 reward audit 的公式期望时延 `Gamma`。

趋势目标：

- latency 随 R 增大。
- 较重负载下 LISDM 低于 baselines。

### Fig. 11 - Throughput

输入：

- 与 Fig. 9 相同。

指标：

- 单位时间投递的 semantic data。
- uplink/downlink throughput bins。

趋势目标：

- throughput 随 R 初期上升。
- congestion 下 OSPF 更早饱和或退化，LISDM 更稳定。

### Fig. 12 - SDE

输入：

- 与 Fig. 9 相同。
- semantic topology scale `rho`。

指标：

- SDE。
- numerator/denominator components。

趋势目标：

- LISDM 的 SDE 最高。
- SDE 在重负载下先达到峰值再下降。
- 论文提到不同机制峰值大致在 R 约 200、250、300、350；在完整 setup 前不能声称精确匹配。

## Run Report 模板

每次 run report 应包含：

- objective。
- paper figure target。
- effective config。
- seed。
- topology profile。
- snapshot count 和 node/edge counts。
- module versions：
  - topology compression。
  - semantic coding。
  - routing。
- metrics summary。
- trend judgment：
  - reproduced。
  - partially reproduced。
  - not reproduced。
- 与论文的已知偏差。
- raw artifact paths。

