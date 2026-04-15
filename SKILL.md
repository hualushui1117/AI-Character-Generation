---
name: pixverse-character-animation-generator
description: >
  Generate AI character reference images and idle/talk/think/listen animation videos
  from a text description or uploaded image using PixVerse CLI.
  Use when: "角色生成", "生成角色", "character animation", "角色素材", "角色图",
  "character reference", "idle talk think listen", "角色视频", "character video sprite".
metadata:
  version: "1.0"
---

# PixVerse Character Animation Generator

一句话描述 → 角色参考图 → Idle / Talk / Think / Listen 四个状态动画视频。

## When to Use

- 用户想生成游戏/虚拟主播/对话 UI 用的角色素材
- 需要绿幕全身角色图 + 4 个标准状态循环视频
- 用户说"帮我生成一个角色"、"做个角色素材"、"character sprite"

## Dependencies

| 工具 | 用途 |
|------|------|
| `pixverse` CLI | 图像生成 + 视频生成 |
| `ffmpeg` | 透明背景处理（可选） |
| `jq` | 解析 JSON 输出 |

安装：
```bash
npm install -g pixverse   # 需要 Node.js >= 20
pixverse auth login       # OAuth 登录
```

---

## Critical Rules

1. **每次 PixVerse 调用必须加 `--json`** — 机器可读输出
2. **图像固定规格**：全身正面图、纯绿色（#00FF00）绿幕背景、9:16
3. **视频使用 Transition 模式**：首帧尾帧都传同一张参考图，通过提示词驱动动作
4. **视频并行提交**：12 个任务用 `--no-wait` 同时提交，统一等待
5. **全程维护内部状态**，跨对话轮次追踪角色名、路径、重生计数
6. **不向用户展示最终 prompt**，直接执行生成

---

## Internal State

在整个对话中保持跟踪：

```
角色名:          [用户命名后填入]
参考图本地路径:  [下载后填入]
绿幕背景:        true（默认）/ false（用户选择生成背景时）
图像重生次数:    0（上限 3）
视频重生次数:    { idle1: 0, idle2: 0, idle3: 0, talk1: 0, talk2: 0, talk3: 0, think1: 0, think2: 0, think3: 0, listen1: 0, listen2: 0, listen3: 0 }（每项上限 1）
```

以下两个 bash 变量在执行命令时使用（随流程推进赋值）：
- `CHAR_NAME` — 角色命名完成后赋值，用于路径构建和 state.json 写入
- `GREEN_SCREEN` — 背景确认后赋值为 `true` 或 `false`，用于 state.json 写入

---

## Phase 0: Install PixVerse Skill

执行任何操作前，用 WebFetch 拉取 PixVerse 官方 skill 文档：

```
1. https://raw.githubusercontent.com/PixVerseAI/skills/main/skills/SKILL.md
2. https://raw.githubusercontent.com/PixVerseAI/skills/main/skills/capabilities/create-and-edit-image.md
3. https://raw.githubusercontent.com/PixVerseAI/skills/main/skills/capabilities/transition.md
4. https://raw.githubusercontent.com/PixVerseAI/skills/main/skills/capabilities/auth-and-account.md
```

fetch 失败（无网络）时继续——本文件已内嵌足够的命令参考。

---

## Pre-flight Checks

```bash
pixverse --version                  # 确认已安装；未安装则提示 npm install -g pixverse
pixverse auth status --json         # 确认已登录；未登录提示运行 pixverse auth login
```

当前目录需有 `docs/` 或 `.claude/`，否则告知用户切换到项目根目录。

---

## Breakpoint Recovery

Pre-flight 通过后，**优先执行**：

```bash
if [ -f .pixverse_state.json ]; then
  python3 -c "
import json
s = json.load(open('.pixverse_state.json'))
name = s.get('character_name', '未知')
desc = {
  'image_done':        '图像已生成，视频待开始',
  'videos_submitted':  '视频已提交（ID 保留），待下载',
  'videos_done':       '视频已下载，待用户验收'
}
print(f'发现未完成任务：角色「{name}」— {desc.get(s[\"phase\"], s[\"phase\"])}')
"
fi
```

