# WBC 算法使用位置总结

本文档用于汇报 `output_project/motion_control_project` 中 WBC 相关算法的分类、实现位置和调用关系。

## 1. 总体调用链

真机运行时，WBC 不是单独直接控制硬件，而是被 `Controller` 调用，并最终通过 `sendCmd()` 输出到电机。

```text
Controller::wbcMove()
    ↓
WbcHost::step()
    ↓
WBCController::compute()
    ↓
TSID / HQP / QP 求解
    ↓
selfCollision->project()
    ↓
RNEA / torque clamp / qDes integration
    ↓
WbcHost ramp / 20D body mapping
    ↓
Controller 写入 low_command
    ↓
Controller::sendCmd()
    ↓
motor_->state_update(...)
```

核心入口：

| 位置 | 作用 |
|---|---|
| `src/controller/controller.cpp` | 真机 Controller 主循环，调用 WBC 并输出硬件命令。 |
| `src/controller/wbc/wbc_host.cpp` | Controller 和 WBCController 之间的接入层，负责状态映射、ramp、freeze、onExit。 |
| `src/controller/wbc/wbc_controller.cpp` | WBC 主体算法，构造 TSID 任务、求解 QP、计算 `ddq/qDes/tau`。 |
| `src/controller/wbc/self_collision.cpp` | 自碰撞 capsule 距离、速度阻尼约束和 ProxQP 投影。 |
| `include/controller/wbc/mit_impedance.hpp` | MIT / CST 路径中的主机侧阻抗力矩层。 |

## 2. WBC 主体算法：TSID / HQP / QP

WBC 主体采用 TSID/HQP/QP 框架。不同 `mode` 本质上是在同一个 TSID 框架中组合不同任务。

包含内容：

| 子模块 | 主要作用 | 实现/使用位置 |
|---|---|---|
| TSID/HQP 求解 | 把多任务全身控制问题构造成 QP/HQP 并求解 `ddq`。 | `src/controller/wbc/wbc_controller.cpp` |
| Pinocchio 运动学/动力学 | 提供 FK、Jacobian、CoM、质量矩阵、RNEA 等模型计算。 | `src/controller/wbc/wbc_controller.cpp`、`src/controller/wbc/self_collision.cpp` |
| 笛卡尔任务 | 左手、右手、torso 的位置/姿态任务。 | `WBCController::addEeTask()` |
| 关节姿态任务 | posture task，mode 1 中也作为关节空间主任务。 | `TaskJointPosture` |
| CoM 任务 | 绝对 CoM 或相对底盘 CoM 约束。 | `wbc_controller.cpp`、`task_com_relative.cpp` |
| 底盘虚拟关节任务 | mode 4 中控制 `base_x/base_y/base_yaw`。 | `WBCController::addChassisTask()` |
| qDes 积分 | 将 QP 输出的 `ddq` 积分成 CSP 位置目标。 | `WBCController::integrateJointPositionCommand()` |

关键调用点：

```text
WBCController::compute()
    → updateReferences()
    → formulation->computeProblemData(...)
    → solver->solve(...)
    → formulation->getAccelerations(...)
```

对应文件：

- `output_project/motion_control_project/src/controller/wbc/wbc_controller.cpp`
- `output_project/motion_control_project/include/controller/wbc/wbc_controller.hpp`
- `output_project/motion_control_project/include/controller/wbc/wbc_config.hpp`

## 3. 安全约束与限幅

这一类包含 WBC 内部约束，也包含输出到真实硬件前的安全门控。

| 安全机制 | 作用 | 主要位置 |
|---|---|---|
| 关节速度硬约束 | 使用 `TaskJointBounds` 限制关节速度。 | `wbc_controller.cpp` |
| 加速度正则 | 使用零增益 posture task 形成软 `||ddq||^2` 正则，使输出更平滑。 | `wbc_controller.cpp` |
| RNEA 力矩计算 | QP 得到 `ddq` 后，用 Pinocchio RNEA 计算 `tau`。 | `WBCController::compute()` |
| 力矩限幅 | 将 `tau` 限制在 `TORQUE_LIMITS` 内。 | `WBCController::compute()` |
| torque bound rows | 为自碰撞投影附加力矩边界约束。 | `WBCController::torqueBoundRows()` |
| CST 幅值/变化率门控 | 真机力矩输出前检查力矩绝对值和单周期变化率。 | `Controller::sendCmd()` |

典型链路：

```text
QP 求解 ddq
    ↓
pinocchio::rnea(q, v, ddq)
    ↓
tau clamp
    ↓
Controller::sendCmd()
    ↓
CST torque gate / dtau gate
```

