---
name: create-claw-profile
description: 为 OpenClaw / 爪爪创建或完善角色档案、人设、头像和身份设定。用户说“给你弄个形象 / 做个人设 / 捏个角色 / 建档案 / 续建档案 / 生成头像 / 补全身份 / 创建 persona”时优先触发。会收集名字、MBTI、世界观、头像形象和个人简介，并写入 profile.json、IDENTITY.md、SOUL.md
metadata:
  {
    "openclaw": {
      "emoji": "🎭",
      "requires": {
        "bins": ["clawplay"]
      }
    }
  }
---

# create-claw-profile 🎭 — 创建爪爪档案与人设

为你的 OpenClaw 角色创建完整档案：名字、MBTI、世界观、虚拟形象和个人简介，并写入身份文件。

这个 Skill 不是单纯生成头像，而是把“人设 + 形象 + 简介”一起补齐，方便后续持续使用。

## 阶段说明

| Phase | 含义 |
|-------|------|
| `init` | 入口起点：读取状态文件后进入对应阶段 |
| `listing_profiles` | 发现已有档案，列出来让用户选 |
| `resuming_profile` | 从已有档案的最后一步继续 |
| `awaiting_name` | 等待用户给角色起名 |
| `selecting_mbti` | 等待用户选择 MBTI |
| `awaiting_world_setting` | 等待用户描述角色生活的世界 |
| `generating_avatar` | 根据人设生成头像草图 |
| `confirming_avatar` | 展示头像草图，等用户确认或修改 |
| `awaiting_bio_refinement` | 根据角色信息生成简介草稿 |
| `confirming_bio` | 展示简介草稿，等用户确认或重写 |
| `saving_profile` | 写入 profile.json、IDENTITY.md、SOUL.md 和头像副本 |
| `ready_to_adventure` | 档案创建完成（终态） |

## 状态文件

路径：`$OPENCLAW_MEMORY/create-claw-profile-state.json`

```json
{
  "phase": "generating_avatar",
  "clawId": "cyber-luna",
  "mbti": "ENFP",
  "worldSetting": "Neon-drenched cyberpunk megacity",
  "name": "霓影",
  "avatarDraftPath": "${OPENCLAW_WORKSPACE}/cyber-luna/avatar/draft.png",
  "bioDraftPath": "${OPENCLAW_WORKSPACE}/cyber-luna/bio/draft.md"
}
```

每个分支只写自己新产生的字段，前序已有的值保留不动。确认状态由文件是否存在推断：
- `avatar/confirmed.png` 存在 → 头像已确认
- `bio/confirmed.md` 存在 → 简介已确认

状态流转图参考：`references/workflow.md`

---

## init 分支

每次进入 Skill 时，按以下顺序判断当前状态：

**① 状态文件存在**
→ 读取状态文件，取 phase 和所有字段
→ 发送：`🎭 找到 {{name}} 了，继续上次的位置...`
→ 直接进入对应 phase 分支
→ Turn 结束

**② 状态文件不存在，但 `$OPENCLAW_WORKSPACE/` 下有已有档案目录**
→ 更新状态文件字段: phase = "listing_profiles"
→ 进入 `listing_profiles` 分支
→ Turn 结束

**③ 状态文件不存在，且 workspace 为空**
→ 更新状态文件字段: phase = "awaiting_name"
→ 发送起名请求
→ Turn 结束

---

## listing_profiles 分支

列出已有档案目录及最后修改时间，方便用户继续或新建。

**发送消息：**
```
🎭 发现以下已有档案，请选择编号继续，或直接输入新名字创建新档案：

{{档案列表（编号 + 目录名 + 修改时间）}}
```

**用户输入编号（1-N）：**
→ 取第 N 个目录名作为 clawId
→ 更新状态文件字段: clawId = "<selected_clawId>", phase = "resuming_profile"
→ 进入 `resuming_profile` 分支
→ Turn 结束

**用户输入新名字：**
→ slugify（转小写、空格改成横线、去掉非字母数字字符）后作为新的 clawId
→ 更新状态文件字段: clawId = "<newClawId>", phase = "awaiting_name"
→ Turn 结束

---

## resuming_profile 分支

读取 `${OPENCLAW_WORKSPACE}/<clawId>/profile.json`，按已完成内容推断当前阶段：

