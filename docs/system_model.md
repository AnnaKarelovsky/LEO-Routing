# System Model

## A. 系统概述与双时标抽象

本文考虑由低轨卫星星座、地面网关和时变语义业务共同构成的星地一体网络。卫星集合、地面网关集合和系统节点集合分别记为

$$
\mathcal{S}
=
\{s_{o,n}\mid o\in\{1,\ldots,O\},n\in\{1,\ldots,N_o\}\},
\quad
\mathcal{G}=\{g_1,\ldots,g_G\},
\quad
\mathcal{N}=\mathcal{S}\cup\mathcal{G},
$$

其中 \(o\) 和 \(n\) 分别表示轨道面编号和轨道面内卫星编号。

系统采用双时标抽象。快时标 \(t\) 对应语义单元的逐跳转发过程，卫星在语义单元到达时根据当前链路、队列和任务语义属性选择下一跳。结构时标 \(k\) 对应星座拓扑快照、星地接入、基础星间链路和拓展光链路状态的更新。拓展链路重构周期 \(T_s\) 与基础拓扑更新周期 \(T_t\) 同步：

$$
T_s=T_t.
$$

在第 \(k\) 个结构周期内，网络表示为动态图

$$
\mathcal{H}_k
=
(\mathcal{N},\mathcal{E}^{\mathrm{act}}_k),
$$

其中 \(\mathcal{E}^{\mathrm{act}}_k\) 为周期 \(k\) 内可用于转发的星地链路和星间链路集合。结构决策在周期内保持不变，语义单元按照快时标连续转发。

## B. LEO 星座、网关接入与链路可达性

每个轨道面 \(o\) 被近似为圆轨道，其轨道高度、倾角和轨道面初始经度分别为 \(h_o\)、\(\delta_o\) 和 \(\epsilon_o\)。记卫星 \(i\in\mathcal{S}\) 在周期 \(k\) 的三维位置为 \(\mathbf{r}_i(k)\)，地面网关 \(g\in\mathcal{G}\) 的固定地理位置对应地心坐标 \(\mathbf{r}_g\)。星地链路由独立的 RF/Ka/Ku 终端承担。

令 \(\mathbf{1}\{\cdot\}\) 表示指示函数，\(\theta_{gi,k}\)、\(d_{gi,k}\)、\(R^{\mathrm{G}}_{gi,k}\) 和 \(R^{\mathrm{G}}_{ig,k}\) 分别表示网关 \(g\) 与卫星 \(i\) 之间的仰角、星地距离、上行速率和下行速率。物理可行性指示量定义为

$$
\chi^{\mathrm{G}}_{gi,k}
=
\mathbf{1}
\left\{
\theta_{gi,k}\ge\theta_{\min},
\ d_{gi,k}\le d_{\max}^{\mathrm{G}},
\ R^{\mathrm{G}}_{gi,k}>0,
\ R^{\mathrm{G}}_{ig,k}>0
\right\},
$$

令二元变量 \(z_{gi,k}=1\) 表示网关 \(g\) 在周期 \(k\) 接入卫星 \(i\)，则网关接入约束为

$$
\sum_{i\in\mathcal{S}}z_{gi,k}=1,
\quad
z_{gi,k}\le\chi^{\mathrm{G}}_{gi,k},
\quad
\sum_{g\in\mathcal{G}}z_{gi,k}\le1.
$$

上述约束分别保证每个网关接入一颗卫星、接入关系满足物理可行性，以及每颗卫星在同一周期最多服务一个网关。实际星地链路集合为

$$
\mathcal{E}^{\mathrm{G}}_k
=
\{(g,i),(i,g)\mid g\in\mathcal{G},i\in\mathcal{S},z_{gi,k}=1\}.
$$

节点 \(i,j\in\mathcal{N}\) 在周期 \(k\) 的距离为

$$
d_{ij,k}
=
\|\mathbf{r}_i(k)-\mathbf{r}_j(k)\|.
$$

