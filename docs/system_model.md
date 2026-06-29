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

系统采用双时标抽象。结构时标 \(k\) 对应星座拓扑快照、星地接入、基础星间链路和拓展光链路状态的更新；快时标 \(t\) 对应语义单元的逐跳转发过程。每个结构周期开始时，集中式上层控制器根据全局或区域汇总状态配置拓展链路，并在周期内保持该配置；给定上层配置，各卫星在快时标上根据本地链路、队列和任务语义属性分布式选择下一跳。拓展链路重构周期 \(T_s\) 与基础拓扑更新周期 \(T_t\) 同步：

$$
T_s=T_t.
$$

在第 \(k\) 个结构周期内，网络表示为动态图

$$
\mathcal{H}_k
=
(\mathcal{N},\mathcal{E}^{\mathrm{act}}_k),
$$

其中 \(\mathcal{E}^{\mathrm{act}}_k\) 为采用上层配置后，周期 \(k\) 内实际用于转发的星地链路和星间链路集合。结构决策在周期内保持不变，语义单元按照快时标连续转发。

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

令 \(b_{ij,k}=\mathbf{1}\{\{i,j\}\in\mathcal{E}^{\mathrm{B}}_k\}\)。当 \(b_{ij,k}=0,x_{ij,k}=1\) 时，E-OISL 提供新的拓扑连接；当 \(b_{ij,k}=1,x_{ij,k}=1\) 时，E-OISL 作为并行链路增加该卫星对的容量。每颗卫星只有一个 E-OISL 端口，因此

$$
\sum_{j:\{i,j\}\in\mathcal{E}^{\mathrm{C}}_k}
x_{ij,k}
\le1,
\quad
\forall i\in\mathcal{S}.
$$

对于星地链路，分别令 \(C_{gi,k}=R^{\mathrm{G}}_{gi,k}\) 和 \(C_{ig,k}=R^{\mathrm{G}}_{ig,k}\)。

## D. E-OISL 并行重构时延与有效容量

由单端口约束可知，每颗卫星在一个结构周期内至多选择一个 E-OISL 对端。记卫星 \(i\) 在周期 \(k\) 选定的活动对端为

$$
\ell_i(k)
=
\begin{cases}
j,
&
x_{ij,k}=1,\\
\varnothing,
&
\displaystyle
\sum_{j:\{i,j\}\in\mathcal{E}^{\mathrm{C}}_k}
x_{ij,k}=0.
\end{cases}
$$

其中 \(\ell_i(k)=\varnothing\) 表示终端在该周期不建立 E-OISL。另记 \(h_i(k-1)\) 为周期 \(k-1\) 结束时终端的实际指向目标；即使旧链路已经释放，该实际指向仍被保留。令 \(\mathbf{u}_{i,h_i(k-1)}\) 和 \(\mathbf{u}_{ij,k}\) 分别为上一实际指向和周期 \(k\) 内由卫星 \(i\) 指向卫星 \(j\) 的单位方向向量，则终端转向新目标 \(j\) 所需角度为

$$
\Delta\theta_{i\rightarrow j,k}
=
\arccos
\left(
\mathbf{u}_{i,h_i(k-1)}^\top
\mathbf{u}_{ij,k}
\right).
$$

记 \(\omega_i^{\mathrm{slew}}\)、\(T_i^{\mathrm{acq}}\)、\(T_{ij,k}^{\mathrm{sync}}\) 和 \(T_i^{\mathrm{rel}}\) 分别为终端 \(i\) 的平均转向角速度、捕获时间、链路同步时间以及释放握手与资源释放时间。终端 \(i\) 在相邻周期之间完成状态迁移所需的墙钟时间为

$$
T_{i,k}^{\mathrm{tr}}
=
\begin{cases}
0,
&
\ell_i(k)=\ell_i(k-1),\\
T_i^{\mathrm{rel}},
&
\ell_i(k)=\varnothing,\
\ell_i(k-1)\ne\varnothing,\\
\mathbf{1}\{\ell_i(k-1)\ne\varnothing\}T_i^{\mathrm{rel}}
+
\dfrac{\Delta\theta_{i\rightarrow \ell_i(k),k}}
{\omega_i^{\mathrm{slew}}}
+
T_i^{\mathrm{acq}}
+
T_{i,\ell_i(k),k}^{\mathrm{sync}},
&
\ell_i(k)\ne\varnothing,\
\ell_i(k)\ne\ell_i(k-1).
\end{cases}
$$

