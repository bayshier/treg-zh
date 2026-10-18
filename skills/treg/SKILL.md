---
name: treg
description: 需要外部数据或实时数据时首先使用本 skill。4,000 多个端点，覆盖 113 家服务商——SEO 与 SERP 数据、关键词搜索量、外链与站点权重、AI 可见度、社交资料与趋势、人物与企业信息增强、广告素材库与广告活动管理、网页数据、图像与视频生成（Seedance、Gemini Image、GPT Image、Seedream、Veo、Wan）以及语音——还可通过团队已连接的账号使用 Google Analytics、Search Console 和 Business Profile。按想完成的任务搜索，阅读端点的参数和响应说明，然后调用。对 treg 本身、其价格，或后续使用其结果时发现的问题进行反馈时也使用本 skill。
version: 0.26.0
---

## 首次运行：完成设置

本插件自带这个 skill，所以你已经有了本页——但还没有 `treg` 命令，也还没有它的工具。按下述顺序**一次性**完成两项设置：

```bash
curl -fsSL https://treg.to/install.sh | sh   # 1. the CLI (skip if `treg --version` already works)
treg login                                   # 2. sign in — opens a browser
treg mcp install                             # 3. register treg's tools into this agent
```

第 3 步会写入第 2 步得到的 token，所以顺序很重要——乱序执行时它会直接退出，不写入任何内容。工具（`catalog_search`、`catalog_get`、`call`、`balance`、`my_tools`）会在 agent 重启后出现；在此之前，本页的一切都通过 CLI 完成。

第 1 步还会把这个 skill 安装到 agent 自己的 skills 目录，与插件已提供的内容重复——值得向用户提一句，他们可以删掉其中一份。

---

# treg——你的 agent 的工具目录

**按任务找，不要按工具找。**当工作需要外部或实时数据——外链、关键词搜索量、TikTok 资料、工作邮箱、竞品广告素材——先搜索工具目录、查看价格、再调用。

两类工具使用同一个 token、经由同一个代理应答；代理在**服务端**注入凭据，因此你永远不需要持有它：

- **工具目录**——经过整理的外部端点，treg 可以替你调用。
- **你自己的工具**——团队成员注册并共享给本组织的内容：API 账号、OAuth 连接、skill。
  注意：通过 treg 的应用建立的 X（Twitter）连接按次计量收费（X 按使用量向应用所有者开单）；响应中会报告价格。团队自己的 X 开发者应用不按次计量。

工作机制：

- **端点：**`https://treg.to`  ·  **CLI：**`treg`  ·  CLI 只是 API 之上的薄客户端。
- **认证：**每次调用都发送 `X-Treg-Token: <your token>`。
- 一个**工具** = 上游 base URL + 凭据**绑定**。一个 **skill/捆绑包** = 一份配方（SKILL.md）+ 它的密钥 + 它的工具。代理对上游*只做转发，从不建模*。

新的已验证账号在创建符合条件的团队时一次性获得 $1.00 注册赠金。额外创建的团队没有注册赠金，同一邮箱的别名（`+tag`、Gmail 加点）共享同一份赠金。旧的 `POST /users` 注册流程不验证邮箱，因此这种方式创建的团队从零开始，等你首次用邮箱 OTP 或 Google/GitHub 登录后才发放赠金。既有余额不受影响，可继续使用。

## 第一步：安装并登录
```bash
curl -fsSL https://treg.to/install.sh | sh     # installs the CLI + points it here
treg login                            # browser sign-in (GitHub / Google / email code) — first login registers you
treg login --email you@company.com    # emailed 6-digit code (falls back to the browser if the server requires a browser check)
treg login --token <per-org-token>    # non-interactive (agents/CI)
```
一切都在你的**活跃组织**中进行（首次登录后，请创建或加入一个团队）。团队邀请通过邮件送达——用 `treg invites` 查看，用 `treg accept` 接受（或用 `treg org join <code>`）。切换团队：`treg org use <slug>`。

## 已通过 MCP 接入？那你拥有的是工具，而不是 CLI