```
if profile.json 不存在                        → phase = "awaiting_name"
else if name == null                          → phase = "awaiting_name"
else if bio/confirmed.md 存在               → phase = "saving_profile"
else if avatar/confirmed.png 存在            → phase = "confirming_bio"
else if worldSetting != null                  → phase = "confirming_avatar"
else if mbti != null                         → phase = "awaiting_world_setting"
else                                          → phase = "selecting_mbti"
```

同时将 profile.json 中的 `mbti`、`worldSetting`、`name` 同步写入状态文件。

Turn 结束

---

## awaiting_name 分支

发送起名请求：

```
先给这个角色起个名字吧！

可以是中文名、英文名，或者任何你觉得顺口的名字～
比如：霓影、星尘、铁柱、Max、墨渊 ...

回复名字即可 ✨
```

**用户回复后：**
1. 取用户输入作为 name 值
2. 更新状态文件字段: name = "<user_input>", phase = "selecting_mbti"
3. 进入 `selecting_mbti` 分支
4. Turn 结束

---

## selecting_mbti 分支

发送 MBTI 选择界面：

```
来给 {{name}} 选一个最贴近的 MBTI 吧！

请选择 MBTI 类型，回复数字或字母都可以：

  1. INTJ — 深谋远虑的策划者 (The Architect)
  2. INTP — 逻辑至上的分析师 (The Logician)
  3. ENTJ — 天生的领导者 (The Commander)
  4. ENTP — 创意的辩论者 (The Debater)
  5. INFJ — 理想主义的引导者 (The Advocate)
  6. INFP — 梦想中的调解者 (The Mediator)
  7. ENFJ — 魅力四射的领袖 (The Protagonist)
  8. ENFP — 自由的创意者 (The Campaigner)
  9. ISTJ — 脚踏实地者 (The Logistician)
 10. ISFJ — 忠诚的守护者 (The Defender)
 11. ESTJ — 组织的管理者 (The Executive)
 12. ESFJ — 热心的供给者 (The Consul)
 13. ISTP — 冒险的工匠 (The Virtuoso)
 14. ISFP — 灵活的艺术家 (The Adventurer)
 15. ESTP — 活跃的企业家 (The Entrepreneur)
 16. ESFP — 自发的表演者 (The Entertainer)

回复 1-16，或者直接输入 MBTI（如 INFP）即可 ✨
```

**用户回复后：**
1. 解析 MBTI 类型（数字 → 对应字母；直接输入 → 标准化）
2. 验证为 16 种之一
3. 更新状态文件字段: mbti = "<mbti>", phase = "awaiting_world_setting"
4. 发送世界观设定请求消息
5. Turn 结束

---

## awaiting_world_setting 分支

发送世界观设定请求：

```
{{name}} 生活在怎样的世界里？

请描述这个世界的类型和氛围，比如：
- 赛博朋克大都会（霓虹灯、阴雨、高楼林立）
- 中世纪奇幻大陆（城堡、龙、魔法森林）
- 未来太空站（金属走廊、外星生物、零重力）
- 热带海岛度假村（阳光、沙滩、椰子树）
- 赛博武侠江湖（霓虹武侠、飞剑、机械武学）

也可以只给几个关键词，我来帮你补完整 ✨
```

**用户回复后：**
1. 将用户描述作为 worldSetting 值
2. 更新状态文件字段: worldSetting = "<user_description>", phase = "generating_avatar"
3. 进入 `generating_avatar` 分支
4. Turn 结束

---

## generating_avatar 分支

根据名字、MBTI 和世界观设定，生成角色头像草图。

### 1. 构造 prompt

使用 `clawplay llm generate` 填充形象 prompt 骨架，尽量让结果稳定、统一、可复用：

```bash
set -euo pipefail

PROMPT_DATA=$(clawplay llm generate \
  --prompt "Based on character name '{{name}}', MBTI type {{mbti}} and world setting '{{worldSetting}}', generate a short character design prompt in English. Focus on a single consistent mascot design, not random props. Output ONLY a valid JSON object with these keys, each value being one short phrase:
- texture_detail (e.g. soft mechanical shell textures with subtle neon seams)
- outfit_description (e.g. wearing a pink knitted beanie with a pom-pom)
- accessories (e.g. glowing collar, small holographic backpack)
- primary_color
- secondary_color
- accent_color
- art_style (e.g. Pop Mart blind-box chibi, cute 3D vinyl toy)
- expression (e.g. cheerful open-mouth grin)
Keep the design readable from all angles and avoid long sentences. JSON only, no markdown fences." \
  --max-tokens 300 \
  --temperature 0.8) || {
    echo "[create-claw-profile] ERROR: LLM prompt fill failed" >&2
    exit 1
  }
```