保持原连接时无需重构；只释放旧连接时仅计释放时间；切换到新目标时，同一终端依次完成旧链路释放、转向、捕获和同步。若终端上一周期空闲，则不计释放时间。状态迁移完成后，终端的实际指向更新为

$$
h_i(k)
=
\begin{cases}
\ell_i(k),
&
\ell_i(k)\ne\varnothing,\\
h_i(k-1),
&
\ell_i(k)=\varnothing.
\end{cases}
$$

首个结构周期使用已知的初始活动对端 \(\ell_i(0)\) 和终端初始指向 \(h_i(0)\)。对于周期 \(k\) 内激活的 E-OISL \(\{i,j\}\)，链路须等待两端均完成状态迁移后才能提供服务，因此其可用前的等待时间为

$$
T_{ij,k}^{\mathrm{ready}}
=
\max
\left\{
T_{i,k}^{\mathrm{tr}},
T_{j,k}^{\mathrm{tr}}
\right\}.
$$

令 \([a]^+=\max\{a,0\}\)，该链路在周期内的有效服务比例为

$$
\eta^{\mathrm{E}}_{ij,k}
=
\left[
1-
\frac{T_{ij,k}^{\mathrm{ready}}}{T_s}
\right]^+.
$$

引入拓扑情形 \(r\in\{\mathrm{B},\mathrm{E}\}\)，其中 \(\mathrm{B}\) 表示仅使用基础拓扑，\(\mathrm{E}\) 表示采用给定 E-OISL 配置的拓展拓扑。两种情形下的星间有效容量分别为

$$
C^{\mathrm{B}}_{ij,k}
=
b_{ij,k}R^{\mathrm{B}}_{ij,k},
\quad
C^{\mathrm{E}}_{ij,k}
=
b_{ij,k}R^{\mathrm{B}}_{ij,k}
+
x_{ij,k}\eta^{\mathrm{E}}_{ij,k}R^{\mathrm{E}}_{ij,k}.
$$

对 GSL，令 \(C^r_{gi,k}=C_{gi,k}\) 和 \(C^r_{ig,k}=C_{ig,k}\)。情形 \(r\) 下的可用链路集合为

$$
\mathcal{E}^{\mathrm{act},r}_k
=
\mathcal{E}^{\mathrm{G}}_k
\cup
\left\{
(i,j)\in\mathcal{S}\times\mathcal{S}
\mid
i\ne j,\ C^r_{ij,k}>0
\right\}.
$$

实际运行图采用拓展情形，故 \(\mathcal{E}^{\mathrm{act}}_k\equiv\mathcal{E}^{\mathrm{act},\mathrm{E}}_k\)。由于不同卫星的 E-OISL 终端相互独立并可并行重构，全网完成本周期配置所需的墙钟时延为

$$
T_k^{\mathrm{reconf}}
=
\max_{i\in\mathcal{S}}
T_{i,k}^{\mathrm{tr}}.
$$

## E. 链路可靠性与时延

本节定义适用于任意给定网络状态的通用单跳性能模型。记 \(Q_{ij,k}\)、\(Q_{ij}^{\max}\) 和 \(A_{ij,k}\) 分别为周期 \(k\) 开始时链路 \((i,j)\) 的出向队列数据量、队列容量和周期内到达数据量，三者均以 bit 计；\(C_{ij,k}\) 为该网络状态下的链路容量。语义单元 \(p\) 的长度为 \(B_p\)，链路误比特率为 \(\varepsilon^{\mathrm{bit}}_{ij,k}\)，则单次传输的信道失败概率为

$$
p^{\mathrm{ch}}_{p,ij,k}
=
1-
\left(
1-\varepsilon^{\mathrm{bit}}_{ij,k}
\right)^{B_p}.
$$