若发现 `.pixverse_state.json`，询问用户：「继续上次进度，还是重新开始？」

```
├── 继续 → 读取 state，根据 phase 跳转：
│     image_done       → 从 CHAR_NAME/GREEN_SCREEN 恢复，直接进入 Phase 2
│     videos_submitted → 从 state.json 中恢复 12 个 video_id，直接进入等待命令
│     videos_done      → 直接进入 Display Results，重新展示 12 个视频供用户验收
│
└── 重新开始 → rm .pixverse_state.json，走完整流程
```

---

## Phase 1: Image Generation

### Entry Recognition

```
├── [入口 A] 用户上传了图片
│     └── 询问："直接使用，还是以它为参考生成新图？"
│           ├── 作为参考 → 入口 B
│           └── 直接使用
│                 └── 软提示："最佳格式为全身正面图/绿幕/9:16，符合吗？"
│                       ├── 符合 → 跳过图像阶段，进入【角色命名】
│                       └── 不符合 → 执行 I2I 转换格式 → 满意度循环
│
├── [入口 B] 参考图 + 文字描述
│     └── 引导追问 → 执行 I2I 生成 2 张 → 满意度循环
│
└── [入口 C] 纯文字描述
      └── 引导追问 → 执行 T2I 生成 2 张 → 满意度循环
```

### Guided Questioning

5 个角色维度（每轮批量问完，最多 3 轮，缺失维度由模型合理补全）：

| 维度 | 示例 |
|------|------|
| 风格 | 写实 / 动漫 / 3D / 像素 |
| 人设基础 | 青年女性、精灵族 |
| 外貌特征 | 银白短发、红色眼睛 |
| 服装装扮 | 黑色皮质战斗服、奇幻皮甲 |
| 气质表情 | 冷酷、活泼、神秘 |

### Background Confirmation

引导追问完成后，生成前询问：

```
"需要为角色生成背景吗？（不需要的话默认绿幕背景，方便后期一键去除）"

├── 是 → 移除固定 prompt 中的"纯绿色（#00FF00）绿幕背景"
│         软提示："带背景的图片将无法使用最后的透明背景处理"
│         可追问用户描述背景（可选）→ 进入图像生成
└── 否 → 保留固定 prompt 中的"纯绿色（#00FF00）绿幕背景" → 进入图像生成
```

```bash
GREEN_SCREEN=true   # 用户选择绿幕背景（否，默认）
# 或
GREEN_SCREEN=false  # 用户选择生成背景（是）
```

### Image CLI Commands

**T2I（入口 C）**：
```bash
# 分析用户完整描述，找出视觉上存在合理分歧的维度（如气质、风格解读），
# 拟定两种不同诠释，分别生成各一张图
RESULT1=$(pixverse create image \
  --prompt "全身正面图，[纯绿色（#00FF00）绿幕背景，]（根据 Background Confirmation 决定是否保留）[角色描述·诠释A]" \
  --model gemini-3.1-flash \
  --quality 1080p \
  --aspect-ratio 9:16 \
  --count 1 \
  --no-wait \
  --json)

RESULT2=$(pixverse create image \
  --prompt "全身正面图，[纯绿色（#00FF00）绿幕背景，]（根据 Background Confirmation 决定是否保留）[角色描述·诠释B]" \
  --model gemini-3.1-flash \
  --quality 1080p \
  --aspect-ratio 9:16 \
  --count 1 \
  --no-wait \
  --json)
```

**I2I（入口 A 不符合 / 入口 B）**：
```bash
RESULT1=$(pixverse create image \
  --prompt "全身正面图，[纯绿色（#00FF00）绿幕背景，]（根据 Background Confirmation 决定是否保留）[角色描述·诠释A]" \
  --image [参考图路径] \
  --model gemini-3.1-flash \
  --quality 1080p \
  --aspect-ratio 9:16 \
  --count 1 \
  --no-wait \
  --json)

RESULT2=$(pixverse create image \
  --prompt "全身正面图，[纯绿色（#00FF00）绿幕背景，]（根据 Background Confirmation 决定是否保留）[角色描述·诠释B]" \
  --image [参考图路径] \
  --model gemini-3.1-flash \
  --quality 1080p \
  --aspect-ratio 9:16 \
  --count 1 \
  --no-wait \
  --json)
```

