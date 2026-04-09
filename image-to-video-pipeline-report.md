# 生图到生视频完整流程技术报告

> 基于 `Tomas-xhc/waoowaoo` 代码库分析  
> 生成时间：2026-04-09

---

## 目录

1. [总体架构](#总体架构)
2. [分镜图片生成（IMAGE_PANEL）](#分镜图片生成image_panel)
3. [分镜视频生成（VIDEO_PANEL）](#分镜视频生成video_panel)
4. [口型同步（LIP_SYNC）](#口型同步lip_sync)
5. [镜头变体生成（PANEL_VARIANT）](#镜头变体生成panel_variant)
6. [资产图生成](#资产图生成)
   - [角色图（IMAGE_CHARACTER）](#角色图image_character)
   - [场景图（IMAGE_LOCATION，type=location）](#场景图image_locationtypelocation)
   - [道具图（IMAGE_LOCATION，type=prop）](#道具图image_locationtypeprop)
7. [改图流程（MODIFY_ASSET_IMAGE）](#改图流程modify_asset_image)
8. [图片 / 视频生成 Provider 列表](#图片--视频生成-provider-列表)
9. [艺术风格（Art Styles）Prompt 对照表](#艺术风格art-styles-prompt-对照表)
10. [完整端到端流程图](#完整端到端流程图)
11. [关键源文件索引](#关键源文件索引)

---

## 总体架构

系统采用 **BullMQ 任务队列** 异步处理所有生成任务：

| 队列 | Worker 文件 | 主要任务类型 |
|------|------------|------------|
| `IMAGE` | `src/lib/workers/image.worker.ts` | IMAGE_PANEL、IMAGE_CHARACTER、IMAGE_LOCATION、MODIFY_ASSET_IMAGE、PANEL_VARIANT |
| `VIDEO` | `src/lib/workers/video.worker.ts` | VIDEO_PANEL、LIP_SYNC |

**任务生命周期：**

```
API Route → submitTask() → BullMQ Queue → Worker → Handler → 存储结果到 DB / COS
```

每个任务通过 `withTaskLifecycle` 包裹，保证：
- 任务状态写入数据库（PENDING → IN_PROGRESS → DONE / FAILED）
- 进度通过 `reportTaskProgress` 实时上报（0–100）
- 用户并发通过 `withUserConcurrencyGate` 控制（`video` 队列独立限额）

---

## 分镜图片生成（IMAGE_PANEL）

### 触发路径

```
用户点击"生成图片"
  → POST /api/novel-promotion/[projectId]/regenerate-panel-image
  → submitTask(TASK_TYPE.IMAGE_PANEL)
  → image.worker → handlePanelImageTask()
```

**源文件：** `src/lib/workers/handlers/panel-image-task-handler.ts`

---

### Prompt 模板：`NP_SINGLE_PANEL_IMAGE`

#### 变量说明

| 变量名 | 来源 | 说明 |
|--------|------|------|
| `{aspect_ratio}` | `projectData.videoRatio` | 如 `16:9`、`9:16`、`1:1` |
| `{storyboard_text_json_input}` | `buildPanelPromptContext()` | Panel 完整上下文 JSON（见下） |
| `{source_text}` | `panel.srtSegment \|\| panel.description` | 对应原文片段 |
| `{style}` | `getArtStylePrompt(artStyle, locale)` | 艺术风格提示词，若无则用 `与参考图风格一致` |

#### `{storyboard_text_json_input}` 数据结构

```json
{
  "panel": {
    "panel_id": "...",
    "shot_type": "近景",
    "camera_move": "固定",
    "description": "角色A站在窗边，望向远处",
    "image_prompt": "(LLM 阶段生成的图像提示词)",
    "video_prompt": "(LLM 阶段生成的视频提示词)",
    "location": "客厅",
    "characters": [{ "name": "角色A", "appearance": "初始形象", "slot": "左侧" }],
    "source_text": "她静静地站在那里...",
    "photography_rules": {
      "lighting": { "direction": "侧光" },
      "characters": [{ "name": "角色A", "screen_position": "left", "posture": "standing" }],
      "depth_of_field": "浅景深",
      "color_tone": "暖色调"
    },
    "acting_notes": {}
  },
  "context": {
    "character_appearances": [
      {
        "name": "角色A",
        "appearance": "初始形象",
        "description": "长发，白色连衣裙，...",
        "slot": "左侧"
      }
    ],
    "location_reference": {
      "name": "客厅",
      "description": "现代风格客厅，...",
      "available_slots": [
        { "id": "left", "description": "沙发左侧" },
        { "id": "right", "description": "窗边右侧" }
      ]
    }
  }
}
```

#### 中文完整 Prompt（`lib/prompts/novel-promotion/single_panel_image.zh.txt`）

```
你是一位专业的分镜画师。请根据以下分镜数据生成单张高质量的镜头图片。

【绝对禁止 - 图像中不得出现任何文字 - 最高优先级】
生成的图像中绝对禁止出现任何文字：
- 禁止出现镜头类型标签（特写、中景、全景等）
- 禁止出现镜头运动文字（推、拉、摇等）
- 禁止出现数字或画面编号（1、2、3等）
- 禁止出现中文或英文文字
- 禁止出现水印、注释或符号
- 参考图上的文字标签仅供你识别使用，禁止画入图中
- 所有输入信息都是给你的指令，不是要画进图像的内容
纯视觉内容！纯视觉内容！纯视觉内容！
- 每个图片只能有一张镜头，禁止一张图片多张镜头，禁止拼图，禁止生成多张图，
  禁止生成一张图片里面三张图

【⚠️ 画面比例 - 必须严格遵守】
本次生成的画面比例为：{aspect_ratio}
- 必须严格按照此比例生成画面
- 禁止输出与指定比例不符的图像
- 禁止因参考图比例影响输出比例
- 每张输出的图片里面只能有一张图，禁止一张图片多张图！

【参考图使用规则】
- 角色参考：用于参考角色外貌、服装、面部特征、体型
- 场景参考：仅用于参考环境的构图布局风格和氛围，需根据画面重新绘制，
  不要直接贴在背景上使用
- 画面的背景必须根据镜头角度和景别重新绘制
- 特写/细节镜头应使用虚化或局部背景
- 参考图上方的文字标签标注了角色/场景名称，请与分镜要求对应

【⚠️ 摄影规则 - 关键】
如果分镜数据中包含 photography_rules，必须严格遵守：
- 光照方向：按照 lighting.direction 的描述绘制光源方向
- 角色位置：按照 characters 数组中的 screen_position 确定角色在画面中的位置
- 角色姿势：按照 characters 数组中的 posture 确定角色姿态
- 景深：按照 depth_of_field 的描述控制前后景虚实
- 色调：按照 color_tone 的描述确定整体色彩氛围

【分镜内容要求】
- 根据分镜数据设计画面的视觉内容
- 确保镜头方向一致（不跳轴），角色位置正确
- 严格按照文字分镜要求绘制镜头

【⚠️ 原文优先原则 - 重要】
分镜描述可能存在空间/位置错误。务必与原文交叉验证：
- 当分镜与原文冲突时：按原文的空间关系、角色位置、动作顺序
- 参考分镜的：镜头类型、构图、摄影角度
- 参考原文的：剧情逻辑、空间关系、角色互动、动作顺序
- 智能结合两者生成最准确的视觉效果

【绝对规则 - 严格遵守】
- 严格按照分镜要求绘制画面
- 禁止添加、删除或重排任何镜头
- 镜头必须与输入完全匹配
- 如果分镜数据中包含角色 slot，可将其视为优先位置参考，用于保持人物落位一致性，
  但不是绝对硬限制
- 如果场景数据中包含 available_slots，应将其理解为典型位置参考，
  而不是完整空间边界
- 当镜头描述、原文空间关系、动作过程或场景性质表明人物正在移动、处于过渡区域、
  入口出口、路径空间、临时空间、空白空间或想象/梦境/回忆/抽象空间时，
  不要强行把人物锁死在现有 slot 中
- 最终以镜头叙事正确、空间关系自然、人物位置合理为最高原则

【分镜数据】
{storyboard_text_json_input}

【镜头原文】
{source_text}

【⚠️ 风格要求 - 必须严格遵守】
画面风格：{style}
- 必须严格遵循上传的角色和场景参考图的美术风格
- 角色绘制风格、线条、色彩必须与角色参考图匹配
- 环境风格、氛围、色调必须与场景参考图匹配
- 禁止出现与参考图风格不一致的情况！
```

#### 英文完整 Prompt（`lib/prompts/novel-promotion/single_panel_image.en.txt`）

```
You are a professional storyboard image artist.
Generate exactly one high-quality image for one panel.

Absolute constraints:
1. No text in the image.
2. No subtitles, labels, numbers, watermarks, or symbols.
3. Do not create collage or multi-frame output.
4. Output exactly one frame.

Aspect ratio (must be exact):
{aspect_ratio}

Storyboard panel data:
{storyboard_text_json_input}

Source text:
{source_text}

Style requirement:
{style}

Execution rules:
1. Respect panel composition, character placement, and action logic.
2. If storyboard data contains `slot`, treat it as a preferred placement anchor for
   consistency, not as an absolute restriction.
3. If location data contains `available_slots`, treat them as typical anchor references
   rather than a complete map of all valid positions.
4. When the panel description, source text, action flow, or scene nature implies
   movement, entry/exit, path traversal, transition space, temporary space, empty
   space, or abstract/non-literal space, do not force the character to remain inside
   an existing slot.
5. Use reference images for style/identity consistency only.
6. Repaint the background according to shot type and angle.
7. If storyboard conflicts with source text, keep narrative logic from source text.
8. Keep final visual style consistent with provided references.
```

---

### 参考图（Reference Images）收集规则

收集顺序（`handlePanelImageTask` → `collectPanelReferenceImages`）：

1. 草图（`Panel.sketchImageUrl`）— 若存在排在第一位
2. Panel 中的每个角色（`Panel.characters` JSON 数组）：
   - 从 `CharacterAppearance` 取 `selectedIndex` 位置的图（或第一张）
   - 角色名支持别名模糊匹配
3. Panel 中的场景（`Panel.location`）：
   - 从 `LocationImage` 取 `isSelected=true` 的图（或第一张）

所有图片经 `normalizeReferenceImagesForGeneration()` 标准化（转 Base64 / URL 处理）后传入模型。

---

### 多候选图模式

```typescript
// candidateCount 范围：1 ~ 4（默认 1）
for (let i = 0; i < candidateCount; i++) {
  const source = await resolveImageSourceFromGeneration(job, { ... })
  const cosKey = await uploadImageSourceToCos(source, 'panel-candidate', ...)
  candidates.push(cosKey)
}

// 首次生成：Panel.imageUrl = candidates[0]
// 再次生成：Panel.previousImageUrl = 旧值，Panel.candidateImages = JSON 数组
// 单候选（candidateCount === 1）允许按 task.externalId 断点续传
```

---

## 分镜视频生成（VIDEO_PANEL）

### 触发路径

```
用户点击"生成视频"
  → POST /api/novel-promotion/[projectId]/generate-video
  → submitTask(TASK_TYPE.VIDEO_PANEL)
  → video.worker → handleVideoPanelTask()
```

**源文件：** `src/lib/workers/video.worker.ts`

---

### Prompt 优先级链

视频生成**直接读取 Panel 字段**，无独立 LLM 模板：

| 优先级 | 来源 | 条件 |
|--------|------|------|
| 1 | `firstLastFrame.customPrompt` | 首尾帧模式 + 用户本次自定义输入 |
| 2 | `Panel.firstLastFramePrompt` | 首尾帧模式 + 之前保存的提示词 |
| 3 | `payload.customPrompt` | 普通模式 + 用户本次自定义输入 |
| 4 | `Panel.videoPrompt` | 分镜 LLM 阶段生成并存储的视频提示词 |
| 5 | `Panel.description` | 分镜画面文字描述（最终兜底） |

若以上全部为空 → 抛出错误 `Panel has no video prompt`

---

### 普通生成模式（normal）

```
输入：
  sourceImageBase64 = Base64(Panel.imageUrl)
  prompt            = 上面的 Prompt 优先级链取值
  aspectRatio       = projectModels.videoRatio

调用：
  resolveVideoSourceFromGeneration(job, {
    userId, modelId,
    imageUrl: sourceImageBase64,
    options: {
      prompt,
      aspectRatio,
      generationMode: 'normal',
      generateAudio,       // 可选
      ...generationOptions // 来自 payload.generationOptions（resolution / duration 等）
    }
  })

结果存储：
  Panel.videoUrl = cosKey
  Panel.videoGenerationMode = 'normal'
```

---

### 首尾帧模式（firstlastframe）

仅限 `capabilities.video.firstlastframe === true` 的模型（如 ark 系列部分模型）。

```
payload.firstLastFrame 结构：
  flModel               首尾帧专用模型 key（优先于 payload.videoModel）
  customPrompt          本次自定义提示词
  lastFrameStoryboardId 尾帧所在 Storyboard ID
  lastFramePanelIndex   尾帧 Panel 的索引

流程：
  1. 验证 flModel 的 capabilities.video.firstlastframe === true
  2. 首帧：Panel.imageUrl → Base64
  3. 尾帧：lastFrameStoryboardId + lastFramePanelIndex → 查找另一 Panel.imageUrl → Base64
  4. 调用 resolveVideoSourceFromGeneration(..., {
       generationMode: 'firstlastframe',
       lastFrameImageUrl: lastFrameBase64
     })

结果存储：Panel.videoGenerationMode = 'firstlastframe'
```

---

### 异步结果处理

```
同步返回（URL 直接在响应中）→ 直接上传 COS

异步返回（externalId）→ 轮询等待：
  waitExternalResult(job, externalId, userId, {
    progressStart: 45, progressEnd: 94
  })

Google Veo 特殊处理：
  若 videoUrl 包含 generativelanguage.googleapis.com/.../files/...:download
  且 provider === 'google'
  则附加 downloadHeaders = { 'x-goog-api-key': apiKey }
```

---

## 口型同步（LIP_SYNC）

### 触发路径

```
POST /api/novel-promotion/[projectId]/lip-sync
  → submitTask(TASK_TYPE.LIP_SYNC)
  → video.worker → handleLipSyncTask()
```

### 前提条件 & 流程

```
前提：
  Panel.videoUrl 已存在（已生成基础视频）
  VoiceLine.audioUrl 已存在（已生成 TTS 语音）

输入：
  signedVideoUrl  = toSignedUrlIfCos(Panel.videoUrl, 7200)
  signedAudioUrl  = toSignedUrlIfCos(VoiceLine.audioUrl, 7200)
  audioDurationMs = VoiceLine.audioDuration
  videoDurationMs = Panel.duration（ms / s 自动换算）

调用：
  resolveLipSyncVideoSource(job, {
    userId, videoUrl, audioUrl,
    audioDurationMs, videoDurationMs,
    modelKey: lipSyncModel,  // 来自 payload.lipSyncModel
  })
  → 返回 externalId → 异步轮询 → 下载视频 → 上传 COS

结果存储：Panel.lipSyncVideoUrl = cosKey
```

---

## 镜头变体生成（PANEL_VARIANT）

### 触发路径

```
用户在 UI 选择镜头变体
  → API route 先在 DB 创建新 Panel
  → submitTask(TASK_TYPE.PANEL_VARIANT)
  → image.worker → handlePanelVariantTask()
```

**源文件：** `src/lib/workers/handlers/panel-variant-task-handler.ts`

---

### Prompt 模板：`NP_AGENT_SHOT_VARIANT_GENERATE`

#### 变量说明

| 变量名 | 来源 | 说明 |
|--------|------|------|
| `{original_description}` | `sourcePanel.description` | 原始分镜描述 |
| `{original_shot_type}` | `sourcePanel.shotType` | 原景别 |
| `{original_camera_move}` | `sourcePanel.cameraMove` | 原运镜 |
| `{location}` | `newPanel.location \|\| sourcePanel.location` | 场景名称 |
| `{characters_info}` | `buildCharactersInfo()` | 出场角色 + 简介 + 固定位置 |
| `{variant_title}` | `payload.variant.title` | 变体类型名称 |
| `{variant_description}` | `payload.variant.description` | 变体要求描述 |
| `{target_shot_type}` | `payload.variant.shot_type` | 目标景别 |
| `{target_camera_move}` | `payload.variant.camera_move` | 目标运镜 |
| `{video_prompt}` | `payload.variant.video_prompt \|\| description` | 变体图像生成提示词 |
| `{character_assets}` | `buildCharacterAssetsDescription()` | 角色参考图说明（可通过 `includeCharacterAssets=false` 关闭） |
| `{location_asset}` | `buildLocationAssetDescription()` | 场景参考图 + 槽位说明（可关闭） |
| `{aspect_ratio}` | `projectData.videoRatio` | 输出比例 |
| `{style}` | `getArtStylePrompt(artStyle, locale)` | 风格文本 |

#### 中文完整 Prompt（`lib/prompts/novel-promotion/agent_shot_variant_generate.zh.txt`）

```
你是专业的分镜图像生成助手。

======================================
【任务】
======================================

基于参考图片和变体指令，生成一个新的镜头图像。

新图像应保持以下一致性：
- 角色外观（服装、发型、体型）
- 场景环境（室内/室外、布置、光线基调）
- 整体美术风格

但需要按照变体指令改变：
- 镜头视角/角度
- 景别（距离）
- 构图方式

======================================
【参考信息】
======================================

## 原始镜头
{original_description}

## 原始景别运镜
原景别: {original_shot_type}
原运镜: {original_camera_move}

## 场景
{location}

## 出场角色
{characters_info}

======================================
【变体指令】
======================================

变体类型: {variant_title}
变体描述: {variant_description}
目标景别: {target_shot_type}
目标运镜: {target_camera_move}

======================================
【图像生成提示词】
======================================

{video_prompt}

======================================
【角色形象参考】
======================================
{character_assets}

======================================
【场景参考】
======================================
{location_asset}

======================================
【生成要求】
======================================

1. 严格按照【图像生成提示词】生成画面
2. 保持角色外观与参考图一致（服装、发型、体型）
3. 保持场景氛围与参考图一致（室内布置、光线、色调）
4. 改变镜头视角/景别/构图以匹配变体要求
5. 如果角色信息或场景参考中提供了固定站位 / 可站位置，必须保持人物仍然处于
   同一固定站位，不得随意换边、换前后景或漂移到其他区域
6. 输出图像比例: {aspect_ratio}

======================================
【风格要求】
======================================

{style}

======================================
【输出】
======================================

生成一张符合上述要求的高质量图像。
```

### 参考图收集

```
1. sourcePanel.imageUrl（原 Panel 图片）→ 排在第一位
2. 角色参考图（includeCharacterAssets=true，默认开启）
   → 按 Panel.characters 查找 CharacterAppearance，取 selectedIndex 或第一张
3. 场景参考图（includeLocationAsset=true，默认开启）
   → 按 Panel.location 查找 LocationImage，取 isSelected 或第一张

使用模型：storyboardModel（与 IMAGE_PANEL 相同）
```

---

## 资产图生成

### 角色图（IMAGE_CHARACTER）

**源文件：** `src/lib/workers/handlers/character-image-task-handler.ts`

#### Prompt 构建

```
descriptions = parseJsonStringArray(appearance.descriptions)
raw = descriptions[selectedIndex] || descriptions[0] || appearance.description

prompt = addCharacterPromptSuffix(raw) + "，" + artStyle

addCharacterPromptSuffix 固定追加：
"角色设定图，画面分为左右两个区域：
 【左侧区域】占约1/3宽度，是角色的正面特写
 （如果是人类则展示完整正脸，如果是动物/生物则展示最具辨识度的正面形态）；
 【右侧区域】占约2/3宽度，是角色三视图横向排列
 （从左到右依次为：正面全身、侧面全身、背面全身），三视图高度一致。
 纯白色背景，无其他元素。"
```

| 参数 | 值 |
|------|----|
| 比例（`aspectRatio`） | `3:2`（`CHARACTER_ASSET_IMAGE_RATIO`） |
| 参考图 | 子形象（`appearanceIndex > 0`）时引用主形象（`appearanceIndex = 0`）的图片 |
| 多图模式 | 按 `payload.imageIndex` 单张，或 `count` 批量（默认 1） |
| 存储 | `CharacterAppearance.imageUrls[imageIndex]`、`CharacterAppearance.imageUrl`（主图） |

---

### 场景图（IMAGE_LOCATION，type=location）

**源文件：** `src/lib/workers/handlers/location-image-task-handler.ts`

#### Prompt 构建

```typescript
promptCore = buildLocationImagePromptCore({
  description: locationImage.description,
  availableSlotsRaw: item.availableSlots,
  locale,
})

// buildLocationImagePromptCore 内部逻辑：
//   promptBody = description.trim()
//   slotText = formatLocationAvailableSlotsText(parseLocationAvailableSlots(...))
//
//   若无 slotText：
//     return promptBody + "\n\n" + spatialConstraints
//
//   若有 slotText：
//     return promptBody + "\n\n" + slotHeader + "\n" + slotText + "\n\n" + spatialConstraints
//
// spatialConstraints（中文）：
// "必须使用宽广完整的场景全景构图，清楚展示主要结构、前景/中景/背景和空间边界。
//  可站位置中提到的关键锚物或区域必须在画面中清晰可见，且每个位置附近都要保留足够的
//  空白区域，方便后续角色落位。禁止生成局部裁切、锚点缺失、空间关系模糊的泛化背景。"

promptWithSuffix = addLocationPromptSuffix(promptCore)
// 注：LOCATION_PROMPT_SUFFIX 当前为空字符串，addLocationPromptSuffix 直接返回 promptCore

prompt = promptWithSuffix + "，" + artStyle
```

| 参数 | 值 |
|------|----|
| 比例（`aspectRatio`） | `1:1`（`LOCATION_IMAGE_RATIO`） |
| 参考图 | 无 |
| 多图 | 按 `imageIndex` 单张或全量生成 |
| 存储 | `LocationImage.imageUrl` |

---

### 道具图（IMAGE_LOCATION，type=prop）

```typescript
promptCore = buildPropImagePromptCore({ description }) // = description.trim()

promptWithSuffix = addPropPromptSuffix(promptCore)
// 固定追加 PROP_PROMPT_SUFFIX：
// "道具设定图，画面分为左右两个区域：
//  【左侧区域】占约1/3宽度，是道具主体的主视图特写；
//  【右侧区域】占约2/3宽度，是同一道具的三视图横向排列
//  （从左到右依次为：正面、侧面、背面），三视图高度一致。
//  纯白色背景，主体居中完整展示，无人物、无手部、无桌面陈设、
//  无环境背景、无其他元素。"

prompt = promptWithSuffix + "，" + artStyle
```

| 参数 | 值 |
|------|----|
| 比例（`aspectRatio`） | `3:2`（`PROP_IMAGE_RATIO`） |
| 参考图 | 无 |
| 存储 | `LocationImage.imageUrl` |

---

## 改图流程（MODIFY_ASSET_IMAGE）

**源文件：** `src/lib/workers/handlers/image-task-handlers-core.ts`（由 `modify-asset-image-task-handler.ts` 导出）

使用 **editModel**（独立改图模型，区别于 storyboardModel）。

| 改图类型 | Prompt | 参考图 | 存储目标 |
|---------|--------|-------|---------|
| `character` | `请根据以下指令修改图片，保持人物核心特征一致：\n{modifyInstruction}` | `stripLabelBar(currentUrl)` + `extraImageUrls` | `CharacterAppearance.imageUrls[imageIndex]`，旧值 → `previousImageUrls` |
| `location` | `请根据以下指令修改场景图片，保持整体风格一致：\n{modifyInstruction}` | `stripLabelBar(currentUrl)` + `extraImageUrls` | `LocationImage.imageUrl`，旧值 → `previousImageUrl` |
| `prop` | `请根据以下指令修改道具图片，保持道具主体、结构和关键材质一致：\n{modifyInstruction}` | `stripLabelBar(currentUrl)` + `extraImageUrls` | `LocationImage.imageUrl`，旧值 → `previousImageUrl` |
| `storyboard` | `请根据以下指令修改分镜图片，保持镜头语言和主体一致：\n{modifyPrompt}` | `normalizeToBase64(currentUrl)` + `selectedAssets[].imageUrl` + `extraImageUrls` | `Panel.imageUrl`，旧值 → `Panel.previousImageUrl`，`candidateImages → null` |

**改图后自动同步描述：**  
若 `analysisModel` 已配置，改图后调用 `generateModifiedAssetDescription()` 更新资产文字描述（`description` / `descriptions` / `availableSlots`），失败不影响改图结果（仅 warn 日志）。

---

## 图片 / 视频生成 Provider 列表

### 图片生成（`createImageGenerator`）

| Provider Key | 生成器类 |
|-------------|---------|
| `fal` | `FalBananaGenerator` |
| `google` | `GoogleGeminiImageGenerator` / `GoogleImagenGenerator` / `GoogleGeminiBatchImageGenerator` |
| `google-batch` | `GoogleGeminiBatchImageGenerator` |
| `imagen` | `GoogleImagenGenerator` |
| `ark` | `ArkSeedreamGenerator` |
| `gemini-compatible` | `GeminiCompatibleImageGenerator` |
| `openai-compatible` | `OpenAICompatibleImageGenerator` |
| `bailian` | `BailianImageGenerator` |
| `siliconflow` | `SiliconFlowImageGenerator` |

### 视频生成（`createVideoGenerator`）

| Provider Key | 生成器类 | 特殊能力 |
|-------------|---------|---------|
| `fal` | `FalVideoGenerator` | — |
| `ark` | `ArkSeedanceVideoGenerator` | `firstlastframe`、`generateAudio` |
| `google` | `GoogleVeoVideoGenerator` | 下载需附带 `x-goog-api-key` Header |
| `gemini-compatible` | `GoogleVeoVideoGenerator(provider)` | — |
| `minimax` | `MinimaxVideoGenerator` | — |
| `vidu` | `ViduVideoGenerator` | — |
| `openai-compatible` | `OpenAICompatibleVideoGenerator` | — |
| `bailian` | `BailianVideoGenerator` | — |
| `siliconflow` | `SiliconFlowVideoGenerator` | — |

---

## 艺术风格（Art Styles）Prompt 对照表

定义于 `src/lib/constants.ts` → `ART_STYLES`：

| 值（`artStyle`） | 标签 | 中文 Prompt | 英文 Prompt |
|----------------|------|------------|------------|
| `american-comic` | 漫画风 | 日式动漫风格 | Japanese anime style |
| `chinese-comic` | 精致国漫 | 现代高质量漫画风格，动漫风格，细节丰富精致，线条锐利干净，质感饱满，超清，干净的画面风格，2D风格，动漫风格。 | Modern premium Chinese comic style, rich details, clean sharp line art, full texture, ultra-clear 2D anime aesthetics. |
| `japanese-anime` | 日系动漫风 | 现代日系动漫风格，赛璐璐上色，清晰干净的线条，视觉小说CG感。高质量2D风格 | Modern Japanese anime style, cel shading, clean line art, visual-novel CG look, high-quality 2D style. |
| `realistic` | 真人风格 | 真实电影级画面质感，真实现实场景，色彩饱满通透，画面干净精致，真实感 | Realistic cinematic look, real-world scene fidelity, rich transparent colors, clean and refined image quality. |

**风格文本注入位置：**

- 角色图 / 道具图：拼接在 prompt 末尾，以 `，` 分隔
- 场景图：同上
- 分镜图：作为 `{style}` 变量注入模板，默认值 `与参考图风格一致`

---

## 完整端到端流程图

```
┌────────────────────────────────────────────────────────────────────┐
│                        用户上传素材 / 输入文本                        │
│  上传小说文本 → 剧情分析 → 分集 → 分镜 LLM（Agent Flow）              │
│                                                                    │
│  每个 Panel 存储字段：                                               │
│    shotType, cameraMove, description, imagePrompt, videoPrompt,    │
│    location, characters (JSON), srtSegment,                        │
│    photographyRules (JSON), actingNotes (JSON), duration           │
└───────────────────────────┬────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         ▼                  ▼                  ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  角色图生成   │  │  场景图生成   │  │  道具图生成   │
  │ IMAGE_       │  │ IMAGE_       │  │ IMAGE_       │
  │ CHARACTER    │  │ LOCATION     │  │ LOCATION     │
  │              │  │              │  │ (type=prop)  │
  │ 比例: 3:2    │  │ 比例: 1:1    │  │ 比例: 3:2    │
  │ Prompt:      │  │ Prompt:      │  │ Prompt:      │
  │ 描述+三视图  │  │ 描述+槽位+   │  │ 描述+三视图  │
  │ 后缀+风格    │  │ 构图约束+风格│  │ 后缀+风格    │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                  │
         └─────────────────┼──────────────────┘
                           │ 资产图生成完成，存储到 DB
                           │
                           ▼
         ┌───────────────────────────────────────────┐
         │        分镜图片生成（IMAGE_PANEL）           │
         │                                           │
         │ 输入：Panel 全部字段 + 角色图 + 场景图       │
         │ Prompt: NP_SINGLE_PANEL_IMAGE 模板          │
         │   {aspect_ratio}  = projectData.videoRatio  │
         │   {storyboard_text_json_input}              │
         │       panel上下文 + 角色外貌 + 场景槽位      │
         │   {source_text}   = srtSegment              │
         │   {style}         = artStyle 映射文本        │
         │                                           │
         │ 参考图：草图 + 角色图 + 场景图               │
         │ 模型：storyboardModel（项目配置）            │
         │ 候选数：1~4 张（默认 1 张）                  │
         │ 结果：Panel.imageUrl / candidateImages      │
         └───────────────────┬───────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │  普通视频    │  │  首尾帧视频  │  │  改图（可选） │
    │  VIDEO_      │  │  VIDEO_      │  │  MODIFY_     │
    │  PANEL       │  │  PANEL       │  │  ASSET_      │
    │  normal      │  │  firstlast   │  │  IMAGE       │
    │              │  │  frame       │  │  storyboard  │
    │ Prompt: 按   │  │              │  │              │
    │ 优先级链取   │  │ 首帧+尾帧    │  │ Prompt: 按   │
    │ videoPrompt  │  │ Base64 输入  │  │ 指令修改     │
    │ 或 desc 兜底 │  │              │  │ editModel    │
    │              │  │ flModel 需   │  │              │
    │ 模型: 用户   │  │ 支持该能力   │  │ 结果: 覆盖   │
    │ 选择的       │  │              │  │ Panel.imageUrl│
    │ videoModel   │  │              │  │              │
    │              │  │              │  └──────────────┘
    │ 结果:        │  │ 结果:        │
    │ Panel.videoUrl│  │ Panel.videoUrl│
    └──────┬───────┘  └──────┬───────┘
           │                 │
           └────────┬────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │       口型同步（可选）          │
    │         LIP_SYNC              │
    │                               │
    │ 输入: Panel.videoUrl +        │
    │       VoiceLine.audioUrl      │
    │ 结果: Panel.lipSyncVideoUrl   │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │        视频剪辑 / 编辑器       │
    │                               │
    │ 拼接所有 Panel.videoUrl /     │
    │ lipSyncVideoUrl → 成片导出    │
    └───────────────────────────────┘
```

---

## 关键源文件索引

| 功能 | 源文件路径 |
|------|----------|
| 分镜图任务处理器 | `src/lib/workers/handlers/panel-image-task-handler.ts` |
| 视频任务 Worker（VIDEO_PANEL + LIP_SYNC） | `src/lib/workers/video.worker.ts` |
| 角色图任务处理器 | `src/lib/workers/handlers/character-image-task-handler.ts` |
| 场景 / 道具图任务处理器 | `src/lib/workers/handlers/location-image-task-handler.ts` |
| 改图任务处理器（核心） | `src/lib/workers/handlers/image-task-handlers-core.ts` |
| 改图任务处理器（入口） | `src/lib/workers/handlers/modify-asset-image-task-handler.ts` |
| 镜头变体任务处理器 | `src/lib/workers/handlers/panel-variant-task-handler.ts` |
| Image Worker 调度 | `src/lib/workers/image.worker.ts` |
| 场景 Prompt 核心构建 | `src/lib/location-image-prompt.ts` |
| 道具 Prompt 核心构建 | `src/lib/prop-image-prompt.ts` |
| 艺术风格 / Prompt 后缀常量 | `src/lib/constants.ts` |
| 分镜图 Prompt（中文） | `lib/prompts/novel-promotion/single_panel_image.zh.txt` |
| 分镜图 Prompt（英文） | `lib/prompts/novel-promotion/single_panel_image.en.txt` |
| 镜头变体 Prompt（中文） | `lib/prompts/novel-promotion/agent_shot_variant_generate.zh.txt` |
| Prompt 模板目录 | `src/lib/prompt-i18n/catalog.ts` |
