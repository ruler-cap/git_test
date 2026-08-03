# Test 1 / Test 2 / Test 3：TSID 任务配置对比

本文比较 `python/test_demo/tsid_pick_place` 修改前后的 TSID 任务配置：

- **Test 1（修改前）**：右手是 level-1 软任务，底盘 XY 只有速度阻尼，yaw 会朝目标方向规划。
- **Test 2（修改后）**：右手位置和姿态升为 level 0；底盘高代价保持启动位姿，不再主动朝向目标。
- **Test 3（旧底盘参数 + 新固定参考）**：保留 Test 2 的右手 level-0、CoM、torso、posture、governor 和启动位姿固定参考，仅恢复底盘 `kp/kd/weight`。这不是完整恢复 Test 1。

由于 `python/test_demo/tsid_pick_place/` 当前没有被 Git 跟踪，Test 1 参数来自修改前读取并保留在会话记录中的配置，不是由 `git diff` 恢复。

## 1. 任务配置对比

| TSID 任务 | Test 1（修改前） | Test 2（强启动位姿弹簧） | Test 3（旧底盘参数 + 固定参考） | 主要变化 |
|---|---|---|---|---|
| 右手位置 | level 1；`kp=70`；`weight=15` | level 0；`kp=70`；`weight=1` | 同 Test 2 | 从可妥协的软任务变成硬等式 |
| 右手姿态 | level 1；`ori_kp=40`；`ori_weight=3` | level 0；`ori_kp=40`；`ori_weight=1` | 同 Test 2 | 与位置共同形成硬 6D 末端任务 |
| 左手保持 | level 1；`kp=45`；`weight=0.5` | 不变 | 同 Test 2 | 继续保持启动时的 base-relative 位姿 |
| relative-CoM | level 1；`kp=50`；`weight=4` | level 1；`kp=60`；`weight=12` | 同 Test 2 | 更强地保持 CoM 相对底盘的 XY 偏置 |
| torso 位置 | level 1；`kp=55`；`weight=1` | level 1；`kp=30`；`weight=0.5` | 同 Test 2 | 允许腰部平移更多，以帮助右手伸展 |
| torso 姿态 | level 1；`ori_kp=30`；`ori_weight=0.25` | level 1；`ori_kp=50`；`ori_weight=4` | 同 Test 2 | 更强地抑制躯干倾斜 |
| 底盘 XY | level 1；`kp=0`；`kd=4`；`weight=1` | level 1；`kp=30`；有效 `kd=2√30≈10.95`；`weight=12` | level 1；`kp=0`；`kd=4`；`weight=1` | 恢复纯速度阻尼，但 XY 参考仍固定为启动位置 |
| 底盘 yaw | level 1；默认 `kp=15`；`kd=4`；`weight=1` | level 1；默认 `kp=30`；有效 `kd≈10.95`；`weight=12` | level 1；默认 `kp=15`；`kd=4`；`weight=1` | 恢复旧增益，但 yaw 仍固定为启动方向，不朝目标重规划 |
| `--no-face-target` | 取消面向目标，yaw 仅有 `kd=4` | yaw 的 `kp=0`，但保留有效 `kd≈10.95` | yaw `kp=0`；`kd=4` | CLI 名称保留，取消启动 yaw 弹簧但保留阻尼 |
| posture 腰腿 | level 1；`kp=80`；总 `weight=2` | level 1；`kp=35`；总 `weight=1` | 同 Test 2 | 腰部更容易偏离 home |
| posture 双臂 | level 1；`kp=10`；总 `weight=2` | level 1；`kp=5`；总 `weight=1` | 同 Test 2 | 双臂成为最容易偏离 home 的关节组 |
| 关节速度边界 | level 0；`v_max=0.5 rad/s` | 不变 | 同 Test 2 | 始终是硬约束 |
| 加速度正则 | level 1；`weight=0.01` | 不变 | 同 Test 2 | 始终提供很弱的加速度平滑偏好 |

Test 1 的 torso 主任务 mask 包含位置和姿态六行，同时还单独添加了 torso 姿态任务，因此姿态被重复加入软目标。Test 2 将 torso 主任务改成纯位置 mask `[1,1,1,0,0,0]`，姿态仅由独立的 `ori_kp=50, ori_weight=4` 任务负责。

### 两次右手末端期望目标

Test 1、Test 2 和 Test 3 使用相同的两个右手 `right_grasp_center`
最终位置目标。`target_for()` 直接使用 marker 位置，没有额外的接近或抓取偏移。

