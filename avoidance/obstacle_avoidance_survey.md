# 机器人障碍物表示、碰撞检测与避障：选型调研

> 调研日期：2026-07-30。对象：轮式、足式、人形与飞行机器人；同时覆盖静态和动态障碍。  
> 本文的“开销”是算法量级和相对等级，不是不同论文在不同硬件上不可直接比较的毫秒数。

## 结论先行

没有一种表示或避障器适合所有平台。工程上最稳妥的组合通常是：**原始传感器量测 → 局部几何/占据表示 → 距离或连续碰撞查询 → 带动力学约束的局部规划器**。对象级检测、跟踪和学习策略应增强它，而不应取代最后一道独立的安全制动/碰撞检查。

|需求|首选表示与避障|理由|主要代价|
|---|---|---|---|
|低算力、近场紧急避障|激光极坐标/局部 2D 栅格 + VFH 或 DWA|只处理传感器可见范围；实现成熟|窄通道、死胡同和遮挡下易局部最优|
|通用地面 AMR|滚动 2D 代价地图/ESDF + TEB 或 DWA；动态物体加跟踪|计算、可解释性和实时性平衡最佳|二维假设不处理悬空物/台阶|
|复杂 3D 空间（无人机、机械臂/人形上半身）|稀疏 TSDF/ESDF + 轨迹优化/MPC|距离和梯度查询快，适合连续轨迹|体素分辨率提高时内存急剧增长|
|多人、多机器人相互让行|对象轨迹 + VO/ORCA，必要时加 MPC|直接在速度空间处理相对运动；ORCA 可给局部互惠安全条件|依赖状态与短期运动预测；非互惠人类会破坏假设|
|语义障碍、非结构化地形|相机/点云检测分割 + 可通行性/对象地图 + 安全规划器|能区分“可跨越、不可碰、可交互”|训练数据、GPU、域偏移和验证成本高|

## 统一的开销口径

令 `N` 为局部栅格/体素数，`P` 为单帧点数（或图像像素数，须注明），`V` 为本周期活跃/受影响体素数，`M` 为正在跟踪的动态对象数，`K` 为候选控制/轨迹数，`H` 为预测时域离散步数，`S` 为采样树节点数，`d` 为机器人配置空间维数。对学习模型另记录网络 MACs。等级是典型在线负担：极低 < 低 < 中 < 高 < 极高。

“内存”只指运行期主要数据结构；相机深网还须加模型参数和特征张量。“计算”包含在线更新与一次规划周期，不含离线训练，除非特别指出。

## 近期代表工作（2017–2024）

下表以完整链路横向比较近期工作；`P`、`V`、`M`、`H` 和 MACs 是应随平台一并报告的规模量，不能用不同硬件上的毫秒数直接排名。除非另注，论文所报运行频率/加速比仅是其测试设置中的结果。