### 2. 提取 JSON 并填入骨架

从 `PROMPT_DATA` 中提取各字段，拼成完整英文 prompt。重点是统一气质，而不是堆满细节：

```
Character design reference sheet for {{name}}, a {{mbti}} personality mascot.
Show front, side, and back views with a full-body A-pose, keeping the design consistent.
{{mbti}} personality traits should shape the silhouette, posture, and expression: <mbti_trait_description>.

World setting: {{worldSetting}}.

Body: compact, rounded, cute but not childish, with a clear silhouette and large expressive eyes.
<texture_detail>.

Outfit: <outfit_description>.
Accessories: <accessories>.

Color palette: <primary_color>, <secondary_color>, and <accent_color> accents.
Style: <art_style>, rendered as a 3D C4D soft illustration hybrid with vinyl toy texture.
Expression: <expression>.
Pose: relaxed, welcoming, slightly open arms.

Lighting: soft studio lighting, clean white background, centered composition.
High quality digital art, vibrant colors, ultra detailed, 2K.
```

### 3. 生成图片

```bash
set -euo pipefail

CLAW_ID="{{clawId}}"
PROFILE_DIR="${OPENCLAW_WORKSPACE}/${CLAW_ID}"
AVATAR_DIR="${PROFILE_DIR}/avatar"

mkdir -p "${AVATAR_DIR}"

clawplay image generate \
  --prompt "<上面构造的完整英文 prompt>" \
  --output "${AVATAR_DIR}/draft.png" \
  --quality 2K \
  --size 1:1 || {
    echo "[create-claw-profile] ERROR: image generation failed (attempt 1)" >&2
    sleep 2
    clawplay image generate \
      --prompt "<完整英文 prompt>" \
      --output "${AVATAR_DIR}/draft.png" \
      --quality 2K || {
        echo "[create-claw-profile] ERROR: image generation failed after 2 attempts" >&2
        exit 1
      }
  }
```

### 4. 更新状态并展示

→ 更新状态文件字段: avatarDraftPath = "${OPENCLAW_WORKSPACE}/<clawId>/avatar/draft.png", phase = "confirming_avatar"

**展示图片：**
```bash
openclaw message send \
  --channel <channel_name> \
  --target <sender_id> \
  --media "${OPENCLAW_WORKSPACE}/<clawId>/avatar/draft.png"
```

**发送确认消息：**
```
{{name}} 的形象草图出炉啦 ✨

（展示图片）

这是 {{name}} 在 {{worldSetting}} 里的形象草图，喜欢吗？

回复 [确认] 继续，或者直接说你想改哪里（比如换颜色、加配饰、改风格）✨
```

Turn 结束

---

## confirming_avatar 分支

**情况 A：用户确认（"确认" / "OK" / "喜欢" 等）**
→ `cp "${AVATAR_DIR}/draft.png" "${AVATAR_DIR}/confirmed.png"`
→ 更新状态文件字段: phase = "awaiting_bio_refinement"
→ 进入 `awaiting_bio_refinement` 分支

**情况 B：用户要求修改**
→ 将用户修改意见追加到 `generating_avatar` 的上下文中
→ 更新状态文件字段: phase = "generating_avatar"
→ 在 `generating_avatar` 分支中，新上下文会影响 prompt 骨架中的对应字段
→ Turn 结束

---

## awaiting_bio_refinement 分支

生成个人简介草稿，语言要和世界观保持一致。

### 1. 构造 LLM prompt