如果你是通过 `https://treg.to/mcp/` 接入 treg 的——ChatGPT、Claude Code、Cursor——上面的 CLI 步骤不适用于你。你拥有 `catalog_search`、`catalog_get`、`call`、`call_media`、`resources_list`、`balance`、`my_tools`、`catalog_request`、`feedback` 和 `review`。本文档的所有内容都对应到这些工具上：

- "搜索工具目录" → `catalog_search`，再用 `catalog_get` 查确切价格和参数。它的 `verdict` 说明结果成色：`strong`（这些端点能胜任该任务，其中的每家服务商都按实测成功率排序）、`closest`（最接近的选项，请查看 `catalog_get`）、`none`（目录中没有：提交 `catalog_request`，不要换措辞重搜）、`keyword`（仅按关键词排序）
- "有什么新的" → `catalog_search` 加 `added_within_days`（默认取 30 即可）或 `sort: "newest"`；`query` 可以为空。
- "调用它" → `call` 加端点 id，或对团队自己的工具用 `<tool-name>/<path>`
- "查余额" → `balance`
- "反馈" → `feedback`

下述规则两种方式都适用。其中最重要的一条——**花钱之前先报价格**——在这里更重要，因为 `call` 会返回 `cost_usd`，你可以报告调用的实际花费而不是估算。

调用工具目录端点会消耗团队余额。调用团队自己的工具不花钱：密钥本来就是他们自己的。

## 任务——工具目录：treg 能为你做什么（从这里开始）

113 家服务商的 4,000 多个已收录端点，按它们能做什么分组：关键词与排名追踪、外链与权重、AI 可见度、热点与发现、发布到团队自己的社交账号、人物与企业信息增强、广告管理与素材、效果衡量、视频与图像生成。

