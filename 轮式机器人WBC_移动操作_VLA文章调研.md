# 轮式机器人 WBC 与移动操作文章调研

调研日期：2026-07-09  
范围：轮式机器人 / 轮式移动机械臂 / 轮式人形 / 轮腿移动操作平台中的 Whole-Body Control（WBC）、Whole-Body Mobile Manipulation、移动操作学习，以及与 VLA（Vision-Language-Action）结合的工作。  

说明：目前公开文献中，**不结合 VLA 的 WBC/移动操作文章更多、更成熟**；而**结合 VLA 的文章多是“VLA 控制或评测移动操作平台”**，真正把 VLA 与经典 WBC/MPC/约束优化控制闭环深度融合的论文还较少。因此第二类中我额外标注了“与 WBC 的关系”。

## 一、未结合 VLA：轮式 WBC / Whole-Body Mobile Manipulation

| 序号 | 论文 | 发表/预印本时间 | 平台/对象 | 与 WBC/移动操作的关系 | 链接 |
|---|---|---:|---|---|---|
| 1 | Whole-Body Control of a Mobile Manipulator using End-to-End Reinforcement Learning | 2020-02-25 | RoyalPanda 轮式移动机械臂 | 用端到端强化学习做移动机械臂 WBC，解决底盘与机械臂同步运动、窄通道避障和任务效率问题。 | https://arxiv.org/abs/2003.02637 |
| 2 | Improved Reinforcement Learning Coordinated Control of a Mobile Manipulator using Joint Clamping | 2021-10-05 | 移动机械臂 | 在移动机械臂全身控制器中研究关节限制、碰撞避障和强化学习协调控制，偏向改进 RL-WBC 的稳定性。 | https://arxiv.org/abs/2110.01926 |
| 3 | Whole-Body Control for Velocity-Controlled Mobile Collaborative Robots Using Coupling Dynamic Movement Primitives | 2022-03-07 | 速度控制移动协作机器人 | 闭式 WBC 框架，把任务运动分配到底盘和机械臂，并结合 Coupling DMP 做避障、示教和柔顺交互。 | https://arxiv.org/abs/2203.03296 |
| 4 | Dynamic Mobile Manipulation via Whole-Body Bilateral Teleoperation of a Wheeled Humanoid | 2023-07-03 | SATYRR 双轮人形机器人 | 面向动态移动操作的全身双边遥操作，展示抓取移动目标和推重箱；核心是轮式人形的躯干/轮/手臂协同。 | https://arxiv.org/abs/2307.01350 |
| 5 | Mobile ALOHA: Learning Bimanual Mobile Manipulation with Low-Cost Whole-Body Teleoperation | 2024-01-04 | Mobile ALOHA 双臂轮式平台 | 低成本全身遥操作采集数据，学习双臂移动操作任务；不是传统优化 WBC，但明确强调 whole-body teleoperation / whole-body mobile manipulation。 | https://arxiv.org/abs/2401.02117 |
| 6 | BEHAVIOR Robot Suite: Streamlining Real-World Whole-Body Manipulation for Everyday Household Activities | 2025-03-07 | 双臂轮式家务机器人，带 4-DoF 躯干 | 面向真实家庭任务的全身移动操作套件，强调双臂协调、精确导航和大范围可达性。 | https://arxiv.org/abs/2503.05652 |
| 7 | Whole-Body Bilateral Teleoperation with Multi-Stage Object Parameter Estimation for Wheeled Humanoid Locomanipulation | 2025-08-13 | 轮式人形机器人 | 全身双边遥操作 + 物体惯性参数在线估计，用于提升搬运/举升/释放等轮式人形移动操作表现；使用 VLM 估计初值，但不是 VLA 控制。 | https://arxiv.org/abs/2508.09846 |
| 8 | Whole-body Motion Control of an Omnidirectional Wheel-Legged Mobile Manipulator via Contact-Aware Dynamic Optimization | 2025-09-17 | 全向轮腿移动操作平台 | 接触感知全身动态优化，同时建模机械臂点接触和轮地线接触，适合工厂、物流等半结构化场景。 | https://arxiv.org/abs/2509.14010 |
| 9 | YOR: Your Own Mobile Manipulator for Generalizable Robotics | 2026-02-11 | 低成本全向轮式双臂移动机械臂 | 开源低成本移动机械臂平台，展示导航、双臂移动操作和 coordinated whole-body control；偏平台与学习基础设施。 | https://arxiv.org/abs/2602.11150 |
| 10 | HoMMI: Learning Whole-Body Mobile Manipulation from Human Demonstrations | 2026-03-03 | 轮式双臂移动操作机器人 | 从无机器人人体示教学习全身移动操作，使用 whole-body controller 将手眼轨迹转换为受机器人约束的全身运动。 | https://arxiv.org/abs/2603.03243 |
| 11 | Whole-Body Mobile Manipulation using Offline Reinforcement Learning on Sub-optimal Controllers | 2026-04-14 | TIAGo++ 轮式双臂移动机械臂 | 先用轻量 WBC 生成数据，再用离线强化学习提升开门、抽屉、柜门等全身移动操作任务。 | https://arxiv.org/abs/2604.12509 |
| 12 | DynaMOMA: Instantaneous Prediction of Grasp Poses for Mobile Manipulation of Dynamic Objects | 2026-06-24 | 移动机械臂 | 用扩散模型预测动态物体短时抓取轨迹，再交给 whole-body RL policy 控制底盘与机械臂抓取动态目标。 | https://arxiv.org/abs/2606.25295 |

