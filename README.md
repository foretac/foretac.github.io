# ForeTac 项目网页

**ForeTac: Steering Robot Actions with Predicted Contact Consequences**

- 在线地址：https://foretac.github.io/
- GitLab 分支：`git.n.xiaomi.com:chenzhiyuan3/vtm.git` → `ProjectPage`
- GitHub 仓库：`git@github-foretac:foretac/foretac.github.io.git` → `main`

---

## 本地预览

```bash
cd ~/projects/foretac_web
python -m http.server 8000
# 浏览器打开 http://localhost:8000
```

---

## 推送部署

```bash
cd ~/projects/foretac_web

# 1. 检查 GitLab 远程是否有新改动（共用仓库，必须先检查）
git fetch origin
git log ProjectPage..origin/ProjectPage --oneline
# 如果有输出 → 远程被改动过，需要先处理冲突

# 2. 提交本地改动
git add -A && git commit -m "描述"

# 3. 推送到两个远程
git push origin ProjectPage        # GitLab
git push github ProjectPage:main   # GitHub (自动部署 Pages)
```

---

## 目录结构

```
foretac_web/
├── index.html              主页面（HTML + 内联 JS）
├── .nojekyll               禁用 Jekyll（GitHub Pages 需要）
├── .gitlab-ci.yml          GitLab Pages CI 配置
├── README.md               本文件
└── static/
    ├── css/
    │   └── style.css       样式表
    ├── images/             图片/SVG 素材
    │   ├── teaser.svg              ✅ 已完成（暗色橙紫流程图）
    │   ├── architecture.svg        ✅ 已完成（推理架构图）
    │   ├── training_pipeline.svg   ✅ 已完成（四阶段训练图）
    │   ├── guidance_mechanism.svg  ✅ 已完成（trust-region 机制图）
    │   ├── main_results.png        ❌ 待补充
    │   ├── ablation.png            ❌ 待补充
    │   └── foresight_quality.png   ❌ 待补充
    └── videos/             视频素材（全部待补充）
        ├── board_real.mp4
        ├── board_viz.mp4
        ├── vase_real.mp4
        ├── vase_viz.mp4
        ├── card_real.mp4
        ├── card_viz.mp4
        ├── chip_real.mp4
        ├── chip_viz.mp4
        └── foresight_pred.mp4
```

---

## 需要补充的视频

所有视频放在 `static/videos/` 目录下。

### 任务 Demo 视频（4 个任务 × 2 = 8 个文件）

每个任务一行两列：左侧是真实机器人执行视频，右侧是实时推理可视化。

| 文件名 | 内容 | 说明 |
|--------|------|------|
| `board_real.mp4` | 擦黑板：真实机器人执行 | 录制 ForeTac 引导下的完整 rollout |
| `board_viz.mp4` | 擦黑板：实时推理可视化 | 画面包含 marker 位移 + 预测触觉 + energy score 实时变化 |
| `vase_real.mp4` | 擦花瓶：真实机器人执行 | 同上 |
| `vase_viz.mp4` | 擦花瓶：实时推理可视化 | 同上 |
| `card_real.mp4` | 刷卡：真实机器人执行 | 同上 |
| `card_viz.mp4` | 刷卡：实时推理可视化 | 同上 |
| `chip_real.mp4` | 夹薯片：真实机器人执行 | 同上 |
| `chip_viz.mp4` | 夹薯片：实时推理可视化 | 同上 |

### 预测可视化视频（1 个文件）

| 文件名 | 内容 | 说明 |
|--------|------|------|
| `foresight_pred.mp4` | Foresight 预测质量 | 展示 predicted vs actual marker displacement field 随时间演化 |

### 可选：Teaser 视频（备选方案）

| 文件名 | 内容 | 说明 |
|--------|------|------|
| `teaser.mp4` | 方法流程动画 | 从左到右逐步展示 Obs→Policy→Foresight→Energy→Guide→Execute |

如果提供了 `teaser.mp4`，可以替换当前的静态 SVG teaser。

---

## 视频规格要求

| 属性 | 要求 |
|------|------|
| 格式 | MP4, H.264 编码 |
| 分辨率 | 720p (1280×720) 或 1080p (1920×1080) |
| 时长 | 10-30 秒，能展示完整一次 rollout |
| 帧率 | 30fps |
| 音频 | 无（网页播放时 muted） |
| 文件大小 | 每个尽量 < 20MB（网页加载速度） |

### 录制建议

- **real 视频**：固定机位录制机器人执行全程，确保画面稳定
- **viz 视频**：用 serving 代码的可视化输出录屏，包含：
  - 实时 marker offset 箭头图
  - Foresight 预测的未来触觉 latent 可视化
  - Energy score 数值/曲线
  - Trust-region refinement 前后的 action 对比（可选）
- **foresight_pred 视频**：双栏对比——左侧 predicted marker field，右侧 ground-truth marker field，随时间步推进

### 视频压缩命令（如果文件过大）

```bash
ffmpeg -i input.mp4 -vcodec libx264 -crf 23 -preset medium -an -vf scale=1280:720 output.mp4
```

---

## 需要补充的图片

| 文件名 | 内容 | 何时补充 |
|--------|------|----------|
| `main_results.png` | 主对比表/柱状图（6 个方法 × 4 个任务） | 实验跑完后 |
| `ablation.png` | 消融实验表 | 实验跑完后 |
| `foresight_quality.png` | Foresight 预测精度图（latent MAE / cosine sim） | 实验跑完后 |

---

## 修改记录

- 2026-07-08：更新 README，匹配实际任务（擦黑板/擦花瓶/刷卡/夹薯片）
- 2026-07-08：全面视觉重设计（深色 hero + 橙紫主题）
- 2026-07-02：添加对比表到飞书文档
- 2026-06-25：初版创建