注意：RNEA 本身是逆动力学算法，不是安全算法；但当前工程中它和力矩输出、力矩限幅、CST 门控绑定使用，因此汇报时可归入“安全与输出约束”。

## 4. 自碰撞算法

自碰撞算法单独实现，不在 `controller.cpp` 或 `wbc_host.cpp` 中。

主要文件：

- `output_project/motion_control_project/include/controller/wbc/self_collision.hpp`
- `output_project/motion_control_project/src/controller/wbc/self_collision.cpp`
- `output_project/motion_control_project/include/controller/wbc/collision_data.hpp`

核心类：

```cpp
moz::SelfCollisionDamper
```

核心逻辑：

```text
capsule 几何体
    ↓
计算 capsule pair 最近距离
    ↓
距离进入 influence 区域时生成 velocity damper 约束
    ↓
约束形式 A * ddq >= b
    ↓
ProxQP 投影修正 TSID 输出的 ddq
```

主要函数：

| 函数 | 作用 |
|---|---|
| `segmentSegmentClosest()` | 计算两个线段之间的最近点。 |
| `SelfCollisionDamper::pairJacobian()` | 计算 capsule pair 距离方向上的 Jacobian。 |
| `SelfCollisionDamper::constraintRows()` | 根据当前 `q/v` 生成自碰撞阻尼约束。 |
| `SelfCollisionDamper::project()` | 用 ProxQP 将 `ddq` 投影到满足自碰撞约束的空间。 |

调用位置：

```text
WBCController 构造
    → std::make_unique<SelfCollisionDamper>(...)

WBCController::compute()
    → selfCollision->project(ddq, q, v, extraA, extraB)
```

## 5. MIT / 力矩阻抗相关算法

MIT 和 mode 7 direct-CST 都属于力矩/CST 路径，但二者不完全相同。

| 路径 | 说明 |
|---|---|
| MIT | WBC 输出 `qDes/vDes/tau_ff`，主机侧 MIT impedance 再生成最终 `tau_cmd`。 |
| mode 7 direct-CST | WBC 的 RNEA/QP 力矩直接作为 CST 命令来源，不叠加 MIT impedance。 |

主要文件：

- `output_project/motion_control_project/include/controller/wbc/mit_impedance.hpp`
- `output_project/motion_control_project/src/controller/wbc/wbc_controller.cpp`
- `output_project/motion_control_project/src/controller/controller.cpp`

主要机制：

| 机制 | 作用 | 位置 |
|---|---|---|
| 测量速度低通滤波 | 力矩模式下用滤波后的实测速度做阻尼，避免噪声和 windup。 | `WBCController::compute()` |
| MIT reference governor | 对 `qDes` 做泄漏治理和误差限幅，避免力矩模式下参考发散。 | `WBCController::mitGovernedCommand()` |
| 主机侧 MIT impedance | `tau_cmd = tau_ff + ramp * (Kp(qDes-q) + Kd(vDes-vFilt))`。 | `MitImpedance::compute()` |
| CST 进入 ramp | 从实测保持力矩种子平滑过渡到阻抗/直接力矩输出。 | `Controller::wbcMove()` |
| 库仑摩擦前馈 | 对 mask 关节添加平滑摩擦补偿。 | `Controller::wbcMove()` |

MIT/direct-CST 真机链路：

```text
WbcHost::step()
    ↓
WBCController::compute()
    ↓
wbc->vDes(), wbc->torque()
    ↓
Controller::wbcMove()
    ↓
MIT: mit_impedance_->compute(...)
mode 7: tau_cmd = tau_ff
    ↓
low_command.tau_ff
    ↓
Controller::sendCmd()
    ↓
CST torque command
```

## 6. WbcHost 接入和平滑机制

`WbcHost` 不是 TSID 主体算法，但是真机接入中非常关键。

主要文件：

- `output_project/motion_control_project/src/controller/wbc/wbc_host.cpp`
- `output_project/motion_control_project/include/controller/wbc/wbc_host.hpp`

主要作用：

| 机制 | 作用 |
|---|---|
| 20D body 和完整模型状态映射 | Controller 使用 20 维身体关节，WBC 内部可能使用 fixed/mobile 完整模型。 |
| 进入时 init | 第一次进入 WBC 时，用当前反馈初始化 WBC，避免跳变。 |
| posture entry anchor | 进入 WBC 时把姿态参考锚定到当前姿态。 |
| ramp | route-A 位置输出从当前反馈平滑插值到 WBC 解。 |
| freeze | 目标流超时时保持最后有效参考。 |
| torque surface | 对 MIT/direct-CST 暴露 WBC 的 `tau/vDes/qDes`。 |
| onExit | 退出 WBC 时清除进入状态、ramp 和过期 posture reference。 |

关键函数：