### 这一类的技术脉络

- 早期重点是底盘-机械臂运动分配：如何在末端任务、底盘速度、机械臂关节、避障和安全约束之间做协调。
- 2023 年以后轮式人形、双臂移动平台和低成本遥操作平台明显增多，研究重点从“控制器”扩展到“数据采集 + 学习策略 + 全身执行”。
- 2025-2026 年的趋势是：用 WBC 作为结构先验或安全执行层，再叠加离线 RL、扩散模型、遥操作数据或人类示教。

## 二、结合 VLA：VLA 与轮式移动操作 / 全身移动操作

| 序号 | 论文 | 发表/预印本时间 | 平台/对象 | 与 VLA 的关系 | 与 WBC 的关系 | 链接 |
|---|---|---:|---|---|---|---|
| 1 | AgentWorld: An Interactive Simulation Platform for Scene Construction and Mobile Robotic Manipulation | 2025-08-11 | 支持轮式底盘和人形运动策略的移动操作仿真平台 | 在家庭移动操作任务上 benchmark imitation learning、ACT、diffusion policy 和 VLA。 | 更偏仿真平台和数据，不是 WBC 控制论文；可用于训练/评测 VLA+全身控制策略。 | https://arxiv.org/abs/2508.07770 |
| 2 | Experiences from Benchmarking Vision-Language-Action Models for Robotic Manipulation | 2025-11-14 | ALOHA Mobile 平台 | 对 ACT、OpenVLA-OFT、RDT-1B、π0 等 VLA/策略模型做真实平台 benchmark。 | 不是 WBC 方法论文，但评测对象是移动操作平台，能反映 VLA 在轮式全身任务中的实际问题。 | https://arxiv.org/abs/2511.11298 |
| 3 | Benchmarking Affordance Generalization with BusyBox | 2026-02-05 | Mobile ALOHA 双臂移动平台 | 用 BusyBox 评测 π0.5、GR00T-N1.6 等 VLA 的 affordance generalization，并发布 Mobile ALOHA 示教数据。 | 不是 WBC 控制器论文；价值在于为轮式双臂移动平台提供 VLA 泛化评测任务。 | https://arxiv.org/abs/2602.05441 |
| 4 | AnoleVLA: Lightweight Vision-Language-Action Model with Deep State Space Models for Mobile Manipulation | 2026-03-16 | 服务机器人/移动操作任务 | 轻量 VLA，用深度状态空间模型处理视觉、语言和动作序列，面向移动操作。 | 摘要未强调经典 WBC；更像 VLA 直接生成轨迹/动作，底层控制作为执行层。 | https://arxiv.org/abs/2603.15046 |
| 5 | SG-VLA: Learning Spatially-Grounded Vision-Language-Action Models for Mobile Manipulation | 2026-03-24 | 家庭移动操作；13 维动作空间，包括底盘、手臂和夹爪 | 明确面向 mobile manipulation 的 VLA，输入多视角 RGB、深度和时间历史，输出协调底盘、机械臂和夹爪的动作。 | 与 WBC 最接近：动作空间包含 base motion + arm articulation + gripper，但论文主线是 VLA 表征和辅助监督，不是约束优化 WBC。 | https://arxiv.org/abs/2603.22760 |
| 6 | DAM-VLA: A Dynamic Action Model-Based Vision-Language-Action Framework for Robot Manipulation | 2026-03-01 | 机器人操作任务，含长时程和接触丰富任务 | 用 VLM 推理 + diffusion action model，动态路由 arm movement / gripper manipulation。 | 主要是机械臂/夹爪动作模型，不是轮式 WBC；可作为“VLA 上臂控制模块”参考。 | https://arxiv.org/abs/2603.00926 |
| 7 | DAM-VLA: Decoupled Asynchronous Multimodal Vision Language Action model | 2026-06-10 | 接触丰富真实操作任务 | 异步多模态 VLA，让语言、视觉、高频传感分别以自身频率更新，动作头连续读取 latent buffer。 | 不是轮式移动操作论文，但对 VLA 与高频底层控制的频率错配问题很有参考价值。 | https://arxiv.org/abs/2606.12105 |
| 8 | Gemini Robotics: Bringing AI into the Physical World | 2025-03-25 | 多机器人形态，通用机器人操作 | Google DeepMind 的 VLA/Embodied Reasoning 系列，强调直接控制机器人、开放词汇指令和少样本迁移。 | 非轮式 WBC 专项；适合作为 VLA 上层任务理解、空间推理、抓取预测和跨平台迁移的背景工作。 | https://arxiv.org/abs/2503.20020 |
| 9 | GR00T N1: An Open Foundation Model for Generalist Humanoid Robots | 2025-03-18 | 人形机器人 | 双系统 VLA：System 2 做视觉语言理解，System 1 用 diffusion transformer 生成实时动作。 | 不是轮式平台，但“慢速 VLA 推理 + 快速动作策略”的架构对上臂 VLA + 底层 WBC 很有借鉴意义。 | https://arxiv.org/abs/2503.14734 |

