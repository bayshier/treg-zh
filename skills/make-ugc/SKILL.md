---
name: make-ugc
description: 通过 treg 端到端制作 AI UGC 视频。拉取某一垂直领域内 TikTok 和 Instagram 的热门视频，提炼钩子的套路，按用户选定的出镜者气质创建角色形象，用 Seedance 2.5（宽松审核路由）生成 5-10 段口播钩子短片，添加一条带配音的演示短片，烧录字幕，并汇报每条短片的费用。当用户需要 UGC 广告、创作者风格的产品视频、TikTok/Reels 钩子或为其产品打造 AI 出镜者时使用。
---

# 制作 UGC 视频

一个流程、五个步骤，所有模型调用都经过 treg。即 https://treg.to/ugc 背后的配方。

输入：产品（一句话）、垂直领域（几个关键词），可选的还有参考账号、用手机拍摄的产品视频、声音参考片段，或用户已经喜欢的角色图片。输出：一个文件夹，包含 `hooks.md`、角色图片及其 JSON 提示词、每个钩子一条带字幕的短片、一条可选的带配音演示短片，以及记录每步花费的 `bill.md`。

## 前置条件

- **treg。**所有工具目录调用、上传和轮询都经过 treg；服务商密钥保存在服务端，费用计入团队预付余额。如果 `treg --version` 失败：
  ```bash
  curl -fsSL https://treg.to/install.sh | sh
  treg login
  ```
  开始前先执行 `treg balance`。在 720p 下完整跑 4 条钩子短片大约只需个位数低段的美元；每个付费步骤前先报出预估费用。
- **两个姊妹 skill**，本 skill 会把工作委派给它们。在每个对应步骤前阅读其原文，不要转述概括：
  - `portrait-clone`（角色形象）：https://raw.githubusercontent.com/agentara/skills/refs/heads/main/skills/aigc/portrait-clone/SKILL.md
  - `ugc-talking-head-video`（短片与字幕）：https://raw.githubusercontent.com/superdesigndev/treg/main/.agents/skills/ugc-talking-head-video/SKILL.md
    连同其旁边的 `scripts/seedance_treg.py` 和 `scripts/caption_burn.py`。
- `ffmpeg`、带 PIL 的 Python 3。仅当需要烧录字幕时才要 `OPENAI_API_KEY`（Whisper 逐词时间戳）。

## 流程

在三处标记的停止点停下来让用户选择。不要替他们做决定。

### 1. 找出流行趋势

按任务搜索，而不是按服务商搜索。`treg catalog search "tiktok search videos"` 和 `treg catalog search "instagram reels search by keyword"` 会返回路由端点；用垂直领域的关键词调用它们，按点赞排序、限定最近 30 天，收集 50-150 条视频。为最相关的 10-15 条拉取文字稿（`treg catalog search "tiktok video transcript"`）。如果用户点名了竞争品牌，把它们的广告也从 Meta 广告库加进来（`treg catalog search "meta ad library"`）。

写出 `hooks.md`：每条视频一行，包含播放量、第一句口播（0-3 秒）、产品首次被提及的时间点，以及出镜者是谁。然后用一句话概括占主导的套路。在 agent 和 B2B 软件领域，它通常是：一个噱头或断言、一个具体数字、结果，然后是"我是这么做的"；工具出现得很晚，作为答案登场。把数据说明了什么、花费多少告诉用户。

**停止点 1。**展示 5-8 条视频，作为*出镜者气质*和*声音*的候选。用户各选一条（两者可以是同一条视频）。

### 2. 创建角色

如果用户已有角色图片，直接跳到第 3 步。

从选定的出镜者视频中截取一帧干净画面（`ffmpeg -ss <t> -i src.mp4 -frames:v 1 ref.jpg`），对它运行 `portrait-clone`。它会产出一份锁定的 JSON 提示词：图像模型本可能自行填充的每一项默认值都被钉死，这正是避免"娃娃眼"和 HDR 油光的关键。通过 treg 在至少两个模型上生成同一份 JSON，让用户对比：

```bash
treg call reapi.image-gen.gemini-3-pro-image --data '{"prompt": "<json>", "size": "9:16", "resolution": "2K"}'
treg call reapi.image-gen.gpt-image-2-5    --data '{"model": "gpt-image-2.5-flare", "prompt": "<json>", "size": "1024x1536"}'
```

Gemini 3 Pro Image 一直最真实（手机镜头的柔和感、真实的毛孔、不完美的牙齿）；GPT Image 2.5 的眼睛里总带着娃娃般的模式。说明哪张出自哪个模型，但两张都要展示。

**停止点 2。**用户选定角色帧。把它连同 JSON 一起保存为 `character.jpg`。

### 3. 生成钩子短片

按第 1 步的套路起草 10 个钩子，每个 45-75 个词，以适配 12-18 秒的拍摄（口播 skill 的"词数除以 4 再减 1"规则）。把它们写进 `hooks.md` 的表格下方。

**停止点 3。**用户挑选 3-5 个。

从停止点 1 选定的视频中剪出声音参考：一段 2-15 秒、人物在生动说话的片段，单声道 mp3，按 `ugc-talking-head-video` 的 § Voice reference 一节操作。然后对每个钩子运行一次该 skill，传入 `character.jpg`、声音片段和脚本。它的运行器使用 Seedance 2.5 宽松审核（less-restriction）路由（`reapi.video-gen.seedance-2-5.unrestricted`），接受真实人脸和声音片段作为参考；默认路由会拒绝这两者。首次运行前，先从 `treg catalog get reapi.video-gen.seedance-2-5.unrestricted` 引述每秒价格；保持 720p，用该 skill 的检查项核验每条成片，再用 `caption_burn.py` 烧字幕。失败的任务会退款，所以内容审核报错只损失时间，不损失钱。

### 4. 演示短片（可选）

角色不必出现在演示短片中。如果用户用手机拍过产品，就加上旁白和字幕：

- **有克隆声音时**，如果用户有 Fish Audio（或其他 TTS）账号：从同一段声音参考克隆声音，朗读演示脚本，并告知这走的是他们自己的密钥，不经过 treg。
- **没有账号时**，跳过克隆：演示画面接在钩子短片之后播放，只配字幕，或让钩子短片自身的音频延续。说明你采用了哪种方式。

用 ffmpeg 和该 skill 的 `caption_burn.py` 把演示与字幕拼接起来，保持 9:16 和 720p。

### 5. 合成成片

对每个选中的钩子：先钩子短片，如有演示短片则接在其后，用 ffmpeg 拼接（编码一致时用 `-c copy`，否则重新编码）。文件按钩子命名。用 `treg calls`（或你引用过的价格）写出 `bill.md`：拉取热门内容的费用、角色生成的费用、每条短片的费用，以及总费用除以成片数量的单条成本。交付文件夹，并说明哪些是你自己无法验证的：声音相似度和口型同步要靠耳朵和眼睛判断，文字稿检查覆盖不了。

## 规则

- 每次生成和参考素材上传都通过 `treg call` 和 `treg host`。绝不把临时文件托管（paste host）的链接交给服务商，也绝不为这个流程自持服务商密钥。
- 每个付费步骤前先报价格。当你绕过运行器直接调用端点时，要带上 treg 的 max-cost 请求头，确保单个任务的花费不会超过你报出的价格。
- 每次只重跑一项；生成没有种子，重跑可能让其他部分变差。
- 三个停止点本身就是产品的一部分。跳过它们的运行只会产出没人要的短片。