令 \(\Pi_{[0,1]}(\cdot)\) 表示截断到 \([0,1]\)。队列溢出造成的丢弃概率为

$$
p^{\mathrm{q}}_{ij,k}
=
\begin{cases}
\displaystyle
\Pi_{[0,1]}
\left(
\frac{
\left[
Q_{ij,k}
+
A_{ij,k}
-
C_{ij,k}T_t
-
Q_{ij}^{\max}
\right]^+
}{
A_{ij,k}
}
\right),
&
A_{ij,k}>0,\\[3mm]
0,
&
A_{ij,k}=0.
\end{cases}
$$

对纳入可用链路集合的链路，假设 \(p^{\mathrm{ch}}_{p,ij,k}<1\)。设链路最多允许 \(M_{ij}\) 次传输，则期望传输次数和单跳成功概率分别为

$$
\overline{M}_{p,ij,k}
=
\frac{
1-
\left(
p^{\mathrm{ch}}_{p,ij,k}
\right)^{M_{ij}}
}{
1-p^{\mathrm{ch}}_{p,ij,k}
},
\quad
q_{p,ij,k}
=
\left(
1-p^{\mathrm{q}}_{ij,k}
\right)
\left[
1-
\left(
p^{\mathrm{ch}}_{p,ij,k}
\right)^{M_{ij}}
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
\frac{B_p}{C_{ij,k}}
+
\frac{d_{ij,k}}{c}
\right).
$$

## F. 分情形任务语义效用

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

其中 \(g_f,d_f\in\mathcal{G}\) 为源网关和目的网关，\(\mathcal{P}_f\) 为任务的语义单元集合，\(\tau_f>0\) 为任务允许的最大端到端时延，\(\beta_f\) 为超时衰减系数，\(S_f^{\min}\) 为最低任务语义保真度，\(t_p^0\) 为语义单元生成时刻。\(w_p\in[0,1]\) 表示语义单元 \(p\) 对任务恢复质量的边际语义贡献，并满足

$$
\sum_{p\in\mathcal{P}_f}w_p=1.
$$

\(w_p\) 可由任务性能下降量、语义相似度变化或语义编解码器的贡献评估模块给出。本文采用语义贡献可加的一阶近似。

基础环境和拓展环境使用相同的业务输入，并分别依据 \(C^{\mathrm{B}}_{ij,k}\) 和 \(C^{\mathrm{E}}_{ij,k}\) 演化链路队列。对 \(r\in\{\mathrm{B},\mathrm{E}\}\)，将 E 节通用单跳模型中的 \(C_{ij,k}\)、\(Q_{ij,k}\)、\(A_{ij,k}\)、\(\varepsilon^{\mathrm{bit}}_{ij,k}\) 和 \(T^{\mathrm{q}}_{ij,k}\) 分别替换为情形 \(r\) 对应的 \(C^r_{ij,k}\)、\(Q^r_{ij,k}\)、\(A^r_{ij,k}\)、\(\varepsilon^{\mathrm{bit},r}_{ij,k}\) 和 \(T^{\mathrm{q},r}_{ij,k}\)，所得单跳成功概率和期望时延记为 \(q^r_{p,ij,k}\) 和 \(\widetilde{D}^{\,r}_{p,ij,k}\)。对情形 \(r\) 下由有序链路组成的路径 \(\pi_p^r\)，端到端期望时延和成功概率为

$$
T_p^r(\pi_p^r)
=
\sum_{(i,j,k)\in\pi_p^r}
\widetilde{D}^{\,r}_{p,ij,k},
\quad
P_p^{\mathrm{succ},r}(\pi_p^r)
=
\prod_{(i,j,k)\in\pi_p^r}
q^r_{p,ij,k}.
$$

因此，情形 \(r\) 下任务 \(f\) 的期望语义保真度定义为

$$
\overline{S}_f^r(\boldsymbol{\pi}^r)
=
\sum_{p\in\mathcal{P}_f}
w_p
P_p^{\mathrm{succ},r}(\pi_p^r)
\mathbf{1}
\left\{
T_p^r(\pi_p^r)\le\tau_f
\right\},
$$

