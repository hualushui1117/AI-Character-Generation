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
2. **图像固定规格**：全身正面图、绿幕背景、9:16
3. **视频使用 Transition 模式**：首帧尾帧都传同一张参考图，通过提示词驱动动作
4. **视频并行提交**：4 个状态用 `--no-wait` 同时提交，统一等待
5. **全程维护内部状态**，跨对话轮次追踪角色名、路径、重生计数
6. **不向用户展示最终 prompt**，直接执行生成

---

## Internal State

在整个对话中保持跟踪：

```
角色名:          [用户命名后填入]
参考图本地路径:  [下载后填入]
图像重生次数:    0（上限 3）
视频重生次数:    { idle: 0, talk: 0, think: 0, listen: 0 }（每项上限 1）
```

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

### Image CLI Commands

**T2I（入口 C）**：
```bash
pixverse create image \
  --prompt "全身正面图，绿幕背景，[角色描述]" \
  --model gemini-3.1-flash \
  --quality 1080p \
  --aspect-ratio 9:16 \
  --count 2 \
  --json
```

**I2I（入口 A 不符合 / 入口 B）**：
```bash
pixverse create image \
  --prompt "全身正面图，绿幕背景，[角色描述]" \
  --image [参考图路径] \
  --model gemini-3.1-flash \
  --quality 1080p \
  --aspect-ratio 9:16 \
  --count 2 \
  --json
```

**等待并下载**：
```bash
# 获取 image_id（count=2 时返回数组）
IMAGE_IDS=$(echo "$RESULT" | jq -r '.image_ids[]')

# 等待完成（如使用了 --no-wait）
for ID in $IMAGE_IDS; do
  pixverse task wait $ID --type image --json
done

# 获取 image_url 展示给用户选择
for ID in $IMAGE_IDS; do
  pixverse asset info $ID --type image --json
done

# 用户选定后下载
mkdir -p output/[角色名]/image
pixverse asset download [选中的image_id] --type image \
  --dest output/[角色名]/image --json
```

### Satisfaction Loop

展示 2 张图（image_url），最多重生 3 次：

```
├── 选中一张 → 进入【角色命名】
├── 局部不满意（有反馈）→ 定向修改维度 → 重新生成（重生次数 +1）
├── 整体不满意 → 重新追问 → 重新生成（重生次数 +1）
└── 达到 3 次上限 → 提示重新描述或上传参考图
      ├── 是 → 重置重生次数，回到入口 B/C
      └── 否 → 从现有图中强制选一张继续
```

### Character Naming

```
Claude: "请为这个角色取个名字～"
用户输入（2-20 字符，如：冰川战士、月光精灵）
→ 记入内部状态，进入 Phase 2
```

---

## Phase 2: Video Generation

**系统自动执行，不向用户询问任何选择。**

### 4 State Prompts

| 状态 | 提示词 |
|------|--------|
| idle | 角色静止站立，放松姿态，自然表情，镜头固定，保持竖屏构图 |
| talk | 角色正在说话，嘴部做出说话动作，热情表达，镜头固定，保持竖屏构图 |
| think | 角色做出思考的手势，若有所思，手指轻点下巴，镜头固定，保持竖屏构图 |
| listen | 角色侧耳倾听，认真聆听的表情，温和友善，镜头固定，保持竖屏构图 |

### Video CLI Commands

```bash
CHAR_IMG="output/[角色名]/image/[文件名]"

# 1. 并行提交 4 个任务
IDLE_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "角色静止站立，放松姿态，自然表情，镜头固定，保持竖屏构图" \
  --quality 720p --duration 3 --no-wait --json | jq -r '.video_id')

TALK_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "角色正在说话，嘴部做出说话动作，热情表达，镜头固定，保持竖屏构图" \
  --quality 720p --duration 3 --no-wait --json | jq -r '.video_id')

THINK_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "角色做出思考的手势，若有所思，手指轻点下巴，镜头固定，保持竖屏构图" \
  --quality 720p --duration 3 --no-wait --json | jq -r '.video_id')

LISTEN_ID=$(pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "角色侧耳倾听，认真聆听的表情，温和友善，镜头固定，保持竖屏构图" \
  --quality 720p --duration 3 --no-wait --json | jq -r '.video_id')

# 2. 统一等待
for ID in $IDLE_ID $TALK_ID $THINK_ID $LISTEN_ID; do
  pixverse task wait $ID --json
done

# 3. 下载
mkdir -p "output/[角色名]/videos"
pixverse asset download $IDLE_ID   --dest "output/[角色名]/videos" --json
pixverse asset download $TALK_ID   --dest "output/[角色名]/videos" --json
pixverse asset download $THINK_ID  --dest "output/[角色名]/videos" --json
pixverse asset download $LISTEN_ID --dest "output/[角色名]/videos" --json
```

### Display Results

```
✅ [角色名] 的视频素材已生成完毕！

📁 output/[角色名]/
   ├── image/   └── character.png
   └── videos/
       ├── idle.mp4 / talk.mp4 / think.mp4 / listen.mp4

请查看 4 个视频，有不满意的可以告诉我重新生成（每个状态最多重生 1 次）。
```

### Single Video Regeneration

```
├── 用户满意全部 → 进入【透明背景处理】
└── 用户指定某个状态重生
      └── 检查该状态重生次数（上限 1）
            ├── 未达上限 → 携带反馈修改提示词 → 重新生成 → 下载替换
            └── 已达上限 → 提示无法继续重生，保留当前版本
```

重生命令（不加 `--no-wait`）：
```bash
pixverse create transition \
  --images "$CHAR_IMG" "$CHAR_IMG" \
  --prompt "[根据用户反馈调整后的提示词]" \
  --quality 720p --duration 3 \
  --json

pixverse asset download [新video_id] --dest "output/[角色名]/videos" --json
```

---

## Post-Processing: Transparent Background

所有视频验收完毕后询问：

```
是否需要将图片和视频处理成透明背景版本？（适合叠加到不同场景使用）
```

```
├── 否 → 跳过，进入【Wrap-up】
└── 是 → 对 image/ 和 videos/ 下所有文件执行 ffmpeg 处理
```

**处理说明**：
- 抠绿幕：`colorkey` 对绿色背景进行去除
- Despill：`colorchannelmixer` 修正边缘绿色溢色
- 输出位置：原文件夹内，文件名加 `_alpha` 后缀
- 图片输出 PNG（rgba），视频输出 WebM VP9（yuva420p）

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
     │   ├── character.png           # 原始参考图
     │   └── character_alpha.png     # 透明背景版（可选）
     └── videos/
         ├── idle.mp4
         ├── talk.mp4
         ├── think.mp4
         ├── listen.mp4
         ├── idle_alpha.webm         # 透明背景版（可选）
         ├── talk_alpha.webm
         ├── think_alpha.webm
         └── listen_alpha.webm
```

---

## Wrap-up

全部完成后询问：

```
这个角色的素材已全部完成 🎉
要继续生成下一个角色吗？
```
