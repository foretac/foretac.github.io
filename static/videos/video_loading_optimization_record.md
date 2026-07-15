# ForeTac 任务视频加载与同步播放优化记录

## 1. 背景与目标

ForeTac 项目网页在四个任务中分别并排展示真实执行视频（`*_real.mp4`）和触觉可视化视频（`*_viz.mp4`）：

- Board Wiping
- Vase Wiping
- Card Swiping
- Chip Grasping

目标是让每组左右视频加载完成后同步播放，并在媒体加载或缓冲期间显示转圈状态。

## 2. 遇到的问题

网页增加同步播放和加载转圈后，实际访问时仍出现以下现象：

1. 刷新页面后，视频区域先显示预览帧，用户不容易判断视频是否正在加载。
2. 左侧 `*_real.mp4` 通常可以进入可播放状态，右侧 `*_viz.mp4` 经常长时间转圈或无法继续播放。
3. 任一视频发生缓冲时，同组两个视频都会暂停，因此右侧卡顿会表现为整组停止播放。
4. 问题在四组任务上均可能发生，且弱网环境下更明显。

## 3. 排查思路与过程

### 3.1 确认文件是否为有效视频

使用 `ffprobe` 检查八个任务视频，确认它们均为浏览器支持的 MP4 视频：

- 视频编码：H.264 High Profile
- 像素格式：`yuv420p`
- 无音轨
- MP4 的 `moov` atom 位于 `mdat` 之前，支持 Fast Start
- 四组左右视频的源时长分别一致

因此，问题不是文件伪装成视频、编码格式不受支持或 MP4 索引位于文件末尾。

### 3.2 检查服务端响应

检查 GitHub Pages 的媒体响应，确认：

- `Content-Type` 为 `video/mp4`
- 支持 `206 Partial Content`
- 支持 HTTP Range 请求

因此，GitHub Pages 本身能够按字节范围传输视频。

### 3.3 检查网络吞吐和视频码率

一次实际网络测试中，从 GitHub Pages 下载媒体的速度约为 `25 KB/s`，约等于 `0.2 Mbps`。原右侧可视化视频为 1920x1080，平均码率约为 `1.7-2.1 Mbps`，明显高于该次测试中的可用吞吐。

原始 `*_viz.mp4` 属性如下：

| 文件 | 分辨率 | 帧率 | 平均码率 | 时长 | 文件大小 |
| --- | ---: | ---: | ---: | ---: | ---: |
| `board_viz.mp4` | 1920x1080 | 24 fps | 1.73 Mbps | 31.167 s | 6.5 MB |
| `vase_viz.mp4` | 1920x1080 | 24 fps | 1.96 Mbps | 20.917 s | 4.9 MB |
| `card_viz.mp4` | 1920x1080 | 24 fps | 1.86 Mbps | 10.084 s | 2.3 MB |
| `chip_viz.mp4` | 1920x1080 | 约 15.79 fps | 2.11 Mbps | 14.125 s | 3.6 MB |

### 3.4 检查关键帧间隔

使用 `ffprobe -skip_frame nokey` 检查关键帧：

- `board_viz.mp4`：仅 3 个关键帧，最大间隔约 10.417 秒
- `vase_viz.mp4`：仅 3 个关键帧，最大间隔约 10.417 秒
- `card_viz.mp4`：整段只有起始关键帧
- `chip_viz.mp4`：整段只有起始关键帧

网页同步逻辑会在恢复播放或发生漂移时把右侧视频的 `currentTime` 对齐到左侧。关键帧过少时，浏览器为了从目标时间恢复解码，需要获取并解码更早的数据；在带宽不足时，这会显著增加等待时间。

### 3.5 检查网页同步逻辑

同步逻辑的设计行为是：

1. 两个视频都达到 `canplay` 后才同时调用 `play()`。
2. 任一视频触发 `waiting` 或 `stalled` 时，同时暂停两个视频。
3. 缓冲恢复后，将右侧时间对齐到左侧，再同时播放。
4. 播放中每 500 ms 检查一次时间差；偏差大于 0.12 秒时校正右侧视频。