其中 \(\boldsymbol{\pi}^r=\{\pi_p^r\}\) 为情形 \(r\) 下的路径决策集合。为反映高语义贡献单元随截止时间临近而增加的调度需求，定义语义时效压力

$$
\Psi_p(t)
=
w_p
\left[
1+
\min
\left\{
1,
\frac{[t-t_p^0]^+}{\tau_f}
\right\}
\right].
$$

相应地，语义单元 \(p\) 在时刻 \(t\) 的剩余时延预算为

$$
\Delta_p(t)
=
\left[
\tau_f-(t-t_p^0)
\right]^+.
$$

在 \(t=t_p^0\) 时，\(\Psi_p(t)=w_p\) 且 \(\Delta_p(t)=\tau_f\)；等待时间达到或超过 \(\tau_f\) 后，\(\Psi_p(t)\) 截断为 \(2w_p\)，剩余时延预算降为 0。二者作为调度状态特征，不替代最终的及时语义效用。

语义单元 \(p\) 在情形 \(r\) 下的期望及时语义效用统一定义为

$$
U_p^r(\pi_p^r)
=
w_p
P_p^{\mathrm{succ},r}(\pi_p^r)
\exp
\left(
-\beta_f
\left[
T_p^r(\pi_p^r)-\tau_f
\right]^+
\right).
$$

因此，基础情形与拓展情形共享业务集合、语义权重和链路可靠性模型，仅可用拓扑、有效容量和据此产生的路径不同。该效用同时刻画语义贡献、可靠交付和时效要求。

## G. 基于 STEE 的层次优化问题

令 \(\mathcal{P}_k\) 表示周期 \(k\) 内等待调度或可能受拓展链路影响的语义单元集合。满足候选链路和单端口约束的 E-OISL 匹配构成上层可行动作集合

$$
\mathcal{X}_k
=
\left\{
\mathbf{x}_k
\ \middle|\
\begin{aligned}
&x_{ij,k}\in\{0,1\},\\
&x_{ij,k}=0,
\quad
\forall\{i,j\}\notin\mathcal{E}^{\mathrm{C}}_k,\\
&\sum_{j:\{i,j\}\in\mathcal{E}^{\mathrm{C}}_k}
x_{ij,k}\le1,
\quad
\forall i\in\mathcal{S}
\end{aligned}
\right\}.
$$

记 \(\Pi_k^r\) 为情形 \(r\) 下的可行路由策略集合。任意 \(\boldsymbol{\pi}^r\in\Pi_k^r\) 均须沿情形 \(r\) 的激活链路转发，并满足链路容量约束：

$$
(i,j)\in\mathcal{E}^{\mathrm{act},r}_k,
\quad
\forall(i,j,k)\in\pi_p^r,
\qquad
\sum_{p\in\mathcal{P}_k:(i,j,k)\in\pi_p^r}
\frac{B_p}{T_t}
\le
C^r_{ij,k},
\quad
\forall(i,j)\in\mathcal{E}^{\mathrm{act},r}_k.
$$

对需要服务保证的任务集合 \(\mathcal{F}^{\mathrm{adm}}\)，可行路由还应满足

$$
\overline{S}_f^r(\boldsymbol{\pi}^r)
\ge
S_f^{\min},
\quad
\forall f\in\mathcal{F}^{\mathrm{adm}}.
$$

基础情形采用当前基础拓扑上的语义效用最优路径作为强基准。令 \(\Pi_{p,k}^{\mathrm{B}}\) 为语义单元 \(p\) 的基础可行路径集合，则

$$
\pi_p^{\mathrm{B},*}
=
\arg\max_{\pi\in\Pi_{p,k}^{\mathrm{B}}}
U_p^{\mathrm{B}}(\pi).
$$

给定上层 E-OISL 配置 \(\mathbf{x}_k\)，快时标下层的理论最优响应为

$$
\boldsymbol{\pi}_k^{\mathrm{E},*}(\mathbf{x}_k)
=
\arg\max_{\boldsymbol{\pi}^{\mathrm{E}}\in\Pi_k^{\mathrm{E}}(\mathbf{x}_k)}
\sum_{p\in\mathcal{P}_k}
U_p^{\mathrm{E}}(\pi_p^{\mathrm{E}}).
$$