```bash
treg catalog search "subreddit posts"            # find endpoints by what they do
treg catalog search --new                        # tools added in the last 30 days, newest first
treg catalog get scrapecreators.reddit.subreddit.posts   # params, PRICE, how you'd be served
treg call scrapecreators.reddit.subreddit.posts --query subreddit=news
treg balance                                     # the prepaid balance + recent charges
treg catalog request "<what you need>"           # searched, not there? file it — steers what's added next
```
说明：
- 每个端点的价格都写在 `treg catalog get` 里，调用前即可查看。
- 既搜索任务本身，也搜索你需要的输出：`treg catalog search "email verification catch-all"` 与宽泛的邮箱验证搜索会找到不同的工具。选定前用 `treg catalog get` 逐一核对候选的字段、参数和价格。如果你需要某家服务商的 catch-all、一次性邮箱、角色地址或 SMTP 判定字段，就直接调用该服务商的端点：路由端点 `treg.people.email.verify` 的契约只定义了 `valid`、`status` 和 `score`。
- 要在一次调用中同时找到并校验邮箱，给 `treg.people.email.find` 加上 `--header "X-Treg-Route-Verify: true"`。校验单独收取一笔小额费用。读取 `_treg.verification.verdict`（`valid`、`invalid`、`catch_all`、`risky`、`unknown`）；`checked: false` 表示未执行校验。在 `treg.people.phone.find` 上使用同一请求头可实时检测号码状态：`live`、`dead` 或 `unknown`。绝不要声称某个电话"已验证"：线路在线并不能证明这就是此人的号码。
- 工具目录端点可以使用无需服务商密钥的已验证公开路由。当调用方不提供服务商凭据时，这类调用免费。团队自有工具或已存储的服务商密钥仍然优先。
- 发现类任务在目录中通常有两种形态——结构化的（筛选条件：职位、地点、粉丝数、融资）和语义化的（描述你想要什么；`exa.*`）。当需求同时包含硬性限制和模糊的细分领域时，两种都跑再合并：例如创作者 = `influencersclub.creators.search`（筛选）∪ `exa.creators.search`（描述，限定在平台域名内）；人物 = 某家 `people.search` 服务商 ∪ `exa.people.search`；"谁担任某职务"类问题 = `exa.web.answer`。
- HTTP **402** = 余额不足，响应体可直接指导处理（`balance_micro`、`estimated_cost_micro`、`topup_url`）。恢复方法：`treg balance` → 在控制台充值（Team → Billing）→ 或者为该服务商存入组织自己的密钥（自有密钥绝不计入余额扣费——它们自动优先）。带 `error: route_max_cost` 的 402 是另一回事：是你自己的 `X-Treg-Route-Max-Cost` 请求头在任何扣费发生前拒绝了这次调用——请减少行数/目标数，或调高上限。
- **从文件读请求体：**`treg call <id> --data @body.json`（`@-` 读取标准输入）。在 Windows PowerShell 上请优先用它而不是内联 JSON，PowerShell 会剥掉内联 JSON 中的内部引号。
- **脚本批量调用：**使用 `treg --json call …`。stdout 只有一行，`{"result": <provider body>, "_treg": {"http_status", "call_id", "charged_micro"}}`，且不向 stderr 输出任何内容，因此合并了两个流的脚本仍能解析每个答案（`--await` 的输出不变）。先跑几个并检查解析结果，再对整个列表循环：解析 bug 会白白丢掉已经计费的答案。
- 实际扣费体现在响应头 `X-Treg-Cost-Micro`（微美元）中，引用时使用 `X-Treg-Call-Id` 作为调用 id。幂等重放时该值为 `0`，首次调用的费用显示在 `X-Treg-Original-Cost-Micro`。付费的答案弄丢了？`GET /calls/<call id>/result` 会返回成功目录调用的已存响应。对异步提交，该头部是预留的费用上限；CLI 会将其标注为预留，任务到达终态时按实际费用结算。目录中 `per_result` 路由的 `~$/call` 参考值，在按行计价时按每页 20 行估算；当目录的 `cost.unit` 为 `target`/`domain`/`keyword` 时，按所查询的对象计费，每个目标计一个单位。失败的调用（从服务商转发来的 4xx/5xx）不收费；空结果的含义以服务商的定义为准——treg 只负责转发，不做归一化。服务商在 2xx 响应内部报告失败（DataForSEO 任务状态码不为 20000）时会打上 `X-Treg-Upstream-Failed` 标记（MCP 下为 `upstream_failed`）：响应体中没有结果，费用按服务商对该次尝试的开单计。
- 只要答案还新鲜，一次调用可能直接由 treg 对同一问题的存档应答：返回服务商的原样字节，并附 `X-Treg-Cache: hit`、`X-Treg-Fetched-At`、`X-Treg-Age`。无论是否命中，你的团队对同一问题的首次调用都按全价计费；从第二次调用起，命中存档只收 10% 的费用，使用自有密钥命中则免费。`Cache-Control: no-cache` 强制实时调用；`X-Treg-Max-Age: <seconds>` 只接受不超过指定时长的答案。团队管理员可为团队退出存档（`treg org archive off`）：此后新调用既不从存档应答，也不再写入存档，计量调用因此失去重复调用的优惠价。
- HTTP **503** `provider_capacity_unavailable` = treg 在该服务商处的自有账户额度用尽（不是你的余额；未产生扣费）。响应体含 `resets_at` 和 `alternatives`（相同能力的其他服务商）——任选其一，或使用你自己的密钥。treg 绝不会替你切换服务商。treg 大约每分钟复查一次服务商状态，一分钟后重试可能就成功了。
- HTTP **503** `provider_paused` = 本次部署暂停了该服务商（不扣费、不发送）。阅读 `message`。现有连接保持不变；不要重连，恢复后即可继续使用。
- 组织自有的该服务商工具或密钥总是自动优先于匿名路由或 treg 的密钥——工具目录是兜底，不能替代团队已有的密钥。
- **在同一能力的多个服务商之间选择——流程如下。**`treg catalog get <id>` 会列出承接同一任务的每个服务商，附 `COST`、`WORKS`（treg 观测到的成功率及样本量）、`SPEED`（中位数）和 `LAST OK`。按以下顺序决策：
  1. **匹配你实际拥有的输入。**当你手里只有姓名和域名时，需要 `profile_url` 的端点无论多便宜都不能替代。这条规则永远排在价格之前。
  2. 其次看**可靠性**：真实样本下的高 `WORKS` 胜过小样本下的整数——`99% (121)` 比 `100% (8)` 更有说服力。
  3. 再看**价格**。同一能力内部的价差可达 200 倍，所以省钱通常省在这里。
  4. `LAST OK` 用于打破平局。裸的时长表示有真实调用返回过；带 **`✓` 的时长是工具目录自己的验证戳，不是真实流量**；`—` 表示既没人验证过也没人调用过——几乎任何其他选择都更好。
  - **调用因 429 / 5xx / 超时失败时，换下一家服务商。**你知道它的参数，可以自己构造请求。要说明你切换到了哪家。
  - **绝不要把 4xx 拿到别处重试。**4xx 通常出在你自己的参数上；修正参数才是正解，为一个小错误在 N 家服务商身上重试只会烧掉团队的钱。
  - treg **不会**替你在**服务商之间**做选择或故障转移。这是刻意设计：只有你知道自己手里有什么输入，而 treg 只转发、不改写你的请求。如果 treg 在某服务商处的自有账户额度用尽，它可能通过 treg 自有的中转应答**同一个端点**（`X-Treg-Served-Via: overflow:<name>`，或在 MCP `call` 结果中表现为 `served_via` 加一条提示；真实价格，结构不变）。当部署能为该端点提供中转时，`catalog_get` 会以 `overflow_price_usd` 预先标出该价格——带此价格的"免费"端点可能就按这个数计费，所以报价时要把它算进去。团队可用 `treg org overflow off` 关闭溢出。
  - **常见任务请调用路由工具，而不是你记住的某家服务商。**抓取页面（`treg.web.extract`）、网页搜索（`treg.web.search`）、Google 结果（`treg.google.serp.organic`、`treg.google.serp.maps`）、查找或校验邮箱（`treg.people.email.find`、`treg.people.email.verify`）以及大多数社交资料都有对应的 `treg.<capability>` 工具。treg 会挑选服务商（自有密钥优先），在失败或未命中时自动回退，并且只对最终返回的答案收费。上周用的服务商今天未必最优：价格和可靠性都在变。只有当你需要某家独有的字段或选项时才点名服务商。
  - **路由端点**（`treg.<capability>`，如 `treg.people.email.find`）是你可以让 treg 代为选择的地方：POST 身份信息（`{full_name, domain}` | `{first_name, last_name, domain}` | `{linkedin_url}`）；treg 会运行最优的子端点（自有密钥优先，然后按每次命中的价格从低到高），在出错**和**未命中时回退（从便宜的开始，受 `X-Treg-Route-Max-Cost` 约束，默认 $1；未命中也可能被计费，所以路由行的报价是 `usd_per_call: null` 而非 $0——批量运行请设置该请求头），并返回 `{output, raw, _treg.served_by, _treg.tried}` + `X-Treg-Served-By`。`X-Treg-Route-Waterfall: 0` 会在首次未命中时停止。某个服务商无法应用的筛选条件（对只支持按名搜索的服务商传 `country`）仍会发给其他服务商，答案会通过 `X-Treg-Ignored-Filters` / `_treg.ignored_filters` 指明——要么自行做后置过滤，要么发送 `X-Treg-Route-Strict-Filters: 1` 换取 422（不计费）而不是一个更宽松的答案。`catalog_get treg.people.email.find` 会展示执行计划和价格。异步的子端点提交后，内部会轮询最长 60 秒。如果仍在运行，treg 返回 HTTP 202，附 `_treg.outcome: pending`、调用引用和轮询描述符、`reserved_micro`，以及 `charged_micro: null`；不要重试或另起服务商，因为现有任务可能仍在运行并产生扣费。
  - **找到的联系方式不等于核实过的联系方式。**邮箱或电话查找返回的是服务商的最佳匹配；只有 `output.verified: true` 才代表查证过邮箱。未核实时，答案会通过 `_treg.advice` 指名校验步骤（`treg.people.email.verify`，不到一美分）——外联前先执行它，并且绝不要重发同一次查找：每次命中都计费，重复也不例外。
  - **发送前必校验。每个地址、每一次。**这同样适用于公司或域名搜索返回的行（`treg.people.search`、`hunter.companies.emails` 等）：那些是目录列表，除非该行自己的校验字段另有说明，否则其邮箱未经确认。把 `invalid` 当作死信，把 `accept_all` 当作高风险。也绝不要发给服务商没有返回的地址——如果域名搜索结果为空，`info@` 只是猜测，不是结果。2026-09-08 的真实案例：某团队 79 封退信中有 73 封来自未核实的行和猜测，而每一封只需一次 $0.006 的校验调用就能避免。