令 \(m\in\{\mathrm{G},\mathrm{B},\mathrm{E}\}\) 分别表示 GSL、基础 ISL 和拓展光学 ISL，\(\varpi\) 表示候选频谱效率。不同链路使用各自的带宽、链路预算和调制编码集合，其速率统一表示为

$$
R^m_{ij,k}
=
W^m
\max_{\varpi\in\mathcal{M}^m}
\left\{
\varpi:
\mathrm{SNR}^m_{ij,k}
\ge
\mathrm{SNR}^m_{\min}(\varpi)
\right\}.
$$

其中 \(W^m\)、\(\mathcal{M}^m\) 和 \(\mathrm{SNR}^m_{\min}(\varpi)\) 分别为链路类型 \(m\) 的带宽、频谱效率集合和可靠通信门限。GSL 和基础 ISL 可采用 RF 自由空间损耗与 DVB-S2 参数；拓展光链路采用包含波长、发射功率、收发孔径、指向损耗和接收灵敏度的光链路预算。若不存在满足门限的频谱效率，则令 \(R^m_{ij,k}=0\)。

## C. 六接口链路架构

每颗卫星配置六个相互独立的物理通信端口：一个 RF/Ka/Ku GSL 端口、四个基础 ISL 端口和一个可重构拓展光学 ISL（Extension Optical ISL, E-OISL）端口。四个基础 ISL 端口维持同轨上/下邻居和跨轨左/右邻居，E-OISL 端口在每个结构周期内至多连接一颗候选卫星。链路资源管理与拓扑控制模块根据语义压力、队列状态和指向捕获跟踪（Pointing, Acquisition, and Tracking, PAT）成本配置 E-OISL。

![六接口星地网络与语义感知拓扑控制示意图](<pictures/初稿/6接口示意图 .jpg>)

*图 1. 六接口架构：1 个 GSL 端口、4 个基础 ISL 端口和 1 个可重构 E-OISL 端口。*

卫星 \(i\) 的基础星间邻居集合为

$$
\mathcal{N}^{\mathrm{B}}_i(k)
=
\{i^{\mathrm{up}},i^{\mathrm{down}},i^{\mathrm{right}},i^{\mathrm{left}}\},
$$

由四个基础端口构成的物理链路集合记为 \(\mathcal{E}^{\mathrm{B}}_k\)，每条双向基础链路在该集合中存储一次。E-OISL 的物理可行性指示量定义为

$$
\chi^{\mathrm{I}}_{ij,k}
=
\mathbf{1}
\left\{
i,j\in\mathcal{S},
\ i\ne j,
\ d_{ij,k}\le d_{\max}^{\mathrm{I}},
\ R^{\mathrm{E}}_{ij,k}>0,
\ R^{\mathrm{E}}_{ji,k}>0
\right\},
$$

其中 \(d_{\max}^{\mathrm{I}}\) 为 E-OISL 的最大作用距离。候选拓展链路集合为

$$
\mathcal{E}^{\mathrm{C}}_k
=
\{\{i,j\}\subseteq\mathcal{S}\mid i\ne j,\chi^{\mathrm{I}}_{ij,k}=1\}.
$$

集合 \(\mathcal{E}^{\mathrm{C}}_k\) 将每条双向 E-OISL 物理链路存储一次。对 \(\{i,j\}\in\mathcal{E}^{\mathrm{C}}_k\)，定义二元变量 \(x_{ij,k}=1\) 表示周期 \(k\) 激活卫星 \(i\) 与 \(j\) 之间的 E-OISL，并令

$$
x_{ij,k}\in\{0,1\},
\quad
x_{ij,k}=0,
\quad
\forall\{i,j\}\notin\mathcal{E}^{\mathrm{C}}_k.
$$

令 \(b_{ij,k}=\mathbf{1}\{\{i,j\}\in\mathcal{E}^{\mathrm{B}}_k\}\)，则卫星对 \(\{i,j\}\) 的有效星间容量为

