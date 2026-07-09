# WBC、轮式机器人与 VLA 上臂控制调研摘要

调研日期：2026-07-09

## 1. WBC 当前工作与亮点

WBC（Whole-Body Control）当前主线是把底盘、躯干、机械臂、末端、接触力、稳定性和安全约束统一到一个实时控制问题中。代表方向包括 QP/HQP 逆动力学、MPC+WBC、阻抗/无源性控制、CBF 安全约束、学习式 WBC。

亮点：

- 从“先移动底盘、再移动机械臂”的串行流程，转向底盘-躯干-上臂的同步协同。
- 从单纯轨迹跟踪，转向同时满足碰撞、力矩、摩擦、关节限位、可达性等约束。
- 学习式 WBC 开始处理真实移动操作任务，例如窄通道、家庭任务、动态搬运和故障容错。

重点链接：

- Mobile manipulator RL-WBC: https://arxiv.org/abs/2003.02637
- Velocity-controlled mobile cobot WBC + DMP: https://arxiv.org/abs/2203.03296
- Passive collaborative transportation: https://arxiv.org/abs/2211.13680
- Deep Whole-Body Control: https://arxiv.org/abs/2210.10044
- CBF-WBC self-collision: https://arxiv.org/abs/2207.00692
- BEHAVIOR Robot Suite: https://arxiv.org/abs/2503.05652

## 2. 国内外相关课题组

国外重点：

- Stanford Robotics Lab / Oussama Khatib: https://robotics.stanford.edu/
- ETH Zurich ASL: https://asl.ethz.ch/
- ETH Zurich RSL: https://rsl.ethz.ch/
- DLR Robotics and Mechatronics: https://www.dlr.de/en/rm
- MIT Biomimetic Robotics Lab: https://biomimetics.mit.edu/
- CMU Pathak Group 相关 Deep WBC 工作: https://arxiv.org/abs/2210.10044
- UIUC / Joao Ramos 相关 SATYRR 双轮人形工作: https://arxiv.org/abs/2307.01350

国内/中国区域线索：

- 港中大/深圳相关团队：移动协作机器人 WBC + DMP，关注底盘与机械臂运动分配。
- 国内轮腿/人形平台团队：轮腿不确定地形 WBC、重肢人形 WBC、接触感知动力学优化。
- 产业侧建议持续跟踪宇树、傅利叶、智元、优必选、小鹏等平台，因为它们会推动 WBC 与上层策略接口落地。

重点链接：

- Velocity-controlled mobile cobot WBC + DMP: https://arxiv.org/abs/2203.03296
- Heavy-limb humanoid WBC: https://arxiv.org/abs/2506.14278
- Wheel-legged impedance WBC: https://arxiv.org/abs/2411.09935
- Contact-aware wheel-legged WBC: https://arxiv.org/abs/2509.14010
- FT-WBC: https://arxiv.org/abs/2606.24466

## 3. 轮式机器人运动控制 + VLA 上臂控制

轮式底盘适合家庭、仓储、工厂等结构化/半结构化环境。核心难点不是底盘或机械臂的单独控制，而是如何在移动中同时考虑可达性、避障、视野、接触力、任务效率和安全约束。

VLA 当前更适合作为上层策略或末端目标生成器，而不是直接绕过底层控制器。建议架构是：

语言/视觉指令 -> VLA 输出子任务、末端目标、抓取意图 -> WBC/MPC 做底盘-上臂协同与安全约束 -> 关节/轮速伺服执行。

重点 VLA 链接：

- RT-2: https://arxiv.org/abs/2307.15818
- OpenVLA: https://arxiv.org/abs/2406.09246
- Octo: https://arxiv.org/abs/2405.12213
- pi0: https://arxiv.org/abs/2410.24164
- GR00T N1: https://arxiv.org/abs/2503.14734
- AnoleVLA mobile manipulation: https://arxiv.org/abs/2603.15046
- DAM-VLA: https://arxiv.org/abs/2603.00926

## 建议研究问题

- VLA 输出末端目标，WBC 负责底盘-手臂协同，是否比“先导航再操作”更高效？
- 如何把 VLA 的不确定性传给 WBC，让低置信度触发降速、扩大安全距离或重规划？
- 轮式底盘能否作为“可移动肩膀”，主动调整底盘位姿以提升手臂可达性和视觉观测？
- 如何采集轮式底盘+上臂的全身遥操作数据，并用于 VLA 微调？
- 如何建立包含语言指令、长距离移动、局部避障、抓取/放置、接触操作和失败恢复的 benchmark？