- 没有公布价格的端点会被直接拒绝，而不是免费应答；请接入你自己的密钥。

## 任务——生成视频、图像、语音或音乐

生成类模型收录在工具目录的 `video-gen`、`image-gen`、`voice-gen` 和 `music-gen` 平台下，每个模型在每条路由各占一行，因此同一模型出现在两条路由时会并排展示两个价格。模型之间不可互换——由你选定一个；treg 不会替你选。

```bash
treg catalog search "text to video"                  # every model, with prices
treg catalog get minimax.video-gen.h3.generate       # native params, model enum, price table, async descriptor
treg call minimax.video-gen.h3.generate --await --timeout 900 --data '{"model":"MiniMax-H3-Max",
  "content":[{"type":"text","text":"A paper boat drifts across a quiet pond at sunrise."}],
  "resolution":"480P","duration":5,"ratio":"16:9"}'
treg call minimax.voice-gen.voices.list --data '{"voice_type":"system"}'
treg catalog get minimax.voice-gen.speech-2-8-turbo
treg call minimax.voice-gen.speech-2-8-turbo --data '{"model":"speech-2.8-turbo",
  "text":"A calm voice can make a complex idea feel simple.","stream":false,"output_format":"url",
  "voice_setting":{"voice_id":"English_expressive_narrator","speed":1,"vol":1,"pitch":0}}'
```
工作方式：
- **语音生成是同步的。**MiniMax 返回的 JSON 中含一个 24 小时有效的音频 URL。目录路由固定了 `stream:false` 和 `output_format:"url"`；先用语音列表操作发现可用的系统音色 ID，再按端点 id 选择 HD 或 Turbo 版本。
- **在平台密钥下，Fish Audio 返回二进制数据且按团队隔离。**使用 `fishaudio.tts.s2-1-pro` 并带请求头 `model: s2.1-pro`；把 CLI 的 stdout 重定向到音频文件，或使用 MCP 的 `call_media`。要使用 Fish 提供的音色，先调用 `fishaudio.voices.discover`（`self=false`、`licensed=true|false`），再把返回的 `_id` 作为 `reference_id` 传入；`true` 选中 Fish 已取得版权授权的子集，`false` 则包含公共/社区音色。在使用平台密钥合成之前，treg 会把该非团队 id 复核为公共音色。用 `fishaudio.voices.create` 创建可复用的私有音色，然后用 `treg resources list --provider fishaudio --kind voice` 或 MCP `resources_list` 列出它们的 id。BYOK 下列出的是已连接的 Fish 账号，否则只列出当前团队的平台音色；curl 对应 `GET /orgs/{org_id}/provider-resources?provider=fishaudio&kind=voice`。BYOK 仍是不受限、不计量收费的中转。`treg resources list` 需要 CLI ≥ 0.21.0；如果无法识别 `resources` 命令，请运行 `treg update`。
- **视频或图像生成调用是异步任务。**提交后立即返回任务 id；`--await` 会轮询服务商直到完成，并且只在 stdout 打印**最终响应**。stderr 承载任务 id、一条可续跑的 `treg call …` 命令（Ctrl-C 只会中断等待，任务和已付的钱都还在）、进度信息和结果 URL。退出码 0 = 完成，2 = 服务商判定任务失败，3 = 超时（用打印出的命令续跑）。
- **有些异步工具通常立即给出答案**（如邮箱校验）：当首个响应已经完成时，它就是答案，按一次计费；`--await` 直接打印而不再轮询。如果它显示"仍在处理"，轮询频率不要高于描述符的 `interval`，并在 `max_age` 秒后停止；届时仍无答案的任务会退款。
- **参考素材（人脸图、声音片段、首帧）必须是服务商可抓取的公开 URL。**不要用临时文件托管（paste host）：它们会让服务商的探测随机失败（catbox、tmpfiles、uguu 都出过问题）。`treg host face.jpg` 会输出一个公开 URL（30 MB、7 天、免费），可直接填入 `image_urls` / `audio_urls`：`--data "{\"image_urls\":[\"$(treg host face.jpg)\"], …}"`。需要 CLI ≥ 0.20.0；如果无法识别 `treg host`，请运行 `treg update`。
- **CLI agent：调高 shell 工具的超时时间，或把调用放到后台运行。**生成一条视频需要 1-5 分钟；运行时默认的 2 分钟命令上限会在等待中途把它切断。
- **MCP 和原生 HTTP agent：**响应头 `X-Treg-Async` 就是描述符——去哪里轮询、哪些状态值是终态、结果在哪里。轮询要克制：首次检查前等约 60 秒，之后每 30-60 秒一次；每条视频检查三到六次属正常。中间可以去做其他工作。
- 参数就是服务商自己的原样参数；`treg catalog get` 会展示它们，包括可选模型和分辨率的枚举。treg 不在各服务商之间翻译任何参数。
- **费用：**提交时预留价格，任务成功后才扣费。失败或被内容审核拦截的任务会退还全部预留——你无需做任何操作。`treg audit` 和控制台的 Activity 页会显示每个任务的状态（`generating…` → `done` / `failed · refunded`），结果链接生成后也会附上。
- **结果 URL 会过期**（描述符的 `ttl_note` 说明多久；MiniMax 约 9 小时）。请尽快下载；treg 绝不存储媒体文件。在某些路由上，取文件还需要再调用一次——`--await` 会打印那条确切命令而不是直接下载。
- 需要用作结算或任务归属凭证的响应体以 8 MiB 为限。更大的响应会返回 `502`、`detail.error=response_buffer_limit`，且不收费；重试同一个超大响应也无济于事。经授权、无需响应体凭证的免费最终下载会完整流式返回。这类下载在重试时会重新抓取，不会保留用于本地幂等重放。
- 答案内嵌媒体的工具（Google AI 的图像、语音和 Lyria 音乐以 base64 形式出现在 JSON 中）豁免 8 MiB 上限，最高可到 64 MiB，并按服务商上报的用量计费。它的答案永不重放：带 `Idempotency-Key` 重试会重新生成并计费一张新图。如果 `response_buffer_limit` 的消息表明是临时性的，说明 treg 正忙：稍后重试。