$$
C_{ij,k}
=
b_{ij,k}R^{\mathrm{B}}_{ij,k}
+
x_{ij,k}R^{\mathrm{E}}_{ij,k}.
$$

当 \(b_{ij,k}=0,x_{ij,k}=1\) 时，E-OISL 提供新的拓扑连接；当 \(b_{ij,k}=1,x_{ij,k}=1\) 时，E-OISL 作为并行链路增加该卫星对的容量。每颗卫星只有一个 E-OISL 端口，因此

$$
\sum_{j:\{i,j\}\in\mathcal{E}^{\mathrm{C}}_k}
x_{ij,k}
\le1,
\quad
\forall i\in\mathcal{S}.
$$

当前可用链路集合为

$$
\mathcal{E}^{\mathrm{act}}_k
=
\mathcal{E}^{\mathrm{G}}_k
\cup
\{(i,j)\in\mathcal{S}\times\mathcal{S}\mid i\ne j,C_{ij,k}>0\}.
$$

对于星地链路，分别令 \(C_{gi,k}=R^{\mathrm{G}}_{gi,k}\) 和 \(C_{ig,k}=R^{\mathrm{G}}_{ig,k}\)。

## D. 链路可靠性与时延

记 \(Q_{ij,k}\)、\(Q_{ij}^{\max}\) 和 \(A_{ij,k}\) 分别为周期 \(k\) 开始时链路 \((i,j)\) 的出向队列数据量、队列容量和周期内到达数据量，三者均以 bit 计。语义单元 \(p\) 的长度为 \(B_p\)，链路 \((i,j)\) 的误比特率为 \(\varepsilon^{\mathrm{bit}}_{ij,k}\)。单次传输的信道失败概率为

$$
p^{\mathrm{ch}}_{p,ij,k}
=
1-
(1-\varepsilon^{\mathrm{bit}}_{ij,k})^{B_p}.
$$

令 \([x]^+=\max\{x,0\}\)，\(\Pi_{[0,1]}(\cdot)\) 表示截断到 \([0,1]\)，\(\epsilon>0\) 用于避免分母为零。队列溢出造成的丢弃概率表示为

$$
p^{\mathrm{q}}_{ij,k}
=
\Pi_{[0,1]}
\left(
\frac{
[Q_{ij,k}+A_{ij,k}-C_{ij,k}T_t-Q_{ij}^{\max}]^+
}{
A_{ij,k}+\epsilon
}
\right),
$$

设链路最多允许 \(M_{ij}\) 次传输，则期望传输次数和单跳成功概率分别为

$$
\overline{M}_{p,ij,k}
=
\frac{
1-(p^{\mathrm{ch}}_{p,ij,k})^{M_{ij}}
}{
1-p^{\mathrm{ch}}_{p,ij,k}+\epsilon
},
\quad
q_{p,ij,k}
=
(1-p^{\mathrm{q}}_{ij,k})
\left[
1-(p^{\mathrm{ch}}_{p,ij,k})^{M_{ij}}
\right].
$$

信道失败触发链路层重传，队列溢出则直接造成语义单元丢弃。令 \(T^{\mathrm{q}}_{ij,k}\) 表示排队时延，\(c\) 表示光速，则单跳期望时延为

$$
\widetilde{D}_{p,ij,k}
=
T^{\mathrm{q}}_{ij,k}
+
\overline{M}_{p,ij,k}
\left(
\frac{B_p}{C_{ij,k}+\epsilon}
+
\frac{d_{ij,k}}{c}
\right),
$$

对由有序链路组成的路径 \(\pi_p\)，端到端期望时延和成功概率为

$$
T_p(\pi_p)
=
\sum_{(i,j,k)\in\pi_p}
\widetilde{D}_{p,ij,k},
\quad
P_p^{\mathrm{succ}}(\pi_p)
=
\prod_{(i,j,k)\in\pi_p}
q_{p,ij,k}.
$$