### 这一类的技术脉络

- 直接相关度最高的是 **SG-VLA**：它明确处理 mobile manipulation，并把底盘、手臂、夹爪放入 13 维动作空间。
- **AnoleVLA** 也明确面向 mobile manipulation，但重点是轻量 VLA 架构和推理效率。
- **AgentWorld、Mobile ALOHA VLA benchmark、BusyBox** 更像数据/平台/benchmark 工作，适合用来评估 VLA 在轮式移动操作上的泛化能力。
- **DAM-VLA、Gemini Robotics、GR00T N1** 不是轮式 WBC 论文，但对“上臂由 VLA 控制、底层由 WBC/MPC 保证安全”的系统架构很有参考价值。

## 三、从这些文章看出的研究空白

1. **VLA 与 WBC 的接口还没有统一范式。**  
   现有 VLA 往往输出末端动作、关节动作或动作 chunk；WBC 需要的是带约束的底盘-躯干-手臂协同控制目标。两者之间需要设计中间层，例如末端目标、可达性评分、底盘参考速度、约束权重或安全边界。

2. **轮式平台的优势没有被 VLA 充分利用。**  
   轮式底盘可以主动改变“肩膀位置”和视角，避免机械臂奇异位形、提升抓取可达性，但很多 VLA 仍像固定机械臂一样处理动作。

3. **安全和实时性仍依赖传统控制层。**  
   VLA 擅长语义泛化，但不保证碰撞、力矩、关节限位、轮速限制、接触力和人机安全。因此更实际的架构是：VLA 输出任务/子目标，WBC/MPC/CBF 做实时约束和安全执行。

4. **全身移动操作数据仍稀缺。**  
   Mobile ALOHA、BEHAVIOR Robot Suite、HoMMI、YOR 等工作的价值在于降低全身数据采集成本。未来 VLA+WBC 需要的不是单纯桌面抓取数据，而是包含底盘移动、躯干调整、双臂操作、失败恢复和接触任务的全身数据。

## 四、建议优先阅读顺序

如果目标是做“轮式机器人 WBC + 移动操作 + 上臂 VLA 控制”，建议按下面顺序读：

1. **WBC/移动操作基础**  
   - Whole-Body Control of a Mobile Manipulator using End-to-End Reinforcement Learning  
   - Whole-Body Control for Velocity-Controlled Mobile Collaborative Robots Using Coupling DMP  
   - Dynamic Mobile Manipulation via Whole-Body Bilateral Teleoperation of a Wheeled Humanoid  

2. **全身数据与平台**  
   - Mobile ALOHA  
   - BEHAVIOR Robot Suite  
   - HoMMI  
   - YOR  

3. **VLA 与移动操作结合**  
   - SG-VLA  
   - AnoleVLA  
   - Experiences from Benchmarking VLA Models on ALOHA Mobile  
   - AgentWorld  

4. **架构参考：VLA 上层 + 控制下层**  
   - GR00T N1  
   - Gemini Robotics  
   - DAM-VLA asynchronous model  

