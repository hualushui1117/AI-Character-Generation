# Workflow — 视频生成阶段

## 概述

基于图像生成阶段确认的角色参考图，为每张图生成 4 个状态的短视频素材，由系统自动生成，用户仅查看最终结果。

---

## 视频规格

### 基础规范
- **分辨率**：1080p（优先），720p（备选，1080p 不可用时降级）
- **尺寸比例**：9:16（竖屏）
- **时长**：3 秒 / 条
- **镜头**：固定不变

### 生成数量
- **每张参考图**生成 4 个状态的视频
- **总产出**：参考图张数 × 4

---

## 4 个视频状态

| 状态 | 英文名 | 说明 |
|------|--------|------|
| 待机 | Idle | 角色静止站立，无动作 |
| 说话 | Talk | 角色做出正常说话口型，口部动作 |
| 思考 | Think | 角色做出思考动作（如挠挠头） |
| 聆听 | Listen | 角色做出聆听表情，面部表情有变动 |

---

## 技术方案：Pixverse Transition 模式

使用 **Pixverse Transition 模式** 完成视频生成：
- **首帧 + 尾帧**：都上传相同的参考图（确认的角色图）
- **各状态提示词**：为每个状态编写专属提示词，指导生成动作/表情
- **镜头**：固定不变，通过提示词控制角色动作
- **输出**：3 秒循环视频

### 实现流程

```
参考图（确认图）
 │
 ├── 状态 1：Idle（待机）
 │    ├── 首帧：参考图
 │    ├── 尾帧：参考图（相同）
 │    ├── 提示词：角色静止站立，放松姿态，自然表情，镜头固定，竖屏
 │    └── 输出：idle.mp4（3s）
 │
 ├── 状态 2：Talk（说话）
 │    ├── 首帧：参考图
 │    ├── 尾帧：参考图（相同）
 │    ├── 提示词：角色正在说话，嘴部做出说话动作，热情表达，镜头固定，竖屏
 │    └── 输出：talk.mp4（3s）
 │
 ├── 状态 3：Think（思考）
 │    ├── 首帧：参考图
 │    ├── 尾帧：参考图（相同）
 │    ├── 提示词：角色做出思考的手势，若有所思，手指轻点下巴，镜头固定，竖屏
 │    └── 输出：think.mp4（3s）
 │
 └── 状态 4：Listen（聆听）
      ├── 首帧：参考图
      ├── 尾帧：参考图（相同）
      ├── 提示词：角色侧耳倾听，认真聆听的表情，温和友善，镜头固定，竖屏
      └── 输出：listen.mp4（3s）
```

### Transition 模式工作原理

- **首尾帧一致**：两个相同的参考图告诉 Pixverse "镜头固定不动"
- **提示词驱动**：Pixverse 在这两帧间根据提示词生成角色的动作/表情变化
- **结果**：角色做出相应动作，背景始终不变，适合作为 UI 素材

### Pixverse 命令调用

```bash
pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "角色正在说话，嘴部做出说话动作，热情表达，镜头固定，竖屏" \
  --duration 3 \
  --quality 1080p \
  --no-wait \
  --json
```

---

## 工作流完整链路

```
[图像生成阶段] ─→ 用户确认参考图
                  │
                  ↓
         [视频生成阶段自动执行]
                  │
         ┌────────┼────────┬────────┐
         ↓        ↓        ↓        ↓
       Idle    Talk    Think   Listen
         │        │        │        │
         └────────┼────────┴────────┘
                  ↓
         [生成完毕，展示 4 个视频]
                  │
                  ├─→ 用户满意 ────────→ 完成 / 导出
                  │
                  └─→ 用户不满意
                       │
                       ├─ 选择重生 Idle（已用×1）
                       ├─ 选择重生 Talk（已用×1）
                       ├─ 选择重生 Think（已用×1）
                       └─ 选择重生 Listen（已用×1）
                            │
                            ↓
                       [单个视频重生]
                            │
                       ┌────┴────┐
                       ↓         ↓
                      满意      不满意
                       │         │
                       ↓         ↓
                      ✓        已达上限
                       │
                       ↓
                  完成 / 导出
```

---

## 用户交互

### 视频展示与选择

- **视频生成过程**：系统自动执行，**不向用户做任何选择**
- **结果展示**：生成完毕后，展示 4 个视频供用户查看（一套完整的 Idle、Talk、Think、Listen）

