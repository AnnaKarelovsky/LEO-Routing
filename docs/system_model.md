# System Model 初稿

本文档基于 `03_代码框架.pdf` 的 System Model、`新思路.md` 的研究设定，以及当前代码实现中的真实网络抽象整理而成。内容分为五部分：

1. 可直接放入论文的 System Model 初稿；
2. 需要后续确认或补充的信息；
3. 原论文/代码实现中值得沿用的建模点；
4. 根据新思路应修改或新增的建模点。
5. 代码实现对应关系。

## 1. 可直接放入论文的 System Model 初稿

### A. 系统概述与双时标抽象

本文考虑由低轨卫星星座、地面网关和时变业务流构成的星地一体网络。卫星集合记为

$$
\mathcal{S}=\{s_{o,n}\mid o\in\{1,\ldots,O\}, n\in\{1,\ldots,N_o\}\},
$$

其中 \(o\) 表示轨道面编号，\(n\) 表示轨道面内卫星编号。地面网关集合记为 \(\mathcal{G}\)，系统节点集合为

$$
\mathcal{N}=\mathcal{S}\cup\mathcal{G}.
$$

本文采用双时标系统抽象。快时标对应数据包级转发过程，即当业务包到达某颗卫星时，卫星基于当前可用邻居、队列状态和业务语义属性选择下一跳，该过程由包到达和逐跳转发事件触发，记为时刻 \(t\)。慢时标对应网络结构级决策，即星座拓扑快照、星地接入关系、基础链路状态以及拓展链路激活状态的更新，离散索引记为 \(k\)，拓扑更新周期为 \(T_t\)。

拓展链路依赖卫星间物理可见性和链路预算。由于这些条件会随星座拓扑快照变化而改变，本文令拓展链路重构周期与基础拓扑更新周期同步，即

$$
T_s=T_t.
$$

为避免符号混淆，本文采用如下上标约定：\(\mathrm{G}\) 表示星地链路或网关相关量，\(\mathrm{I}\) 表示星间物理可达性，\(\mathrm{B}\) 表示基础星间链路资源，\(\mathrm{E}\) 表示拓展链路资源，\(\mathrm{C}\) 表示候选集合，\(\mathrm{act}\) 表示当前实际可用于转发的链路集合。相应地，\(\chi\) 类变量表示物理或链路预算可行性，\(z\) 和 \(x\) 类变量表示实际决策，\(C\) 类变量表示决策后的有效容量。

在第 \(k\) 个拓扑周期内，网络图记为

$$
\mathcal{H}_k=(\mathcal{N},\mathcal{E}^{\mathrm{act}}_k),
$$

\(\mathcal{E}^{\mathrm{act}}_k\) 为当前可用于转发的星间链路和星地链路集合。拓展链路变量在周期 \(k\) 内保持不变；在该周期内部，数据包仍按照快时标 \(t\) 逐跳转发。该部分仅给出系统建模，具体学习算法在后续章节展开。

### B. LEO 星座、网关接入与链路可达性

每个轨道面 \(o\) 被近似为圆轨道，其轨道高度为 \(h_o\)，轨道倾角为 \(\delta_o\)，升交点经度或轨道面初始经度为 \(\epsilon_o\)。卫星 \(s_{o,n}\) 在轨道面内均匀分布，其轨道周期可表示为

$$
T_o = 2\pi\sqrt{\frac{(R_E+h_o)^3}{\mu}},
$$

其中 \(R_E\) 为地球半径，\(\mu\) 为地球引力常数。记卫星 \(i\in\mathcal{S}\) 在周期 \(k\) 的三维位置为 \(\mathbf{r}_i(k)=(x_i(k),y_i(k),z_i(k))\)，其经纬度为 \((\phi_i(k),\lambda_i(k))\)。

地面网关 \(g\in\mathcal{G}\) 位于固定地理位置 \((\phi_g,\lambda_g)\)。网关接入卫星必须同时满足可见性约束和分配约束。定义

$$
\chi^{\mathrm{G}}_{g i,k}
=
\mathbf{1}\{\theta_{g i,k}\ge \theta_{\min},\ d_{g i,k}\le d^{\mathrm{G}}_{\max},\ R^{\mathrm{G}}_{g i,k}>0\},
$$

其中 \(\theta_{g i,k}\) 为网关 \(g\) 在周期 \(k\) 相对卫星 \(i\) 的仰角，\(d_{g i,k}\) 为星地距离，\(R^{\mathrm{G}}_{g i,k}\) 为星地链路速率，\(\theta_{\min}\) 和 \(d^{\mathrm{G}}_{\max}\) 分别为最小仰角和最大星地通信距离。变量 \(\chi^{\mathrm{G}}_{g i,k}\) 为星地接入的物理可行性指示量：\(\chi^{\mathrm{G}}_{g i,k}=1\) 表示网关 \(g\) 可以被卫星 \(i\) 服务，但并不表示二者一定被分配连接。实际接入决策由二元变量 \(z_{g i,k}\) 给出：

$$
z_{g i,k}=
\begin{cases}
1, & \text{若网关 }g\text{ 在周期 }k\text{ 接入卫星 }i,\\
0, & \text{否则},
\end{cases}
$$

则网关接入约束为

$$
\sum_{i\in\mathcal{S}}z_{g i,k}=1,\quad
z_{g i,k}\le \chi^{\mathrm{G}}_{g i,k},\quad
\sum_{g\in\mathcal{G}}z_{g i,k}\le 1,
$$

上述三个约束分别具有如下含义：第一项 \(\sum_{i\in\mathcal{S}}z_{g i,k}=1\) 表示每个网关 \(g\) 在周期 \(k\) 有且仅有一个接入卫星；第二项 \(z_{g i,k}\le \chi^{\mathrm{G}}_{g i,k}\) 表示只有满足仰角、距离和链路预算约束的可见卫星才能被分配给该网关；第三项 \(\sum_{g\in\mathcal{G}}z_{g i,k}\le 1\) 表示卫星侧的星地接入资源约束，即同一颗卫星在同一周期内最多服务一个网关，从而避免超过其星地链路终端、波束或调度能力。

由 \(z_{g i,k}\) 可得到网关 \(g\) 在周期 \(k\) 的实际接入卫星 \(a_k(g)\)，即唯一满足 \(z_{g i,k}=1\) 的卫星 \(i\)。

若采用最近可见卫星接入作为具体分配规则，则可进一步令

$$
a_k(g)=\arg\min_{i\in\mathcal{S}:\chi^{\mathrm{G}}_{g i,k}=1} d_{g i,k}.
$$

星地链路集合为

$$
\mathcal{E}^{\mathrm{G}}_k
=
\{(g,i),(i,g)\mid g\in\mathcal{G}, i\in\mathcal{S}, z_{g i,k}=1\}.
$$

其中 \(\mathcal{E}^{\mathrm{G}}_k\) 是完成网关分配后的实际星地链路集合，而不是全部可见星地候选集合。集合中同时写入 \((g,i)\) 和 \((i,g)\)，表示数据可在建模中按上行和下行两个方向使用该星地接入关系。

可调整假设：当前代码中，GSL 分配由 `linkSats2GTs("Optimize")` 完成，并在小星座实验中对可见距离阈值做了放宽处理；论文正文建议采用上述标准可见性与分配约束，仿真设置中再说明实现细节。

为后续链路集合定义，先给出链路几何距离、可达性与速率。记节点 \(i,j\in\mathcal{N}\) 在周期 \(k\) 的空间距离为

$$
d_{ij,k}=\|\mathbf{r}_i(k)-\mathbf{r}_j(k)\|.
$$

若节点 \(i\) 与 \(j\) 满足地球遮挡、最大可达距离和链路预算约束，则认为二者在周期 \(k\) 物理可达。其可用速率由当前 SNR 和调制编码门限决定：

