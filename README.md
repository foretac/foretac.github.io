# ForeTac Project Page

项目网页：**ForeTac: Steering Robot Actions with Predicted Contact Consequences**

## 本地预览

```bash
cd ProjectPage
python -m http.server 8000
# 浏览器打开 http://localhost:8000
```

## 部署

推送到 GitHub 仓库后，开启 GitHub Pages（Settings → Pages → Source: root of `main` branch 或 `/docs`），或推送到独立的 `foretac.github.io` 仓库。

---

## 需要补充的素材

### 图片（放入 `static/images/`）

| 文件名 | 内容 | 优先级 | 建议尺寸 |
|--------|------|--------|----------|
| `teaser.png` | ForeTac pipeline 示意图（obs → DP → action → foresight → energy → refine → execute），一图看懂方法 | **P0 必须** | 1600×600 或 16:6 |
| `architecture.png` | 详细方法架构图：TacVAE + Foresight Transformer + Energy Scorer + Trust-Region Refiner | **P0 必须** | 1600×800 |
| `training_pipeline.png` | 四阶段训练流程：Stage ①②③④ 以及模块冻结关系 | P1 推荐 | 1400×500 |
| `guidance_mechanism.png` | Trust-region refinement 过程可视化：action chunk 修正前后 + 能量梯度方向 | P1 推荐 | 1200×600 |
| `main_results.png` | 主对比实验表格/柱状图（success rate across tasks） | **P0 必须** | 1200×500 |
| `ablation.png` | 消融实验表格 | P1 推荐 | 1200×400 |
| `foresight_quality.png` | Foresight 预测精度：predicted vs GT tactile latent 对比 | P1 推荐 | 1400×500 |
| `favicon.png` | 网页 favicon，可以用 ForeTac logo 缩略版 | P2 可选 | 32×32 或 64×64 |

### 视频（放入 `static/videos/`）

| 文件名 | 内容 | 优先级 | 建议格式 |
|--------|------|--------|----------|
| `board_foretac.mp4` | Board wiping：ForeTac guidance 下的 rollout | **P0 必须** | mp4, H.264, 720p, <30s |
| `board_baseline.mp4` | Board wiping：Base DP (无 guidance) 的 rollout 对比 | **P0 必须** | 同上 |
| `insertion_foretac.mp4` | Peg insertion：ForeTac guidance 下的 rollout | **P0 必须** | 同上 |
| `insertion_baseline.mp4` | Peg insertion：Base DP 的 rollout | **P0 必须** | 同上 |
| `guidance_viz.mp4` | Guidance 过程可视化：action trajectory + predicted tactile + energy score 随 refinement step 变化 | P1 推荐 | 同上 |

### 论文/链接

| 项目 | 当前状态 |
|------|----------|
| Paper PDF link | 占位（`#`），论文上传后替换 |
| arXiv link | 占位（`#`），提交后替换 |
| GitHub Code link | 占位（`#`），代码公开后替换 |
| 作者名 + 机构 | 匿名占位，去匿名后替换 |
| Venue badge | 当前写 "Under Review"，接收后改为会议名 |

---

## 素材制作建议

### Teaser 图
- 建议用 Figma/PPT 画一个左到右的 pipeline 图
- 风格参考 TacForeSight 的 overview figure
- 关键元素：Robot arm + 触觉传感器 → DP 生成 action → Foresight 预测触觉 → Energy 打分 → 梯度修正 → 执行

### 视频录制
- 从 `for_show_xiaomi/` serving 代码 rollout 时录屏
- 建议画面包含：机器人画面 + 实时 marker 可视化 + guidance report
- 对比视频建议同角度、同起始条件

### 实验结果图
- 建议用 matplotlib/seaborn 出图后导出 PNG
- 表格可以直接 LaTeX 渲染后截图，或用代码生成 SVG

---

## 目录结构

```
ProjectPage/
├── index.html              主页面
├── .nojekyll               GitHub Pages 禁用 Jekyll
├── README.md               本文件
└── static/
    ├── css/
    │   └── style.css       样式表
    ├── images/             图片素材（待补充）
    │   ├── teaser.png
    │   ├── architecture.png
    │   └── ...
    └── videos/             视频素材（待补充）
        ├── board_foretac.mp4
        ├── board_baseline.mp4
        └── ...
```

## 修改记录

- 2026-06-25：初版创建，所有图片/视频为占位，结构完成