其中 \(\pi_p^{\mathrm{E},*}(\mathbf{x}_k)\) 表示该最优路由策略中语义单元 \(p\) 的路径。周期 \(k\) 的系统级语义拓扑拓展效率（Semantic-aware Topology Extension Efficiency, STEE）定义为归一化重构时延惩罚下的语义效用增益：

$$
\mathrm{STEE}_k
\left(
\mathbf{x}_k,
\boldsymbol{\pi}_k^{\mathrm{E},*}(\mathbf{x}_k)
\right)
=
\frac{
\displaystyle
\sum_{p\in\mathcal{P}_k}
\left[
U_p^{\mathrm{E}}
\left(
\pi_p^{\mathrm{E},*}(\mathbf{x}_k)
\right)
-
U_p^{\mathrm{B}}
\left(
\pi_p^{\mathrm{B},*}
\right)
\right]
}{
1+T_k^{\mathrm{reconf}}/T_s
}.
$$

当本周期保持上一周期的 E-OISL 配置时，\(T_k^{\mathrm{reconf}}=0\)，STEE 分母为 1；发生拓扑重构时，较长的全网完成时延将降低相同语义效用增益对应的 STEE。

集中式慢时标上层据此求解

$$
\mathbf{x}_k^*
=
\arg\max_{\mathbf{x}_k\in\mathcal{X}_k}
\mathrm{STEE}_k
\left(
\mathbf{x}_k,
\boldsymbol{\pi}_k^{\mathrm{E},*}(\mathbf{x}_k)
\right).
$$

对于固定的 \(\mathbf{x}_k\)，\(T_k^{\mathrm{reconf}}\) 和基础效用项均与下层逐跳动作无关，因此下层最大化拓展情形的总语义效用，与提高该拓扑配置下的 STEE 一致。该嵌套结构将拓扑配置和逐跳路由分别置于慢、快时标，避免将两个时标写成一次性联合动作。

## H. 集中式上层与分布式下层接口

令 \(t_k=kT_s\) 表示结构周期 \(k\) 的起始时刻。慢时标上层在每个结构周期执行一次集中决策。首先定义基础 ISL 邻接矩阵、基础 ISL 速率矩阵、GSL 接入矩阵和上一周期 E-OISL 配置矩阵：

$$
\mathbf{B}_k
=
\left[b_{ij,k}\right]_{|\mathcal{S}|\times|\mathcal{S}|},
\quad
\mathbf{R}^{\mathrm{B}}_k
=
\left[R^{\mathrm{B}}_{ij,k}\right]_{|\mathcal{S}|\times|\mathcal{S}|},
\quad
\mathbf{Z}_k
=
\left[z_{gi,k}\right]_{|\mathcal{G}|\times|\mathcal{S}|},
\quad
\mathbf{X}_{k-1}
=
\left[x_{ij,k-1}\right]_{|\mathcal{S}|\times|\mathcal{S}|}.
$$

对任意卫星对 \(i<j\)，定义候选 E-OISL 特征

$$
\widehat{T}_{ij,k}^{\mathrm{ready}}
=
T_{ij,k}^{\mathrm{ready}}
\big|_{\ell_i(k)=j,\ \ell_j(k)=i},
\quad
\{i,j\}\in\mathcal E_k^{\mathrm C},
\quad
\mathbf{f}_{ij,k}^{\mathrm C}
=
\begin{cases}
\left(
\chi^{\mathrm I}_{ij,k},
R^{\mathrm E}_{ij,k},
\frac{\widehat{T}_{ij,k}^{\mathrm{ready}}}{T_s}
\right),
&
\{i,j\}\in\mathcal E_k^{\mathrm C},\\[2mm]
\mathbf 0,
&
\{i,j\}\notin\mathcal E_k^{\mathrm C},
\end{cases}
\quad
\mathbf{F}^{\mathrm C}_k
=
\left\{
\mathbf{f}_{ij,k}^{\mathrm C}
\right\}_{i<j}.
$$