$$
R_{ij,k}
=
W\max_{\varpi\in\mathcal{M}}
\left\{
\varpi:
\mathrm{SNR}_{ij,k}\ge \mathrm{SNR}_{\min}(\varpi)
\right\},
$$

其中 \(W\) 为链路带宽，\(\mathcal{M}\) 为可用频谱效率集合，\(\varpi\) 为候选频谱效率，\(\mathrm{SNR}_{\min}(\varpi)\) 表示频谱效率 \(\varpi\) 对应的最小可靠通信 SNR。若节点对不可达或链路预算不满足，则令 \(R_{ij,k}=0\)。

### C. 基础链路、候选拓展链路与已激活链路

本文将星间链路资源划分为基础链路资源与拓展链路资源。

本文考虑每颗卫星配置四个基础 ISL 终端和一个额外可重构通信终端。四个基础终端用于维持同轨上/下邻居和跨轨左/右邻居；额外可重构终端在同一拓扑周期内只能承担一种任务：要么用于星地服务，要么用于建立一条拓展 ISL。也就是说，已经接入地面网关的卫星不再作为拓展 ISL 的候选端点。为刻画卫星 \(i\) 的额外终端是否已被星地服务占用，根据前文网关接入变量 \(z_{g i,k}\) 定义

$$
y^{\mathrm{G}}_{i,k}
=
\sum_{g\in\mathcal{G}}z_{g i,k}.
$$

其中，\(z_{g i,k}\) 区分具体“哪个网关接入哪颗卫星”；\(y^{\mathrm{G}}_{i,k}\) 是卫星侧汇总占用变量，只关心卫星 \(i\) 是否已经服务某个网关。由于前文约束 \(\sum_{g\in\mathcal{G}}z_{g i,k}\le 1\)，故 \(y^{\mathrm{G}}_{i,k}\in\{0,1\}\)：当 \(y^{\mathrm{G}}_{i,k}=1\) 时，卫星 \(i\) 的额外可重构终端已被星地服务占用；当 \(y^{\mathrm{G}}_{i,k}=0\) 时，该终端可作为拓展 ISL 候选资源。

基础稳定链路集合记为 \(\mathcal{E}^{\mathrm{B}}_k\)。该集合包括同轨道面内相邻卫星之间的前向/后向链路，以及由当前星座几何关系选择的东西向跨轨链路。对卫星 \(i\)，基础邻居集合记为

$$
\mathcal{N}^{\mathrm{B}}_i(k)
=
\{i^{\mathrm{up}}, i^{\mathrm{down}}, i^{\mathrm{right}}, i^{\mathrm{left}}\},
$$

其中 \(i^{\mathrm{up}}\) 和 \(i^{\mathrm{down}}\) 表示同轨道面内相邻卫星，\(i^{\mathrm{right}}\) 和 \(i^{\mathrm{left}}\) 表示跨轨方向上的邻近卫星。若某方向上不存在可行链路，则对应邻居为空。\(\mathcal{E}^{\mathrm{B}}_k\) 表示由四个基础 ISL 终端维持的默认转发骨干，其链路随星座运动和跨轨匹配结果在周期 \(k\) 更新。

候选拓展链路不是简单定义为基础拓扑之外的边，而是由额外可重构通信终端提供的临时链路资源。若两颗卫星 \(i,j\in\mathcal{S}\) 在周期 \(k\) 满足物理可见、最大距离和链路预算约束，且两端额外终端均未被星地服务占用，则它们可以成为候选拓展端点。定义

$$
\chi^{\mathrm{I}}_{ij,k}
=
\mathbf{1}\{i,j\in\mathcal{S}, i\ne j,\ d_{ij,k}\le d^{\mathrm{I}}_{\max},\ R^{\mathrm{E}}_{ij,k}>0\},
$$

其中 \(d^{\mathrm{I}}_{\max}\) 为拓展 ISL 的最大可达距离，\(R^{\mathrm{E}}_{ij,k}\) 表示使用拓展终端建立链路可获得的速率。变量 \(\chi^{\mathrm{I}}_{ij,k}\) 是候选拓展 ISL 的物理可行性指示量：\(\chi^{\mathrm{I}}_{ij,k}=1\) 表示卫星对 \((i,j)\) 在周期 \(k\) 具备建立拓展链路的物理条件。结合额外终端的互斥使用逻辑，周期 \(k\) 内真正可被拓展控制器选择的候选拓展集合定义为

$$
\mathcal{E}^{\mathrm{C}}_k
=
\{(i,j)\mid i,j\in\mathcal{S}, i\ne j,\ \chi^{\mathrm{I}}_{ij,k}=1,\ y^{\mathrm{G}}_{i,k}=0,\ y^{\mathrm{G}}_{j,k}=0\}.
$$

因此，只有在当前周期同时满足物理可达性、链路预算约束以及两端额外终端未被星地服务占用的卫星对，才进入候选拓展链路集合。这里的 \(\mathcal{E}^{\mathrm{C}}_k\) 已经完成 GSL 占用端点的剔除，表示后续优化模型可直接选择的额外链路资源集合；若基础链路已存在，额外拓展终端仍可作为并行资源提升该卫星对的有效容量。对每条候选拓展链路定义激活变量

$$
x_{ij,k}=
\begin{cases}
1, & \text{若候选拓展链路 }(i,j)\text{ 在周期 }k\text{ 被激活},\\
0, & \text{否则}.
\end{cases}
$$

其中 \(x_{ij,k}\) 是拓展链路的实际激活决策变量。只有当 \((i,j)\in\mathcal{E}^{\mathrm{C}}_k\) 且 \(x_{ij,k}=1\) 时，拓展终端才真正占用两端卫星资源并参与后续容量计算。令

$$
x_{ij,k}=0,\quad \forall (i,j)\notin\mathcal{E}^{\mathrm{C}}_k.
$$

该约定表示非候选卫星对在周期 \(k\) 不能被激活。由于 \(\mathcal{E}^{\mathrm{C}}_k\) 已经剔除了物理不可达链路和额外终端被星地服务占用的端点，后续优化模型不再额外添加星地占用相关约束。

$$
b_{ij,k}=\mathbf{1}\{(i,j)\in\mathcal{E}^{\mathrm{B}}_k\},
$$

其中 \(b_{ij,k}\) 是基础链路存在性指示量：\(b_{ij,k}=1\) 表示卫星对 \((i,j)\) 在基础拓扑中已经存在四方向 ISL，\(b_{ij,k}=0\) 表示二者之间没有基础 ISL。则卫星对 \((i,j)\) 在周期 \(k\) 内的有效星间容量为

$$
C_{ij,k}
=
b_{ij,k}R^{\mathrm{B}}_{ij,k}
+
x_{ij,k}R^{\mathrm{E}}_{ij,k}.
$$

其中 \(R^{\mathrm{B}}_{ij,k}\) 为基础 ISL 速率，\(R^{\mathrm{E}}_{ij,k}\) 为额外拓展终端提供的链路速率，\(C_{ij,k}\) 为完成基础链路选择和拓展链路激活后的总可用星间容量。若 \(b_{ij,k}=0,x_{ij,k}=1\)，拓展链路表现为新增连接；若 \(b_{ij,k}=1,x_{ij,k}=1\)，拓展链路表现为并行传输或容量增强。

由于 \(\mathcal{E}^{\mathrm{C}}_k\) 已经排除了额外终端被 GSL 占用的卫星，后续只需限制同一颗卫星不能同时激活多条拓展 ISL，即拓展链路互斥约束为

$$
\sum_{j:(i,j)\in\mathcal{E}^{\mathrm{C}}_k} x_{ij,k}
\le 1,\quad \forall i\in\mathcal{S}.
$$

