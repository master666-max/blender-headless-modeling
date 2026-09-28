# blender-headless-modeling

> **CodeBuddy / WorkBuddy 用户级 Skill** —— Blender 5.2 headless 建模、机验门禁、呈现渲染的完整工作流与踩坑册（真机实测）。

一个让 AI 助手"开箱就会" Blender 5.2 无头建模的技能包：所有结论出自真机项目（Blender 5.2.1 LTS headless + 门禁验收 + 多样性标定），每条坑都带 现象 → 根因 → 修法 与可抄代码。

## 结构

```
blender-headless-modeling/
├── SKILL.md                    # 主册：跑法 + TOP 8 硬坑速查 + 建模 SOP + 呈现配方 + 回退范式
└── references/
    ├── pitfalls.md             # API 坑全量册（103 行，现象→根因→修法，含代码片段）
    └── calibration.md          # 标定结论：门禁判据总表 / 多样性双尺 / WAL 回退链 / 生态杂项
```

## 覆盖什么

- **Headless 跑法**：`blender -b --factory-startup -P script.py`、清场骨架、Windows venv 布局
- **5.2 API 变更**：`display.shading` 单入口、`color_type 'SINGLE'`、`BLENDER_EEVEE_NEXT`、shade_auto_smooth
- **坐标语义**（最坑）：`to_mesh()`/`BVHTree` 是本地坐标 → origin 归零范式
- **布尔与曲线融合**：EXACT solver、刀穿面防共面、附属体嵌入、口沿 bevel 防击穿
- **机验门禁**：BVHTree 穿透射线（壁厚/底厚/指孔）+ 散度定理体积质心 + 渲染谓词（非空/方差/**过曝<5%**）
- **目验纪律**：G1 剪影铁门 / 证据在图·判据在册·结论可追溯 / 修正指令必须归因
- **呈现渲染**：10cm 级三点光配方（米级配方贴小目标必爆的教训）、Filmic+MHC、EEVEE→Cycles 降级
- **WAL 回退**：哈希链事件 + 白名单重放 + 三重对账（指纹/逐顶点 sha/链校验）
- **多样性双尺**：128³ 体素 1-IoU + 三轴 EMD（Spearman 0.9639）× LPIPS 8 视角（阈值 t_same=0.0464 / t_diff=0.1875，灰区保守判不同）

## 安装（CodeBuddy / WorkBuddy）

把本目录放到用户级技能目录即可：

```bash
# Windows
git clone https://github.com/master666-max/blender-headless-modeling.git "%USERPROFILE%\.workbuddy\skills\blender-headless-modeling"
```

新会话中凡涉及 Blender headless 脚本 / 建模门禁 / 5.x API 调试，skill 自动触发。

## 来源

蒸馏自 [blender-ai-console](https://github.com/master666-max/blender-ai-console)（Blender AI 建模控制台，LLM 零可执行代码 + plan-compile-verify 闭环）的真机开发过程——每个数字与坑都对应一次真实报错与修复。

## License

[MIT](LICENSE)