## E. 任务语义业务模型

业务层采用给定的语义编码器将任务数据转换为若干语义单元，目的端语义解码器根据成功接收的语义单元恢复任务信息。本文优化语义单元的拓扑资源分配与路由，不涉及语义编解码器的训练。

任务 \(f\) 及其语义单元 \(p\) 分别表示为

$$
f
=
(g_f,d_f,\mathcal{P}_f,\tau_f,\beta_f,S_f^{\min}),
\quad
p
=
(f,B_p,t_p^0,w_p),
$$

其中 \(g_f,d_f\in\mathcal{G}\) 为源网关和目的网关，\(\mathcal{P}_f\) 为任务的语义单元集合，\(\tau_f\) 为任务截止时间，\(\beta_f\) 为超时衰减系数，\(S_f^{\min}\) 为最低任务语义保真度，\(t_p^0<\tau_f\) 为语义单元生成时间。\(w_p\in[0,1]\) 表示语义单元 \(p\) 对任务恢复质量的边际语义贡献，并满足

$$
\sum_{p\in\mathcal{P}_f}w_p=1.
$$

\(w_p\) 可由任务性能下降量、语义相似度变化或语义编解码器的贡献评估模块给出。本文采用语义贡献可加的一阶近似。令 \(\boldsymbol{\pi}=\{\pi_p\}\) 表示路径决策集合，\(\mathbf{x}=\{\mathbf{x}_k\}\) 表示全部结构周期的拓展链路决策。任务 \(f\) 的期望语义保真度定义为

$$
\overline{S}_f(\boldsymbol{\pi},\mathbf{x})
=
\sum_{p\in\mathcal{P}_f}
w_p
P_p^{\mathrm{succ}}(\pi_p)
\mathbf{1}\{T_p(\pi_p)\le\tau_f\}.
$$

为反映高语义贡献单元随截止时间临近而增加的调度需求，定义语义时效压力

$$
\Psi_p(t)
=
w_p
\left[
1+
\min
\left\{
1,
\frac{[t-t_p^0]^+}{\tau_f-t_p^0+\epsilon}
\right\}
\right].
$$

令 \(\mathbf{x}_k=\{x_{ij,k}\}\) 表示周期 \(k\) 的拓展链路决策。语义单元 \(p\) 的期望及时语义效用为

$$
U_p(\pi_p,\mathbf{x}_k)
=
w_p
P_p^{\mathrm{succ}}(\pi_p)
\exp
\left(
-\beta_f
[T_p(\pi_p)-\tau_f]^+
\right),
$$

该效用同时刻画语义贡献、可靠交付和时效要求。

## F. E-OISL 重构成本

E-OISL 的重构成本由建立、保持和释放三部分组成。建立成本 \(c^{\mathrm{set}}_{ij,k}\) 包含终端转向、PAT 捕获同步时间及相应能耗；保持成本 \(c^{\mathrm{hold}}_{ij,k}\) 包含持续跟踪能耗和端口占用成本；释放成本 \(c^{\mathrm{off}}_{ij,k}\) 表示链路释放和终端复位开销。

为覆盖相邻周期内候选链路的状态变化，定义过渡集合

$$
\mathcal{E}^{\mathrm{tr}}_k
=
\mathcal{E}^{\mathrm{C}}_k
\cup
\mathcal{E}^{\mathrm{C}}_{k-1}.
$$

对 \(\{i,j\}\in\mathcal{E}^{\mathrm{tr}}_k\)，新激活变量和释放变量为

$$
u_{ij,k}
=
[x_{ij,k}-x_{ij,k-1}]^+,
\quad
v_{ij,k}
=
[x_{ij,k-1}-x_{ij,k}]^+.
$$

周期 \(k\) 的拓展链路总成本为

