# Blender 5.2.1 LTS Headless 坑册（全量，真机实测）

> 每条都来自真机报错/回执，非文档转述。条目格式：现象 → 根因 → 修法。

## 1. 坐标语义：to_mesh() / BVHTree 是本地坐标（最坑，症状隐蔽）

- **现象**：射线 0 命中或命中位置荒谬；bbox/质心数字离谱。
- **根因**：`obj.evaluated_get(dg).to_mesh()` 与 `BVHTree.FromBMesh(bm)` 返回/消费的都是**网格本地坐标**。若 object origin 不在世界原点（如锥台建模时 `location=(0,0,0.0475)`），本地 z∈[-0.0475,+0.0475]，而射线 origin 按世界语义写在 (0,0,-0.005)——实际落在内腔空中。
- **修法**：建模后立即把几何抬到世界、origin 归零：
  ```python
  from mathutils import Matrix
  body.data.transform(Matrix.Translation((0, 0, 0.0475)))
  body.location = (0, 0, 0)     # 之后 to_mesh()/BVHTree 的本地 = 世界
  ```
- 布尔刀等工具物体的 `matrix_world` 不受影响（布尔在 object 层面解算），无需同改。

## 2. modifier_apply 双坑

- **poll 失败**（headless 常见 `context is incorrect`）：primitive ops 会抢 active object。修法：
  ```python
  def apply_mod(ob, name):
      bpy.ops.object.select_all(action='DESELECT')
      ob.select_set(True)
      bpy.context.view_layer.objects.active = ob
      bpy.ops.object.modifier_apply(modifier=name)
  ```
- **apply 后引用失效**：`modifier_apply` 会把 modifier 从栈里删掉，之后读 `mod.width` 得 0。门禁断言要在 **apply 前快照**：`snap = (mod.width, mod.segments)`。
- 注意：bevel 的 `width` 是单边偏移。口沿厚 4.44mm 时 bevel 2.5 会击穿——先算 `width < 沿厚/2`。

## 3. Blender 5.2 API 变更（4.x → 5.x）

| 变更点 | 旧 | 新（5.2 实测） |
|---|---|---|
| Workbench 渲染着色 | `scene.display.render_shading` | **已移除**，统一 `scene.display.shading`（单入口管视口+渲染） |
| color_type 枚举 | `'UNIFORM'` | `'SINGLE'`（枚举：MATERIAL/OBJECT/RANDOM/VERTEX/TEXTURE/SINGLE） |
| EEVEE 引擎 id | `'BLENDER_EEVEE'` | `'BLENDER_EEVEE_NEXT'`（探测循环后用） |
| auto smooth | `use_auto_smooth` | `bpy.ops.object.shade_auto_smooth(angle=...)`（包 try/except 降级 shade_smooth） |

- 探测引擎 id 的安全写法：
  ```python
  for eid in ('BLENDER_EEVEE_NEXT', 'BLENDER_EEVEE'):
      try:
          scene.render.engine = eid; break
      except TypeError: continue
  ```

## 4. 布尔与曲线融合

- 一律 `solver = 'EXACT'`（FAST 会在融合处出破面）。
- 凹腔刀**顶面必须穿出主体面**（贴面 = 共面 = 不切割）："贴面不融合教训"。
- 曲线把手路线：`curves.new(..., 'CURVE')` → `bevel_depth`（管半径）+ `bevel_resolution` → POLY spline 点列（48 段近似椭圆弧足够光滑）→ link 到 scene → convert(target='MESH') → EXACT UNION。端点**嵌入主体 3-4mm**。
- 曲线 convert 前要 select + active；convert 后对象名取 `bpy.context.active_object`。

## 5. BVH 穿透采样范式

```python
def cast_all(tree, origin, direction, max_d=1.0):
    hits = []
    p = Vector(origin); d = Vector(direction).normalized()
    for _ in range(16):
        loc, _n, _i, _dist = tree.ray_cast(p, d, max_d)
        if loc is None: break
        hits.append(loc.copy())
        p = loc + d * 1e-5          # hit+ε 继续穿透
    return hits
```