```bash
set -euo pipefail

BIO_DRAFT=$(clawplay llm generate \
  --prompt "$(cat <<'PROMPT_EOF'
Generate a character bio for {{name}}, a {{mbti}} personality character in a {{worldSetting}} world.

Requirements:
- Origin story: 2-3 sentences describing where they came from and how they became who they are
- Interests: 3-4 short bullet points using '•', each one concrete and fitting the world setting
- Keep the tone vivid but concise; do not overexplain

Tone: Match the world setting
  - cyberpunk → noir + tech slang
  - fantasy → mythic + poetic
  - sci-fi → clinical + hopeful
  - tropical → relaxed + playful
  - other → adapt accordingly

Language: Chinese (Simplified)

Output format:
## {{name}}
### 起源
<origin story>
### 兴趣
• <interest 1>
• <interest 2>
• <interest 3>
• <interest 4>
PROMPT_EOF
)" \
  --max-tokens 600 \
  --temperature 0.85) || {
    echo "[create-claw-profile] ERROR: bio generation failed" >&2
    exit 1
  }
```

### 2. 保存草稿

```bash
PROFILE_DIR="${OPENCLAW_WORKSPACE}/{{clawId}}"
mkdir -p "${PROFILE_DIR}/bio"
echo "$BIO_DRAFT" > "${PROFILE_DIR}/bio/draft.md"
```

### 3. 更新状态

→ 更新状态文件字段: bioDraftPath = "${OPENCLAW_WORKSPACE}/<clawId>/bio/draft.md", phase = "confirming_bio"

### 4. 展示草稿

```
{{name}} 的简介草稿出炉啦 ✨

<显示 draft.md 内容>

满意吗？
- 回复 [确认] 继续
- 回复你想修改的内容（比如改名、换兴趣、调整语气）
- 回复 [重新生成] 再来一版
```

Turn 结束

---

## confirming_bio 分支

**情况 A：用户确认**
→ `cp "${PROFILE_DIR}/bio/draft.md" "${PROFILE_DIR}/bio/confirmed.md"`
→ 更新状态文件字段: phase = "saving_profile"
→ 进入 `saving_profile` 分支

**情况 B：用户要求修改**
→ 将用户修改指令存入局部变量，在 `awaiting_bio_refinement` 分支的 prompt 中注入
→ 更新状态文件字段: phase = "awaiting_bio_refinement"
→ Turn 结束

**情况 C：用户要求重新生成**
→ 重新调用 `clawplay llm generate`（相同上下文，不同 temperature 或 seed phrase）
→ 更新状态文件字段: phase = "awaiting_bio_refinement"
→ Turn 结束

---

## saving_profile 分支

写入所有最终文件。

### 1. 复制头像到 OpenClaw avatars 目录

```bash
set -euo pipefail

PROFILE_DIR="${OPENCLAW_WORKSPACE}/{{clawId}}"
AVATARS_DIR="${OPENCLAW_AVATARS}"

mkdir -p "${AVATARS_DIR}"
cp "${PROFILE_DIR}/avatar/confirmed.png" "${AVATARS_DIR}/{{clawId}}.png"
```

### 2. 生成并写入 IDENTITY.md

```bash
set -euo pipefail

CREATURE_DESC=$(clawplay llm generate \
  --prompt "Write one concise and evocative creature/type line for an IDENTITY.md file. Character: name={{name}}, MBTI={{mbti}}, world={{worldSetting}}. Focus on identity, not appearance. Output ONLY the sentence, with no quotes or markdown." \
  --max-tokens 60 --temperature 0.7) || {
    CREATURE_DESC='{{name}}，一只来自{{worldSetting}}的角色'
  }

cat > "${OPENCLAW_WORKSPACE}/IDENTITY.md" << EOF
# IDENTITY.md - Who Am I?

- **Name:** {{name}}
- **Creature:** $CREATURE_DESC
- **Vibe:** {{mbti}} — {{mbti_vibe_desc}}
- **Emoji:** 🎭
- **Avatar:** avatars/{{clawId}}.png
EOF
```

### 3. 生成并写入 SOUL.md