$$
C^{\mathrm{ext}}_k
=
\sum_{\{i,j\}\in\mathcal{E}^{\mathrm{tr}}_k}
\left(
c^{\mathrm{set}}_{ij,k}u_{ij,k}
+
c^{\mathrm{hold}}_{ij,k}x_{ij,k}
+
c^{\mathrm{off}}_{ij,k}v_{ij,k}
\right).
$$

## G. 基于 STEE 的联合优化问题

令 \(\mathcal{P}_k\) 表示周期 \(k\) 内等待调度或可能受拓展链路影响的语义单元集合，\(\pi_p^{\mathrm{B}}\) 表示仅使用基础拓扑时的路径，并定义 \(U_p^{\mathrm{B}}=U_p(\pi_p^{\mathrm{B}},\mathbf{0})\)。周期 \(k\) 的系统级语义拓扑拓展效率（Semantic-aware Topology Extension Efficiency, STEE）定义为

$$
\mathrm{STEE}_k(\mathbf{x}_k,\boldsymbol{\pi})
=
\frac{
\displaystyle
\sum_{p\in\mathcal{P}_k}
\left(
U_p(\pi_p,\mathbf{x}_k)
-
U_p^{\mathrm{B}}
\right)
}{
C^{\mathrm{ext}}_k+\epsilon
},
$$

系统联合优化拓展链路激活和逐跳路由：

$$
\max_{\{\mathbf{x}_k\},\boldsymbol{\pi}}
\quad
\mathbb{E}
\left[
\sum_k
\mathrm{STEE}_k(\mathbf{x}_k,\boldsymbol{\pi})
\right].
$$

拓展链路变量及端口约束为

$$
x_{ij,k}\in\{0,1\},
\quad
x_{ij,k}=0\ \ \forall\{i,j\}\notin\mathcal{E}^{\mathrm{C}}_k,
\quad
\sum_{j:\{i,j\}\in\mathcal{E}^{\mathrm{C}}_k}x_{ij,k}\le1.
$$

每个语义单元的每一跳只能使用对应周期内的激活链路：

$$
(i,j)\in\mathcal{E}^{\mathrm{act}}_k,
\quad
\forall(i,j,k)\in\pi_p.
$$

链路容量约束为

$$
\sum_{p\in\mathcal{P}_k:(i,j,k)\in\pi_p}
\frac{B_p}{T_t}
\le
C_{ij,k},
\quad
\forall(i,j)\in\mathcal{E}^{\mathrm{act}}_k.
$$

对需要提供服务保证的任务集合 \(\mathcal{F}^{\mathrm{adm}}\)，语义保真度约束为

$$
\overline{S}_f(\boldsymbol{\pi},\mathbf{x})
\ge
S_f^{\min},
\quad
\forall f\in\mathcal{F}^{\mathrm{adm}}.
$$

系统级 STEE 给出统一优化方向。慢时标控制器配置 E-OISL，快时标路由器在该拓扑上转发语义单元，两层决策共同决定 \(U_p\) 和 \(C_k^{\mathrm{ext}}\)。

## H. 快时标逐跳路由接口

慢时标控制器完成 E-OISL 配置后，每颗卫星在快时标上最多具有五个星间转发端口。令 \(e_i^d(k)\) 表示端口 \(d\) 在周期 \(k\) 对应的出向链路，则卫星 \(i\) 的可用动作集合为

$$
\mathcal{A}_{i,k}
=
\left\{
d\in\{\mathrm{U},\mathrm{D},\mathrm{L},\mathrm{R},\mathrm{E}\}
\mid
e_i^d(k)\in\mathcal{E}^{\mathrm{act}}_k
\right\},
$$

其中 \(\mathrm{U},\mathrm{D},\mathrm{L},\mathrm{R}\) 对应四个基础 ISL 端口，\(\mathrm{E}\) 对应当前激活的 E-OISL 端口。不可用端口通过 action mask 屏蔽；当卫星与目的网关直接连接时，语义单元通过 GSL 下行。
