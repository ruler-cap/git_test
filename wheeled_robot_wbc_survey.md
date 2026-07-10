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