| 函数 | 作用 |
|---|---|
| `WbcHost::makeConfig()` | 根据 mode 和 MIT/commanded 标志生成 WBC 配置。 |
| `WbcHost::buildState()` | 把 Controller 的 20D body 状态扩展成 WBC 模型状态。 |
| `WbcHost::step()` | 每周期调用 WBCController，处理 ramp 和输出映射。 |
| `WbcHost::onExit()` | WBC 退出时重置接入状态。 |

真机调用点：

```text
Controller::wbcMove()
    → wbc->setBaseState(...)
    → wbc->setPostureReferenceBody(...)
    → wbc->step(...)
```

退出调用点：

```text
Controller::mainLoop()
    current_cmd.mode != WBC_CONTROL
        → wbc->onExit()
```

## 7. 汇报用简化分类

最终汇报时可以按下面 5 类讲：

| 类别 | 包含内容 | 汇报重点 |
|---|---|---|
| WBC 主体算法 | TSID/HQP/QP、Pinocchio、末端任务、posture、CoM、底盘虚拟关节、qDes 积分 | 不同 mode 是不同任务组合。 |
| 安全约束与限幅 | 速度约束、加速度正则、RNEA torque clamp、CST 幅值/变化率门控 | 保证输出平滑、不过限、可上真机。 |
| 自碰撞算法 | capsule 距离、velocity damper、ProxQP 投影 | 自碰撞会实际修改 `ddq`，不是只报警。 |
| MIT / 力矩算法 | MIT impedance、reference governor、速度滤波、mode 7 direct torque | 支持 CST 力矩和顺应控制。 |
| WbcHost 接入机制 | ramp、freeze、entry anchor、onExit | 保证 Controller 切入/退出 WBC 不跳变。 |

一句话总结：

```text
当前 WBC 使用 TSID/HQP/QP 作为主体控制框架，Pinocchio 提供模型计算，
通过不同 mode 组合末端、躯干、姿态、CoM 和底盘任务；求解后再经过
自碰撞投影、RNEA 力矩计算、限幅门控、MIT/direct-CST 力矩处理和
WbcHost ramp 接入，最终由 Controller 转成 CSP/CSV/CST 硬件命令。
```

## 8. 技术描述

本项目面向具备双臂、躯干和移动底盘的真实机器人，构建了一套可在真机上运行的全身控制系统。系统以 WBC（Whole-Body Control，全身控制）为核心，将末端执行器控制、躯干控制、关节姿态控制、质心约束、移动底盘控制、自碰撞约束和力矩顺应控制统一到同一个控制框架中，并通过 ROS 2 节点、真机 Controller、电机 SDK、底盘接口和数据录制工具形成完整的工程闭环。

系统采用 TSID/HQP/QP 作为主体算法框架，通过 Pinocchio 进行机器人运动学和动力学建模。控制器根据不同任务模式构造不同的 QP 任务组合，包括关节空间跟踪、双手末端跟踪、躯干跟踪、移动底盘参与的全身控制、base-sensed 上身补偿控制以及力矩顺应控制。QP 求解得到关节加速度后，系统可根据实际运行模式转换为 CSP 位置命令、CSV 底盘轮速命令或 CST 力矩命令，从而适配真实机器人上的多种电机控制模式。

在真机部署层面，项目设计了 `Controller + WbcHost + WBCController` 的分层结构。`Controller` 负责电机状态读取、安全门控、模式切换和硬件命令下发；`WbcHost` 负责 WBC 与真机状态之间的映射、进入 ramp、冻结保持和退出重置；`WBCController` 负责构建和求解全身控制问题。该结构使算法层和硬件层解耦，既支持 ROS 正式集成路径，也支持 examples 直接运行真机，同时支持离线 replay 和无电机验证。

系统还针对真实机器人运行引入了多层安全机制，包括关节速度硬约束、加速度正则、力矩限幅、CST 力矩幅值和变化率门控、自碰撞投影、底盘超速保护、WBC 进入/退出无冲击切换等。自碰撞部分采用 capsule 几何近似和 velocity damper 约束，在 TSID 求解后通过 ProxQP 对 `ddq` 进行投影修正，使机器人在执行末端和全身任务时能够主动避开危险的自碰撞区域。

对于力矩和顺应控制，系统实现了 MIT impedance 和 mode 7 direct-CST 两类路径。MIT 路径基于 WBC 输出的 `qDes/vDes/tau_ff`，在主机侧叠加阻抗反馈生成最终力矩；direct-CST 路径则将 WBC 通过 RNEA 计算出的力矩直接作为电机 CST 命令。两类路径均配套速度滤波、reference governor、进入力矩 ramp、摩擦前馈和力矩门控，提升了真实硬件上的顺应性和安全性。