该逻辑的目标是保证左右内容一致，但进一步的浏览器运行时检查发现，原实现存在同步 seek 与缓冲恢复相互重入的问题。

### 3.6 浏览器运行时诊断

通过 Chrome DevTools Protocol 分别检查 `file://` 直接打开和本地 HTTP 服务两种场景，得到相同结果：

- `board_viz.mp4` 和 `vase_viz.mp4` 已完整缓冲，`buffered` 覆盖整个视频。
- `video.error` 为 `null`，不存在媒体解码或文件读取错误。
- 右侧视频却持续处于 `readyState=1`、`seeking=true`、`paused=true`，播放时间停在开头。
- 3 秒内记录到 1128 个媒体事件，循环频率约为每 20 ms 一轮。
- Chrome renderer CPU 占用一度约为 168%。

事件循环路径为：

1. `playTogether()` 无条件将右侧 `currentTime` 设置为左侧时间，即使两者时间几乎相同。
2. 设置 `currentTime` 触发右侧 `seeking`，随后 `play()` 触发 `waiting`。
3. `waiting` handler 再次执行 `pauseTogether()` 和 `playTogether()`。
4. 新一轮 `playTogether()` 再次设置 `currentTime`，形成 seek/waiting 重入死循环。

运行时仅阻止原 `waiting` handler 重入后进行对照测试，左右视频立即正常播放：3 秒后 `board_real` 和 `board_viz` 的播放时间差约为 9 ms，两个 spinner 均消失。这证明持续转圈的直接原因是同步逻辑重入，而不是视频文件或本地打开协议。

## 4. 根因结论

右侧 `*_viz.mp4` 经常卡住包含两个层面：

1. **持续转圈的直接原因**：同步 seek 触发 `waiting`，而 `waiting` handler 重入同步流程，形成高频事件死循环。
2. **弱网下的放大因素**：右侧视频为 1080p，码率高于实际传输能力；浏览器可能优先调度 DOM 中更早出现的左侧媒体；原右侧视频 GOP 很长，进一步延长 seek 后的恢复时间。

## 5. 解决方案

### 5.1 保留原始文件

四个原始可视化视频逐字节保留为：

- `board_viz_raw.mp4`
- `vase_viz_raw.mp4`
- `card_viz_raw.mp4`
- `chip_viz_raw.mp4`

通过 SHA-256 对比确认，`*_viz_raw.mp4` 与重新编码前相应的 `*_viz.mp4` 完全一致。

### 5.2 生成网页优化版本

网页继续引用原文件名 `*_viz.mp4`，但文件内容重新编码为：

- 分辨率：1280x720
- 固定帧率：15 fps
- 编码：H.264 High Profile, Level 4.0
- 像素格式：`yuv420p`
- CRF：22
- 最大码率：900 kbps
- VBV Buffer：1800 kb
- GOP：15 帧，即每 1 秒一个关键帧
- 场景切换关键帧：关闭，保持稳定 GOP
- MP4 Fast Start：启用
- 音频：不写入

核心编码参数：

```bash
ffmpeg -i INPUT_RAW.mp4 \
  -map 0:v:0 -an \
  -vf "scale=1280:720:flags=lanczos,fps=15" \
  -c:v libx264 -preset slow -crf 22 \
  -maxrate 900k -bufsize 1800k \
  -profile:v high -level 4.0 -pix_fmt yuv420p \
  -g 15 -keyint_min 15 -sc_threshold 0 \
  -movflags +faststart OUTPUT_WEB.mp4
```

### 5.3 修复同步播放重入

同步 JavaScript 采用以下修复：

1. 增加单次同步状态，两个视频同时产生 `waiting/stalled` 时只允许一个恢复流程运行。
2. 仅在左右时间差超过 0.12 秒时设置右侧 `currentTime`，避免无意义 seek。
3. 同步 seek 期间忽略 `waiting/stalled` 重入。
4. seek 完成后再次等待两个视频达到 `HAVE_FUTURE_DATA`，再同时调用 `play()`。
5. 周期漂移修正不再直接修改 `currentTime`，统一进入带防重入保护的同步流程。