对候选卫星对，\(\widehat{T}_{ij,k}^{\mathrm{ready}}\) 表示假设本周期选择 \(\{i,j\}\) 时，按照 D 节终端迁移模型计算的链路就绪时间；非候选卫星对的特征置零。因此 \(\mathbf F_k^{\mathrm C}\) 是固定维度的候选特征张量，同时包含可行性、光链路速率和归一化 PAT 重构代价。

令 \(\mathcal P_{i,g}(t_k)\) 表示在 \(t_k\) 时刻缓存于卫星 \(i\)、尚待转发且目的网关为 \(g\) 的语义单元集合。按卫星和目的网关汇总的待传数据量及平均语义压力定义为

$$
\overline Q_{ig,k}^{\mathrm H}
=
\sum_{p\in\mathcal P_{i,g}(t_k)}
B_p,
\qquad
\overline\Psi_{ig,k}^{\mathrm H}
=
\begin{cases}
\displaystyle
\frac{
\sum_{p\in\mathcal P_{i,g}(t_k)}
B_p\Psi_p(t_k)
}{
\overline Q_{ig,k}^{\mathrm H}
},
&
\overline Q_{ig,k}^{\mathrm H}>0,\\[3mm]
0,
&
\overline Q_{ig,k}^{\mathrm H}=0.
\end{cases}
$$

记 \(\overline{\mathbf Q}_k^{\mathrm H}=[\overline Q_{ig,k}^{\mathrm H}]\) 和 \(\overline{\boldsymbol\Psi}_k^{\mathrm H}=[\overline\Psi_{ig,k}^{\mathrm H}]\)，二者的维度均为 \(|\mathcal S|\times|\mathcal G|\)。上层状态、动作和周期奖励正式定义为

$$
s_k^{\mathrm H}
=
\left(
\mathbf B_k,
\mathbf R_k^{\mathrm B},
\mathbf Z_k,
\mathbf F_k^{\mathrm C},
\overline{\mathbf Q}_k^{\mathrm H},
\overline{\boldsymbol\Psi}_k^{\mathrm H},
\mathbf X_{k-1}
\right),
\quad
a_k^{\mathrm H}
=
\mathbf x_k\in\mathcal X_k,
\quad
r_k^{\mathrm H}
=
\mathrm{STEE}_k.
$$

目的网关条件汇总使上层能够区分数据量相同但流向不同的业务。上层动作在整个 \(T_s\) 内保持不变。连续特征的归一化、组合动作的编码方式和上层策略网络类型属于算法设计，将在算法章节中确定。

给定 \(\mathbf{x}_k\)，各卫星基于本地观测执行分布式 DDQN 逐跳路由。每颗卫星在快时标上最多具有五个星间转发端口。令 \(e_i^d(k)\) 表示端口 \(d\) 在周期 \(k\) 对应的出向链路，则卫星 \(i\) 的可用动作集合为

$$
\mathcal D
=
\left\{
\mathrm U,\mathrm D,\mathrm L,\mathrm R,\mathrm E
\right\},
\quad
j_i^d(k)
\in
\mathcal S\cup\{\varnothing\},
\quad
e_i^d(k)
=
\begin{cases}
\left(i,j_i^d(k)\right),
&
j_i^d(k)\ne\varnothing,\\
\varnothing,
&
j_i^d(k)=\varnothing,
\end{cases}
\quad
m_{i,d,k}
=
\mathbf 1
\left\{
e_i^d(k)\in\mathcal E_k^{\mathrm{act},\mathrm E}
\right\}.
$$

其中 \(j_i^d(k)\) 为端口 \(d\) 当前连接的邻居；端口未连接时 \(j_i^d(k)=\varnothing\)，\(m_{i,d,k}=0\)。相应的可用动作集合为

$$
\mathcal{A}_{i,k}
=
\left\{
d\in\mathcal D
\ \middle|\
m_{i,d,k}=1
\right\}.
$$

其中 \(\mathrm{U},\mathrm{D},\mathrm{L},\mathrm{R}\) 对应四个基础 ISL 端口，\(\mathrm{E}\) 对应当前激活的 E-OISL 端口。不可用端口通过 action mask 屏蔽；当卫星与目的网关直接连接时，语义单元通过 GSL 下行。下层观测仅包含本地邻接关系、链路与队列状态、目的节点和当前语义时效压力。