项目同时保留了完整的数据闭环能力。真机运行过程可以录制 `task.csv`、`motor_state.csv`、`motor_cmd.csv`、`odom.csv` 和 `metadata.yaml` 等数据，用于离线 replay、Python/C++ 对拍、算法回归验证和现场问题定位。这使得系统不仅能完成单次演示，也具备持续迭代、验证和工程维护能力。

## 9. 创新点描述

1. **面向真实机器人的全身控制工程化集成**

   本项目不是单一算法验证程序，而是将 WBC 算法、真机 Controller、电机 SDK、底盘控制、力矩控制、ROS 接口和数据录制集成为一套可部署的真实机器人控制系统。系统能够在同一框架下支持关节控制、双手末端控制、躯干控制、移动底盘控制和力矩顺应控制，具备较强的工程完整性和可转化价值。

2. **多模式 WBC 任务组织机制**

   系统将不同控制需求抽象为 WBC mode，通过同一 TSID/HQP/QP 框架组合不同任务。mode 1-7 分别覆盖关节空间安全跟踪、双手末端跟踪、躯干跟踪、底盘参与的全身控制、主流 EEF/torso 流式控制、base-sensed 上身补偿控制和 direct-CST 力矩顺应控制。该设计提高了控制框架的复用性，使不同演示和任务不需要重复开发独立控制器。

3. **真机友好的 WBC 接入与无冲击切换机制**

   系统设计了 `WbcHost` 作为 Controller 和 WBCController 之间的接入层，专门处理真机运行中的状态映射、入口姿态锚定、ramp、freeze 和 onExit 重置问题。该机制使机器人从普通 CSP 保持切入 WBC 时能够从当前反馈状态平滑进入，降低目标突变带来的冲击风险，提升了真实机器人演示和调试的可靠性。

4. **自碰撞约束的后投影安全机制**

   系统采用 capsule 几何体描述机器人关键连杆，通过最近距离计算构造 velocity damper 自碰撞约束，并在 TSID 求解后利用 ProxQP 对关节加速度进行投影修正。该方法能够在保持原有任务解的基础上，对可能导致自碰撞的运动进行主动修正，相比单纯的距离报警或事后停止更适合连续控制场景。

5. **移动底盘与上身 WBC 的统一建模**

   在 mode 4 和 mode 6 中，系统将移动底盘状态纳入 WBC 控制或感知链路。mode 4 通过虚拟 base joints 使底盘参与全身 QP 求解，mode 6 则允许底盘由外部速度驱动，同时 WBC 感知底盘运动并补偿上身末端任务。该机制使双臂、躯干和底盘能够在同一控制框架下协同工作，适合移动操作机器人场景。

6. **位置控制与力矩顺应控制的统一输出框架**

   系统同时支持 CSP 位置输出、CSV 底盘速度输出和 CST 力矩输出。对于常规 WBC 任务，QP 输出经过积分生成 `qDes` 并由电机位置环跟踪；对于力矩顺应任务，系统可通过 MIT impedance 或 mode 7 direct-CST 输出真实力矩。该设计兼顾了位置控制的稳定性和力矩控制的顺应性，有利于在不同风险等级和应用场景之间切换。

7. **面向真机安全的多层门控设计**

   系统在算法层、Controller 层和硬件输出层均设置安全机制，包括 QP 速度约束、加速度正则、RNEA 力矩限幅、CST 幅值门控、CST 单周期力矩变化率门控、底盘超速保护、反馈有效性检查和故障锁存。多层门控降低了复杂 WBC 算法直接作用于真实硬件时的风险，提高了系统可演示性和可维护性。

8. **离线 replay 与真机数据闭环**

   系统设计了录制、replay 和对拍机制，能够将真机运行数据保存为结构化数据集，并用当前 C++ WBC 重新计算和验证。这种闭环能力有助于定位真机问题、验证算法修改影响、保证版本迭代稳定性，也为后续产品化测试和技术验收提供了数据基础。

9. **支持 ROS 集成和独立 example 运行的双入口架构**

   系统既可以通过 `control_node` 接入 ROS topic/service，实现与视觉、规划、遥操作和上位机系统的协同；也可以通过 examples 直接构造 Controller，在不依赖 ROS 通信的情况下完成真机 bring-up、单项演示和专项测试。双入口架构提升了系统调试效率和实际部署灵活性。

10. **从算法验证到真机部署的完整迁移路径**

    项目保留了无电机 WBC 验证、离线 replay、真机 example、ROS 正式节点和部署文档等多级验证路径。算法可以先在无电机环境中验证，再通过 example 进行真机专项测试，最终接入 ROS 系统运行。这种分阶段迁移方式降低了复杂机器人控制算法从仿真到真机落地的风险。