| 阶段 | 场景世界坐标 `(x, y, z)` / m | 实际送入 TSID 的 Pinocchio 坐标 `(x, y, z)` / m |
|---|---|---|
| 第一次：front 目标 | `(-0.1800, -2.7900, 0.9450)` | `(-0.1800, -2.7900, 0.8349)` |
| 第二次：side 目标 | `(-0.7200, -2.0500, 0.9450)` | `(-0.7200, -2.0500, 0.8349)` |

两组 z 坐标相差 `0.1101 m`，是 MuJoCo 自由底座启动高度。CSV 中
`box_x/y/z` 记录场景坐标，`target_x/y/z` 记录实际送入右手任务的 Pinocchio 坐标。

### Test 3 的唯一变量

Test 3 的非底盘任务与 Test 2 完全一致。底盘有效参数为：

| 运行模式 | XY `kp/kd` | yaw `kp/kd` | `weight/priority` | 参考生成 |
|---|---|---|---|---|
| 默认 | `0/4` | `15/4` | `1/1` | XY 和 yaw 都固定为启动位姿 |
| `--no-face-target` | `0/4` | `0/4` | `1/1` | XY 和 yaw 都固定为启动位姿 |

目标从 front 切换到 side 时，不更新底盘 XY 参考，也不把 yaw 重规划为面向新目标。因此这组实验隔离的是 chassis `Kp/Kd/weight` 本身，不等同于完整恢复 Test 1 的参考生成逻辑。

## 2. 修改前后的 HQP 层级

### Test 1

```text
Level 0（硬约束）
└── 关节速度边界

Level 1（加权软任务）
├── 右手位置和姿态
├── relative-CoM
├── torso 位置和姿态
├── chassis
├── posture
├── 左手保持
└── 加速度正则
```

右手、CoM、torso 和底盘都在同一软层中竞争。右手权重最大，因此优化器偏向末端精度，但在冲突时仍可留下右手误差来满足稳定性、姿态或关节边界要求。

### Test 2

```text
Level 0（硬约束）
├── 右手位置 3D
├── 右手姿态 3D
└── 关节速度边界

Level 1（只在 Level 0 的可行解空间内优化）
├── relative-CoM         weight 12
├── chassis 启动位姿    weight 12
├── torso 姿态           weight 4
├── posture              weight 1
├── torso 位置           weight 0.5
├── 左手保持
└── 加速度正则           weight 0.01
```

右手 6D 轨迹现在优先于全部姿态偏好。Level 1 只能在不破坏右手硬等式和关节速度边界的前提下，决定使用手臂、腰还是底盘完成动作。

Test 3 的层级与 Test 2 相同，但 level-1 的 chassis 从 `weight=12`
降为 `weight=1`；XY 从启动位置弹簧改为纯速度阻尼，yaw 则改为较弱的启动方向弹簧。

## 3. Test 2 / Test 3 同条件 70 秒 A/B 结果

| 指标 | Test 2（强启动位姿弹簧） | Test 3（旧底盘参数 + 固定参考） |
|---|---:|---:|
| 前 2 s 最大 XY 偏移 | 0.0317 m | 0.1117 m |
| 前 2 s 最大 yaw 偏移 | 0.00870 rad | 0.0721 rad |
| 全程底盘路径 | 2.8337 m | 3.0878 m |
| 全程最大 XY 偏移 | 1.4156 m | 2.3326 m |
| 全程最大 yaw 偏移 | 0.2141 rad | 0.0721 rad |
| 右臂最大偏移 | 1.9370 rad | 0.9608 rad |
| 腰部最大偏移 | 1.6042 rad | 0.0926 rad |
| 最大 relative-CoM 误差 | 0.2933 m | 0.0207 m |
| 最大 torso 姿态误差 | 1.9163 rad | 0.1102 rad |
| 目标到达 | front 未到达；side 未切换 | front 24.356 s；side 未到达 |
| QP failure | 240 | 0 |

两组完整验收都未全部通过。Test 2 在右手 level-0 跟踪后期出现 QP failure；Test 3 消除了 QP failure，且 CoM 和 torso 恢复到当前阈值内，但底盘启动偏移超限且 side 目标未在 70 s 内到达。这些结果只支持底盘参数 A/B 的影响分析，不改变现有验收阈值。

## 4. `kp` 是什么

`kp` 是任务空间或关节空间的**位置误差反馈增益**。当前实现使用临界阻尼规则：

```text
kd = 2√kp
```

一个典型 TSID 位置任务生成的期望加速度近似为：

```text
a_des = a_ref + kp · (p_ref - p) + kd · (v_ref - v)
```

姿态任务采用同样思想，只是位置误差换成 SO(3) 旋转误差。posture 任务则把 `p` 换成关节位置 `q`。

因此，`kp` 主要决定“任务自身希望多快纠正误差”：