|方向与工作|传感器/检测输入|障碍物表示|避障或安全输出|在线计算瓶颈与适用边界|
|---|---|---|---|---|
|点云目标检测：[PointPillars (Lang et al., 2019)](https://openaccess.thecvf.com/content_CVPR_2019/html/Lang_PointPillars_Fast_Encoders_for_Object_Detection_From_Point_Clouds_CVPR_2019_paper.html)|LiDAR 点云 `P` → pillar 特征|柱状 BEV 伪图像与 3D box|供对象跟踪、预测或规划使用；本身不产生安全动作|pillar 化近似 `O(P)`，主要成本为 BEV CNN MACs；原文在其硬件上报 62 Hz、快速版 105 Hz。适合有 GPU 的对象级感知，漏检仍须由几何占据兜底。见[§2.1](#21-从传感器到障碍物)。|
|检测、跟踪与速度估计：[CenterPoint (Yin et al., 2021)](https://openaccess.thecvf.com/content/CVPR2021/html/Yin_Center-Based_3D_Object_Detection_and_Tracking_CVPR_2021_paper.html)|LiDAR 点云/BEV 特征|物体中心、尺寸、朝向、速度和 track|检测与最近邻关联形成动态对象状态，供时空碰撞检查|BEV 网络主导；朴素关联约 `O(M²)`，实际常因目标数小而可控；原文在其设置中报 13 FPS。适合动态对象支路，不替代自由空间/占据图。见[§2.1](#21-从传感器到障碍物)、[§2.2](#22-碰撞检测不是单一点查询)。|
|稠密几何与距离场：[nvblox (Millane et al., 2024)](https://doi.org/10.1109/ICRA57147.2024.10611532)|深度或 LiDAR|GPU 哈希 TSDF、occupancy 和 ESDF|`O(1)` 距离/梯度查询，供连续碰撞和轨迹优化|更新随可见/受影响体素 `V` 增长；原文报告 GPU 相对 CPU ESDF 更新最高 31× 加速。适合 CUDA/GPU 平台，代价是显存、同步和硬件依赖。见[§1](#1-障碍物如何表示)。|
|免 ESDF 局部避障：[EGO-Planner (Zhou et al., 2021)](https://arxiv.org/abs/2008.08835)|占据地图/点云与引导路径|只为发生碰撞的轨迹段抽取必要障碍信息|B 样条局部优化与动力学可行性调整|成本随控制点、碰撞段和迭代次数增长，避免维护完整 ESDF；论文实现报约 1 ms 规划时间。适合高速四旋翼局部重规划，不提供全局距离图。见[§1](#1-障碍物如何表示)、[§3](#3-如何避开方法时间与资源对比)。|
|未知环境安全规划：[FASTER (Tordesillas et al., 2020)](https://arxiv.org/abs/2001.04420)|局部已知自由空间与未知空间|安全走廊、激进轨迹及已知自由空间中的备份轨迹|MIQP 同时生成可探索轨迹和始终可执行的安全备份|时域离散、走廊数与整数变量决定成本；最坏情形高于连续优化。适合高速未知环境；安全论断依赖已知自由空间和备份轨迹均正确。见[§3](#3-如何避开方法时间与资源对比)。|
|动态障碍/多机协同：[MADER (Tordesillas & How, 2021)](https://arxiv.org/abs/2010.11061)|静态地图、动态物体与其他机器人的承诺轨迹|每个轨迹区间的外包多面体|优化分离平面并重检轨迹，实现异步 3D 多机避碰|每个对象、轨迹段和预测时域增加约束；优化成本通常高于 ORCA。适合可交换承诺轨迹的 UAV，非合作目标和通信失效仍需几何检查。见[§3](#3-如何避开方法时间与资源对比)。|
|遮挡动态障碍：[OA-MPC (Firoozi et al., 2023)](https://arxiv.org/abs/2211.09156)|可见目标跟踪、LiDAR/相机遮挡区|遮挡区潜在行人的前向可达集|把可达集纳入 NMPC，并以终端停车约束保持递归可行|可达集传播、预测时域 `H` 和 NMPC 求解是附加成本。适合“看不见不等于安全”的室内/人机混行；保证依赖可达模型和感知遮挡建模。见[§2.2](#22-碰撞检测不是单一点查询)、[§3](#3-如何避开方法时间与资源对比)。|
|语义可通行性：[WayFAST (Gasparino et al., 2022)](https://arxiv.org/abs/2203.12071)|RGB-D 与在线牵引力状态估计|自监督预测的可通行性/代价图|MPC 规避打滑、下陷等“无碰撞但不可走”区域|网络 MACs 加在线状态估计和 MPC；比纯几何地图昂贵。适合雪、沙、泥等野外地形，域偏移时仍须以几何碰撞层保护。见[§1](#1-障碍物如何表示)、[§3](#3-如何避开方法时间与资源对比)。|
|安全过滤：[CBF-QP (Ames et al., 2017)](https://doi.org/10.1109/TAC.2016.2638961)|距离/安全集与上层控制命令|由障碍距离定义的控制 barrier 约束|QP 将上层动作投影到安全控制集合|低维底盘多为小 QP，约束数随障碍数增加。适合作为 MPC 或学习策略的末端过滤；不能替代感知和未知区策略。见[§3](#3-如何避开方法时间与资源对比)。|
|学习式人群避障：[SARL (Chen et al., 2019)](https://arxiv.org/abs/1809.08835)|机器人和人群相对状态|注意力聚合的人群交互状态|RL 输出局部避障动作|在线推理固定于网络 MACs，通常低于显式长时域优化；离线训练昂贵且没有几何安全保证。适合社交导航，应叠加 CBF 或急停。见[§3](#3-如何避开方法时间与资源对比)。|

## 1. 障碍物如何表示

|表示（代表工作，时间）|状态内容与适用范围|更新/查询量级|计算|内存|优点与限制|
|---|---|---|---|---|---|
|量测点、2D scan、3D 点云|传感器坐标中的点/射线；适合紧急制动、局部聚类|存储 `O(P)`；朴素最近距离 `O(P)`，KD-tree 建树约 `O(P log P)`、查询平均近似 `O(log P)`|低–中|低–中|无建图延迟、保留几何细节；噪声/遮挡未显式表达，时序融合和大点云会增负担|
|概率占据栅格（[Elfes, 1989](https://doi.org/10.1109/2.30720)）|每个 2D/3D cell 的占据概率（常用 log-odds）；轮式局部代价地图的基线|稠密存储 `O(N)`；射线更新与经过 cell 数成正比；单 cell 查询 `O(1)`|低（2D）/中–高（3D）|中（2D）/高–极高（3D）|天然融合多帧与不确定性；分辨率翻倍使 2D 内存约 ×4、3D 约 ×8；未观测与空闲要分开处理|
|八叉树占据图（[Hornung et al., 2013, OctoMap](https://doi.org/10.1007/s10514-012-9321-0)）|多分辨率 3D 占据、空闲和未知空间|节点数 `K_o` 时内存 `O(K_o)`、寻址约 `O(log N)`；射线更新为路径长度乘树高|中|中（稀疏场景）|比稠密 3D 体素省内存，适用于无人机/机械狗；指针与概率节点有常数开销，梯度/距离查询不如 ESDF 直接|
|TSDF/ESDF 距离场（[Oleynikova et al., 2017, Voxblox](https://arxiv.org/abs/1611.03631)；[Han et al., 2019, FIESTA](https://arxiv.org/abs/1903.02144)；[Millane et al., 2024, nvblox](https://doi.org/10.1109/ICRA57147.2024.10611532)）|每个体素保存到表面的截断有符号距离（TSDF）或最近障碍欧氏距离及梯度（ESDF）；nvblox 在 GPU 上维护哈希 TSDF、occupancy 与 ESDF|稠密/哈希体素 `O(N)`；距离查询、梯度 `O(1)`；增量更新随受影响体素 `V` 增长，最坏 `O(N)`|中–高|高|连续优化的碰撞代价最友好；nvblox 以显存和 CUDA 依赖换取更新吞吐，动态障碍频繁出现/消失时仍需控制更新成本|
|几何基元/网格/BVH（[Gottschalk et al., 1996, OBBTree](https://www.cs.unc.edu/~geom/OBB/OBBT.html)；[Pan et al., 2012, FCL](https://doi.org/10.1109/ICRA.2012.6225337)）|圆、胶囊、凸包、三角网格；机器人自身也可精确建模|基元数 `G`；BVH 构建约 `O(G log G)`，平均碰撞/距离查询常为近似 `O(log G)`，最坏仍可线性|低–中（查询）|低–中|对机械臂、人形自碰撞及精确车体 footprint 有利；从稀疏感知量测生成可靠网格较难|
|对象级 4D 状态/轨迹分布（[Yin et al., 2021, CenterPoint](https://openaccess.thecvf.com/content/CVPR2021/html/Yin_Center-Based_3D_Object_Detection_and_Tracking_CVPR_2021_paper.html)）|类别、3D box/姿态、速度、协方差、预测轨迹；人、车、其他机器人|存储 `O(MH)`；滤波/数据关联依检测数而定，朴素最近邻关联约 `O(M²)`，预测模型约 `O(MH)`|中–高|低–中|直接支持社交距离、不同物体行为和速度障碍；检测网络和关联是额外支路，漏检、ID switch、错误预测会直接影响安全，必须配几何兜底|
|语义/可通行性图、神经隐式场（[Gasparino et al., 2022, WayFAST](https://arxiv.org/abs/2203.12071)）|类别、材质、坡度、可通行概率，或神经网络的隐式占据/SDF；WayFAST 用 RGB-D 和在线牵引力估计预测可通行性|取决于网络 MACs 和查询采样数；一般为 `O(Q × 网络推理)`|高–极高|中–高|表达“草地可走、玻璃不可见、桌下可穿”和无碰撞的低牵引风险；需自监督/标注数据并面对域偏移，难提供严格几何安全界|

### 表示选型要点

1. **二维不是低级版本，而是强假设。** 轮式底盘在平整地面、障碍可投影到地面时，滚动 2D 栅格最划算；台阶、悬空桌面、坡度或无人机绕飞时必须保留高度或可通行性。
2. **避障图应是局部、滚动和带时间戳的。** 全局静态图适合路径搜索；局部障碍层需快速遗忘动态物体，不能把移动人永久烙进地图。
3. **规划器需要的不是同一种地图。** DWA/VFH 需要障碍距离或极坐标障碍密度；连续优化最受益于 ESDF 梯度；VO/ORCA 最需要对象的相对速度与半径。

## 2. 如何检测、跟踪并判定碰撞

### 2.1 从传感器到障碍物

|路线|在线主要量级|计算/内存|强项|主要失效模式|
|---|---|---|---|---|
|2D LiDAR：去地/聚类/线段或圆拟合|`O(P)` 到 `O(P log P)`|低–中 / 低|尺度精确、光照不敏感、容易接入栅格|玻璃、黑色吸光材质、低矮/悬空障碍、遮挡|
|3D LiDAR/深度：下采样、地面分割、聚类、跟踪|`O(P)` 到邻域搜索 `O(P log P)`|中–高 / 中|保留高度和形状，适合 3D 机动|点云密度随距离下降；雨雾、反射与自遮挡|
|RGB/RGB-D：检测、分割、单目/立体深度|CNN 推理约与输入像素、层数、通道数成正比；对象后处理与检测数有关|中–极高 / 中–高|语义最强，能识别人体、门、线缆等|光照/纹理/域偏移；单目尺度不确定；端到端漏检不可接受|
|多模态融合|感知端为各支路之和，外加时空标定、关联和滤波|高 / 中–高|互补盲区与置信度，可用视觉语义 + LiDAR 几何|时间同步、外参漂移、冲突量测；工程复杂度高|

推荐把感知输出拆成两条：一条将**保守几何占据**送入安全层；另一条给出对象类别、运动状态和不确定性供预测规划。视觉网络“没看到”的区域不能自动当作自由空间。

以 PointPillars 为例，点云先聚合为 pillar/BEV，再由 2D CNN 回归 3D box；其计算应记作 `O(P) + O(MACs)`，而不是笼统写成“点云检测耗时”。CenterPoint 则进一步回归中心、朝向和速度，并用检测间关联形成轨迹。二者产生的是对象支路：它可补足类别和速度，但不能把未检测到的点云/深度量测从几何安全支路中删除。

### 2.2 碰撞检测不是单一点查询

|判定方式|适合表示|一次查询量级与资源|说明|
|---|---|---|---|
|footprint 膨胀/Minkowski 和|2D 栅格、圆/多边形|栅格膨胀约 `O(N)`（可预处理）；轨迹采样为 `O(H)` 次查图|把机器人半径及安全边界并入障碍，最容易实现；对姿态变化大的非圆形机器人会过保守|
|离散姿态/轨迹检查|任意地图/几何模型|`O(H × C_q)`，`C_q` 是一次距离/碰撞查询成本|必须用足够细的步长，否则两采样点间会穿透；速度高时要按制动距离自适应加密|
|连续碰撞检测（CCD）|凸包、网格/BVH|依几何对数与保守推进迭代数；通常高于离散查询|适合机械臂、人形和高速飞行器，避免 tunneling；实现复杂，需运动模型和几何质量|
|ESDF 安全余量|ESDF|每个状态 `O(1)` 距离/梯度查询，整条轨迹 `O(H)`|可直接构造可微碰撞代价；距离场分辨率、未知区策略决定真正安全性|
|时空碰撞/概率风险|预测轨迹与协方差|约 `O(MH)`；若做多模态预测或联合场景采样则更高|动态障碍必需；应将预测不确定性扩张为时间相关安全边界，而非只用均值轨迹|
|遮挡可达集风险（[Firoozi et al., 2023, OA-MPC](https://arxiv.org/abs/2211.09156)）|可见目标轨迹、遮挡几何与潜在目标动力学|前向可达集传播 + `H` 步 NMPC/碰撞约束；复杂度受可达集表示和求解迭代主导|把“不可见”视为潜在动态障碍，并以终端停车条件维持递归可行；安全界仅覆盖所假设的遮挡区、目标类型和运动界|

安全层应独立计算**可停性**：当前速度下，制动距离加反应延迟、定位误差和感知盲区边界内是否仍无碰撞。DWA 的可达且可安全刹停速度思想正是这一基线 [Fox, Burgard & Thrun, 1997](https://doi.org/10.1109/100.580977)。

## 3. 如何避开：方法、时间与资源对比

|族类与代表工作（时间）|输入/核心机制|单周期主导复杂度|计算|内存|动态障碍与安全性|适用判断|
|---|---|---|---|---|---|---|
|人工势场 APF（[Khatib, 1986](https://doi.org/10.1177/027836498600500106)）|目标吸引、障碍排斥的局部势/梯度|对近邻障碍 `O(M)` 或对图 `O(N)`|极低–低|极低|可对瞬时障碍反应；无全局完备性，局部极小/振荡|只适合简单反应层或与全局规划混合，不能单独承担复杂避障|
|VFH/VFH*（[Borenstein & Koren, 1991](https://doi.org/10.1109/70.88137)；[Ulrich & Borenstein, 2000](https://doi.org/10.1109/70.881174)）|局部栅格投影为角度直方图，选自由方向；VFH* 加前视搜索|构图 `O(N)`，`B` 个方向 bin 搜索 `O(B)`；VFH* 额外依前视节点数|低|低–中|对瞬时动态障碍可反应；没有严格预测安全保证|廉价 LiDAR 平台的优先候选；高维 3D 与复杂动力学不合适|
|DWA（[Fox et al., 1997](https://doi.org/10.1109/100.580977)）|在可达 `(v, ω)` 窗口采样，评估朝向、速度、净空与可刹停性|`O(K × H × C_q)`|低–中|低|对短时动态障碍可重算；安全性只在模型、感知和采样充分时成立|差速/阿克曼轮式机器人首选基线；采样太粗会漏掉可行解，窄通道易保守|
|速度障碍 VO / ORCA（[Fiorini & Shiller, 1998](https://doi.org/10.1177/02783649922066289)；[van den Berg et al., 2011](https://doi.org/10.1007/978-3-642-19457-3_1)）|相对位置、速度、半径形成禁忌速度锥；ORCA 解低维线性规划|邻居筛选后约 `O(M)` 构造约束；ORCA 期望线性 LP [原文讨论](https://gamma.cs.unc.edu/ORCA/publications/ORCA.pdf)|低–中|低|擅长多主体动态互避；局部保证依“相互承担一半责任”、准确状态和有限预测时域|多机器人/无人机编队、人群绕行的强基线；对静态复杂地图需叠加地图规划|
|采样/搜索：RRT、Kinodynamic RRT、RRT*（[LaValle & Kuffner, 2001](https://doi.org/10.1177/02783640122067453)；[Karaman & Frazzoli, 2011](https://doi.org/10.1177/0278364911406761)）|在配置/状态空间采样并做碰撞检查；RRT* 重连趋于最优|`O(S × (log S + C_q))`；高维、窄通道和动态重规划会显著变慢|中–高|中–高|可加入动力学、复杂形体；经典版本不天然处理移动障碍，安全取决于时空扩展与检查|高维人形/机械臂或复杂 3D 可行性强；不宜作为严格周期短的唯一局部控制器|
|轨迹优化/TEB/CHOMP/MPC（[Rösmann et al., 2015](https://doi.org/10.1109/ECC.2015.7331052)；[Ratliff et al., 2009](https://doi.org/10.1109/ICRA.2009.5152817)）|最小化时间、平滑、控制和 ESDF 障碍代价，满足动力学/约束|稀疏问题每轮近似随 `H(d+u)` 和约束数增长；迭代次数 `I` 决定实际耗时，即约 `O(I × 求解器成本)`|中–高|中|能自然加入预测轨迹、舒适性和动力学；非凸问题会陷局部解，硬实时须限时和提供备份控制|追求平滑、贴近约束、动力学一致时首选；建议 ESDF + warm start + 紧急制动后备|
|免 ESDF 轨迹优化（[Zhou et al., 2021, EGO-Planner](https://arxiv.org/abs/2008.08835)）|占据/点云与无碰撞引导路径；仅在碰撞轨迹段提取障碍信息|随 B 样条控制点、碰撞段和迭代次数增长|低–中|低–中|不维护全量 ESDF，碰撞代价由引导路径提供方向；仅覆盖局部优化子空间，仍需独立碰撞重检|高速四旋翼的快速局部再规划；没有全局距离场需求时很合适|
|未知环境的备份安全规划（[Tordesillas et al., 2020, FASTER](https://arxiv.org/abs/2001.04420)）|已知自由空间、未知空间与局部目标；MIQP 联合时段分配|走廊数、离散时域和整数变量共同决定；最坏情形高于连续优化|高|中|激进轨迹可进入未知区，同时维护已知自由空间内的安全备份轨迹|高速探索、不能只靠“在已知自由空间停车”时；需给 MIQP 设时限和可行备份|
|动态多机的多面体分离（[Tordesillas & How, 2021, MADER](https://arxiv.org/abs/2010.11061)）|静态地图、动态对象/其他机器人承诺轨迹；每段轨迹构造外包多面体并优化分离平面|每个对象、时段和分离面增加变量/约束|中–高|中|可表达 3D 动态障碍和异步承诺轨迹；安全依赖通信到的轨迹与重检机制|多 UAV/机器人协调；目标数大或非合作对象多时比 ORCA 更重|
|遮挡感知 NMPC（[Firoozi et al., 2023, OA-MPC](https://arxiv.org/abs/2211.09156)）|可见目标跟踪与遮挡区潜在行人可达集进入 NMPC，终端状态满足停车约束|可达集传播 + `H` 步非线性优化及碰撞子问题|高|中|在遮挡中保持递归可行，而非把无观测误判为空闲|室内人机混行与盲角；可达集的速度/加速度界需保守且可信|
|CBF-QP 安全过滤（[Ames et al., 2017](https://doi.org/10.1109/TAC.2016.2638961)）|上层控制命令和障碍距离/安全集；将 CBF 不等式写入 QP 并投影到安全控制集|低维底盘常为小 QP；约束数约随近邻障碍数增长|低–中|低|可为学习策略或 MPC 增加控制级安全约束；感知漏检、错误距离场和不可行 QP 不会被它自动解决|适合作为最后一层，配合保守障碍输入、松弛处理和急停|
|深度模仿/RL 直接策略（[Tai, Li & Liu, 2016](https://doi.org/10.1109/IROS.2016.7759428)；[Chen et al., 2017](https://arxiv.org/abs/1609.07845)）|scan/图像/对象状态经网络直接输出动作或局部目标|推理约 `O(网络 MACs)`，通常与 `N_px`/点数及模型大小成正比；训练成本远高于推理|中–高（边缘推理）/极高（训练）|中–高|可学习社交与难建模策略；单独使用通常没有可验证安全保证，受 sim-to-real/长尾影响|适合把语义、行为偏好或局部启发式注入系统；部署时必须用速度/距离安全滤波器约束输出|
|注意力式学习人群避障（[Chen et al., 2019, SARL](https://arxiv.org/abs/1809.08835)）|机器人与行人的相对状态经注意力聚合，RL 网络输出局部动作|在线成本固定于网络 MACs，通常低于显式长时域优化；训练成本不计入控制周期|低–中（推理）/高（训练）|中|能表达不同人对当前决策的重要性；没有对未检测人、几何障碍或长尾交互的安全保证|社交导航的行为层；应接 CBF、可停性检查或急停层|
|学习感知 + 安全 MPC/CBF/规划器|网络产生语义、可通行性、预测或代价，几何安全模块裁决|网络推理 + `O(MH)` 预测 + 优化/查询成本|高|中–高|在性能与可审计性间最实用；安全结论仍取决于保守占据、误差界和后备控制|研究与产品推荐方向，尤其是人机共享场景；接口必须显式传递置信度和未知区|

## 4. 静态与动态场景的推荐流水线

```text
LiDAR / 深度 / 相机 / 触觉
          │
          ├─ 保守几何支路：局部滚动占据图 → 膨胀 / ESDF → 独立碰撞与可停性检查
          │                                                    │
          └─ 语义动态支路：检测/分割 → 多目标跟踪 → 多模态轨迹与协方差预测
                                                               │
全局任务路径 ─────────────────────────────────────→ 局部规划（DWA / TEB / MPC / VO）
                                                               │
                                      安全过滤、急停/降速 ← 控制命令
```

- **平面轮式机器人**：以激光更新滚动 2D occupancy/costmap；把车体 footprint 和定位误差膨胀进去。动态对象以 2D box/circle + 速度/协方差表示，局部规划用 DWA/TEB；人群密集时增添 VO/ORCA 或预测约束。
- **机械狗、人形**：环境障碍仍可用 3D ESDF，但碰撞体应为每个 link 的 capsule/凸包；规划状态须含关节与接触可行性。地图避障和自碰撞查询应分层，后者不能用仅 2D 地图代替。
- **无人机**：局部稀疏 TSDF/ESDF 加连续轨迹优化较合适；将速度、姿态误差和制动/转向能力转成时空安全裕量。对相向飞行器可用 3D ORCA，但仍须处理非合作目标和感知盲区。

四类压力场景可直接检验选型是否自洽：静态稠密空间优先比较 ESDF/nvblox 与 EGO-Planner 的更新和再规划延迟；动态横穿比较 CenterPoint 对象支路与 `O(MH)` 时空约束；遮挡突现必须测试 OA-MPC 式可达集或可停性后备；雪、沙、泥等非结构化地面则比较 WayFAST 的可通行性预测与纯几何路线。四类场景中，保守 occupancy/ESDF、停车约束或 CBF 都应保留为独立几何安全支路。

## 5. 评测：不要只报“成功率”

统一用同一地图、传感器噪声和机器人动力学，对每条方案报告：

1. **安全**：碰撞率、最小净空、TTC（time-to-collision）违规、急停次数，以及漏检/预测误差下的最坏情形。
2. **任务**：到达率、路径长度与时间、平均速度、重规划次数；人机共处另报舒适距离和不必要干扰。
3. **资源**：峰值 RAM、地图大小、每模块延迟的 p50/p95/p99、CPU/GPU 利用率；随 `N`、`P`、`M`、`H` 递增的曲线。
4. **场景**：静态密集障碍、窄通道/U 形死胡同、横穿与相向动态目标、遮挡后突然出现、玻璃/弱反射和 3D 悬空障碍。训练方法另加未见环境、光照/天气变化和传感器掉帧。

## 6. 参考文献时间线（代表而非穷尽）

|年份|工作|在本调研中的位置|
|---:|---|---|
|1986|[Khatib, *Real-Time Obstacle Avoidance for Manipulators and Mobile Robots*](https://doi.org/10.1177/027836498600500106)|人工势场反应式避障|
|1989|[Elfes, *Using Occupancy Grids for Mobile Robot Perception and Navigation*](https://doi.org/10.1109/2.30720)|概率占据表示|
|1991|[Borenstein & Koren, *The Vector Field Histogram*](https://doi.org/10.1109/70.88137)|直方图式局部避障|
|1996|[Gottschalk, Lin & Manocha, *OBBTree*](https://www.cs.unc.edu/~geom/OBB/OBBT.html)|层次包围盒碰撞检测|
|1997|[Fox, Burgard & Thrun, *The Dynamic Window Approach to Collision Avoidance*](https://doi.org/10.1109/100.580977)|动力学受限速度采样|
|1998|[Fiorini & Shiller, *Motion Planning in Dynamic Environments using Velocity Obstacles*](https://doi.org/10.1177/02783649922066289)|动态物体的速度空间表示|
|2000|[Ulrich & Borenstein, *VFH+*](https://doi.org/10.1109/70.881174)|带运动学约束的 VFH 改进|
|2001|[LaValle & Kuffner, *Randomized Kinodynamic Planning*](https://doi.org/10.1177/02783640122067453)|动力学采样规划|
|2009|[Ratliff et al., *CHOMP*](https://doi.org/10.1109/ICRA.2009.5152817)|函数空间轨迹优化|
|2011|[Karaman & Frazzoli, *Sampling-based Algorithms for Optimal Motion Planning*](https://doi.org/10.1177/0278364911406761)|RRT* 最优采样规划|
|2011|[van den Berg et al., *Reciprocal n-Body Collision Avoidance*](https://doi.org/10.1007/978-3-642-19457-3_1)|ORCA 多主体互惠避障|
|2012|[Pan et al., *FCL: A General Purpose Library for Collision and Proximity Queries*](https://doi.org/10.1109/ICRA.2012.6225337)|通用几何碰撞/距离查询|
|2013|[Hornung et al., *OctoMap*](https://doi.org/10.1007/s10514-012-9321-0)|稀疏概率 3D 占据图|
|2015|[Rösmann et al., *Timed Elastic Bands*](https://doi.org/10.1109/ECC.2015.7331052)|时间最优局部轨迹优化|
|2016|[Tai, Li & Liu, *A Deep-Network Solution towards Model-less Obstacle Avoidance*](https://doi.org/10.1109/IROS.2016.7759428)|深度端到端避障|
|2017|[Oleynikova et al., *Voxblox*](https://arxiv.org/abs/1611.03631)|增量 TSDF→ESDF|
|2017|[Chen et al., *Decentralized Non-communicating Multiagent Collision Avoidance with Deep RL*](https://arxiv.org/abs/1609.07845)|学习式动态多主体避障|
|2017|[Ames et al., *Control Barrier Function Based Quadratic Programs for Safety Critical Systems*](https://doi.org/10.1109/TAC.2016.2638961)|把安全集约束为控制 QP，作为末端安全过滤|
|2019|[Han et al., *FIESTA*](https://arxiv.org/abs/1903.02144)|快速增量 ESDF|
|2019|[Lang et al., *PointPillars: Fast Encoders for Object Detection From Point Clouds*](https://openaccess.thecvf.com/content_CVPR_2019/html/Lang_PointPillars_Fast_Encoders_for_Object_Detection_From_Point_Clouds_CVPR_2019_paper.html)|pillar/BEV 点云 3D 检测（CVPR）|
|2019|[Chen et al., *SARL: A Deep Reinforcement Learning Based Approach for Crowd Navigation*](https://arxiv.org/abs/1809.08835)|注意力式人群状态表示与局部 RL 避障|
|2020|[Tordesillas et al., *FASTER: Fast and Safe Trajectory Planner for Navigation in Unknown Environments*](https://arxiv.org/abs/2001.04420)|未知空间探索与已知自由空间安全备份轨迹|
|2021|[Yin et al., *Center-Based 3D Object Detection and Tracking*](https://openaccess.thecvf.com/content/CVPR2021/html/Yin_Center-Based_3D_Object_Detection_and_Tracking_CVPR_2021_paper.html)|中心式 BEV 检测、速度估计与跟踪（CVPR）|
|2021|[Zhou et al., *EGO-Planner: An ESDF-free Gradient-based Local Planner for Quadrotors*](https://arxiv.org/abs/2008.08835)|仅提取碰撞段障碍信息的局部 B 样条优化（RA-L/ICRA）|
|2021|[Tordesillas & How, *MADER: Trajectory Planner in Multi-Agent and Dynamic Environments*](https://arxiv.org/abs/2010.11061)|外包多面体和分离平面的异步多机避碰|
|2022|[Gasparino et al., *WayFAST: Navigation with Predictive Traversability in the Field*](https://arxiv.org/abs/2203.12071)|RGB-D、牵引力估计和预测可通行性（RA-L）|
|2023|[Firoozi et al., *OA-MPC: Occlusion-Aware MPC for Guaranteed Safe Robot Navigation with Unseen Dynamic Obstacles*](https://arxiv.org/abs/2211.09156)|遮挡目标可达集、NMPC 与终端停车约束|
|2024|[Millane et al., *nvblox: GPU-Accelerated Incremental Signed Distance Field Mapping*](https://doi.org/10.1109/ICRA57147.2024.10611532)|GPU TSDF/occupancy/ESDF 增量更新（ICRA）|
|2024|[de Carvalho, Simoni & Yoshioka, *Autonomous Navigation and Collision Avoidance for Mobile Robots: Classification and Review*](https://arxiv.org/abs/2410.07297)|近期分类综述，适合补充检索入口|

## 落地时的最小决策清单

在选择算法前，先固定：障碍是否必须 3D、最大速度与可停距离、传感器盲区/最坏漏检、是否需要区分物体语义、动态对象最大数量，以及能接受的控制周期。若这些量未定，不应从论文的单一成功率反推方案优劣。

对一个普通轮式平台，建议先建立“2D 滚动栅格 + footprint 膨胀 + DWA/TEB + 独立急停”的可测基线，再按评测结果增加对象跟踪、ESDF、VO/MPC 或学习感知；这能让每一项额外算力带来的安全/任务收益可量化。
