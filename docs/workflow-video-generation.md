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
- **每张参考图**生成 4 个状态 × 每状态 3 个视频
- **总产出**：参考图张数 × 12

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
 ├── 状态 1：Idle（待机）— 3 条提示词变体，生成 3 个视频
 │    ├── 首帧/尾帧：参考图（相同）
 │    ├── 提示词变体1→idle1.mp4 / 变体2→idle2.mp4 / 变体3→idle3.mp4
 │    └── 固定前缀：固定镜头，保持人物形象一致性，背景不变，保持竖屏构图：
 │
 ├── 状态 2：Talk（说话）— 3 条提示词变体，生成 3 个视频
 │    ├── 首帧/尾帧：参考图（相同）
 │    └── talk1.mp4 / talk2.mp4 / talk3.mp4
 │
 ├── 状态 3：Think（思考）— 3 条提示词变体，生成 3 个视频
 │    ├── 首帧/尾帧：参考图（相同）
 │    └── think1.mp4 / think2.mp4 / think3.mp4
 │
 └── 状态 4：Listen（聆听）— 3 条提示词变体，生成 3 个视频
      ├── 首帧/尾帧：参考图（相同）
      └── listen1.mp4 / listen2.mp4 / listen3.mp4
```

### Transition 模式工作原理

- **首尾帧一致**：两个相同的参考图告诉 Pixverse "镜头固定不动"
- **提示词驱动**：Pixverse 在这两帧间根据提示词生成角色的动作/表情变化
- **结果**：角色做出相应动作，背景始终不变，适合作为 UI 素材

### Pixverse 命令调用

```bash
pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "固定镜头，保持人物形象一致性，背景不变，保持竖屏构图：人物面向镜头生动说话，口型清晰贴合发音，伴随富有情绪的手势动作，神态鲜活有感染力" \
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
      (×3)    (×3)    (×3)    (×3)
         │        │        │        │
         └────────┼────────┴────────┘
                  ↓
         [生成完毕，展示 12 个视频]
                  │
                  ├─→ 用户满意 ────────→ 完成 / 导出
                  │
                  └─→ 用户不满意
                       │
                       └─ 指定某个视频重生（E.g. idle2）（每个视频最多×1）
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
- **结果展示**：生成完毕后，展示 12 个视频供用户查看（每个状态 3 个：idle1~3、talk1~3、think1~3、listen1~3）

### 满意度循环 — 单视频重生机制

用户可对每个视频进行 **一次性重生**：

```
展示 12 个视频
 │
 ├── 用户满意 ──────────────────→ 完成 / 导出
 │
 └── 用户不满意（有具体反馈）
      └── 对单个视频重生（E.g., 只重生 idle2）
           │
           ├── [重生次数限制]
           │  每个视频最多重生1次
           │  （idle1 ×1、idle2 ×1、idle3 ×1 … listen3 ×1）
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
           ├── idle1.mp4 / idle2.mp4 / idle3.mp4
           ├── talk1.mp4 / talk2.mp4 / talk3.mp4
           ├── think1.mp4 / think2.mp4 / think3.mp4
           └── listen1.mp4 / listen2.mp4 / listen3.mp4
```

### 命名示例

```
output/
 ├── 冰川战士/
 │    ├── image/
 │    │   └── character.png
 │    └── videos/
 │         ├── idle1.mp4 / idle2.mp4 / idle3.mp4
 │         ├── talk1.mp4 / talk2.mp4 / talk3.mp4
 │         ├── think1.mp4 / think2.mp4 / think3.mp4
 │         └── listen1.mp4 / listen2.mp4 / listen3.mp4
 │
 └── 月光精灵/
      ├── image/
      │   └── character.png
      └── videos/
           ├── idle1.mp4 / idle2.mp4 / idle3.mp4
           ├── talk1.mp4 / talk2.mp4 / talk3.mp4
           ├── think1.mp4 / think2.mp4 / think3.mp4
           └── listen1.mp4 / listen2.mp4 / listen3.mp4
```

---

## 透明背景处理（可选后处理）

所有视频验收完毕后，**仅当图像阶段使用了纯绿色（#00FF00）绿幕背景时**，系统询问：

> "是否需要将图片和视频处理成透明背景版本？（适合叠加到不同场景使用）"

若图像阶段选择了生成背景，跳过此询问，直接结束。

### 处理说明

- **抠绿幕**：`colorkey` 滤镜去除 `#00FF00` 背景
- **Despill**：`colorchannelmixer` 修正边缘绿色溢色
- **输出位置**：新建 `[image无背景版]/` 和 `[videos无背景版]/` 文件夹
- **图片输出**：PNG（RGBA）；**视频输出**：WebM VP9（yuva420p）

### 处理命令

```bash
mkdir -p "output/$CHAR_NAME/image无背景版"
mkdir -p "output/$CHAR_NAME/videos无背景版"

# 处理参考图
ffmpeg -i "output/$CHAR_NAME/image/character.png" \
  -vf "colorkey=color=0x00FF00:similarity=0.30:blend=0.08,\
       colorchannelmixer=rg=-0.15:bg=-0.15" \
  -pix_fmt rgba \
  "output/$CHAR_NAME/image无背景版/character无背景版.png"

# 处理 12 个视频
for STATE in idle talk think listen; do
  for N in 1 2 3; do
    ffmpeg -i "output/$CHAR_NAME/videos/${STATE}${N}.mp4" \
      -vf "colorkey=color=0x00FF00:similarity=0.30:blend=0.08,\
           colorchannelmixer=rg=-0.15:bg=-0.15" \
      -c:v libvpx-vp9 \
      -pix_fmt yuva420p \
      "output/$CHAR_NAME/videos无背景版/${STATE}无背景版${N}.webm"
  done
done
```

### 处理后文件结构

```
output/[角色名]/
├── image/
│   └── character.png
├── image无背景版/
│   └── character无背景版.png
├── videos/
│   ├── idle1.mp4 / idle2.mp4 / idle3.mp4
│   ├── talk1.mp4 / talk2.mp4 / talk3.mp4
│   ├── think1.mp4 / think2.mp4 / think3.mp4
│   └── listen1.mp4 / listen2.mp4 / listen3.mp4
└── videos无背景版/
    ├── idle无背景版1.webm / idle无背景版2.webm / idle无背景版3.webm
    ├── talk无背景版1.webm / talk无背景版2.webm / talk无背景版3.webm
    ├── think无背景版1.webm / think无背景版2.webm / think无背景版3.webm
    └── listen无背景版1.webm / listen无背景版2.webm / listen无背景版3.webm
```

---

## 后续可扩展方向

- 支持多个参考图时的批量生成
- 视频的导出、下载功能
- 不同角色间的组合视频生成
- 其他视频状态的扩展（如互动、动作等）
