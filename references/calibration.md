# 标定结论与工程纪律（真机实测数字）

## 1. 质量门判据总表（马克杯实例：350ml / 高95 / 口径80）

| 层 | 判据（machine，不可降格） | 实测 | 采样法 |
|---|---|---|---|
| 1.1 容量 | ≥ 320ml | 328.8 | 内腔锥台解析 π·h/3·(R1²+R1R2+R2²)，R2 取刀在杯口处的实际半径 |
| 1.2 壁厚 | p05..p95 ⊆ [4,7]mm | [4.37,4.44] | evaluated mesh BVHTree 水平射线穿透，同线前两 hit 差 |
| 1.3 底厚 | ∈ [8,10]mm 且体积>0 | 8.0 | 轴线竖直穿透（origin 外 → 内底两 hit）+ 散度定理体积 |
| 2.1 指孔 | ≥ 28mm | 32.0 | 把手弧顶高度水平穿透：把手内缘 − 杯外壁 |
| 2.2 管径 | ∈ [12,16]mm | 14.0 | y 向横穿弧顶两壁差（= bevel_depth×2） |
| 2.3 跨度 | ∈ [28,72]% 杯高 | 30–70 | 解析（端点 z ± 半短轴）；融合后无缝故按 build 参数断言 |
| 2.4 总宽 | ∈ [125,150]mm | 125.0 | 全 mesh bbox x 范围 |
| 3.1 高径比 | ∈ [1.09,1.29] | 1.188 | bbox z / bbox y（y 向不含把手） |
| 3.3 质心 | xy 投影 ⊂ 底面圆 | 6.26mm | 散度定理：V=Σdot(v0,cross(v1,v2))/6，C=Σ(v0+v1+v2)·d/(24V)（先 triangulate） |
| 4.1 材质 | 参数在册 | rough .32/coat .8 | apply 前快照 + 节点实读回 |
| 4.2 渲染 | 非空+方差+过曝<5% | luma .63/.34，2.7% | PNG 回读 numpy：mean/std/((l>0.98).mean) |

**目验门**：G1 剪影（正交 Workbench FLAT 侧视，512²；机侧仅证前景占比>8%，"剪影即目标"留人侧）；G2 细节清单（把手段差/口沿高光/底足站姿，逐项目验不接受整体印象）。

## 2. 多样性双尺标定（同 prompt N 产出两两比较）

- **几何尺**：128³ 体素 1-IoU + 三轴 W1-EMD。参数级域 Spearman(V,L)=0.9639。
- **视觉尺**：LPIPS(alex) 8 视角（方位角 45°×8 / 仰角 30° / 距离冻结）batch 一次前向均值。**纯推理零训练**——预训练骨干+人眼校准线性层，喂图出分。
- **三档分离**（28 对实测）：unchanged≈0 / near ∈[0.0112,0.0464] / far ∈[0.1875,0.3114]，严格 max-min 分离（4 倍间隔）。
- **阈值**：t_same=0.0464 / t_diff=0.1875，灰区保守判不同。
- **诚实登记**：全域 Spearman(V,L)=0.7203 <0.80 未过——跨模态排序非精确（体素尺对"同体积异形"钝感，视觉尺剪影互补）。两把尺互补使用，不互相替代。
- **双脚本分工**：torch 不进 Blender。Blender 侧出 8 视角 PNG + 几何 V（JSON 落盘），venv 侧 LPIPS batch 前向（28 对 ~2.5s）。跨脚本 JSON 交接时**字段结构显式对账**（读侧 KeyError = 写侧包装层约定没对齐）。
- LPIPS AlexNet 骨干权重 233MB 首跑自动下载，之后离线。

## 3. 渲染 diff 管线（确定性地面真值）

- 相机按求值网格 bbox 拟合后**冻结**；Workbench + FLAT 确定性渲染，同场景重渲 **0 像素漂移** → diff 门禁零误报。
- 三度量：dHash / pHash / SSIM，阈值经三档变体（未变/小改/大改）标定。
- **呈现档与 diff 档两档并存**：呈现帧给人看（EEVEE 三点光+DOF），diff 帧给机器（Workbench 确定性）。呈现渲染后**必须恢复** engine/camera/film_transparent/resolution，防污染 diff 状态。

## 4. WAL 回退事件链样例

```json
{"seq": 0, "kind": "checkpoint", "payload": {"stage": "A"}, "prev": "000…0", "hash": "sha256…"}
{"seq": 1, "kind": "build.body",  "payload": {"op": "cone+cavity"}, "prev": "<seq0.hash>", "hash": "…"}
{"seq": 2, "kind": "build.handle","payload": {"op": "union"},       "prev": "<seq1.hash>", "hash": "…"}
{"seq": 3, "kind": "modify.oops", "payload": {"op": "gouge"},       "prev": "<seq2.hash>", "hash": "…"}
```

- 回退 = `replay()` 取事件 → 只执行 `kind.startswith("build.")`，跳过 `modify.*` → 指纹对账（verts/faces/volume + 全量坐标 sha16）→ `verify()` 哈希链。
- 同进程同参数重建是确定性的：对账要求**严格相等**（不是近似）。

## 5. 生态与工程杂项（Windows）

- **路径**：Git Bash 的 `/tmp` ≠ Windows Python 的 `/tmp`（前者映射 %TEMP%，后者解析成 C:/tmp）——跨工具中间文件一律写工作区绝对路径。
- **pip 镜像**：清华镜像 torch 索引可能空窗（install 报 No matching distribution）→ 先 `pip index versions torch` 快速诊断，再换阿里云源；别死磕 install 重试。
- **帧条合成**：系统 Python + Pillow；Windows 字体 `C:/Windows/Fonts/segoeui.ttf`（中文环境用微软雅黑 msyh.ttc 需另配），`ImageFont.truetype(size=30)`。
- **git push**：SSH port 22 被拒时 `git remote set-url --push origin https://...` 切 HTTPS（fetch/push 可混合配置）。
- **验收纪律**：断言数字全部落盘 result JSON；负结果（未过/未采纳）也是交付，带数据留档。