该约束表示卫星 \(i\) 在周期 \(k\) 内最多参与一条已激活拓展 ISL，右端的 1 表示每颗卫星只有一个额外可重构终端。若实现中将一条双向物理拓展链路存储为两个有向边，则该求和应按物理端点去重，避免同一条链路在资源约束中被重复计数。

当前可用于转发的链路集合为

$$
\mathcal{E}^{\mathrm{act}}_k
=
\mathcal{E}^{\mathrm{G}}_k
\cup
\{(i,j)\mid i,j\in\mathcal{S}, C_{ij,k}>0\}.
$$

其中 \(\mathcal{E}^{\mathrm{act}}_k\) 是路由层在周期 \(k\) 真正可以使用的边集合，由已分配星地链路和容量为正的星间链路共同组成。对于星地链路，可在后续时延与容量约束中令 \(C_{g i,k}=C_{i g,k}=R^{\mathrm{G}}_{g i,k}\)。

可调整假设：若星地链路由独立 RF/Ka 终端承担，则可令 \(y^{\mathrm{G}}_{i,k}=0\)，表示额外光终端不受星地服务占用；若存在多个额外可重构终端，可将候选端点条件 \(y^{\mathrm{G}}_{i,k}=0\) 放宽为 \(y^{\mathrm{G}}_{i,k}<L_i^{\mathrm{E}}\)，并将拓展链路互斥约束右端由 1 替换为可用额外终端数。

### D. 链路速率与单跳时延模型

基于上述 \(d_{ij,k}\) 与 \(C_{ij,k}\)，定义链路拥塞、丢包和单跳时延。对链路 \((i,j)\)，记 \(Q_{ij,k}\) 为周期 \(k\) 开始时节点 \(i\) 指向节点 \(j\) 的出向队列长度，\(Q_{ij}^{\max}\) 为该队列容量，\(A_{ij,k}\) 为周期 \(k\) 内到达该队列的数据量，\(F_{ij,k}\) 为周期 \(k\) 内链路实际承载流量。定义链路拥塞压力

$$
\Omega_{ij,k}
=
\omega_q\frac{Q_{ij,k}}{Q_{ij}^{\max}}
+
\omega_f\frac{F_{ij,k}}{C_{ij,k}+\epsilon},
$$

其中 \(\omega_q,\omega_f\ge 0\) 为队列占用和负载强度的权重，\(\epsilon>0\) 用于避免分母为零。\(\Omega_{ij,k}\) 越大，表示链路 \((i,j)\) 的排队压力或容量占用越高。该指标后续用于三类建模：作为链路重构成本学习状态的一部分，刻画候选链路所处的拥塞环境；用于计算候选拓展链路带来的拥塞缓解量；并进入系统级 STEE 目标函数，衡量拓展链路对全网拥塞状态的改善。

本文只关注路由层链路选择，不区分承载数据的具体编码类型。参考一般丢包-重传建模，业务包 \(p\) 在链路 \((i,j)\) 上因链路误码导致的一次发送失败概率可写为

$$
p^{\mathrm{ch}}_{p,ij,k}
=
1-(1-\varepsilon^{\mathrm{bit}}_{ij,k})^{B_p}(1-\varepsilon^{\mathrm{bit}}_{ji,k})^{B_p^{\mathrm{ack}}},
$$

其中 \(\varepsilon^{\mathrm{bit}}_{ij,k}\) 为链路 \((i,j)\) 在周期 \(k\) 的误比特率，区别于前文基础链路指示变量 \(b_{ij,k}\)；\(B_p^{\mathrm{ack}}\) 为确认信息大小。该式只刻画链路层传输失败事件，不对业务包的内容类型、具体编码方式或应用层恢复机制做额外假设。

队列溢出丢包概率定义为

$$
p^{\mathrm{q}}_{ij,k}
=
\Pi_{[0,1]}
\left(
\frac{[Q_{ij,k}+A_{ij,k}-C_{ij,k}T_t-Q_{ij}^{\max}]^+}
{A_{ij,k}+\epsilon}
\right),
$$

其中 \([x]^+=\max\{x,0\}\)，\(\Pi_{[0,1]}(\cdot)\) 表示截断到 \([0,1]\)，\(C_{ij,k}T_t\) 表示链路 \((i,j)\) 在一个结构周期内最多可服务的数据量。因此，\(p^{\mathrm{q}}_{ij,k}\) 表示因队列容量不足而被丢弃的到达流量比例。综合信道错误和队列溢出，单跳丢包概率为

$$
\rho_{p,ij,k}
=
1-
(1-p^{\mathrm{ch}}_{p,ij,k})
(1-p^{\mathrm{q}}_{ij,k}).
$$

其中 \(\rho_{p,ij,k}\) 表示业务包 \(p\) 在链路 \((i,j)\) 上经历一次发送时未能成功交付的概率。若采用丢包重传机制，则业务包 \(p\) 通过链路 \((i,j)\) 的期望单跳时延为

$$
\widetilde{D}_{p,ij,k}
=
\left(
T^{\mathrm{q}}_{ij,k}
+
\frac{B_p}{C_{ij,k}+\epsilon}
+
\frac{d_{ij,k}}{c}
+
\frac{B_p^{\mathrm{ack}}}{C_{ji,k}+\epsilon}
\right)
\frac{1}{1-\rho_{p,ij,k}+\epsilon}.
$$

其中 \(T^{\mathrm{q}}_{ij,k}\) 为链路 \((i,j)\) 的排队等待时延，\(B_p/(C_{ij,k}+\epsilon)\) 为数据包传输时延，\(d_{ij,k}/c\) 为传播时延，\(c\) 为光速，\(B_p^{\mathrm{ack}}/(C_{ji,k}+\epsilon)\) 为确认信息返回时延，乘子 \(1/(1-\rho_{p,ij,k}+\epsilon)\) 表示重传导致的期望发送次数。对路径 \(\pi_p\)，端到端期望时延和成功概率分别为

$$
T_p(\pi_p)
=
\sum_{(i,j,k)\in\pi_p}\widetilde{D}_{p,ij,k},
\quad
P_p^{\mathrm{succ}}(\pi_p)
=
\prod_{(i,j,k)\in\pi_p}(1-\rho_{p,ij,k}).
$$

其中 \(\pi_p\) 表示业务包 \(p\) 经过的有序跳序列，\((i,j,k)\in\pi_p\) 表示该包在周期 \(k\) 通过链路 \((i,j)\)。若一个包在同一拓扑周期内完成多跳转发，则这些跳具有相同的 \(k\)；若跨越多个拓扑周期，则每一跳使用其实际发生时对应的周期索引。

### E. 业务流、语义价值与时效压力

业务包集合记为 \(\mathcal{P}\)，周期 \(k\) 内等待调度或可能受拓展链路影响的业务包集合记为 \(\mathcal{P}_k\subseteq\mathcal{P}\)。每个包定义为

$$
p=(g_p,d_p,B_p,t_p^{0},v_p,\tau_p,\beta_p,\ell_p,\vartheta_p),
$$

其中 \(g_p,d_p\in\mathcal{G}\) 分别为源网关和目的网关，\(B_p\) 为包长，\(t_p^{0}\) 为生成时间，\(v_p\) 为语义价值，\(\tau_p\) 为任务截止时间，\(\beta_p\) 为超时敏感系数，\(\ell_p\) 为丢失造成的任务损失，\(\vartheta_p\) 为任务语义特征。普通业务可视为低语义价值、弱时效约束的特例；灾害监测、热点遥感回传、海事应急、目标跟踪等业务则具有较高 \(v_p\)、较小 \(\tau_p-t_p^0\)、较大 \(\beta_p\) 或较大 \(\ell_p\)。