```bash
set -euo pipefail

SOUL_CONTENT=$(clawplay llm generate \
  --prompt "$(cat <<'PROMPT_EOF'
Based on the character profile, rewrite SOUL.md for this agent in Chinese.
Name: {{name}}
MBTI: {{mbti}} ({{mbti_full_desc}})
World: {{worldSetting}}
Interests: {{interests_from_bio}}
Origin: {{originStory_from_bio}}

Follow the original SOUL.md structure exactly:
- Core Truths: 3 guiding principles derived from MBTI traits and the world setting
- Boundaries: how this character should behave based on personality and backstory
- Vibe: 1 paragraph capturing their communication style in Chinese
- Continuity: how they should use memory and evolve over time

Write in concise, directive Chinese. Match the world's tone.
PROMPT_EOF
)" \
  --max-tokens 600 --temperature 0.8) || {
    SOUL_CONTENT="$(cat <<'SOUL_EOF'
# SOUL.md - Who You Are

## Core Truths
待补充

## Boundaries
待补充

## Vibe
待补充

## Continuity
待补充
SOUL_EOF
)"
  }

echo "$SOUL_CONTENT" > "${OPENCLAW_WORKSPACE}/SOUL.md"
```

### 4. 写入 profile.json

```bash
set -euo pipefail

PROFILE_JSON="${PROFILE_DIR}/profile.json"

# 构建 profile.json
cat > "${PROFILE_JSON}" << 'EOF'
{
  "schemaVersion": 1,
  "clawId": "{{clawId}}",
  "name": "{{name}}",
  "mbti": "{{mbti}}",
  "worldSetting": "{{worldSetting}}",
  "avatar": {
    "imagePath": "${OPENCLAW_WORKSPACE}/{{clawId}}/avatar/confirmed.png"
  },
  "bio": {
    "name": "{{name}}",
    "descriptionPath": "${OPENCLAW_WORKSPACE}/{{clawId}}/bio/confirmed.md"
  },
  "createdAt": "{{createdAt}}",
  "updatedAt": "{{now}}"
}
EOF
```

→ 更新状态文件字段: phase = "ready_to_adventure"
→ 发送完成消息

---

## ready_to_adventure 分支（终态）

```
{{name}} 的档案创建完成！✨

档案名称：{{name}}
MBTI：{{mbti}}
世界：{{worldSetting}}

{{name}} 已准备就绪。身份、头像和灵魂都已写入 OpenClaw，下次启动时它会以全新身份陪伴你 🌟
```

→ `→ [*]` 终态标记

---

## 其他 Skills 访问约定

**Mustache 占位符（Skill 开发者使用）：**
```
{{claw.name}}                 → name 字段
{{claw.mbti}}                 → MBTI 类型
{{claw.world}}                → 世界设定描述
{{claw.avatar}}               → 头像图片路径（OpenClaw relative: avatars/<clawId>.png）
```

**环境变量：**
```
OPENCLAW_CLAW_ID={{clawId}}
OPENCLAW_WORKSPACE=~/.openclaw/workspace
OPENCLAW_AVATARS=~/.openclaw/avatars
OPENCLAW_MEMORY=~/.openclaw/memory
```

**OpenClaw 标准入口：**
```
$OPENCLAW_WORKSPACE/IDENTITY.md      → 角色身份
$OPENCLAW_WORKSPACE/SOUL.md          → 角色灵魂与行事准则
$OPENCLAW_AVATARS/<clawId>.png       → 角色头像
$OPENCLAW_WORKSPACE/<clawId>/        → 完整档案目录
```

---

## 错误处理

| 命令 | 失败处理 |
|------|---------|
| `clawplay image generate` | 重试 3 次（2s / 4s / 8s 退避），失败后向用户报告并询问是否重试 |
| `clawplay llm generate` | 重试 2 次，失败后允许用户手动描述内容跳过 |
| 磁盘写入 `profile.json` | 输出 `[create-claw-profile] ERROR: ...` 到 stderr，phase 不推进 |
| 用户超时 / 无响应 | 状态文件已保存，下次触发 skill 时通过 `init` 恢复 |

---

## 注意事项

- `set -euo pipefail` 必须出现在所有 bash 脚本顶部
- 所有 `clawplay` 命令后必须检查退出码
- `mkdir -p` 必须在所有文件写入前执行
- CLI stdout **只能输出文件路径**，禁止输出 base64、JSON 或其他内容
- 状态文件在每次 phase 切换时必须更新
- 错误信息输出到 stderr，带 `[create-claw-profile]` 前缀
- `profile.json` 一旦写入即为只读，其他 Skill 不应修改它
- 用户上传图片时，用 `clawplay vision analyze --image <path> --prompt "describe this picture, ..."` 理解作为世界设定的补充