- 壁厚 = 同一直线前两命中差；底厚 = 轴线竖直穿透。
- **采样线避让**：附属件（把手端头 = 端点 ± 管半径）占的空间先算出来，采样高度整体避开。实测教训：z=0.022/0.070 两条线穿进把手端头，样本 [4.37, 11.75, 26.3, ...] 直接 FAIL。
- 体积/质心用散度定理（先 `bmesh.ops.triangulate` + `recalc_face_normals`）：
  `V = Σ dot(v0, cross(v1,v2))/6`；`C = Σ(v0+v1+v2)·d / (24·V)`。单位：m³ ×1e6 = ml（**不是** ×1e9，那是 mm³）。

## 6. 渲染管线

- **BMCP-ERR-007**：不设 `scene.camera` 渲染报 No camera found。TRACK_TO 相机创建后必须回写。
- Workbench 剪影（G1 目验帧）：
  ```python
  scene.render.engine = 'BLENDER_WORKBENCH'
  rs = scene.display.shading          # 5.2 单入口
  rs.light = 'FLAT'; rs.color_type = 'SINGLE'
  rs.single_color = (0.10, 0.10, 0.12)
  rs.background_type = 'VIEWPORT'; rs.background_color = (0.92, 0.92, 0.94)
  # 正交相机：type='ORTHO', ortho_scale 按目标宽 ×1.3；rotation (pi/2,0,0) 侧视
  ```
- 像素回读：`bpy.data.images.load` → `img.pixels.foreach_get(np.empty)` → reshape(-1,4) → luma 点积 [0.299,0.587,0.114] → 用完 `bpy.data.images.remove(img)`。
- EEVEE headless 需要 GPU/EGL；失败降级 `engine='cycles'` + `cycles.device='CPU'`。
- 渲染恢复纪律（diff 管线互证）：渲染后恢复 engine/camera/film_transparent/resolution——呈现档不得污染确定性 diff 状态。

## 7. 光照尺度（米级配方 ≠ 小目标配方）

- 米级 rig（灯距 2×radius、300W、size 2m）照 10cm 目标 = 贴脸爆白（全画面 luma 0.80、过曝 27%+）。
- 10cm 级实测配方：Key 85W @(0.42,-0.52,0.52) size 0.25 暖(1.0,0.96,0.90) / Fill 22W size 0.6 冷(0.85,0.9,1.0) / Rim 260W size 0.2 正后上。
- 地板：albedo 0.045 + rough 0.9（rough 0.6 会有 key 镜面长条亮带）；world 0.03 深棚。
- 曝光收敛顺序：先加**过曝谓词**（blown<5% 进门禁）→ 再调 energy/size → 最后 exposure 旋钮。三轮实测：27.6% → 11.7% → 2.7%。

## 8. 环境杂项

- venv python：Windows 布局 `envs/default/Scripts/python.exe`（不是 `envs/default/python.exe`）。
- Bash 的 `/tmp` 与 Windows Python 的 `/tmp` **不是同一目录**（Git Bash /tmp → %TEMP%，Python 解析成 C:/tmp）——中间文件一律写工作区绝对路径。
- GitHub release 大文件：`gh-proxy.com/https://github.com/...` 可用；最后 5% 掉速属服务端特性，Range 续传。
- 清华镜像 torch 索引空窗：`pip index versions torch` 快速诊断（返 none 即空窗），换阿里云镜像，别死磕 install 重试。
- LPIPS AlexNet 权重 233MB 首跑自动下载；**torch 不进 Blender**（Blender Python 与 venv 分离，双脚本走文件交接）。
- PIL/Blender 分工：Blender 出帧，系统 Python + Pillow 12.x 拼帧条（Windows 字体 segoeui.ttf / arialbd.ttf，ImageFont.truetype 30px）。
