---
name: blender-headless-modeling
description: Blender 5.2 headless 建模、机验门禁、呈现渲染的完整工作流与踩坑册（真机实测）。This skill should be used when writing or running Blender headless scripts (bpy / blender -b -P), building models via booleans and curve bevels, designing machine-verified quality gates (silhouette / ray-cast / render predicates), rendering presentation frames, or debugging Blender 5.x API changes (display.shading single entry, EEVEE_NEXT id, BVHTree local coordinates, modifier apply snapshots, blown-highlight predicates). NOT for interactive/GUI-driven Blender sessions or non-Blender 3D tools.
agent_created: true
---

# Blender 5.2 Headless 建模工作流（真机实测版）

所有结论出自真机项目（Blender 5.2.1 LTS headless + 门禁验收 + 多样性标定）。两份参考册：

- `references/pitfalls.md` —— API 坑全量册（含代码片段）。**跑脚本前先扫 TOP 坑**
- `references/calibration.md` —— 门禁判据、多样性双尺标定、WAL 回退范式、生态坑

## 使用边界（触发校准）

- 触发：写/跑 blender -b -P 无头脚本、布尔/曲线建模、机验门禁、呈现渲染 → 用本技能
- 不触发：GUI 交互式操作会话 → 直接指导；非 Blender 的 3D 工具 → 另查

## 环境与跑法

```bash
# headless 跑脚本（--factory-startup 防用户配置污染）
"C:/Program Files/Blender Foundation/Blender 5.2/blender.exe" -b --factory-startup -P script.py

# 脚本内定位仓库根（脚本在 <root>/xxx/ 下时）
ROOT = Path(__file__).resolve().parents[1]

# venv python（Windows 布局——别猜 envs/default/python.exe，Scripts 子目录才是真的）
#   C:/Users/<u>/.workbuddy/binaries/python/envs/default/Scripts/python.exe
```

脚本开头固定骨架：

```python
import bpy, bmesh, json, math, time
from pathlib import Path
from mathutils import Matrix, Vector
from mathutils.bvhtree import BVHTree

bpy.ops.wm.read_factory_settings(use_empty=True)   # 首动作清场
scene = bpy.context.scene
```

## TOP 硬坑速查（跑脚本前必扫；全量+代码见 pitfalls.md）

1. **坐标语义**：`to_mesh()`/`BVHTree` 全是**本地坐标**。建模后立即 origin 归零，否则射线/bbox/质心全错位：
   ```python
   body.data.transform(Matrix.Translation((0, 0, 0.0475)))  # 几何抬到世界
   body.location = (0, 0, 0)                                 # origin 归零 → 本地=世界
   ```
2. **modifier_apply 双坑**：headless 下 poll 需要显式 active（`select_set(True)` + `view_layer.objects.active = ob`）；apply 后 modifier 被移除，旧引用读属性得 0——**apply 前快照参数**。
3. **5.2 API 变更**：`scene.display.render_shading` 已移除（并入 `scene.display.shading` 单入口）；`color_type` 枚举 `'UNIFORM'`→`'SINGLE'`；EEVEE 引擎 id 是 `'BLENDER_EEVEE_NEXT'`（探测后使用）。
4. **布尔**：一律 `solver='EXACT'`；凹腔刀顶必须**穿出面**（共面 = 不融合）；附属体（把手等）端点**嵌入主体 3-4mm**（体积重叠才融合）。
5. **射线采样避让**：穿透采样（hit 后从 hit+ε 继续）前，先算附属件空间范围（端头 = 端点 ± 管半径），采样线整体避开，否则前两个命中是附属件而非主体壁。
6. **渲染前必设 `scene.camera`**；Workbench 吃不到场景灯——剪影走 `display.shading`（FLAT + SINGLE + background_color）。
7. **白瓷/浅色材质**：AgX 压灰 → `view_transform='Filmic'` + `look='Medium High Contrast'`。
8. **光照尺度**：灯距/尺寸/能量按目标尺度配，米级配方贴小目标必爆。10cm 级：灯距 0.5m、size 0.25m、Key ~85W + exposure 旋钮。

## 建模 SOP（逻辑树驱动）

1. **分解**：一句话需求 → 逻辑树。每叶节点 = build 打法 + machine 判据 + visual 备注 三元组，四层递进（容器功能 / 人机 / 形态 / 材质呈现），锚定真实规格（电商实售尺寸）。
2. **几何自洽预检**：写代码前解方程验证参数互不冲突（案例：圆柱内腔与容量/壁厚双门禁矛盾 → 内腔跟随外壁锥度解）。
3. **build 顺序**：主体（锥台/立方）→ EXACT DIFFERENCE 挖腔 → 曲线 bevel 转 mesh + EXACT UNION 融合附属 → bevel 收边（口沿宽 < 沿厚/2，防击穿）→ shade_auto_smooth。
4. **每动作记账**：append JSONL 账本（ts/actor/act/detail/gate/ok），偏离树 build 的打法记"偏离声明"，判据数值永不降格。
5. **门禁行走**：G1 剪影目验（正交 Workbench FLAT 侧视帧；机侧只证非空，"可辨认"留人侧）→ G3 机验（evaluated mesh：射线测壁厚/底厚/指孔 + 散度定理体积质心 + bbox；FAIL 即 raise）→ G4 呈现谓词（非空 + 方差 + 过曝 <5%，渲染后回读像素验）。
6. **目验纪律**：机验绿 ≠ 过关。目验与机验同构——证据在图、判据在册、结论可追溯；修正指令必须引用所见并归因（"地板爆白 = 大面光贴脸"），禁止无归因的"再调调"。

## 呈现渲染配方（10cm 级目标实测）

- 三点光：Key 85W @(0.42,-0.52,0.52) size 0.25 暖光 / Fill 22W size 0.6 冷光 / Rim 260W size 0.2 背后轮廓
- 相机 50mm + DOF f/2.8（focus 挂空物体 TRACK_TO）；TRACK_TO 后**必须** `scene.camera = cam`
- 深棚：world 0.03 + 地板 albedo 0.045 / rough 0.9（rough 0.6 在大面光下拉镜面条亮带）
- Filmic + MHC + exposure −0.4；渲染后回读 luma：`mean>0.01`、`std>0.005`、过曝(luma>0.98)<5%
- EEVEE 优先（~4s/帧），异常降级 Cycles CPU；帧条合成用系统 Python + PIL（Windows 字体 segoeui.ttf）

## 回退范式（WAL + 哈希链）

- 每动作先落 WAL（JSONL 哈希链：append/replay/verify）；回退 = **确定性重放白名单事件（build.*），跳过破坏事件（modify.*）**。
- 对账三重实锤：网格指纹（verts/faces/体积）+ 逐顶点 sha16 + WAL `verify()=True`；破坏帧与体积跳水留档，不装没犯过错。

## 标定结论速查（详见 calibration.md）

- 多样性双尺：几何（128³ 体素 1-IoU + 三轴 W1-EMD，Spearman 0.9639）+ 视觉（LPIPS-alex 8 视角均值，纯推理零训练）；LPIPS 阈值 t_same=0.0464 / t_diff=0.1875，灰区保守判不同。
- torch 不进 Blender（双脚本分工：Blender 侧出图、venv 侧打分）；清华镜像 torch 索引可能空窗 → `pip index versions torch` 诊断后换源，别死磕 install。