本文将 \(v_p\) 视为由业务层随数据包一同提供的属性，即路由层不负责定义任务语义本身，而是利用业务层给出的语义价值进行资源调度。一个可行实现是：业务层根据数据包所属任务类型、文本描述、事件类别或元数据，通过 NLP 模型或任务分类器得到未归一化语义分数 \(\eta_p\)，再在同一调度窗口的业务集合 \(\mathcal{P}_k\) 内做 softmax 归一化：

$$
v_p
=
\frac{\exp(\eta_p)}
{\sum_{q\in\mathcal{P}_k}\exp(\eta_q)}.
$$

在本文建模和仿真中，\(v_p\) 暂由数据包属性直接给出；上述 NLP 评分和归一化过程可作为业务层接口解释，不作为本文核心研究内容。这里使用 \(\eta_p\) 表示业务层评分，是为了避免与前文网关接入决策变量 \(z_{g i,k}\) 混淆。

为刻画业务对拓展链路资源的需求强度，定义语义时效压力

$$
\Psi_p(t)
=
\alpha_v v_p
+
\alpha_w \frac{t-t_p^{0}}{\tau_p-t_p^{0}+\epsilon}
+
\alpha_d \frac{1}{\tau_p-t+\epsilon}
+
\alpha_\ell \ell_p,
$$

其中 \(\alpha_v,\alpha_w,\alpha_d,\alpha_\ell\) 为权重，\(\epsilon>0\) 用于避免分母为零。该指标同时反映任务重要性、已等待时间、接近截止时间的紧迫程度以及潜在任务损失。

可调整假设：若后续已有具体任务收益函数或信息年龄模型，可将 \(\Psi_p(t)\) 替换为业务层输出的任务价值、效用提升量或信息新鲜度。

### F. 可重构链路成本模型

前文已经给出了周期 \(k\) 内可被选择的候选拓展集合 \(\mathcal{E}^{\mathrm{C}}_k\) 以及激活变量 \(x_{ij,k}\)。本节不重新定义链路集合，也不额外施加候选链路可行性约束，而是围绕相邻结构周期之间 \(x_{ij,k}\) 的状态变化，刻画可重构链路建立、保持和释放所产生的开销。具体而言，若 \(x_{ij,k-1}=0,x_{ij,k}=1\)，系统需要完成终端转向、捕获、同步和链路建立；若 \(x_{ij,k-1}=1,x_{ij,k}=1\)，链路继续占用额外可重构终端，并产生跟踪能耗和机会成本；若 \(x_{ij,k-1}=1,x_{ij,k}=0\)，系统需要释放链路并回收或重新指向终端。因此，成本模型必须与前文的链路选择变量 \(x_{ij,k}\) 及后续定义的新激活变量 \(u_{ij,k}\)、释放变量 \(v_{ij,k}\) 保持一致，用于度量上述三类重构行为。对于上一周期已激活而当前周期因不可见或端点星地占用不再属于 \(\mathcal{E}^{\mathrm{C}}_k\) 的链路，后文通过过渡集合统一计算其释放成本。

该成本项的作用是为后续 STEE 指标和联合优化目标提供“收益-开销”比较基准：拓展链路只有在其带来的语义效用提升、时延改善、可靠性改善或拥塞缓解足以覆盖重构开销时，才应被优先选择。拓展链路重构成本既可以由物理参数计算，也可以由历史观测自适应估计。本文将成本建模为“物理成本先验 + 学习型成本估计”：物理成本先验由终端指向变化、捕获同步时间、能耗和机会成本计算；学习型成本估计允许系统根据历史观测用 Q-table 或函数逼近修正不同链路在不同状态下的长期重构代价。

记卫星 \(i\) 的额外可重构终端在上一结构周期 \(k-1\) 指向的目标为 \(h_i(k-1)\)，该目标可以是某颗卫星、某个网关或空闲参考方向。定义从原目标切换到候选卫星 \(j\) 所需的指向角变化

$$
\Delta\theta_{i\to j,k}
=
\arccos
\left(
\mathbf{u}_{i,h_i(k-1)}^\top
\mathbf{u}_{ij,k}
\right),
$$

其中 \(\mathbf{u}_{ij,k}\) 为周期 \(k\) 内卫星 \(i\) 指向卫星 \(j\) 的单位方向向量，\(\mathbf{u}_{i,h_i(k-1)}\) 为卫星 \(i\) 的额外终端在上一周期的指向单位向量。若终端在上一周期空闲，可令 \(\Delta\theta_{i\to j,k}=0\) 或设为固定初始捕获角。候选链路 \((i,j)\) 的建立时间为

$$
T^{\mathrm{set}}_{ij,k}
=
T^{\mathrm{acq}}_0
+
\frac{\Delta\theta_{i\to j,k}}{\omega_i^{\mathrm{slew}}}
+
\frac{\Delta\theta_{j\to i,k}}{\omega_j^{\mathrm{slew}}}
+
T^{\mathrm{sync}}_{ij,k},
$$

其中 \(T^{\mathrm{acq}}_0\) 为捕获与握手基础时间，\(\omega_i^{\mathrm{slew}}\) 和 \(\omega_j^{\mathrm{slew}}\) 分别为两端终端最大转动角速度，\(T^{\mathrm{sync}}_{ij,k}\) 为同步、捕获和跟踪稳定时间。对应建立能耗为

$$
E^{\mathrm{set}}_{ij,k}
=
P_i^{\mathrm{slew}}\frac{\Delta\theta_{i\to j,k}}{\omega_i^{\mathrm{slew}}}
+
P_j^{\mathrm{slew}}\frac{\Delta\theta_{j\to i,k}}{\omega_j^{\mathrm{slew}}}
+
(P_i^{\mathrm{acq}}+P_j^{\mathrm{acq}})T^{\mathrm{acq}}_0.
$$

链路保持期间的跟踪能耗为

$$
E^{\mathrm{hold}}_{ij,k}
=
(P_i^{\mathrm{trk}}+P_j^{\mathrm{trk}})T_s.
$$

因此物理成本先验定义为

$$
c^{\mathrm{phy}}_{ij,k}
=
\omega_T\frac{T^{\mathrm{set}}_{ij,k}}{T_s}
+
\omega_E\frac{E^{\mathrm{set}}_{ij,k}+E^{\mathrm{hold}}_{ij,k}}{E_0}
+
\omega_O O_{ij,k},
$$

其中 \(O_{ij,k}\) 为额外终端被占用造成的机会成本，例如激活链路 \((i,j)\) 后无法同时选择其他候选拓展链路；\(E_0\) 为能耗归一化常数；\(\omega_T,\omega_E,\omega_O\) 分别控制建链时间、能耗和机会成本在总成本中的权重。若后续只考虑链路切换开销，可令 \(\omega_E=\omega_O=0\)。

为便于进入优化约束，进一步将物理成本拆分为建立成本、保持成本和释放成本：

$$
c^{\mathrm{set,phy}}_{ij,k}
=
\omega_T\frac{T^{\mathrm{set}}_{ij,k}}{T_s}
+
\omega_E\frac{E^{\mathrm{set}}_{ij,k}}{E_0}
+
\omega_O O^{\mathrm{set}}_{ij,k},
$$

$$
c^{\mathrm{hold,phy}}_{ij,k}
=
\omega_E\frac{E^{\mathrm{hold}}_{ij,k}}{E_0}
+
\omega_O O^{\mathrm{hold}}_{ij,k},
$$

$$
c^{\mathrm{off,phy}}_{ij,k}
=
\omega_T\frac{T^{\mathrm{off}}_{ij,k}}{T_s}
+
\omega_E\frac{E^{\mathrm{off}}_{ij,k}}{E_0}.
$$