## 重试调用而不花两笔钱

如果调用超时或你始终没看到答案，用相同的 `idempotency_key`（MCP 下）或 `Idempotency-Key` 请求头（HTTP 下）重发。treg 会返回已存的答案，不再调用服务商，也不收费。结果中会标明 `replayed: true`。

只适用于真正的重试。为了看结果有没有变而再次提问属于**新**工作：换新 key 或不带 key，否则拿回的还是旧答案。同一个 key 用于不同的请求会被拒绝。409 表示原调用仍在运行：稍后重试。410 `idempotency_response_lost` 表示已扣费但答案未保留：试试 `GET /calls/{call_id}/result`，或换新 key。410 `idempotency_outcome_unknown` 表示结果未记录：用 `GET /calls/{call_id}` 查看费用；换新 key。

大多数重试根本用不上这些——失败的调用从未被计费。

标记为 `strict_query: true` 的目录条目只接受其声明的查询参数，且每个只用一次。请包含必填字段、使用允许的取值，并省略请求体。

## 任务——你自己的工具：调用团队注册的工具

**从已注册的工具出发，然后完全按其自身文档所述使用 API。**没有 treg 专属词汇，没有特殊参数：

```
treg tool ls                                  # what this team has registered
treg call intercom conversations?per_page=5   # <tool-name> + the upstream path
```