### 满意度循环 — 单视频重生机制

用户可对每个状态进行 **一次性重生**：

```
展示 4 个视频
 │
 ├── 用户满意 ──────────────────→ 完成 / 导出
 │
 └── 用户不满意（有具体反馈）
      └── 对单个状态重生（E.g., 只重生 Talk）
           │
           ├── [重生次数限制]
           │  每个状态最多重生1次
           │  （Idle ×1、Talk ×1、Think ×1、Listen ×1）
           │
           └── 重生完成
                └── 展示更新后的视频
                     ├── 满意 ──────────→ 完成 / 导出
                     └── 不满意 ────────→ 无法继续重生，提示已达上限
```

---

## 文件存放规范

生成的视频文件按以下结构存放，**文件夹名使用用户输入的角色名**：

> 注意：`pixverse asset download` 保存的文件名由 API 决定（通常为随机 ID）。下载后应手动重命名为下方规范名称，或在脚本中用 `mv` 完成重命名。

```
output/
 └── [角色名]/              # E.g., 冰川战士、月光精灵
      ├── image/
      │   └── character.png            # 参考图
      │
      └── videos/
           ├── idle.mp4
           ├── talk.mp4
           ├── think.mp4
           └── listen.mp4
```

### 命名示例

```
output/
 ├── 冰川战士/
 │    ├── image/
 │    │   └── character.png
 │    └── videos/
 │         ├── idle.mp4
 │         ├── talk.mp4
 │         ├── think.mp4
 │         └── listen.mp4
 │
 ├── 月光精灵/
 │    ├── image/
 │    │   └── character.png
 │    └── videos/
 │         ├── idle.mp4
 │         ├── talk.mp4
 │         ├── think.mp4
 │         └── listen.mp4
 │
 └── 暗夜探手/
      ├── image/
      │   └── character.png
      └── videos/
           ├── idle.mp4
           ├── talk.mp4
           ├── think.mp4
           └── listen.mp4
```

---

## 透明背景处理（可选后处理）

所有视频验收完毕后，**仅当图像阶段使用了纯绿色（#00FF00）绿幕背景时**，系统询问：

> "是否需要将图片和视频处理成透明背景版本？（适合叠加到不同场景使用）"

若图像阶段选择了生成背景，跳过此询问，直接结束。

### 处理说明

- **抠绿幕**：`colorkey` 滤镜去除 `#00FF00` 背景
- **Despill**：`colorchannelmixer` 修正边缘绿色溢色
- **输出位置**：原文件夹内，文件名加 `_alpha` 后缀
- **图片输出**：PNG（RGBA）；**视频输出**：WebM VP9（yuva420p）

### 处理命令

```bash
# 处理 image/ 下所有图片
for IMG in output/[角色名]/image/*.{png,jpg,jpeg}; do
  [ -f "$IMG" ] || continue
  FILENAME=$(basename "${IMG%.*}")
  ffmpeg -i "$IMG" \
    -vf "colorkey=color=0x00FF00:similarity=0.30:blend=0.08,\
         colorchannelmixer=rg=-0.15:bg=-0.15" \
    -pix_fmt rgba \
    "output/[角色名]/image/${FILENAME}_alpha.png"
done

# 处理 videos/ 下所有视频
for VID in output/[角色名]/videos/*.mp4; do
  [ -f "$VID" ] || continue
  FILENAME=$(basename "${VID%.*}")
  ffmpeg -i "$VID" \
    -vf "colorkey=color=0x00FF00:similarity=0.30:blend=0.08,\
         colorchannelmixer=rg=-0.15:bg=-0.15" \
    -c:v libvpx-vp9 \
    -pix_fmt yuva420p \
    "output/[角色名]/videos/${FILENAME}_alpha.webm"
done
```

### 处理后文件结构

```
output/[角色名]/
├── image/
│   ├── character.png           # 原始参考图
│   └── character_alpha.png     # 透明背景版
└── videos/
    ├── idle.mp4
    ├── talk.mp4
    ├── think.mp4
    ├── listen.mp4
    ├── idle_alpha.webm         # 透明背景版
    ├── talk_alpha.webm
    ├── think_alpha.webm
    └── listen_alpha.webm
```

---

## 后续可扩展方向

- 支持多个参考图时的批量生成
- 视频的导出、下载功能
- 不同角色间的组合视频生成
- 其他视频状态的扩展（如互动、动作等）