其中 \(c^{\mathrm{set,phy}}_{ij,k}\) 表示从未激活状态切换到激活状态所需的物理建立成本，\(c^{\mathrm{hold,phy}}_{ij,k}\) 表示链路在周期 \(k\) 继续占用额外终端的保持成本，\(c^{\mathrm{off,phy}}_{ij,k}\) 表示释放链路或切回默认指向的关闭成本。\(O^{\mathrm{set}}_{ij,k}\) 表示启动链路时占用额外终端造成的即时机会成本，\(O^{\mathrm{hold}}_{ij,k}\) 表示保持链路期间放弃其他候选连接的机会成本，\(T^{\mathrm{off}}_{ij,k}\) 和 \(E^{\mathrm{off}}_{ij,k}\) 表示释放链路、回收终端或重新对准默认姿态的时间与能耗。

在上述物理先验之外，为了体现成本的历史可学习性，使用当前拓展链路决策周期内的拥塞、距离和拓展链路速率定义链路重构状态

$$
\zeta_{ij,k}
=
[\Delta\theta_{i\to j,k},\Delta\theta_{j\to i,k},d_{ij,k},R^{\mathrm{E}}_{ij,k},
\Omega_{ij,k},x_{ij,k-1}],
$$

其中 \(\zeta_{ij,k}\) 为候选链路 \((i,j)\) 在周期 \(k\) 的成本学习状态，包含两端指向角变化、卫星间距离、拓展链路速率、当前拥塞压力以及上一周期是否已激活。由于 \(\mathcal{E}^{\mathrm{C}}_k\) 已经在集合定义中剔除了额外终端被星地服务占用的端点，网关占用状态不再作为候选链路成本状态的独立输入。动作 \(a\in\{0,1\}\) 表示是否尝试激活候选链路，其中 \(a=1\) 表示激活或保持该链路，\(a=0\) 表示不激活或释放该链路。令 \(Q^{\mathrm{C}}_k(\zeta,a)\) 表示在状态 \(\zeta\) 下执行动作 \(a\) 的长期重构成本估计。若周期 \(k\) 观测到即时成本

$$
r^{\mathrm{C}}_k
=
\omega_T\frac{\widehat{T}^{\mathrm{set}}_{ij,k}}{T_s}
+
\omega_E\frac{\widehat{E}^{\mathrm{set}}_{ij,k}+\widehat{E}^{\mathrm{hold}}_{ij,k}}{E_0}
+
\omega_F\mathbf{1}_{\mathrm{fail}},
$$

其中 \(\widehat{T}^{\mathrm{set}}_{ij,k}\)、\(\widehat{E}^{\mathrm{set}}_{ij,k}\) 和 \(\widehat{E}^{\mathrm{hold}}_{ij,k}\) 为实际观测或仿真反馈得到的建链时间、建链能耗和保持能耗，\(\mathbf{1}_{\mathrm{fail}}\) 表示本次建链或保持是否失败，\(\omega_F\) 为失败惩罚权重。则可采用 Q-table 或函数逼近形式更新

$$
Q^{\mathrm{C}}_{k+1}(\zeta_{ij,k},a)
=
(1-\alpha_C)Q^{\mathrm{C}}_{k}(\zeta_{ij,k},a)
+
\alpha_C
\left[
r^{\mathrm{C}}_k
+
\gamma_C\min_{a'\in\{0,1\}}Q^{\mathrm{C}}_{k}(\zeta_{ij,k+1},a')
\right].
$$

最终用于优化的成本估计可写为

$$
\widehat{c}_{ij,k}
=
(1-\lambda_C)c^{\mathrm{phy}}_{ij,k}
+
\lambda_C Q^{\mathrm{C}}_k(\zeta_{ij,k},1),
$$

其中 \(\lambda_C\in[0,1]\) 控制物理模型和经验学习的权重。若希望完全采用 Q-table 学习成本，可令 \(\lambda_C=1\)，并用 \(c^{\mathrm{phy}}_{ij,k}\) 初始化 \(Q^{\mathrm{C}}\)。

对应地，\(\widehat{c}^{\mathrm{set}}_{ij,k}\)、\(\widehat{c}^{\mathrm{hold}}_{ij,k}\) 和 \(\widehat{c}^{\mathrm{off}}_{ij,k}\) 可分别由 \(c^{\mathrm{set,phy}}_{ij,k}\)、\(c^{\mathrm{hold,phy}}_{ij,k}\)、\(c^{\mathrm{off,phy}}_{ij,k}\) 初始化，并使用同样的 Q-table 更新机制进行迭代估计。

由于候选集合会随星座几何关系和星地接入结果变化，为计算链路状态变化，定义周期 \(k\) 的过渡集合

$$
\mathcal{E}^{\mathrm{tr}}_k
=
\mathcal{E}^{\mathrm{C}}_k\cup\mathcal{E}^{\mathrm{C}}_{k-1}.
$$

其中 \(\mathcal{E}^{\mathrm{tr}}_k\) 不是新的候选拓展集合，也不改变周期 \(k\) 的链路可行域；它仅用于在成本统计中同时覆盖当前可新激活或保持的链路，以及上一周期已激活但当前需要释放的链路。

定义链路新激活变量 \(u_{ij,k}\) 和释放变量 \(v_{ij,k}\)，\((i,j)\in\mathcal{E}^{\mathrm{tr}}_k\)。其中 \(u_{ij,k}=1\) 表示链路 \((i,j)\) 在周期 \(k-1\) 未激活而在周期 \(k\) 被新激活，\(v_{ij,k}=1\) 表示链路在周期 \(k-1\) 已激活而在周期 \(k\) 被释放：

$$
u_{ij,k}\ge x_{ij,k}-x_{ij,k-1},\quad
u_{ij,k}\le x_{ij,k},\quad
u_{ij,k}\le 1-x_{ij,k-1},
$$

$$
v_{ij,k}\ge x_{ij,k-1}-x_{ij,k},\quad
v_{ij,k}\le x_{ij,k-1},\quad
v_{ij,k}\le 1-x_{ij,k}.
$$

其中 \(u_{ij,k},v_{ij,k}\in\{0,1\}\)。这些线性约束用于把 \(x_{ij,k}\) 的状态变化转化为建立成本和释放成本。采用 \(\mathcal{E}^{\mathrm{tr}}_k\) 可以覆盖“上一周期已激活、当前周期因不可见或端点接入 GSL 而不再属于候选集合”的释放情形。周期 \(k\) 的拓展链路总成本为

$$
C^{\mathrm{ext}}_k
=
\sum_{(i,j)\in\mathcal{E}^{\mathrm{tr}}_k}
\left(
\widehat{c}^{\mathrm{set}}_{ij,k}u_{ij,k}
+
\widehat{c}^{\mathrm{hold}}_{ij,k}x_{ij,k}
+
\widehat{c}^{\mathrm{off}}_{ij,k}v_{ij,k}
\right),
$$

在本文采用 \(T_s=T_t\) 的同步更新设定下，拓展链路可以在每个拓扑周期重新决策，因此默认取 \(H^{\min}_{ij}=1\)。若后续实验需要刻画链路建立后的最小服务时间，可仅在连续可见窗口内引入可选保持约束：

$$
\sum_{\mu=k}^{k+H^{\min}_{ij}-1}x_{ij,\mu}
\ge
H^{\min}_{ij}u_{ij,k}.
$$

该可选约束需同时满足链路在保持窗口内始终属于候选集合，即

$$
(i,j)\in\mathcal{E}^{\mathrm{C}}_\mu,
\quad \mu=k,\ldots,k+H^{\min}_{ij}-1,
$$

其中 \(H^{\min}_{ij}\) 为链路 \((i,j)\) 一旦新激活后至少保持的结构周期数。默认 \(H^{\min}_{ij}=1\) 时，该约束退化为 \(x_{ij,k}\ge u_{ij,k}\)，已经由新激活变量定义隐含满足；当 \(H^{\min}_{ij}>1\) 时，该约束才强制链路在连续候选窗口内保持多个周期，以避免在卫星运动导致链路不可见或端点被星地服务占用后仍强制保持拓展链路。