**等待并获取预览 URL**：
```bash
# 获取两张图各自的 image_id
IMG1_ID=$(echo "$RESULT1" | jq -r '.image_ids[0]')
IMG2_ID=$(echo "$RESULT2" | jq -r '.image_ids[0]')

# 等待完成
pixverse task wait $IMG1_ID --type image --json
pixverse task wait $IMG2_ID --type image --json

# 获取 image_url 展示给用户选择（仅展示 URL，不下载）
pixverse asset info $IMG1_ID --type image --json
pixverse asset info $IMG2_ID --type image --json
```

> 中间各轮重生只通过 URL 预览，**不落盘**。最终选定后统一下载一次（见 Satisfaction Loop）。

### Satisfaction Loop

展示 2 张图（image_url），附上一行对比说明，最多重生 3 次：

> 例：「图一偏冷酷内敛，图二偏强势凌厉——两种气质诠释，选你更想要的方向。」
> 维度由 Claude 自行从用户描述中识别，选视觉上存在合理分歧的点。

```
├── 选中一张（最终定稿）
│     └── → 进入【角色命名】（先命名，路径才有角色名）
│           → 命名完成后清空并下载：
│                 rm -f "output/$CHAR_NAME/image/"*
│                 mkdir -p "output/$CHAR_NAME/image"
│                 TMP_IMG=$(mktemp -d)
│                 pixverse asset download "$CHOSEN_IMG_ID" --type image --dest "$TMP_IMG" --json
│                 mv "$TMP_IMG"/*.* "output/$CHAR_NAME/image/character.png"
│                 rm -rf "$TMP_IMG"
│           → 写入断点状态：
│                 cat > .pixverse_state.json << EOF
│                 { "phase": "image_done", "character_name": "$CHAR_NAME",
│                   "green_screen": $GREEN_SCREEN,
│                   "image_path": "output/$CHAR_NAME/image/character.png" }
│                 EOF
│           → 进入 Phase 2（视频生成）
├── 局部不满意（有反馈）
│     └── 主动询问："还有什么想修改或增加的关键词吗？"
│           └── 整合用户反馈 → 定向修改对应维度 → 重新生成（重生次数 +1）
├── 整体不满意
│     └── 主动询问："有什么想修改或增加的关键词吗？"
│           └── 整合用户反馈 → 重新追问缺失维度 → 重新生成（重生次数 +1）
└── 达到 3 次上限 → 提示重新描述或上传参考图
      ├── 是 → 重置重生次数，回到入口 B/C
      └── 否 → 从现有图中强制选一张继续（同样执行上方的清空并下载步骤）
```

### Character Naming

```
Claude: "请为这个角色取个名字～"
用户输入（2-20 字符，如：冰川战士、月光精灵）
→ 记入内部状态，进入 Phase 2
```

```bash
CHAR_NAME="[用户输入的角色名]"
```

---

## Phase 2: Video Generation

**系统自动执行，不向用户询问任何选择。**

### 4 State Prompts

提示词从 `prompt.md` 读取，每个状态3条变体，固定前缀为：
`固定镜头，保持人物形象一致性，背景不变，保持竖屏构图：`

每次生成将3条变体分别分配给视频1/2/3，确保每个视频动作各有差异。

### Video CLI Commands

执行前先读取 `prompt.md`，将各状态变体文本赋值到以下变量：

```bash
# 从 prompt.md 读取各状态3条变体（固定前缀已含在完整文本中）
IDLE_P1="[prompt.md Idle 变体1完整文本]"
IDLE_P2="[prompt.md Idle 变体2完整文本]"
IDLE_P3="[prompt.md Idle 变体3完整文本]"

TALK_P1="[prompt.md Talk 变体1完整文本]"
TALK_P2="[prompt.md Talk 变体2完整文本]"
TALK_P3="[prompt.md Talk 变体3完整文本]"

THINK_P1="[prompt.md Think 变体1完整文本]"
THINK_P2="[prompt.md Think 变体2完整文本]"
THINK_P3="[prompt.md Think 变体3完整文本]"

LISTEN_P1="[prompt.md Listen 变体1完整文本]"
LISTEN_P2="[prompt.md Listen 变体2完整文本]"
LISTEN_P3="[prompt.md Listen 变体3完整文本]"
```