为刻画一跳下游拥塞，令 \(Q_{ij}(t)\) 表示快时标时刻 \(t\) 链路 \((i,j)\) 的当前出向队列数据量，并令 \(Q_{ij,k}=Q_{ij}(t_k)\)。卫星 \(j\) 向一跳邻居广播的归一化出向队列向量定义为

$$
\left[
\mathbf q_j(t)
\right]_{d'}
=
\begin{cases}
\displaystyle
\frac{
Q_{j,j_j^{d'}(k)}(t)
}{
Q_{j,j_j^{d'}(k)}^{\max}
},
&
m_{j,d',k}=1,\\[3mm]
0,
&
m_{j,d',k}=0,
\end{cases}
\quad
d'\in\mathcal D.
$$

定义相对位置向量 \(\boldsymbol\rho_{ij,k}=\mathbf r_j(k)-\mathbf r_i(k)\)。对于可用端口 \(d\) 及其邻居 \(j=j_i^d(k)\)，端口特征由 action mask、链路质量、本地出向队列、邻居队列和相对位置组成：

$$
\boldsymbol\phi_{i,d}(t)
=
\begin{cases}
\left(
m_{i,d,k},
C^{\mathrm E}_{ij,k},
\varepsilon^{\mathrm{bit},\mathrm E}_{ij,k},
d_{ij,k},
\dfrac{Q_{ij}(t)}{Q_{ij}^{\max}},
\mathbf q_j(t),
\boldsymbol\rho_{ij,k}
\right),
&
m_{i,d,k}=1,\\[3mm]
\mathbf 0,
&
m_{i,d,k}=0.
\end{cases}
$$

由网关接入约束可知，目的网关 \(d_f\) 在周期 \(k\) 唯一接入一颗卫星。记该卫星为 \(i_f^{\mathrm{dst}}(k)\)，即 \(z_{d_f,i_f^{\mathrm{dst}}(k),k}=1\)。语义单元特征和卫星 \(i\) 针对语义单元 \(p\) 的局部观测分别定义为

$$
\boldsymbol\sigma_p(t)
=
\left(
B_p,
w_p,
\beta_f,
\frac{\Delta_p(t)}{\tau_f},
\Psi_p(t)
\right),
\quad
o_{i,p}^{\mathrm L}(t)
=
\left(
\mathbf r_i(k),
\boldsymbol\rho_{i,i_f^{\mathrm{dst}}(k),k},
\left\{
\boldsymbol\phi_{i,d}(t)
\right\}_{d\in\mathcal D},
\boldsymbol\sigma_p(t)
\right).
$$

该观测只依赖卫星自身信息、可由一跳邻居交换的信息和当前语义单元属性，不包含全网瞬时状态。各连续特征在送入 DDQN 前的数值归一化方式属于算法实现，不在系统模型中固定。

令 \(T_p^{\mathrm{real}}\) 表示一次传输中的实际端到端时延，\(\mathsf{Succ}_p\in\{0,1\}\) 表示实际交付结果，则语义单元的终止回报定义为

$$
R_p^{\mathrm{L}}
=
\mathbf{1}\{\mathsf{Succ}_p=1\}
w_p
\exp
\left(
-\beta_f
\left[
T_p^{\mathrm{real}}-\tau_f
\right]^+
\right),
\quad
\mathbb{E}
\left[
R_p^{\mathrm{L}}
\mid
\pi_p^{\mathrm{E}}
\right]
=
U_p^{\mathrm{E}}(\pi_p^{\mathrm{E}}).
$$

在采用 F 节端到端期望时延近似，即以 \(T_p^{\mathrm{E}}\) 代替随机实现 \(T_p^{\mathrm{real}}\) 计算衰减项时，上式的期望等于 \(U_p^{\mathrm{E}}\)。因此，分布式下层学习目标与 G 节的内层语义效用目标保持一致。用于加速训练的中间奖励塑形不属于系统模型，将在算法章节中单独定义。