### G. 语义拓扑拓展效率与预测收益

为将语义价值、时延敏感性、路径改善、可靠性改善、拥塞缓解和链路激活成本统一到同一指标中，本文定义语义拓扑拓展效率（Semantic-aware Topology Extension Efficiency, STEE）。

在结构周期 \(k\) 内，令 \(t_k\) 表示该周期的起始时刻或用于预测的代表时刻，业务 \(p\) 的预测语义时效压力为

$$
\widehat{\Psi}_{p,k}=\widehat{\Psi}_p(t_k).
$$

其中 \(\widehat{\Psi}_{p,k}\) 是链路拓展控制器在周期 \(k\) 对业务 \(p\) 紧迫程度和语义重要性的预测值，可由当前等待时间、截止时间和业务层语义属性计算。设 \(\widehat{T}^{\mathrm{B}}_p\)、\(\widehat{P}^{\mathrm{B}}_p\) 分别表示仅使用基础拓扑时业务 \(p\) 的预测端到端时延和成功概率；\(\widehat{T}^{\mathrm{E}}_p(e_{ij})\)、\(\widehat{P}^{\mathrm{E}}_p(e_{ij})\) 分别表示激活候选拓展链路 \(e_{ij}\) 后的对应预测值。定义业务效用

$$
\widehat{U}^{r}_p
=
v_p\widehat{P}^{r}_p
\exp\left[-\beta_p[\widehat{T}^{r}_p-\tau_p]^+\right]
-
\ell_p(1-\widehat{P}^{r}_p),
\quad r\in\{\mathrm{B},\mathrm{E}\}.
$$

其中 \(r=\mathrm{B}\) 表示基础拓扑情形，\(r=\mathrm{E}\) 表示激活候选拓展链路 \(e_{ij}\) 后的情形；\(\widehat{U}^{r}_p\) 越大，表示业务 \(p\) 在该拓扑情形下获得的语义交付收益越高。第一项刻画成功交付且按时到达时的语义收益，第二项刻画未成功交付导致的任务损失。候选链路 \(e_{ij}\) 的受益业务集合定义为

$$
\mathcal{B}_{ij,k}
=
\left\{
p\in\mathcal{P}_k
\mid
\widehat{U}^{\mathrm{E}}_p(e_{ij})-\widehat{U}^{\mathrm{B}}_p>0
\right\}.
$$

其中 \(\mathcal{B}_{ij,k}\subseteq\mathcal{P}_k\) 表示周期 \(k\) 内可能从候选拓展链路 \(e_{ij}\) 获益的业务集合。只有当激活 \(e_{ij}\) 后业务效用相对基础拓扑提高时，该业务才计入候选链路收益。记候选链路带来的预测路径改善量和可靠性改善量为

$$
\Delta T_{p,ij,k}
=
[\widehat{T}^{\mathrm{B}}_p-\widehat{T}^{\mathrm{E}}_p(e_{ij})]^+,
\quad
\Delta P_{p,ij,k}
=
[\widehat{P}^{\mathrm{E}}_p(e_{ij})-\widehat{P}^{\mathrm{B}}_p]^+.
$$

其中 \(\Delta T_{p,ij,k}\) 表示候选链路 \(e_{ij}\) 对业务 \(p\) 的预测端到端时延下降量，\(\Delta P_{p,ij,k}\) 表示其对成功交付概率的提升量。记拥塞缓解量为

$$
\Delta\Omega_{ij,k}
=
\sum_{(a,b)\in\mathcal{E}^{\mathrm{act}}_k}
[\widehat{\Omega}^{\mathrm{B}}_{ab,k}-\widehat{\Omega}^{\mathrm{E}}_{ab,k}(e_{ij})]^+.
$$

其中 \(\widehat{\Omega}^{\mathrm{B}}_{ab,k}\) 表示基础拓扑下链路 \((a,b)\) 的预测拥塞压力，\(\widehat{\Omega}^{\mathrm{E}}_{ab,k}(e_{ij})\) 表示激活候选链路 \(e_{ij}\) 后该链路的预测拥塞压力，\(\Delta\Omega_{ij,k}\) 表示候选拓展链路对全网拥塞的总缓解量。定义候选链路评估成本

$$
\widehat{C}^{\mathrm{eval}}_{ij,k}
=
\widehat{c}^{\mathrm{set}}_{ij,k}(1-x_{ij,k-1})
+
\widehat{c}^{\mathrm{hold}}_{ij,k}.
$$

其中，\(\widehat{C}^{\mathrm{eval}}_{ij,k}\) 是用于评估是否选择候选链路 \(e_{ij}\) 的边级成本。若链路上一周期未激活，则评估成本包含建立成本和保持成本；若链路已经处于保持状态，则只计算继续占用终端的保持成本。候选链路 \(e_{ij}\) 的语义拓扑拓展效率定义为

$$
\mathrm{STEE}_{ij,k}
=
\frac{
\sum_{p\in\mathcal{B}_{ij,k}}
\widehat{\Psi}_{p,k}
\left(
\omega_U[\widehat{U}^{\mathrm{E}}_p(e_{ij})-\widehat{U}^{\mathrm{B}}_p]^+
+
\omega_{\Delta T}\Delta T_{p,ij,k}
+
\omega_P\Delta P_{p,ij,k}
\right)
+
\omega_\Omega\Delta\Omega_{ij,k}
}{
\widehat{C}^{\mathrm{eval}}_{ij,k}
+
\epsilon
}.
$$

其中 \(\omega_U,\omega_{\Delta T},\omega_P,\omega_\Omega\ge0\) 分别为业务效用提升、时延改善、可靠性改善和拥塞缓解的权重。这里使用 \(\omega_{\Delta T}\) 而不是 \(\omega_T\)，是为了避免与成本模型中的建链时间权重 \(\omega_T\) 混淆。该指标的分子刻画拓展链路对高语义价值、高时效压力业务的效用提升、时延改善、可靠性提升和拥塞缓解；分母刻画激活和保持该链路所需的可重构终端成本。慢时标控制器可优先选择 \(\mathrm{STEE}_{ij,k}\) 较高且满足终端约束的候选链路。

### H. 快时标语义感知逐跳路由模型

在快时标上，当包 \(p\) 到达卫星 \(i\) 时，若 \(i\) 当前与目的网关 \(d_p\) 直接相连，则包直接通过 GSL 下行；否则卫星 \(i\) 在当前激活星间邻居集合

$$
\mathcal{A}_{i,k}
=
\{j\in\mathcal{S}\mid (i,j)\in\mathcal{E}^{\mathrm{act}}_k\}
$$

中选择下一跳

$$
a_p(t)\in\mathcal{A}_{i,k}.
$$

由于拓展链路可以连接物理可达范围内任意候选卫星，\(\mathcal{A}_{i,k}\) 不再局限于固定的 `U/D/R/L` 四个方向，而是由基础邻居和当前已激活拓展邻居共同组成的可变集合。System Model 需要明确这一物理动作空间；至于学习算法如何处理可变动作数，可在算法章节中通过候选邻居排序、Top-\(K\) 截断、padding 与 action mask，或基于边评分的策略网络进一步实现。

为了与分布式路由实现一致，卫星 \(i\) 只利用一跳邻居信息、目的网关接入卫星位置和本地队列状态进行决策。可将包 \(p\) 在卫星 \(i\) 处的状态表示为

$$
s_i^p(t)
=
\left[
\mathbf{q}_{\mathcal{N}_i}(t),
\Delta\mathbf{r}_{\mathcal{N}_i}(t),
\mathbf{r}_i(t),
\Delta\mathbf{r}_{a_k(d_p)}(t),
\Psi_p(t),
\mathbf{x}_{\mathcal{N}_i,k}
\right],
$$