```bash
CHAR_IMG="output/$CHAR_NAME/image/character.png"

# 1. 并行提交 12 个任务
IDLE1_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$IDLE_P1" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

IDLE2_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$IDLE_P2" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

IDLE3_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$IDLE_P3" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

TALK1_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$TALK_P1" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

TALK2_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$TALK_P2" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

TALK3_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$TALK_P3" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

THINK1_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$THINK_P1" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

THINK2_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$THINK_P2" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

THINK3_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$THINK_P3" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

LISTEN1_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$LISTEN_P1" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

LISTEN2_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$LISTEN_P2" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

LISTEN3_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "$LISTEN_P3" \
  --quality 1080p --duration 3 --no-wait --json | jq -r '.video_id')

# 写入断点状态（12 个 ID 均已拿到）
cat > .pixverse_state.json << EOF
{
  "phase": "videos_submitted",
  "character_name": "$CHAR_NAME",
  "green_screen": $GREEN_SCREEN,
  "image_path": "output/$CHAR_NAME/image/character.png",
  "video_ids": {
    "idle1": "$IDLE1_ID", "idle2": "$IDLE2_ID", "idle3": "$IDLE3_ID",
    "talk1": "$TALK1_ID", "talk2": "$TALK2_ID", "talk3": "$TALK3_ID",
    "think1": "$THINK1_ID", "think2": "$THINK2_ID", "think3": "$THINK3_ID",
    "listen1": "$LISTEN1_ID", "listen2": "$LISTEN2_ID", "listen3": "$LISTEN3_ID"
  }
}
EOF

# 2. 统一等待
for ID in $IDLE1_ID $IDLE2_ID $IDLE3_ID \
          $TALK1_ID $TALK2_ID $TALK3_ID \
          $THINK1_ID $THINK2_ID $THINK3_ID \
          $LISTEN1_ID $LISTEN2_ID $LISTEN3_ID; do
  pixverse task wait $ID --json
done

# 3. 下载并重命名（逐一下载后立即重命名，避免文件名冲突）
mkdir -p "output/$CHAR_NAME/videos"

for PAIR in \
  "$IDLE1_ID:idle1"     "$IDLE2_ID:idle2"     "$IDLE3_ID:idle3" \
  "$TALK1_ID:talk1"     "$TALK2_ID:talk2"     "$TALK3_ID:talk3" \
  "$THINK1_ID:think1"   "$THINK2_ID:think2"   "$THINK3_ID:think3" \
  "$LISTEN1_ID:listen1" "$LISTEN2_ID:listen2" "$LISTEN3_ID:listen3"; do
  VID_ID="${PAIR%%:*}"
  VID_NAME="${PAIR##*:}"
  TMP=$(mktemp -d)
  pixverse asset download "$VID_ID" --dest "$TMP" --json
  mv "$TMP"/*.mp4 "output/$CHAR_NAME/videos/${VID_NAME}.mp4"
  rm -rf "$TMP"
done

# 写入断点状态（视频全部下载完成）
cat > .pixverse_state.json << EOF
{
  "phase": "videos_done",
  "character_name": "$CHAR_NAME",
  "green_screen": $GREEN_SCREEN,
  "image_path": "output/$CHAR_NAME/image/character.png",
  "video_ids": {
    "idle1": "$IDLE1_ID", "idle2": "$IDLE2_ID", "idle3": "$IDLE3_ID",
    "talk1": "$TALK1_ID", "talk2": "$TALK2_ID", "talk3": "$TALK3_ID",
    "think1": "$THINK1_ID", "think2": "$THINK2_ID", "think3": "$THINK3_ID",
    "listen1": "$LISTEN1_ID", "listen2": "$LISTEN2_ID", "listen3": "$LISTEN3_ID"
  }
}
EOF
```

### Display Results