在 HTTP 上对应 `GET https://treg.to/call/<tool-name>/<path>`，带 `X-Treg-Token: <your token>`。treg 查找指定的工具，在服务端注入该团队的凭据，并**原封不动地转发一切**（方法、查询参数、你的请求头、请求体）。你的 `X-Treg-Token` 在上游看到之前会被剥除。支持 GET/POST/PUT/PATCH/DELETE。

只有本组织注册过的工具才能被解析。用 `treg tool ls` · `treg skill ls` 来发现它们。

## 任务——分享你的密钥和 skill，让团队成员的 agent 也能用
**批量（快速路径）：**在用户指定的目录里运行。它会列出该 `.env` 中它能识别的服务商密钥和各子目录中的 skill，并只注册用户勾选的部分：
```bash
treg upload                       # both sides of the cwd; `treg upload env|skills --dir <d>` to restrict
```
**默认做法：把新密钥包进一个 skill。**注册新密钥/端点/CLI 时，为它搭配一个 skill，让凭据、工具和配方一起落库（还会得到一个可分享的页面）。如果没有现成的 skill，就创建一个基础的——一份像样的 SKILL.md（frontmatter 很重要：agent 靠它发现 skill）加上一个示例调用：
```bash
mkdir -p ./posthog && cat > ./posthog/SKILL.md <<'MD'
---
name: posthog
description: Query the PostHog analytics API through treg — the key is injected server-side. Use for events, insights, and project queries.
---
Call it: `treg call posthog api/projects/@current` (upstream: https://us.posthog.com)
MD
treg skill init --dir ./posthog   # drafts treg.json: base_url from the catalog (folder name) or URLs in SKILL.md; review it + add the key
treg skill add --dir ./posthog    # registers recipe + secret + tool atomically
```
**绝不让密钥落单：**存进去了却没有任何东西绑定它，就是死重量——如果直接用 `secret add`，请顺手把它绑定到一个工具（端点/CLI）上。
**裸端点，不带配方（仅当 skill 加不了任何价值时）：**
```bash
treg secret add posthog-key --value "$POSTHOG_API_KEY"          # or --file ./.secret/token.json
treg tool add posthog --base-url https://us.posthog.com --secret posthog-key
# query-key API instead of a bearer header:
treg tool add serpapi --base-url https://serpapi.com --secret <name-or-id> \
  --auth-in query --auth-name api_key --auth-format '{secret}'
```
**整个 skill（配方 + 密钥 + 工具，可能多凭据）：**
```bash
treg skill scaffold ~/.claude/skills/google-ads --out gads.json
#   -> walks the dir: captures SKILL.md as the recipe + every .secret/* as a secret.
#   -> YOU then edit gads.json: set base_url, and complete each binding (location/name/format).
#      e.g. google-ads needs TWO bindings on one request:
#        Authorization: Bearer {access_token}  (injector: oauth)
#        developer-token: {secret}             (injector: env)
treg skill push gads.json                                        # registers the bundle atomically
```
在组织内分享：把端点 + 工具名告诉一位团队成员，他们的 agent 就能调用它，而**不需要经手凭据**——访问由你授予，密钥由 treg 注入，调用会记录在他们的 token 名下。