其中 \(\mathcal{N}_i(k)=\mathcal{A}_{i,k}\) 表示卫星 \(i\) 在周期 \(k\) 的当前可用星间邻居集合，\(\mathbf{q}_{\mathcal{N}_i}(t)\) 为这些邻居对应出向队列的状态向量，\(\Delta\mathbf{r}_{\mathcal{N}_i}(t)\) 为邻居相对当前卫星的位置向量，\(\mathbf{r}_i(t)\) 为当前卫星位置，\(\Delta\mathbf{r}_{a_k(d_p)}(t)\) 为目的网关当前接入卫星相对位置，\(\Psi_p(t)\) 为当前业务包的语义时效压力，\(\mathbf{x}_{\mathcal{N}_i,k}\) 表示卫星 \(i\) 与各可用邻居之间的拓展链路激活标志。路由动作 \(a_p(t)\) 是包级下一跳选择变量，不同于网关接入卫星 \(a_k(g)\)。

若沿用当前 DDQN 接口，基础状态可对应为 28 维向量：四个方向邻居各 4 个队列编码和 2 个相对坐标，加上当前卫星 2 个绝对坐标和目的接入卫星 2 个相对坐标。新论文可在此基础上加入 \(\Psi_p(t)\)、拓展链路状态标志以及额外候选邻居特征，使路由策略能够区分普通业务与高语义时效业务。

### I. 基于 STEE 的联合优化问题

在全局形式下，系统联合优化结构级拓展链路激活变量 \(\mathbf{x}=\{x_{ij,k}\}\) 与快时标路由决策 \(\pi=\{\pi_p\}\)，其中 \(\mathbf{x}_k\) 表示周期 \(k\) 的全部拓展链路激活决策，\(\pi_p\) 表示业务包 \(p\) 的实际转发路径或逐跳路由策略。定义周期 \(k\) 的系统级语义拓扑拓展效率

$$
\mathrm{STEE}_{k}(\mathbf{x}_k,\pi)
=
\frac{
\sum_{p\in\mathcal{P}_k}
\left[
U_p(\pi_p,\mathbf{x}_k)-U_p^{\mathrm{B}}
\right]^+
+
\omega_\Omega
\sum_{(i,j)\in\mathcal{E}^{\mathrm{act}}_k}
[\Omega^{\mathrm{B}}_{ij,k}-\Omega_{ij,k}]^+
}{
C^{\mathrm{ext}}_k+\epsilon
},
$$

其中

$$
U_p(\pi_p,\mathbf{x}_k)
=
v_pP_p^{\mathrm{succ}}(\pi_p)
\exp[-\beta_p(T_p(\pi_p)-\tau_p)^+]
-
\ell_p(1-P_p^{\mathrm{succ}}(\pi_p)),
$$

\(U_p^{\mathrm{B}}\) 表示仅使用基础拓扑时的业务效用，\(\Omega^{\mathrm{B}}_{ij,k}\) 表示基础拓扑下的链路拥塞压力，\(C^{\mathrm{ext}}_k\) 表示周期 \(k\) 内拓展链路建立、保持和释放的总成本。系统级 \(\mathrm{STEE}_{k}\) 衡量在付出拓展链路成本后，全网业务效用和拥塞状态相对基础拓扑的综合改善程度。联合优化问题可写为

$$
\max_{\mathbf{x},\pi}
\quad
\mathbb{E}
\left[
\sum_k \mathrm{STEE}_{k}(\mathbf{x}_k,\pi)
\right].
$$

拓展链路激活变量只在候选集合上定义，即

$$
x_{ij,k}\in\{0,1\},
\quad \forall (i,j)\in\mathcal{E}^{\mathrm{C}}_k.
$$

由于 \(\mathcal{E}^{\mathrm{C}}_k\) 已经在集合定义中剔除了物理不可达链路和额外终端被星地服务占用的卫星端点，优化问题只需在该候选集合上选择实际激活的拓展 ISL。

拓展链路互斥约束

$$
\sum_{j:(i,j)\in\mathcal{E}^{\mathrm{C}}_k}x_{ij,k}
\le 1,\quad \forall i\in\mathcal{S},
$$

该约束表示卫星 \(i\) 的额外可重构终端在同一周期内最多参与一条拓展 ISL，即只刻画不同拓展链路之间对同一额外终端的竞争。

链路容量约束

$$
F_{ij,k}\le C_{ij,k},
\quad \forall (i,j)\in\mathcal{E}^{\mathrm{act}}_k,
$$

其中 \(F_{ij,k}=\sum_{p\in\mathcal{P}_k}f^p_{ij,k}\) 表示周期 \(k\) 内所有业务在链路 \((i,j)\) 上的总流量，\(f^p_{ij,k}\) 为业务 \(p\) 分配到链路 \((i,j)\) 的流量。容量约束要求总承载流量不超过链路有效容量。

流守恒约束

$$
\sum_{j:(j,i)\in\mathcal{E}^{\mathrm{act}}_k} f^p_{ji,k}
-
\sum_{j:(i,j)\in\mathcal{E}^{\mathrm{act}}_k} f^p_{ij,k}
=
\begin{cases}
-r_{p,k}, & i=a_k(g_p),\\
r_{p,k}, & i=a_k(d_p),\\
0, & \text{otherwise},
\end{cases}
\quad \forall i\in\mathcal{S},
$$

其中 \(r_{p,k}\) 表示业务 \(p\) 在周期 \(k\) 需要传输的流量需求，\(a_k(g_p)\) 为源网关 \(g_p\) 当前接入卫星，\(a_k(d_p)\) 为目的网关 \(d_p\) 当前接入卫星。等式左侧采用“入流减出流”的形式，因此源接入卫星为 \(-r_{p,k}\)，目的接入卫星为 \(r_{p,k}\)，其他中继卫星为 0。

以及可选最小保持时间约束

$$
\sum_{\mu=k}^{k+H^{\min}_{ij}-1}x_{ij,\mu}
\ge
H^{\min}_{ij}u_{ij,k}.
$$

在本文默认同步更新模型中，取 \(H^{\min}_{ij}=1\)，表示拓展链路每个拓扑周期均可根据当前可见性和业务状态重新配置；若取 \(H^{\min}_{ij}>1\)，则必须额外满足 \((i,j)\in\mathcal{E}^{\mathrm{C}}_\mu,\ \mu=k,\ldots,k+H^{\min}_{ij}-1\)，即该链路在保持窗口内始终物理可达且两端额外终端未被星地服务占用。

上述问题同时包含慢时标二元拓展链路选择、快时标路径选择、拥塞状态演化和语义效用评估。后续算法章节可将其分解为：慢时标上根据 \(\mathrm{STEE}_{ij,k}\) 或其学习估计值选择拓展链路，快时标上在 \(\mathcal{A}_{i,k}\) 内执行带 action mask 的语义感知逐跳路由。

## 2. 需要后续确认或补充的信息

1. 语义属性来源。

   建议论文中写为：语义价值 \(v_p\)、截止时间 \(\tau_p\)、超时敏感系数 \(\beta_p\)、任务损失 \(\ell_p\) 和语义特征 \(\vartheta_p\) 由业务层作为数据包属性提供，路由层只负责使用这些属性进行链路激活和路由决策。代码实现上可以先采用随机生成、按业务类型表生成，或按热点任务场景生成。

   这样处理的优点是不会把论文范围扩展到语义识别或任务理解模型，研究重点保持在“语义属性驱动的拓扑增强和路由”。缺点是需要在实验中说明语义属性的生成方式，否则读者可能质疑 \(v_p\) 的来源和可信度。