```
✅ [角色名] 的视频素材已生成完毕！

📁 output/[角色名]/
   ├── image/   └── character.png
   └── videos/
       ├── idle1.mp4 / idle2.mp4 / idle3.mp4
       ├── talk1.mp4 / talk2.mp4 / talk3.mp4
       ├── think1.mp4 / think2.mp4 / think3.mp4
       └── listen1.mp4 / listen2.mp4 / listen3.mp4

请查看 12 个视频，有不满意的可以告诉我重新生成（每个视频最多重生 1 次）。
```

### Single Video Regeneration

```
├── 用户满意全部 → 进入【透明背景处理】
└── 用户指定某个视频重生（如 "idle2 不满意"）
      └── 检查该视频重生次数（上限 1）
            ├── 未达上限 → 携带反馈修改提示词 → 重新生成 → 下载重命名覆盖
            └── 已达上限 → 提示无法继续重生，保留当前版本
```

重生命令（不加 `--no-wait`）：
```bash
# 重新声明变量（bash 状态不跨 tool call 持久化）
CHAR_IMG="output/$CHAR_NAME/image/character.png"

NEW_RESULT=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "[根据用户反馈调整后的提示词]" \
  --quality 1080p --duration 3 \
  --json)
NEW_VIDEO_ID=$(echo "$NEW_RESULT" | jq -r '.video_id')

TMP=$(mktemp -d)
pixverse asset download "$NEW_VIDEO_ID" --dest "$TMP" --json
mv "$TMP"/*.mp4 "output/$CHAR_NAME/videos/[状态名][编号].mp4"
rm -rf "$TMP"
```

---

## Post-Processing: Transparent Background

所有视频验收完毕后，**仅当图像阶段使用了纯绿色（#00FF00）绿幕背景时**询问：

```
是否需要将图片和视频处理成透明背景版本？（适合叠加到不同场景使用）
```

若图像阶段选择了生成背景，跳过此询问，直接进入 Wrap-up。

```
├── 否 → 跳过，进入【Wrap-up】
└── 是 → 对 image/ 和 videos/ 下所有文件执行 ffmpeg 处理
```

**处理说明**：
- 抠绿幕：`colorkey` 对绿色背景进行去除
- Despill：`colorchannelmixer` 修正边缘绿色溢色
- 输出位置：新建 `[image无背景版]/` 和 `[videos无背景版]/` 文件夹
- 图片输出 PNG（rgba），视频输出 WebM VP9（yuva420p）

```bash
# 创建无背景版文件夹
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

---

## Error Handling

| 退出码 | 含义 | 处理方式 |
|--------|------|---------|
| 2 | 超时 | 提示网络较慢，询问是否重试 |
| 3 | 认证过期 | 提示运行 `pixverse auth login`，完成后重试 |
| 4 | Credits 不足 | 运行 `pixverse account info --json` 显示余额，提示充值 |
| 5 | 生成失败 | 可能是内容审核，建议调整描述后重试 |
| 6 | 参数错误 | 检查命令参数后重试 |

---

## Output Structure

```
output/
└── [角色名]/
     ├── image/
     │   └── character.png                 # 原始参考图
     ├── image无背景版/                     # 透明背景版（可选）
     │   └── character无背景版.png
     ├── videos/
     │   ├── idle1.mp4 / idle2.mp4 / idle3.mp4
     │   ├── talk1.mp4 / talk2.mp4 / talk3.mp4
     │   ├── think1.mp4 / think2.mp4 / think3.mp4
     │   └── listen1.mp4 / listen2.mp4 / listen3.mp4
     └── videos无背景版/                   # 透明背景版（可选）
         ├── idle无背景版1.webm / idle无背景版2.webm / idle无背景版3.webm
         ├── talk无背景版1.webm / talk无背景版2.webm / talk无背景版3.webm
         ├── think无背景版1.webm / think无背景版2.webm / think无背景版3.webm
         └── listen无背景版1.webm / listen无背景版2.webm / listen无背景版3.webm
```

---

## Wrap-up

全部完成后，清理断点文件并告知用户：

```bash
rm -f .pixverse_state.json
```

```
这个角色的素材已全部完成 🎉
要继续生成下一个角色吗？
```