## 6. 解决后的验证结果

| 文件 | 优化后码率 | 优化后大小 | 原始大小 | 关键帧最大间隔 |
| --- | ---: | ---: | ---: | ---: |
| `board_viz.mp4` | 0.82 Mbps | 3.1 MB | 6.5 MB | 1.000 s |
| `vase_viz.mp4` | 0.81 Mbps | 2.1 MB | 4.9 MB | 1.000 s |
| `card_viz.mp4` | 0.80 Mbps | 0.98 MB | 2.3 MB | 1.000 s |
| `chip_viz.mp4` | 0.90 Mbps | 1.6 MB | 3.6 MB | 1.000 s |

四个网页视频合计由约 17.3 MB 降至约 7.8 MB，减少约 55%。

重新编码造成的时长取整误差均小于一个 15 fps 帧：

| 任务 | 左侧时长 | 右侧时长 | 差值 |
| --- | ---: | ---: | ---: |
| Board Wiping | 31.167 s | 31.200 s | 0.033 s |
| Vase Wiping | 20.917 s | 20.934 s | 0.017 s |
| Card Swiping | 10.084 s | 10.067 s | 0.017 s |
| Chip Grasping | 14.125 s | 14.134 s | 0.009 s |

抽取四个优化版视频在 5 秒位置的画面进行人工检查，触觉图、曲线、图例和数值仍可辨认。优化后预期效果是：

1. 右侧视频首播和缓冲恢复所需的数据量显著下降。
2. 同步校时后最多只需回溯约 1 秒的关键帧，而不是 10 秒以上或回到视频开头。
3. 弱网下出现长时间转圈和整组停播的概率降低。
4. 原始 1080p 文件仍随仓库保留，后续可以重新调整编码参数。

初次本地网页冒烟测试确认页面、视频容器和 spinner 可以渲染，但右侧持续转圈；该结果触发了上述 Chrome DevTools Protocol 深入诊断，不能视为播放验证通过。

同步代码修复后，使用全新 Chrome profile、1280x900 viewport，分别在 `file://` 和本地 HTTP 下依次滚动到四个任务，每组持续观察 3 秒。结果如下：

- 四组左右视频全部达到 `readyState=4`。
- 四组视频均为 `paused=false`、`seeking=false`、`error=null`。
- 所有 spinner 均正常消失，左右播放时间持续推进。
- `file://` 下四组最大同步误差低于 1.2 ms。
- 本地 HTTP 下四组最大同步误差低于 0.9 ms。
- 修复后每个视频在 3 秒内通常只产生 11-12 个正常 `timeupdate`，未再出现事件风暴。
- Headless Chrome renderer 平均 CPU 约为 26%，较修复前约 168% 明显下降。

本地两种打开方式均验证通过。线上播放结果将在发布后补充。

## 7. Git 同步记录

开始修改前执行了远端检查：

1. 两个 GitHub 仓库的 `main` 均位于提交 `17d84c2`。
2. GitLab `origin/ProjectPage` 已有合作者新增的两个提交：
   - `ec59f1e Add foresight prediction quality figure`
   - `d04232a Add main comparison table placeholder`
3. 本地分支和 GitLab 已分叉，因此先 fetch GitLab，再使用普通 merge 将合作者内容与本地视频同步功能合并。
4. 合并由 Git 的 `ort` 策略自动完成，没有发生文本冲突；合作者新增的图、JSON、绘图工具和网页内容均得到保留。
5. 提交前分别 fetch 两个 GitHub `main`；两个远端均无本地尚未包含的新提交。

待本次媒体提交和发布完成后，在此补充 GitLab、ForeTac GitHub 项目仓库和 GitHub Pages 仓库的最终提交哈希及在线验证结果。