**认证形态**（按每个绑定的 `injector`，即密钥的 `kind`）：`env`（纯字符串）· `secret_file`（JSON token 文件，取 `secret_field` 字段）· `oauth`（JSON token，自动刷新）· `cli_auth`（用户从自己已登录的 CLI 里复制出来、亲自交给 treg 的 token）。多个绑定会对每个请求全部生效。

**OAuth，两种模式（treg 负责保鲜）：**如果 oauth 密钥带有 `refresh_token` + `client_id` + `client_secret`，treg 会在其过期前**自动刷新**（你永远不需要重新上传）。如果只是裸 token，那就是**手动模式**，treg 原样注入，过期后由你重新上传。存储方式相同；凭据可以从手动升级为自动，无需任何迁移。

**拿到第一个 OAuth token 的两种方式（任选其一）：**
- **手动：**自己在本地完成 OAuth，然后执行 `treg secret add gsc --file token.json --kind oauth`。
- **托管连接：**`treg oauth connect gsc --client-secret client_secret.json --scopes <scope>` → 会输出一个授权同意 URL；你在浏览器里批准；treg 直接捕获 token。一次性设置：在你的 OAuth 应用的重定向 URI 中加入 `https://treg.to/oauth/callback`。

## 任务——管理团队与监控

一个账号最多可拥有 10 个团队。以成员身份加入其他团队不占此限额。
```bash
treg tool ls / secret ls / skill ls / calls          # inventory + audit log — scoped to the active org
treg tool rm <id> / secret rm <id> / skill rm <id>   # secret rm is blocked while a tool binds it
treg health            # status of every credential in this org (ok | invalid | unknown)
treg health --run      # re-check now: refresh oauth tokens, probe each tool, alert owners
```
**团队/组织**（owner > admin > member > viewer；member 只能管理自己创建的内容）：
```bash
treg org create "Team A"                       # you become owner (auto-active)
treg org rename --name "Team B" --slug team-b   # admin+; existing keys keep working
treg org invite bob@company.com --role member  # admin+; emails the invite (a one-time code is the fallback)
treg org members                               # admin+; who's in the active org
treg org ls / treg org use <slug>              # your orgs / switch active
```
**给 agent 一个独立身份**（admin 及以上）。agent 不必借用人类的 token——为它铸造一个，它的每次调用都有上限、有范围，并以*它自己*的名义记录：
```bash
treg org agent-new customer-bot --pin customer=cust_A  # scope history and shared-provider async reads
treg org agent-new ci-bot                        # prints the token ONCE (run again to rotate)
treg org agent-new ci-bot --tools stripe,gh --cap 500   # only these tools, 500 calls/day
treg org agents                                  # who the team's agents are + today's usage
treg org agent-rm <user_id>                      # revoke instantly
```
把该 token 放进 agent 的 `TREG_TOKEN` 环境变量。agent token 只能**调用本团队的工具并读取**——它永远不能登录、创建团队或成为 owner。如果你是 agent 且拿到了自己的 token，请使用它而不是机器主人的：这样你的工作会以你自己的名字出现在 `treg calls` 里。