2. 拓展链路物理含义。

   建议采用本文当前版本的建模：每颗卫星有四个基础 ISL 终端和一个额外可重构通信终端。四个基础终端用于维持基础邻居，额外终端可用于星地服务或一条拓展 ISL。因此拓展链路不是无限额外链路，而是由额外可重构终端状态决定的临时链路资源。

   该方案的优点是物理约束清楚：主模型中 \(\mathcal{E}^{\mathrm{C}}_k\) 先剔除已服务网关的卫星端点，再用 \(\sum_{j:(i,j)\in\mathcal{E}^{\mathrm{C}}_k}x_{ij,k}\le1\) 表示每星最多参与一条拓展链路。缺点是如果实际系统中星地链路不是光链路，而是独立 RF/Ka 天线，则应在论文中说明 \(y^G_{i,k}=0\) 的替代实现，即额外光终端不受星地服务占用。

3. 建立时延、保持成本和切换成本的量级。

   当前版本已给出物理成本先验 \(c^{\mathrm{phy}}_{ij,k}\) 和 Q-table 型学习成本 \(Q^C_k(\zeta,a)\)。后续需要确认的主要是 \(T^{acq}_0\)、\(\omega_i^{slew}\)、\(P_i^{slew}\)、\(P_i^{trk}\)、\(E_0\) 以及学习权重 \(\lambda_C\) 的取值。若暂时缺少真实硬件参数，实验中可先用归一化参数扫描，并明确说明成本量级用于刻画不同链路重构难度。

4. 是否显式建模丢包。

   当前版本已显式建模信道丢包、队列溢出丢包和重传时延。后续需要确认的是队列容量 \(Q_{ij}^{\max}\)、ACK 大小 \(B_p^{ack}\) 以及误比特率 \(\varepsilon^{\mathrm{bit}}_{ij,k}\) 的仿真生成方式。

   需要注意的是，当前代码主要记录队列长度和时延，并用 `infQueue` 做状态截断，尚未完整实现本文公式中的随机链路丢包和队列溢出丢包。因此论文中的丢包模型应作为扩展建模，代码实现时需要补充相应仿真逻辑。

5. 拓展链路切换周期与拓扑更新周期。

   当前建议确定采用 \(T_s=T_t\)。这里的“慢时标”是相对于包级逐跳转发而言，而不是相对于星座拓扑更新而言。也就是说，数据包转发可以在一个拓扑周期内部多次发生；拓展链路激活则与拓扑快照、星地接入和链路可见性同步更新。

   这样建模的优点是不会在卫星运动导致物理可见性变化后，仍强制保持上一周期的拓展链路决策；同时，拓展链路候选集、基础拓扑和 GSL 接入可以在同一次拓扑更新中计算，便于实现和公平对比。

   若需要降低控制开销，可以在 \(T_s=T_t\) 的基础上引入事件触发规则，即每个拓扑周期都允许重构，但只有当热点队列、语义压力或 STEE 超过阈值时才实际改变链路配置。该策略不改变系统模型中的同步周期，只改变算法层面的触发条件。

6. 拓展链路超过当前四方向动作空间。

   这件事需要在 System Model 中先定义清楚物理可行动作空间，即 \(\mathcal{A}_{i,k}\) 是基础邻居与已激活拓展邻居的并集，而不是固定四方向。算法章节再决定如何编码可变动作集合。

   可选实现包括：固定 Top-\(K\) 候选邻居并使用 action mask；对每条候选边计算边评分并选择最大者；或者将慢时标链路激活看作先筛选动作集合，快时标路由只在基础邻居加少量已激活拓展邻居中选择。建议建模部分明确“物理可达、链路预算满足且额外终端未被星地服务占用的卫星对”才进入拓展候选集合，算法部分再说明为了可训练性采用 Top-\(K\)、mask 或边评分网络。

## 3. 原论文/代码实现中值得沿用的建模点

1. 动态图抽象可以沿用：系统节点包括卫星和网关，边包括 ISL 和 GSL。

2. LEO 星座建模可以沿用：轨道面、轨道高度、倾角、轨道周期、卫星三维坐标和经纬度位置均在代码中显式实现。

3. 基础 ISL 结构可以沿用：每颗卫星拥有同轨上/下邻居和跨轨左/右邻居，对应当前 DDQN 的 `U/D/R/L` 动作空间。

4. GSL 接入抽象可以沿用：每个网关在当前周期接入一颗卫星，卫星也保存其连接网关引用。

5. 链路速率模型可以沿用：代码通过自由空间损耗、SNR 和 DVB-S2 频谱效率门限计算 ISL/GSL 速率。

6. 数据包与队列模型可以沿用：`DataBlock` 包含源、目的、大小、生成时间、路径、队列时延、传输时延和传播时延；每条出向链路对应一个 FIFO 发送队列。

7. 快时标逐跳路由机制可以沿用：RL 版本中数据包在传播过程中逐跳决定下一跳，而不是在源端一次性固定完整路径。

8. DDQN 状态设计可以沿用：邻居队列、邻居相对坐标、当前卫星坐标和目的接入卫星坐标已经形成可复用的本地观测接口。

9. 奖励结构可以沿用并扩展：当前奖励由队列等待、接近目的接入卫星、到达奖励、回环惩罚和不可用链路惩罚组成，可自然加入语义效用项。

## 4. 根据新思路应修改或新增的建模点

1. 从单一当前可用链路集合扩展为基础链路资源、候选拓展链路资源和已激活拓展链路资源，并允许拓展链路表现为新增连接或已有链路容量增强。

2. 新增慢时标链路激活变量 \(x_{ij,k}\)，并明确候选集合筛选、拓展链路互斥约束、建立/释放变量和最小保持约束。

3. 新增物理成本先验 \(c^{phy}_{ij,k}\) 与 Q-table 型长期成本估计 \(Q^C_k(\zeta,a)\)，使链路激活成本既可由硬件参数计算，也可由历史观测迭代学习。

4. 新增语义时效压力 \(\Psi_p(t)\)，使高价值、临近截止、等待时间长、任务损失大的业务在建链和路由中获得更高权重。

5. 新增语义拓扑拓展效率 \(\mathrm{STEE}_{ij,k}\)，统一刻画语义价值、时延敏感性、路径改善、可靠性改善、拥塞缓解和链路重构成本。

6. 快时标路由状态需要加入语义压力和邻接拓展链路激活状态。

7. 优化目标从普通端到端时延最小化扩展为 \(\mathrm{STEE}_k\) 最大化，同时显式考虑链路重构成本、丢包概率、重传时延和任务损失。

8. 若代码后续实现真实拓展链路，需要扩展 `createGraph` 中的候选边管理、卫星链路 buffer、可变动作空间和状态编码。System Model 中应保留“物理可达、链路预算满足且额外终端未被星地服务占用的卫星对才可能成为候选拓展邻居”的定义，算法实现再通过 Top-\(K\)、mask 或边评分网络降低复杂度。

## 5. 代码实现对应关系

当前代码中与 System Model 直接相关的模块如下：

1. 星座和卫星节点：`SimulationRL.py` 中的 `create_Constellation`、`OrbitalPlane`、`Satellite`。

2. 网关节点和 GSL 接入：`Gateway`、`linkSats2GTs`、`link2Sat`、`adjustDataRate`。

3. ISL 生成与动态图：`createGraph`、`greedyMatching`、`markovianMatchingTwo`、`establishRemainingISLs`。

4. 链路速率计算：`RFlink`、`get_data_rate`、`adjustDataRate`、`adjustDownRate`。

5. 星座运动和拓扑更新：`moveConstellation`、`rotate`、`updateSatelliteProcessesRL`。

6. 数据包和队列：`DataBlock`、`Gateway.sendBlock`、`Satellite.sendBlock`、`Satellite.receiveBlock`、`getQueues`。

7. DDQN 状态、动作和奖励：`DDQNAgent`、`getDeepStateDiff`、`getDeepLinkedSats`、`getNextHop`、`makeDeepAction`、`getQueueReward`、`getDistanceRewardV4`。
