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

## 专题：高动态一体化底盘-上身控制与高精度末端任务

本节针对以下更严格的目标筛选：

- 底盘与上身进入同一优化变量或闭环控制器，而非串行“停车后操作”。
- 支持快速运动、在线重规划、动态障碍或接触力扰动中的至少一项。
- 末端具有明确的位姿连续跟踪、接触力控制、物体稳定或精细作业目标。

### 优先级最高的工作

| 匹配度 | 工作 | 平台与任务 | 核心方法 | 高动态与末端精度证据 | 发表时间与开源情况 |
| --- | --- | --- | --- | --- | --- |
| A | Minniti et al., *Whole-Body MPC for a Dynamically Stable Mobile Manipulator* | 带机械臂的球式动态平衡机器人（ballbot；单球滚动底盘）。同时进行平衡、末端 SE(3) 位姿跟踪和开门接触力操作。 | 以完整动力学建立**单一全身非线性 MPC**；在末端空间写入位置、姿态与接触力目标，预测它们对未来平衡的影响；关节/力矩约束以 relaxed log barrier 处理，使用 SLQ 求解。 | 这是“上身动作直接影响底盘稳定”的强耦合基准；末端目标同时包含 pose 与 force，而非只跟踪底盘路径。注意：球式底盘不是常见差速轮，但更能暴露高动态耦合问题。 | 2019-10，RA-L 4(4)。论文开放获取；论文专用配置未公开，但其 MPC 基础设施 [OCS2](https://github.com/leggedrobotics/ocs2) 开源。[论文](https://arxiv.org/abs/1902.10415) |
| A | Pankert & Hutter, *Perceptive Model Predictive Control for Continuous Mobile Manipulation* | 轮式移动操作臂执行连续轨迹任务；利用视觉避障、触觉/力觉实现接触力控制，面向移动建造等需要末端持续精确作业的场景。 | 递推时域全身 MPC：直接跟踪末端任务空间参考，将视觉障碍、力觉交互、机械稳定与关节限位一并纳入约束。 | 任务目标就是连续末端轨迹和接触力，且系统可边移动边操作；作者报告相较采样式规划更快，并在多种实机平台验证。 | 2020-10，RA-L 5(4)。**论文配套软件开源**。[论文/项目](https://www.research-collection.ethz.ch/items/503d607c-b158-4f97-9c23-8ed95022d1bb)；[代码](https://github.com/leggedrobotics/perceptive_mpc) |
| A | Minniti et al., *Model Predictive Robot-Environment Interaction Control for Mobile Manipulation Tasks* | 同一动态平衡移动操作臂执行开门、提起物体，并与未知刚度/动力学环境接触。 | 在全身 MPC 外接在线系统辨识与自适应控制，以修正未知机器人-环境交互模型；同时规划全身运动与接触力。 | 对高精度接触特别有价值：不必预先标定环境参数或反复人工调参，仍可稳定执行门把手/物体交互。 | 2021-05，ICRA。论文开放获取；可基于 [OCS2](https://github.com/leggedrobotics/ocs2) 复现方法基础，未发现论文专用开源配置。[论文](https://arxiv.org/abs/2106.04202) |
| A | Spahn, Brito & Alonso-Mora, *Coupled Mobile Manipulation via Trajectory Optimization with Free Space Decomposition* | **非完整差速轮式**移动操作臂在杂乱、含动态障碍场景中执行取放；底盘与手臂联合规划。 | 递推时域全身轨迹优化 / MPC。用每个主要连杆周围的凸自由空间替代“每障碍物一条约束”，并预测动态障碍轨迹。 | 明确联合求解底盘与上臂；实验中相较解耦方案总执行时间降低 **48%**，且碰撞约束数量不随障碍数量增长，适合板端实时部署。末端是取放目标，而非高带宽力控。 | 2021-05，ICRA。论文开放获取；未发现官方代码。[论文](https://repository.tudelft.nl/record/uuid%3Ad901a4ce-f1c6-447c-99e2-8ced87264572) |
| A | Du et al., *An Efficient Representation of Whole-body MPC for Online Compliant Dual-arm Mobile Manipulation* | EVA：**四麦克纳姆轮** 3-DoF 底盘 + 15-DoF 上身（胸、头、双臂）；在窄通道中取放/搬运、躲避动态障碍，并进行双末端柔顺交互。 | 双层连续 MPC。第一层以 B\'ezier 曲线在 SE(3) 规划双末端长时域轨迹；第二层用 B\'ezier 参数化全身轨迹并加入 predictive admittance、末端/底盘避障和硬约束。 | 底盘速度和上身位置命令来自同一连续轨迹；任务层 MPC 平均 **6.2 ms**，全身 MPC 约 **9-12.5 ms**，控制周期 20 ms；腕部力传感器支持末端力偏差到运动响应的闭环优化。 | 2024-10-30，arXiv 预印本。论文开放获取；未发现官方代码。[论文](https://arxiv.org/abs/2410.22910) |
| A | Wang, Chen & Zhao, *Whole-Body MPC for Mobile Manipulation with Task Priority Transition* | 非完整移动操作臂抓门、开门、由底盘挡住自闭门并穿行；底盘与末端任务在执行中交接。 | 全身 MPC 将任务优先级和时间优先级合并为统一权重；预测时域内联合优化底盘和手臂，改善可操作度并平滑切换任务。 | 适合“底盘持续运动、末端保持高精度约束”的流程；项目报告单次计算 **4.3 ms**，并提高可操作度轨迹跟踪表现。 | 2025-05，ICRA。项目页提供论文和视频；未发现代码。[项目/论文](https://wbmpc.github.io/) |

### 高价值补充

| 工作 | 价值与边界 | 发表时间与开源情况 |
| --- | --- | --- |
| Wu et al., *Real-time Whole-body Motion Planning for Mobile Manipulators Using Environment-adaptive Search and Spatial-temporal Optimization*（REMANI-Planner） | 面向轮式移动操作臂的实时全身路径搜索和时空轨迹优化；把全身安全、敏捷性、动力学可行性和任务约束统一考虑。它更偏全局/局部轨迹生成，末端力控和物体动力学不如 `Keep it Upright` 完整，但可作为后者的高质量可行初值或上层规划器。 | 2024-05，ICRA。**完整代码开源，GPL-3.0**。[论文项目页](https://robotics-star.com/REMANI-Planner/)；[代码](https://github.com/Robotics-STAR-Lab/REMANI-Planner) |
| Dietrich et al., *Whole-body Impedance Control of Wheeled Mobile Manipulators* | Rollin' Justin 的非完整轮式底盘与上身阻抗控制。通过底盘 admittance interface 和上/下身动力学耦合补偿，建立被动且渐近稳定的全身闭环。它不是高动态 MPC，但适合作为高精度柔顺接触的低层控制基座。 | 2016-03-01，Autonomous Robots 40(3)。论文作者稿可读；未发现代码。[论文](https://portal.fis.tum.de/de/publications/whole-body-impedance-control-of-wheeled-mobile-manipulators-stabi/) |
| Chen et al., *Safe Expeditious Whole-Body Control of Mobile Manipulators for Collision Avoidance* | 对飞球、摆动障碍等快速外部扰动，CBF + ACI + QP 能给出反应很快的全身避障修正。它不负责末端物体平衡，但适合作为 `Keep it Upright` 的动态安全层候选。详见上一节。 | 2026-03 在线发表；论文开放获取，未发现代码。[论文](https://arxiv.org/abs/2409.14775) |

### 对目标系统的落地判断

如果你的优先级是“移动中托盘/未抓取物体不失稳”，`Keep it Upright` 仍是首选主线；其 2025 鲁棒扩展解决载荷惯性未知。若更强调“运动中精确接触、插接、打磨、喷涂或开门”，应优先研究 Pankert & Hutter (2020)、Minniti et al. (2019, 2021) 和 Du et al. (2024)。

一个可行的工程组合是：`Keep it Upright` 的全身约束 MPC 管理底盘-手臂协同和物体稳定；采用 Pankert/Minniti 路线增加视觉、力觉与末端阻抗/导纳；使用 Chen et al. 的 CBF/ACI 作为短时域动态障碍安全层。三者直接叠加会产生约束冲突，实施时应在同一个 QP/MPC 中定义硬安全约束、软任务约束和优先级，而不是串联三个互不感知的控制器。