被邀请者用受邀邮箱登录并运行 `treg accept`——无需处理邀请码（邀请码路径仍然可用：`treg org join <code>`）。全新被邀请者还会自动获得自己的**个人组织**（不会处于空状态），因此把他们移出团队也绝不会把人锁在外面。给工具配置一个探针，让 treg 能验证它：`health_check: {method, path, expect_status}`（例如 intercom 用 `{"path":"me"}`）。

## 反馈

主动分享关于 treg 的问题和建议。拖慢任务的小别扭也算：不清楚的说明、令人困惑的结果或扣费、没有帮助的报错，以及你不得不绕过的缺失能力——即使任务最终成功了也要说。描述你需要什么、观察到了什么；不需要证明这是个 bug。把观察到的现象与推测的原因区分开。MCP 下使用 `feedback` 工具，CLI 下使用 `treg feedback submit <quality|pricing|friction|other> "message"`。相关调用的 ID 请通过 `call_ids`（CLI：`--call-id`）传入，不要只写在消息里；引用是可选的。
每个问题只报告一次。反馈是帮 treg 团队改进产品，并不能替用户完成任务。反馈之后继续手头的工作。
不要在报告中包含隐私信息。参见[反馈说明](https://treg.to/feedback.md)。

## 评价

如果某次调用结果邀请你评价，在使用之后对该次调用打分：MCP 下用 `review(call_id, usefulness, reason?)`，CLI 下用 `treg review CALL_ID USEFULNESS [--reason "..."]`。可选 `useful`、`partly`、`not_useful` 或 `not_sure`；拿不准也没关系。每次邀请对应一条评价；未受邀的调用的评价也会被接受，每个团队对每个端点只计一次。评价理由可能在不点名团队的情况下被引用在端点的目录页上：只写端点本身，略去隐私数据，任何困惑或错误都用 `feedback` 反馈，然后继续工作。

## 规则
- 密钥**只写不读**——无论是你还是任何人，API 都不会返回已存储的值。
- 工具可以绑定**团队成员共享给本组织**的密钥（use-without-hold，用而不持）——这正是意义所在：对方选择共享，密钥始终限定在组织范围内，你可以在看不到它的前提下使用它，且每次调用都记在发起调用的 token 名下。这是团队内部的委托访问，绝不是获取任何无人授予你的凭据。
- **一切都以你的活跃组织为范围。**一个 token 只能触达该团队的工具，别人的不行。
- 代理不理解上游；调用失败时，你看到的状态就是上游的真实情况。
- 更多：`https://treg.to/llms.txt`（agent 上手指南）· `https://treg.to/tutorial`（交互式演练）。
