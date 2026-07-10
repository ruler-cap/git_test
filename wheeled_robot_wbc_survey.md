# 轮式机器人 Whole-Body Control（WBC）调研

调研日期：2026-07-10

## 筛选口径

本文将 WBC 限定为：**底盘自由度与躯干、机械臂等上身自由度进入同一控制分配或预测优化问题**，而非先导航、后操作的串行系统。

轮式系统分为两类：

- **动态平衡轮式人形 / WIP（Wheeled Inverted Pendulum）**：上身动作会直接扰动底盘稳定性，适合研究平衡与操作耦合。
- **常规轮式移动操作臂**：重点是非完整约束、避障、接触操作与任务优先级切换。

“论文开放”与“控制器代码开源”分开记录；可下载论文不等于可复现源码。

## 代表性工作

| 工作 | 任务与底盘-上身协同 | 核心方法 | 发表时间 | 开源情况与链接 |
| --- | --- | --- | --- | --- |
| Dietrich et al., *Reactive Whole-Body Control: Dynamic Mobile Manipulation Using a Large Number of Actuated Degrees of Freedom* | 在 Rollin' Justin 上完成远距离抓取、人机物理交互、自碰撞规避；移动底盘、躯干、双臂共同完成多目标。 | 力矩控制的分层任务控制：动态一致零空间投影与全身阻抗控制。高优先级安全、接触任务优先，冗余度用于低优先级操作。 | 2012-06，IEEE Robotics & Automation Magazine | 论文预印本可读；未发现公开可复现控制器代码。[论文](https://elib.dlr.de/81232/)；[DLR 项目说明](https://www.dlr.de/en/rm/research/robotic-systems/humanoids/rollin-justin/compliant-whole-body-manipulation) |
| Zafar & Christensen, *Whole Body Control of a Wheeled Inverted Pendulum Humanoid* | 两轮倒立摆人形在保持平衡时执行操作空间末端任务；上身动作显式影响底盘平衡。 | 基于 operational-space control 的全身运动/力矩控制，处理底盘欠驱动与关节冗余。 | 2016-11，IEEE-RAS Humanoids | 论文可获取；未发现官方控制代码或项目仓库。[论文](https://doi.org/10.1109/HUMANOIDS.2016.7803259) |
| Zafar, Hutchinson & Theodorou, *Hierarchical Optimization for Whole-Body Control of Wheeled Inverted Pendulum Humanoids* | Krang 类 WIP 人形同时平衡、保持躯干姿态或视线、搬运载荷、控制末端位姿。 | 双层优化：长时域简化模型为轮子零动态生成 CoM 目标；短时域全动力学优化负责上身操作，并处理关节角和力矩约束。 | 2019-05，ICRA；预印本 2018-10-07 | 论文开源；未发现作者公开控制代码。[论文](https://arxiv.org/abs/1810.03074) |
| Zambella et al., *Dynamic Whole-Body Control of Unstable Wheeled Humanoid Robots* | ALTER-EGO 两轮不稳定人形在上身运动和外部扰动下维持直立，突出上、下身动力学耦合。 | 不采用在线约束优化；在 quasi-velocity 下建立受约束动力学内部模型，再用 computed-torque 稳定全身，降低实时计算负担。 | 2019-10，IEEE Robotics and Automation Letters 4(4) | 有公开作者稿；未发现控制器代码。[论文与项目页](https://iliad-project.eu/publications/2019-2/dynamic-whole-body-control-of-unstable-wheeled-humanoid-robots/) |
| Kim et al., *Whole-body Control of Non-holonomic Mobile Manipulator Based on Hierarchical Quadratic Programming and Continuous Task Transition* | 非完整差速底盘与机械臂共同执行复杂末端任务，并在任务优先级变化时保持控制连续。 | HQP 同时编码等式与不等式任务，将底盘运动学约束并入求解；连续任务过渡避免优先级切换造成指令跳变。 | 2019-07，ICARM | 有[项目页](https://ggory15.github.io/ARM2019)，主要提供论文材料；未发现公开源码。[论文 DOI](https://doi.org/10.1109/ICARM.2019.8834269) |
| Kindle et al., *Whole-Body Control of a Mobile Manipulator Using End-to-End Reinforcement Learning* | RoyalPanda 在狭窄走廊中移动并操作；底盘和机械臂不再串行执行，并支持在线避障。 | 端到端强化学习策略直接输出联合控制；与采样式规划比较，缩短任务时间并扩大可达工作空间。 | 2020-02-25，arXiv 预印本 | 论文开源；未发现官方训练或部署代码。[论文](https://arxiv.org/abs/2003.02637) |
| Heins & Schoellig, *Keep it Upright: Model Predictive Control for Nonprehensile Object Transportation with Obstacle Avoidance on a Mobile Manipulator* | “服务员问题”：移动时托盘上无抓取平衡物体，同时规避静态和动态障碍；包括避开抛来物体。 | 将底盘和机械臂统一进约束 MPC；把物体接触的最小静摩擦系数纳入约束，兼顾平衡鲁棒性、碰撞规避和在线重规划。 | 2023-12，IEEE Robotics and Automation Letters 8(12) | **控制代码开源，MIT License**。最适合直接复现实机或仿真 WBC-MPC 的项目。[论文](https://arxiv.org/abs/2305.17484)；[代码](https://github.com/utiasDSL/upright) |
| Wang, Chen & Zhao, *Whole-Body Model Predictive Control for Mobile Manipulation with Task Priority Transition* | 非完整移动操作臂开并穿过自闭门：手抓、拉门后，底盘接替“挡门”任务，最后穿行。 | WBMPC 在预测时域统一优化底盘和机械臂；统一权重矩阵同时表示任务优先级与时间优先级，使末端与底盘任务平滑交接。 | 2025-05，ICRA | 有完整[项目页](https://wbmpc.github.io/)及论文、视频；截至调研未见代码仓库。 |

## 建议阅读路线

1. 从 Dietrich et al. (2012) 开始，建立力矩级任务层级、阻抗、零空间投影和安全约束的基础。
2. 若底盘为两轮自平衡，重点阅读 Zafar et al. (2019) 与 Zambella et al. (2019)：前者偏分层优化与预测，后者偏全动力学和实时 computed-torque。
3. 若平台是普通移动操作臂，优先比较 Kim et al. (2019) 的 HQP、Heins & Schoellig (2023) 的约束 MPC 与 Wang et al. (2025) 的时变任务优先级。
4. 想快速落地实现，优先从 [utiasDSL/upright](https://github.com/utiasDSL/upright) 开始；它是本文确认公开、且明确实现底盘-机械臂联合 MPC 的项目。

## 方法选型摘要

| 场景 | 优先方法 | 原因 |
| --- | --- | --- |
| 两轮自平衡人形，强上身扰动 | 全动力学 WBC、分层优化、CoM/零动态规划 | 上身惯性会直接决定轮子平衡裕度。 |
| 需要柔顺接触、人机协作 | 力矩级分层阻抗 WBC | 可在严格优先级下同时管理接触柔顺性、姿态和安全。 |
| 常规移动操作臂，多任务与约束 | HQP | 易于表达非完整约束、关节限位、避障与任务优先级。 |
| 动态障碍、物体平衡、预测任务切换 | 约束 MPC / WBMPC | 能显式处理预测时域内的动力学、接触、碰撞和阶段任务。 |
| 模型不准且任务分布可仿真 | 端到端 RL 或模型学习 | 可直接学习底盘-机械臂联合策略，但需要关注安全约束与 sim-to-real。 |

## 与 Keep it Upright 最相近的扩展

以下工作依据 `Keep it Upright` 的参考脉络及后续引用链筛选。优先保留“轮式移动操作臂 + 联合控制/规划 + 物体或环境约束”的工作；其中前两项最值得直接接续阅读。

| 工作 | 任务与联合求解 | 核心方法 | 与 Keep it Upright 的关系 | 发表时间 | 开源情况与链接 |
| --- | --- | --- | --- | --- | --- |
| Heins & Schoellig, *Robust Nonprehensile Object Transportation with Uncertain Inertial Parameters* | 同为 waiter’s problem：移动操作臂托盘运输未抓取物体；对象质量、质心、惯量未知或误差很大时仍须避免滑动/倾倒。 | 在轨迹优化中加入对惯性参数不确定性的鲁棒平衡约束；用 moment relaxation 刻画给定包围几何体下物理可实现的惯性参数集合，并验证最坏情形约束满足。 | **最直接的后续工作**：从 2023 工作的动态避障与快速在线 MPC，扩展到未知载荷惯性的鲁棒运输。 | 2025-05，IEEE Robotics and Automation Letters 10(5) | **代码沿用并开源于 `upright`，MIT License**。[论文](https://arxiv.org/abs/2411.07079)；[代码](https://github.com/utiasDSL/upright) |
| Haviland, Suenderhauf & Corke, *A Holistic Approach to Reactive Mobile Manipulation* | 9-DoF 轮式移动操作臂的闭环抓取、取放与末端跟踪；底盘和机械臂作为一个整体，避开关节位置/速度限制并保持可操纵性。 | 微分运动学 QP：以全身 Jacobian 联合分配底盘速度和关节速度；代价项最大化可操作度、限制底盘-末端相对朝向，并动态调整底盘/机械臂贡献。 | 同样追求快速反应和全身协同，但它是瞬时 QP，不显式预测物体平衡、摩擦或动态障碍。 | 2022-04，IEEE Robotics and Automation Letters 7(2) | **代码开源**，支持非完整和全向底盘。[论文](https://arxiv.org/abs/2109.04749)；[项目与代码](https://jhavl.github.io/holistic/) |
| Rizzi et al., *Robust Sampling-Based Control of Mobile Manipulators for Interaction With Articulated Objects* | 10-DoF 移动操作臂操纵门、家具等可动铰接物体；需要协调底盘与机械臂，并应对接触模式切换和扰动。 | 采样式最优控制处理不可微接触动力学；结合 CBF 与 passivity theory，为安全和稳定提供约束/保证；可在 CPU 多线程实时部署。 | 与 `Keep it Upright` 一样处理“运动中物体/环境接触”和实时重规划，但采用采样控制而非梯度型 MPC，目标是铰接物体交互。 | 2023-06，IEEE Transactions on Robotics 39(3) | 作者明确公开通用多线程实现；论文页提供实现入口。[论文与项目](https://www.research-collection.ethz.ch/items/b348fad3-a20c-49ec-9811-bcd1aca83ccd) |
| Chen et al., *Like a Martial Arts Dodge: Safe Expeditious Whole-Body Control of Mobile Manipulators for Collision Avoidance* | 9-DoF 轮式移动操作臂在全身空间避开摆动杆、飞球等快速障碍，同时到达目标并规避自碰撞。 | 两层优化：CBF 生成初始安全约束；Adaptive Cyclic Inequality (ACI) 结合障碍位置、速度和运动方向，消除传统 CBF 的伪平衡问题；最终由 QP 联合分配底盘/机械臂运动。 | 与 `Keep it Upright` 的“飞来物体避障”实验最相近，但不包含托盘物体平衡约束；适合作为 MPC 外层快速安全滤波或替代反应层。 | 2026-03 在线发表；预印本 2024-09-23 | 论文开放获取；未发现公开代码。[论文](https://arxiv.org/abs/2409.14775)；[期刊页](https://doi.org/10.1016/j.birob.2026.100301) |
| Benzi, Mancus & Secchi, *Whole-Body Control of a Mobile Manipulator for Passive Collaborative Transportation* | 人与移动操作臂共同搬运不便搬运的载荷；移动底盘和机械臂均作为全身模型的一部分参与接触交互。 | 基于扩展 Jacobian 的全身速度控制与被动性方法；在线调节交互参数，同时保持闭环鲁棒稳定。 | 同样是“运输任务 + 全身联合控制”，但对象由人协作接触而非托盘非抓取平衡；核心是柔顺/被动性，不是 MPC。 | 2022-10，IFAC SYROCO；论文集 2023 | 预印本公开；未发现公开代码。[论文](https://arxiv.org/abs/2211.13680) |
| Patra, Sinha & Guha, *Motion Planning of Nonholonomic Cooperative Mobile Manipulators* | 多个非完整轮式移动操作臂协同运输物体，在静态和动态障碍中联合规划底盘和机械臂轨迹。 | 离线以 visibility vertices 生成全局路径及凸安全区域；在线 NMPC 同时考虑底盘、手臂、运动学/动力学约束和障碍。 | 将单机器人 waiter’s problem 扩展为多机协同刚体运输；仍是显式约束的预测联合规划，但不是托盘非抓取任务。 | 2025-02-08，arXiv 预印本 | 未发现公开代码。[论文](https://arxiv.org/abs/2502.05462) |

### 针对“托盘运输 + 动态避障”的实现建议

1. 以 `upright` 为主线复现已公开的约束 MPC，并先复现实验中的托盘平衡与静态障碍场景。
2. 需要处理未知载荷时，直接对接同仓库支持的 2025 鲁棒惯性约束，而不是只靠降低速度或增大摩擦安全裕度。
3. 需要应对飞来物、人员或快速移动障碍时，可把 Chen et al. 的 CBF + ACI 视为 MPC 之外的高速安全过滤层；这需要重新处理两个优化器之间的可行性和优先级，不能直接串联使用。
4. 如果实时性优先于长时域预测，可将 Haviland et al. 的全身 QP 作为局部跟踪层，再由高层 MPC 给出末端或底盘参考。