- `kp` 越大，同样误差产生的期望加速度越大，响应更快、更硬。
- `kp` 越小，任务允许误差缓慢消除，表现更柔和。
- 大多数任务的 `kd` 按临界阻尼随 `kp` 变化；Test 3 的 chassis 是例外，显式固定为 `kd=4`。
- `kp` 不决定任务层级；高 `kp` 的 level-1 任务仍必须服从 level-0。
- `kp` 也不能保证任务一定实现。如果自由度不足、达到关节速度边界或与同层任务冲突，实际加速度仍会偏离 `a_des`。

例如：

- relative-CoM 从 `kp=50` 提高到 `60`，相同 CoM 偏差会要求更强的腰腿纠偏加速度。
- torso 位置从 `55` 降到 `30`，腰部偏离启动位置时的恢复要求变弱，更愿意参与伸展。
- 底盘从 `kp=0` 改为 `30` 后，底盘偏离启动 XY 会产生明确的回位加速度；原来只会消除底盘速度，不会主动回到启动位置。

## 5. `weight` 是什么

`weight` 是**同一优先级内**任务误差在 QP 目标函数中的代价系数。简化表示为：

```text
minimize Σ weight_i · ‖J_i·ddq - a_des_i‖²
```

其中 `J_i·ddq` 是给定关节加速度 `ddq` 后任务实际获得的加速度，`a_des_i` 是由参考、`kp` 和 `kd` 生成的期望加速度。

`weight` 的影响规则是：

- 同一 level 内发生冲突时，weight 较大的任务通常获得更小的任务加速度残差。
- weight 只表达相对偏好；所有同层 weight 同比例放大，理想数学解通常不变，但数值条件可能变化。
- weight 不能跨越优先级。无论 level-1 weight 多大，都不能牺牲 level-0 来改善自身误差。
- 对 level-0 硬等式而言，weight 不再表达“可以少满足一点”的软权衡。只要问题可行，硬等式必须满足；所以 Test 2 的右手 `weight=1` 不表示右手比原来的 `weight=15` 弱。

例如 Test 2 中 relative-CoM 和 chassis 都是 `weight=12`：在满足右手硬任务后，两者会强烈竞争剩余自由度。torso 位置只有 `weight=0.5`，因此更容易留下位置误差；torso 姿态为 `weight=4`，会比 torso 位置更积极地保持直立。

## 6. `kp` 与 `weight` 的区别

可以把二者概括为：

```text
kp：这个任务希望系统怎样运动，以及误差多快收敛。
weight：多个同层任务无法同时做到时，更愿意听谁的。
priority：某任务是否允许为了低层任务而被破坏。
```

以两个同层任务 A、B 为例：

- 增大 A 的 `kp`：A 会提出更大的纠错加速度要求。
- 增大 A 的 `weight`：QP 会更努力逼近 A 已经提出的加速度要求。
- 把 A 从 level 1 提到 level 0：只要问题可行，B 不再有权通过权衡破坏 A。

所以 `kp` 和 `weight` 不能互相替代。高 `kp`、低 weight 的任务可能要求很激进，但优化器允许它留下较大残差；低 `kp`、高 weight 的任务则会非常忠实地执行一个较柔和的纠偏要求。

## 7. 对最终输出的影响

TSID 首先求出广义加速度 `ddq`，随后积分得到关节位置命令，并从底盘三个虚拟关节提取世界系底盘速度，再转换成四个全向轮的速度命令。因此任务参数最终会影响：

```text
任务参考与实测状态
        ↓ kp/kd
各任务期望加速度
        ↓ priority/weight 组成 HQP
广义加速度 ddq
        ↓ 积分与限幅
身体关节位置命令 + 底盘速度命令
        ↓ 位置伺服和全向轮被控对象
机器人实际运动
```

Test 2 预期表现为：右手轨迹最优先，手臂最容易离开 home，腰部其次；底盘因启动位姿任务的高 weight 而尽量不动，但在满足右手 level-0 任务所需时仍可参与。

不过，右手 6D 和关节速度边界同时位于 level 0。当远距离目标要求的右手任务加速度与 `v_max=0.5 rad/s` 的硬边界不相容时，二者不能通过 weight 妥协，QP 可能不可行。Test 2 的 70 秒 baseline 出现了 240 拍 QP failure，而 Test 3 在同条件下为 0；因此不能把 Test 2 的失败简化为仅由 level-0 结构决定。

此外，Test 2 前 2 秒底盘 XY 最大偏移约 `0.032 m`，仍高于计划的 `0.02 m`。这说明 `chassis weight=12` 只能形成软偏好，并不能表达严格的底盘静止死区。
