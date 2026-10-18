# treg — CLI 使用手册

`treg` 是 **tools-registry** 的命令行客户端：无需持有凭据即可调用团队共享的工具，并把你本地的 skill（技能）变成可共享的工具。它是构建在 API（`https://treg.to`）之上的一层薄客户端；API 才是唯一的大脑。

每条命令都会读取 `~/.treg/config.json` 来获取端点地址和你的令牌。用 `treg <command> --help` 查看单条命令的详细说明（例如 `treg secret --help`，而不是 `treg "secret add"`）。

## 安装（一次即可）

```bash
cd tools-registry
uv tool install --editable . --python 3.13     # puts `treg` on your PATH, tracking the dev source
# code edits reflect with no reinstall; re-run with --reinstall only after a dependency change
```

## 核心理念

**两类可调用的工具都位于同一个 `/call/` 入口之后。**

- **工具目录（catalogue）** — 约 2,600 个经过精选的外部端点，覆盖约 40 家数据服务商（TikTok、Instagram、Reddit、SEO 与 SERP 数据、信息补全、网页抓取……）。通过**端点 id** 调用。如果团队里没人持有该服务商的密钥，treg 可以用它**自己的密钥**为你提供服务，按次计费，从团队预付余额中扣除。无需在服务商处注册。
- **保险库（vault）** — 团队自己的工具。成员注册的任何 API 密钥、OAuth 连接或厂商 CLI，都会对每位队友及其 agent 可调用，而密钥永远不会离开服务器。

保险库部分的几个术语：

- **工具（tool）** = 一个上游 `base_url` + 一组凭据**绑定（binding）**（每个绑定向请求注入一个机密）。一个请求可以携带多个（例如 google-ads：OAuth bearer + 一个 `developer-token` 请求头）。
- **skill / bundle（捆绑包）** = 一份配方（SKILL.md）+ 其机密 + 其工具。
- 代理对上游**只做转发、从不建模**，并且**在服务端注入认证**，因此调用方永远不需要持有密钥。你的 `X-Treg-Token` 会在上游看到请求之前被剥除。

---

## 设置 / 身份

认证采用**身份优先**：先证明一次你是谁（GitHub 或邮箱验证码——首次验证同时会注册你的账号并创建你的个人组织），之后即可在你的所有组织下工作。Agent/CI 则可以直接出示按组织颁发的令牌。

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg config` | `--base-url URL` | 查看或设置端点（打印基础 URL、邮箱、当前组织、登录状态）——用 `treg org ls` 查看完整的组织列表 |
| `treg login` | _（无）_ | GitHub 浏览器登录（注册即登录），保存一个身份令牌 |
| `treg login` | `--email EMAIL` | 邮箱一次性验证码登录（注册即登录）；会提示输入验证码。若服务器要求浏览器校验，则会改为在浏览器中完成登录 |
| `treg login` | `--token TOKEN` | agent/CI：直接使用按组织颁发的令牌 |
| `treg logout` | — | 清除已保存的凭据 |
| `treg onboard` | `--mode guided\|quick` · `--name N` · `--yes` · `--reset` | 生动友好的首次运行引导——选择**引导式搭建**（一步步创建团队并邀请一位队友）或**快速演示**（我们为你预置一个完整的演示团队 + 工具 + 活动），最后以一次免密钥调用收尾。首次登录后会提示 `[Y/n]`；`--reset` 会移除演示队友 |

```bash
treg config --base-url https://treg.to
treg login                                 # GitHub; or:
treg login --email you@example.com        # emailed 6-digit code (register-or-login)
```

## 团队 / 组织

**组织（org）**（即团队）拥有工具和机密。执行 `treg login` 后，CLI 会在 `~/.treg/config.json` 中保存一个**身份令牌**，并以 `X-Treg-Org` 请求头发送你**当前**的组织；`treg org use <slug>` 用于切换。（若使用 `treg login --token`，则直接使用那个按组织颁发的令牌。）其余所有命令都在当前组织下运行。角色：**owner > admin > member**（member 只能调用并管理自己创建的内容；admin/owner 可管理组织内的任何内容；admin 及以上可以邀请/移除成员）。

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg org create` | `"NAME"` | 创建新组织；你成为它的 owner（生成新令牌，自动激活） |
| `treg org ls` | — | 列出你的组织（标记当前组织） |
| `treg org use` | `SLUG` | 切换当前组织 |
| `treg org rename` | `--name "NAME"`, `--slug SLUG` | （admin 及以上）重命名当前组织并/或修改其 slug；现有密钥继续有效 |
| `treg org invite` | `EMAIL`, `--role member\|admin` | （admin 及以上）创建一个可分享的一次性邀请**码** |
| `treg org members` | — | （admin 及以上）列出成员及其角色 |
| `treg org join` | `CODE`, `--email EMAIL` | 兑换邀请码：新用户会先注册，然后加入组织，并保存组织令牌 |

```bash
# owner side
treg org create "Team A"                       # active org is now team-a
treg org invite bob@company.com --role member  # prints e.g. inv_7Kd9x2LmQpR4 — send it to Bob

# Bob's side (no email is sent; he gets the code over Slack/DM)
treg config --base-url https://treg.to
treg org join inv_7Kd9x2LmQpR4 --email bob@company.com   # joins + mints HIS own token
treg tool ls            # sees only Team A's tools
treg org use team-a     # switch orgs anytime; one token per org, never mixed
```

## 机密（凭据——只写；API 永不返回已存储的值）

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg secret add NAME` | `--value V` \| `--file PATH`, `--kind env\|secret_file\|oauth\|cli_auth` | 上传一个凭据（一个字符串值，或一个文件的内容） |
| `treg secret ls` | — | 列出（名称 / 类型 / 所有者） |
| `treg secret rm ID` | — | 删除（仍被某个工具绑定时会被阻止） |

```bash
treg secret add posthog-key --value "$POSTHOG_API_KEY"
treg secret add gsc --file ./.secrets/token.json --kind oauth
```

## 工具

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg tool add NAME` | `--base-url URL`（必填） | 注册一个工具 |
| — 单绑定 | `--secret ID`, `--injector`, `--auth-in`, `--auth-name`, `--auth-format`, `--secret-field` | 最常见的情形 |
| — 多绑定（友好格式） | `--bind 'secret=ID,injector=,location=,name=,format=,secret_field='`（可重复） | 只有 `secret=` 是必填的 |
| — 多绑定（原始格式） | `--binding '<json>'`（可重复） | 完整的绑定字典，面向高级用法 |
| — 健康探测 | `--health '{"path":"me","expect_status":200}'` | 可选的验证探测 |
| `treg tool ls` | — | 列出工具及其绑定 |
| `treg tool rm ID` | — | 删除 |

单个 `--secret` 和每个 `--bind` 的**默认值**为：`injector=env`、`location`/`auth-in=header`、`name`/`auth-name=Authorization`、`format`/`auth-format=Bearer {secret}`、`secret-field=access_token`。

```bash
# single credential
treg tool add posthog --base-url https://us.posthog.com --secret 3
# query-key API instead of a bearer header
treg tool add gsc --base-url https://searchconsole.googleapis.com --secret 3 \
  --injector oauth --secret-field token
# two credentials on one request (google-ads)
treg tool add google-ads --base-url https://googleads.googleapis.com \
  --bind "secret=4,injector=oauth" \
  --bind "secret=5,name=developer-token,format={secret}"
```

## 工具目录（调用你没有密钥的 API）

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg catalog search` | `"你想做的事"`, `--new [DAYS]`, `--sort newest` | 按能力查找端点，与 agent 使用的搜索相同（指明服务商名，如 "exa search"，可让其工具排在前面）；`--new` 只保留最近 DAYS 天内新增的工具（不带值时默认 30 天，最大 365 天），`--sort newest` 按最新在前排列；使用其中任一选项时，搜索词可省略 |
| `treg catalog get` | `ENDPOINT_ID` | 文档、参数、**价格**，以及你会以何种方式获得服务 |
| `treg call ENDPOINT_ID` | `--query K=V`, `--data STR` 或 `--data @FILE` | 调用它 |
| `treg --json call ENDPOINT_ID` | 同上 | 供脚本使用：在 stdout 输出一行 JSON `{"result": <body>, "_treg": {http_status, call_id, charged_micro}}`，stderr 无输出 |
| `treg call ENDPOINT_ID --await` | `--timeout N`（默认 900） | 生成类调用（视频/图像）：提交、轮询服务商、输出最终响应 |
| `treg host FILE` | `--content-type T`, `--json` | 将参考图像/音频/视频托管在一个厂商可抓取的公开 URL 上（30 MB，7 天有效期，免费）；打印用于 `image_urls` / `audio_urls` 的 URL |
| `treg catalog request` | `"缺少什么"` | 搜了却没有？提交一个请求——这类请求会引导后续收录的方向 |
| `treg resources list` | `--provider fishaudio --kind voice` | 使用 BYOK 时列出 Fish 账号的音色，否则列出当前团队的平台创建音色 |

```bash
treg catalog search "instagram profile"
treg catalog search --new                            # tools added in the last 30 days, newest first
treg catalog search phone --new 60                   # phone tools added in the last 60 days
treg catalog get tikhub.tiktok.user.profile          # shows the price BEFORE you spend
treg call tikhub.tiktok.user.profile --query uniqueId=tiktok
```

**视频与图像生成**（`video-gen` / `image-gen` 平台）是异步任务：调用会返回一个任务 id，`--await` 会轮询直到完成。stdout 只输出最终响应；stderr 会显示任务 id、一条可恢复执行的 `treg call …` 命令、进度以及结果 URL。退出码 0 = 完成，2 = 服务商使任务失败，3 = 超时（用打印出的命令恢复执行）。费用在提交时预留，仅在成功时扣收；失败的任务会退回预留金额。结果 URL 有时效性（请及时下载；treg 从不存储媒体文件）。在编码 agent 中使用时，请调高 shell 工具的超时时间或将调用放到后台运行——一段视频需要 1-5 分钟。`treg audit` 可查看每个任务的状态。

Fish Audio TTS 是同步的，返回音频字节。保持 stdout 干净并将其重定向：

```bash
treg call fishaudio.tts.s2-1-pro --method POST --header model=s2.1-pro \
  --data '{"text":"Hello","format":"mp3"}' > speech.mp3
```

可以通过重复的 `--upload voices=@clip.wav` 参数以及 `treg catalog get fishaudio.voices.create` 所示的固定私有字段来创建音色（voice）；然后把它的 id 用作 TTS 的 `reference_id`。平台创建的音色会出现在 `treg resources list --provider fishaudio --kind voice` 中。使用 BYOK 时，同一命令列出的则是已连接的 Fish 账号。使用 curl 的调用者可访问 `GET /orgs/{org_id}/provider-resources?provider=fishaudio&kind=voice`；它返回同样的规范化行，并带有 `X-Treg-Resource-Source: byok|platform`。BYOK 用户也可以直接调用 Fish 原生的账号级列表。要选用一个可复用的 Fish 官方音色，可调用 `fishaudio.voices.discover`，传入 `self=false` 并显式指定 `licensed=true|false`，然后把返回的某个 `_id` 用作 TTS 的 `reference_id`。`licensed=true` 选取 Fish 已保障版权的子集；`false` 则包含更广泛的公共/社区工具目录。在平台密钥上，treg 会在合成前校验非本团队的 id 仍是公开的。`treg catalog get fishaudio.tts.s2-1-pro` 列出了所支持的生成、韵律、输出格式、比特率、采样率、延迟与分块控制参数。

**一次工具目录调用是如何被服务的——凭据阶梯（credential ladder），按顺序：**

1. 团队为该服务商注册了自己的工具 → 使用该工具、该密钥；
2. 团队存储了该服务商的机密 → 通过一个虚拟工具注入；
3. 两者都没有 → 使用 **treg 自己的密钥**，按次计量并从团队预付余额中扣费。

第 3 级仅适用于 treg 对该端点既有密钥、又有公开定价的情况；任何未定价的端点都会被拒绝服务而不是免费提供，并提示你接入自己的密钥。你自己的密钥永远不会被计入余额计费。

## 评价一次工具目录调用

`treg review <call_id> <useful|partly|not_useful|not_sure> [--reason "text"]` 用于在你使用结果之后对一次工具目录调用打分。第一阶段仅邀请在 treg 自有平台密钥上服务的直接目录调用；被抽样的调用会在 stderr 上打印邀请。路由调用和自有密钥的目录调用即使未收到邀请也可以评价。分不清时选 `not_sure` 即可。可选的 reason 去除首尾空白后为 1-200 个字符。请不要包含私密数据和原始载荷。遇到任何令人困惑或错误的地方请使用 `feedback`，然后继续手头的任务。回执包含 `review_id` 和 `status`（`received` 或 `already_reviewed`）。如果调用记录尚未写入，请稍后用同一个调用 ID 重试。

## 反馈

`treg feedback submit <quality|pricing|friction|other> "message"` 会以当前团队的身份向所配置的注册中心提交一个问题或建议。如有可用的 `--call-id ID`（可重复）或 `--endpoint-id ID` 请一并附上。消息为 `-` 时会从 stdin 读取一段准备好的、经过脱敏的描述。该命令会返回一个 `feedback_id`。请不要包含私密信息以及原始请求、响应或日志。
用 `treg feedback get <feedback_id>` 取回一份报告。
详情参见 `treg feedback --help` 以及你的注册中心的 `/feedback.md`。

## 余额与充值

只有使用 **treg 密钥**（上文第 3 级）的调用才消耗余额。其他一切——你自己的密钥、你自己的工具、厂商 CLI——都不消耗余额。

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg balance` | `--limit N`, `--json` | 剩余额度、进行中的调用、近期支出 |
| `treg topup` | | 充值，或设置自动充值 |

```bash
treg whoami                  # the account, the active team and your role, the server
treg balance                 # every new team starts with $1.00 of free credit
treg balance --json          # integer micro-USD (1e-6 USD) — the unit the ledger uses
treg topup
```

余额不足时返回 HTTP **402**，其中携带 `balance_micro`、`estimated_cost_micro` 和一个 `topup_url`，agent 无需阅读文字说明即可据此行动。对 treg 密钥的每日支出还设有上限，因此失控 agent 的影响范围是有界的。

如果 treg 在某服务商处的**自有**账号额度用尽，计量调用会在任何预留发生之前被拒绝，返回 HTTP **503** `provider_capacity_unavailable`——不会扣取任何费用，响应体在已知时会给出 `resets_at`，并给出 `alternatives`（提供同一能力的其他服务商）。你自己在该服务商的密钥永远不受影响，treg 也不会替你切换服务商。treg 大约每分钟重新检查一次该服务商，因此一分钟后的重试可能成功。

在部署开启了溢出（overflow）中继的情况下，treg 可能改为通过一个 treg 自有的聚合账号来服务**同一端点**——相同的请求、相同的响应结构，按中继的真实价格计费（0% 加价），响应中带有 `X-Treg-Served-Via: overflow:<name>`。`treg org overflow off` 可让你的团队退出（此后调用会遇到上面的 503）；`treg org overflow` 可查看该设置。自有密钥永远不会经由中继转发。

`treg org archive off` 会把你的团队移出 treg 的答案存档：你的新调用不再以已存答案作答，也不再被记录进存档，因此计量调用会失去重复调用的优惠价（自有密钥照旧免费）。`treg org archive` 可查看该设置；`treg org archive on` 可重新加入。团队已存入的答案会在团队删除时或应要求被清除；调用记录和异步任务完成凭证仍照旧保留。

### 路由端点——让 treg 替你选择服务商

`treg.<capability>` 端点（目前有 `treg.people.email.find`）由同一能力下适配器通过验证的服务商生成。像调用任何端点一样调用它：

```bash
treg call treg.people.email.find --body '{"full_name": "Patrick Collison", "domain": "stripe.com"}'
treg call treg.people.email.find --body '{"linkedin_url": "https://www.linkedin.com/in/patrickcollison"}' \
  --header "X-Treg-Route-Max-Cost: 0.05"     # cap the per-call spend (default ceiling $1)
treg catalog get treg.people.email.find      # the ranked plan with prices — spends nothing
```

treg 会运行最优的子调用——优先用你自己的密钥，其次按每次命中的预期成本从低到高——并返回 `{output, raw, _treg: {served_by, tried, charged_micro}}`，外加 `X-Treg-Served-By` 和 `X-Treg-Providers-Tried`。服务商报错时会回退到下一个候选（最多再多两个）；厂商返回 4xx 说明是你的请求有问题，随即停止。**未命中（miss）**同样会尝试下一个服务商（瀑布模式，默认开启），从最便宜的开始，并受 `X-Treg-Route-Max-Cost` 限制（默认每次调用 $1）；每次尝试都按其真实价格结算，在按成功计费的服务商那里未命中是免费的。`X-Treg-Route-Waterfall: 0` 会在第一次未命中时停止。`X-Treg-Route-Prefer` / `X-Treg-Route-Exclude` 用于指定服务商。实际服务的服务商无法应用的过滤条件会在 `X-Treg-Ignored-Filters`（以及 `_treg.ignored_filters`）中列出；`X-Treg-Route-Strict-Filters: 1` 则会以 422（不计费）拒绝这类调用。厂商端点仍然原样转发；只有 `treg.*` 条目会对 API 建模。

在 `treg.people.email.find` 上使用 `X-Treg-Route-Verify: true` 会在同一次调用中用 `treg.people.email.verify` 校验命中的结果：先使用你自己的校验密钥，然后是 BounceBan，再按常规顺序。该校验是一次独立计费的调用，包含在同一个 `X-Treg-Route-Max-Cost` 之内；`X-Treg-Cost-Micro` 是 find 加 check 的总和。响应会附加 `_treg.verification: {verdict, checked, served_by, cost_micro}`（`valid` / `invalid` / `catch_all` / `risky` / `unknown`）。`checked: false` 加上一个 `reason`（`no_hit`、`over_cost_limit`、`checker_failed`、`no_checker`、`not_allowed`、`spend_limit`）表示没有执行校验，也未产生任何费用。未命中永远不会被校验：其 reason 为 `no_hit`。其他路由工具会拒绝该请求头（422，不计费）。

在 `treg.people.phone.find` 上，同一个请求头会通过 HLR Lookup 的实时网络查询（`hlrlookup.people.phone.verify`）检查线路：先用你自己的 HLR 密钥，`usa_status` 仅适用于 `+1` 号码，不读取缓存。判定结果为 `live` / `dead` / `unknown`；线路存活并不代表这个号码属于此人（`trestleiq.people.contact.verify` 可校验美国姓名）。未按国际格式书写的号码永远不会被发送，因为 HLR 会把它的前几位当作国家代码读取：此时 reason 为 `not_international`，不收取任何费用。

```bash
treg call treg.people.email.find --body '{"full_name": "Patrick Collison", "domain": "stripe.com"}' \
  --header "X-Treg-Route-Verify: true"
```

当路由的子调用为异步时，treg 会在内部提交并轮询它，最长 60 秒。如果它仍在处理中，路由会返回 HTTP 202，附带 `_treg.outcome: "pending"`、子调用的引用和轮询描述符、`reserved_micro`，以及 `charged_micro: null`。该次尝试会就此终止瀑布模式；在第一次调用仍可能完成的情况下，再启动另一个计费服务商是不安全的。

## 调用

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg call TOOL PATH` | `--method GET`, `--query K=V`（可重复）, `--data STR`（或 `@PATH`、`@-`）, `--file PATH` | 以代理方式调用（具名形式） |
| `treg calls` | `--limit N` | 审计日志（谁在何时调用了哪个工具、状态如何） |

```bash
treg call intercom conversations --query per_page=5
treg call posthog api/projects --method POST --data '{"name":"x"}'
treg call dataforseo.google.keywords.volume --data @body.json   # @PATH reads a file, @- stdin
```

看起来像 JSON 但无法解析的请求体会被拒绝而不是发送：PowerShell 会剥掉带引号参数内部的引号，所以请把请求体放进文件（`--file PATH` 或 `--data @PATH`）。带有字节顺序标记（BOM）保存的 JSON 文件（PowerShell 的 `>` 写出的是 UTF-16）会以 UTF-8 JSON 发送，并在 stderr 上给出提示；其他任何文件都会按字节原样发送。传入 `--content-type` 可以让请求体严格按书写内容发送。

**Agent 原生（原始 HTTP）形式** — 构造真实的上游 URL 并加上前缀；无需 CLI：

```
GET https://treg.to/call/https://api.intercom.io/conversations?per_page=5
    + header:  X-Treg-Token: <your token>
```

treg 根据上游主机名解析出工具，注入凭据，并忠实地转发一切（方法、所有查询参数（含重复项）、请求头、cookie、请求体）。

## 运行厂商 CLI

`treg call` 代理的是 **HTTP API**。`treg run` 是它的命令行补充：它会运行一个**厂商 CLI**（stripe、gh、vercel、gcloud、flyctl 等），并在服务端注入该工具的凭据，因此成员运行 CLI 时**完全无需持有或登录该密钥**。仅含配方的工具目录 CLI skill（例如 `stripe-cli` 的 SKILL.md）会自动变为可运行——`treg upload` 会通过服务商工具目录识别它并接好凭据，因此 `treg run` 无需 `treg.json` 即可工作。

工具所有者需要按工具为**本地**运行选择加入（密钥会到达成员的机器）：使用仪表盘的 `⌘ run` 开关，或 `treg tool update <id> --local-run on|off`。**服务器**运行无需选择加入——密钥永远不会离开注册中心，且服务器上的可执行文件允许列表会限制什么可以执行。工具上的一个 `cli` 配置同时驱动这两个层级（`bin`、`inject`、`deny`、`enabled`）。

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg run TOOL -- <cli args>` | `--local`（默认）\| `--server`, `--timeout N`（仅 [--server]） | 运行该工具的 CLI 并注入其凭据；`--` 之后的所有内容原样传给 CLI |
| `treg runs` | `--limit N` | CLI 运行审计日志（谁在何时运行了哪个工具、退出码） |
| `treg setup-local-run` | `--run-proof VALUE`, `--member USER` | （admin、Linux、`sudo`、仅需一次）安装隔离的 `treg-run` 运行器 |

**两个层级：**

- `--local`（默认）— 在成员**自己的机器**上运行。在 Linux 上，管理员需运行一次 `sudo treg setup-local-run`，使 CLI 在专用的 `treg-run` 系统用户下运行，凭据永远不会触及成员自己的 uid；在 macOS 上是尽力而为（以成员身份运行，并给出警告）。成员可以运行密钥**归自己所有**的工具；**共享**（队友/admin 的）密钥则需要隔离运行器**以及**服务器的 `TREG_RUN_PROOF`（安装时以 `--run-proof` 传入）。
- `--server` — 在注册中心**服务器**上运行（Tier 0）；只有工具目录已知的 CLI（或列入 `TREG_RUN_ALLOWED_BINS` 的）可以执行；stdout/stderr + 退出码会流式传回。当密钥绝对不能到达本机时使用这种方式。

```bash
treg run stripe -- get /v1/balance          # local: CLI runs here, key injected, never held
treg run gh -- pr list
treg run --server agentmail-cli inboxes list # server-side: key never leaves the registry
sudo treg setup-local-run --run-proof "$TREG_RUN_PROOF"   # Linux admin, once
treg runs --limit 20
```

## 捕获你自己的调用

`treg run` 覆盖厂商 **CLI**，`treg call` 覆盖你让 treg 代为发起的 HTTP 调用。但当程序自己向 `api.stripe.com` 发起请求时，两者都帮不上忙——treg 对它不可见，它也没有密钥。

### 常规方式：`treg <command>`

在任何命令前面加上 `treg`：

```bash
treg claude                  # a Claude Code session using the team's shared credentials
treg codex                   # same for Codex
treg node server.js          # your app, with the team's keys, without holding any
treg python train.py
treg with -- npm test        # the explicit form, for anything that confuses the parser
```

代理需要一个编译包（`cryptography`）来生成本机的证书颁发机构。`install.sh` 安装器已包含它；如果你是用普通 `pip` 安装的且缺少它，首次使用该功能时 treg 会主动提出按你的安装方式正确补装。

**这是按命令选择加入的，而这正是设计的重点。** treg 是父进程，因此该设置只作用于那条命令及其子进程。`treg claude` 使用团队的共享访问权限；直接运行 `claude` 完全不受影响，使用的是你自己本地的密钥。不会向任何配置文件写入内容，所以无需撤销任何东西，也不会有任何会话在你不知情的情况下被改变。

运行期间，发往**已注册**主机的 HTTPS 调用会经过注册中心，由它**在服务器上**添加凭据。所有其他地址——包括你的 agent 自己对 `api.anthropic.com` 或 `api.openai.com` 的调用——都直接发出，我们无法读取。

如果 `treg serve` 代理已在运行，就会复用它且不去动它；否则 treg 会在操作系统挑选的端口上启动一个私有代理（因此两个会话永远不会冲突），并在你的命令退出时将其停止。

**如果你写 Node，有一个注意事项。** Node 内置的 `fetch` 直到 **Node 24** 才会理会代理设置。在较旧的 Node 上，普通的 `fetch()` 不会被捕获，调用会在没有凭据的情况下直接发出。请改用会读取环境变量的客户端（`axios`、`got`、undici 的 `ProxyAgent`），或升级 Node。curl、git、Python 的 `requests`/`httpx` 以及 Deno 在任何版本下都可用。[`examples/proxy-demo/`](examples/proxy-demo/) 中有一个可运行的示例，同时展示了两种情况，并解释了旧版 Node 的变通办法。

### 子 shell：`treg shell --proxy`

`treg run` 覆盖厂商 **CLI**，`treg call` 覆盖你让 treg 代为发起的 HTTP 调用。但当 agent 自己写脚本直接与 `api.stripe.com` 通信时，两者都帮不上忙——treg 对它不可见，脚本也没有密钥。

`treg shell start --proxy` 弥补了这个空缺。在这个 shell 内，发往**已注册**主机的 HTTPS 调用会经由注册中心路由，由它**在服务器上**添加凭据，并原样返回厂商的应答。你的代码不需要密钥，也不需要任何 treg 专属的代码：

```bash
treg shell start --proxy                 # the banner lists the hosts being captured
curl https://api.stripe.com/v1/balance   # no key anywhere — treg injected it server-side
python my_script.py                      # same for anything the script calls
exit                                     # the proxy stops with the shell
```

工作原理：该 shell **只为这个 shell** 设置 `HTTPS_PROXY` 和一个信任包，使用在你机器上生成的证书颁发机构（私钥权限 `0600`，有效期两年，绝不共享）。**系统信任库永远不会被修改**——这个 shell 之外的任何东西都不信任它，你的浏览器不信任，操作系统也不信任。

它**不会**触及的：所有不是已注册工具的地址，包括你的 agent 自己对 `api.anthropic.com` 或 `api.openai.com` 的调用。这些调用只是被隧道转发而不会被读取，也永远不会为它们生成证书。

| 选项 | 作用 |
|---|---|
| `--proxy` | 开启该功能（默认关闭） |
| `--proxy-port N` | 监听 127.0.0.1:18791 以外的地址 |
| `--renew-ca` | 启动前重新生成这台机器的证书颁发机构 |

### 作为后台服务（`treg serve`）

如果你不想进入子 shell，而是想保留自己的 shell，可以把同一个代理作为服务运行：

```bash
treg serve start                  # starts in the background, prints the next line for you
eval "$(treg serve env)"          # point THIS terminal at it (repeat in any other terminal)
curl https://api.stripe.com/v1/balance
treg serve status                 # port, team, and the hosts being captured
eval "$(treg serve env --unset)"  # stop using it in this terminal
treg serve stop                   # stop the service
```

`treg serve start --foreground` 会保持前台附着而不是脱离，这正是使用服务管理器或查看其日志（`~/.treg/proxy/serve.log`）时所需要的方式。

有一个值得了解的区别。`treg shell --proxy` 把它的访问令牌保存在子 shell 的环境变量里，两者同生共灭。而服务必须能被其他终端找到，所以它会把端口和令牌写入 `~/.treg/proxy/proxy.json`（仅所有者可读，权限 `0600`）。该文件保存的是代理自己的令牌，绝不是厂商密钥——但它毕竟是磁盘上的一个文件，这是子 shell 方式所没有的。请据此权衡选择。

一些值得了解的限制：固定证书（certificate pinning）的客户端会拒绝这种拦截（这种情形请用 `treg run` 或 `treg call`）；每个被捕获的调用都要多经过注册中心一跳，因此注册中心宕机时它会失败；**托管式** MCP 服务器是在别人的机器上发起调用的，因此不在覆盖范围内。如果某个调用失败，treg 会用自己的话说明——例如 "no tool is registered for this host"（此主机未注册任何工具）、"ask an admin"（请联系管理员）——而不是让你面对一个裸 404，把它当成厂商的应答。

## skill（捆绑包）

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg skill scaffold DIR` | `--out FILE` | 扫描一个 skill 目录 → 生成清单草稿（自动发现配方 + 机密；由你填写 `base_url` 和绑定） |
| `treg skill push FILE` | — | 注册一份填写完成的清单（原子式地创建 bundle + 机密 + 工具） |
| `treg skill ls` | — | 列出 bundle |
| `treg skill rm ID` | — | 删除一个 bundle（级联删除其工具 + 机密） |

```bash
treg skill scaffold ~/.claude/skills/intercom --out intercom.json
#   edit intercom.json: set base_url + bindings
treg skill push intercom.json
```

## 批量上传（scan → upload）

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg scan [env\|skills]` | `--dir D`, `--env-file F`, `--skills-dir D` | 只读预览：列出上传将会注册的密钥和 skill（不会有任何数据离开本机） |
| `treg upload [env\|skills]` | `--dir D`, `--select a,b` \| `--all`, `--replace`, `--no-oauth`, `--llm …` | 一次性注册一个 `.env` 中的服务商密钥和/或一个 skills 文件夹 |

```bash
treg scan                        # what's here? (keys + skills, read-only)
treg upload --all                # register everything the scan found
treg upload env --select openai,stripe
treg upload skills --dir ~/.claude/skills --all
```

幂等——随时可以重复运行（跳过已注册的；`--replace` 会更新）。非交互（agent/CI）运行在缺少 `--all`/`--select` 时会拒绝执行。

## OAuth（连接流程）+ 健康检查

| 命令 | 选项 | 作用 |
|---|---|---|
| `treg oauth connect NAME` | `--client-secret PATH`, `--scopes S1 S2 …` | 浏览器授权 → treg 捕获首个令牌并将其存为 oauth 机密 |
| `treg connections connect --provider P` | `--key-stdin` | 基于密钥的服务商（Moz、Hunter 等）：以隐藏输入的方式提示输入密钥，或用 `--key-stdin` 从 stdin 读取；treg 会先向该服务商验证密钥，然后才存储 |
| `treg health` | `--run` | 显示每个凭据的状态；`--run` 立即重新检查（刷新 oauth、探测工具、提醒所有者） |

```bash
# one-time: add https://treg.to/oauth/callback to your OAuth app's redirect URIs
treg oauth connect gads --client-secret ./client_secret.json \
  --scopes https://www.googleapis.com/auth/adwords
treg health --run
```

---

## OAuth：两种模式（treg 负责保持新鲜度）

- **自动刷新** — 如果 oauth 机密带有 `refresh_token` + `client_id` + `client_secret`，treg 会在其过期前自动刷新（你无需重新上传）。`oauth connect` 流程总是落入这种模式。
- **手动** — 直接上传的令牌会按原样注入；过期后由你重新上传。

存储方式相同；凭据可以从手动升级为自动，无需任何迁移。`treg health` 会标记任何失效的凭据，并通过 webhook 通知所有者（如果注册时设置了 `webhook_url`）。

## 认证形态（每个绑定的 `injector`）

- `env` — 纯字符串（API 密钥）
- `secret_file` — JSON 令牌文件；从中提取 `secret_field`
- `oauth` — JSON OAuth 令牌；从中提取 `secret_field`（可刷新时自动刷新）
- `cli_auth` — 从 CLI 密钥链中提取的材料（像字符串一样放置）

## `treg hub` — 发布一个由工具组成的工具（受 `TREG_HUB_ENABLED` 控制）

hub 工具是你的团队发布给他人 agent 使用的工具：一段 JSON 步骤配方，或沙箱中的一个脚本。每个步骤都经由你自己的工具和密钥运行；调用方支付计量步骤的费用加上你设定的价格，而这份价格会作为额度进入你的余额。发布后立即可调用，id 为 `<team-slug>.<name>`；通过页面 `/hub/<id>` 分享。新工具在你上架（list）之前不会出现在搜索结果中。

```
treg hub init <name> [--script] [--dir D]   write the files: recipe.json, run.js (scripts), check.json, README.md
treg hub run <dir> --input k=v …            run the folder for real on your token; nothing stored; read the trace
treg hub publish <dir>                      validate, run check.json once on your balance, live on pass (a new version)
treg hub ls                                 your team's hub tools, every version, with health and 30-day numbers
treg hub earnings <id> [--days N] [--csv]   what one tool earned, per day (sales only; never who called)
treg hub price <id> <usd>                   change the price for later runs; no version bump; 0 = free
treg hub retire <id>                        every version off the call road; history and earnings stay readable
treg hub list <id> · treg hub unlist <id>   ask for a place in catalog search (treg approves), or take it out
treg hub log <id> --public on|off           show or hide the run log on the share page (default on)
```

**上架与公开运行日志。**两者都不会使版本号增加。`list` 是一个请求：一旦 treg 批准，该工具就会进入 `treg catalog search` 和 `catalog_search`，标记为 `hub` 并显示你的团队、价格标签和 30 天成功率，按相关性排名且没有加权。拒绝会附带原因（见 `treg hub ls` 与仪表盘的 Listing 标签页）；再次上架即再次申请。上架状态属于工具本身；`unlist` 会将其撤下。在 recipe.json 中加入 `"capability": "<a catalog capability id>"` 来指明你的工具承担的任务：获批后，在该任务下任何服务商的 `treg catalog get` 结果都会把你的工具显示在这些服务商旁边。新工具的成功率从 90% 起步（按 5 次运行计），其他团队的真实运行会逐渐替换这一估计值。工具一旦上架，每个新版本和每次 `treg hub price` 都要等待 treg 审核：publish 会返回 `review`，调用方继续使用已批准的版本和价格，而你可以自己用 `<id>@N` 调用新版本。被拒绝的更新会保留已批准版本，并告诉你原因。`unlist` 之后依然如此：treg 一旦批准过某个工具，它就一直处于审核监管之下，`list` 无需重新审核即可让它重回搜索。看起来像 treg、像官方或已认证、或像工具目录中某家服务商的团队名（包括形近字母）不能发布。被 treg 拒绝的工具只能由其制作者所在的团队调用。check.json 可以包含 `{"cases": [...]}`，最多 5 个样例，发布时全部运行。所有输出字段均为空的应答不支付卖方价格。未上架、未批准、失败和已退役的工具永远不会出现。分享页的运行日志显示最近 20 次运行和 30 天内每天的运行数：时间、结果、耗时、步骤、用量和支付的价格；绝不显示调用者是谁、输入或输出。仪表盘在 Hub → 某个工具 → Listing 下有同样的两个开关。

文件夹可以携带第五个文件 `data.csv`（≤ 50 MB）：脚本以 `ctx.data` 读取它。脚本可以获得 `ctx.inputs`、`ctx.call(target, {method, query, body, headers})`、`ctx.charge(usd, note)`、`ctx.csv(text)`、`ctx.data`、`ctx.log(text)`；`target` 是一个工具目录 id、你自己的某个工具（写作 `<tool>/<path>`），或该工具基础 URL 之下的完整 URL。绝不要把凭据粘贴进脚本：先注册它（`treg secret add`、`treg tool add`），在 `uses` 中列出该工具，并在 `ctx.call` 中指名使用。调用方用 `treg call <id> --data '{…}'` 运行它；用 `treg catalog get <id>` 查看其契约。

**定价。**只有你的价格：服务商费用（工具目录步骤）会额外向调用方计收，失败的运行不收取任何费用。

| 类型 | recipe.json | 每次成功运行的收益 |
|---|---|---|
| 步骤配方 | `"pricing": {"price_usd": 0.02}` | 固定价格 |
| 脚本 | `"pricing": {"max_price_usd": 0.05}` | run.js 中各 `ctx.charge(usd, note)` 行之和，绝不超过上限 |

脚本的计费行可以是一笔手续费（`ctx.charge(0.01, "fee")`）、按结果计费（`rows.length * 0.002`）、工具目录调用的加价（`r.cost_usd * 0.2`），或你自己厂商的成本。调用方在运行前就能看到上限。所有界面都会向调用方展示近期成功运行的花费（从低到高），服务商费用与你的价格合并计算。`treg hub price <id> <usd>` 为后续运行设定配方的价格，或脚本的上限。

## 匿名使用统计

使用 treg.to 时，CLI 会通过 PostHog 发送基本使用数据：命令名、成功/退出码、耗时、CLI 版本和操作系统，并关联到一个随机的本地安装 ID。它不会发送命令参数、凭据、请求/响应内容、邮箱或团队标识。帮助信息和参数解析错误不会被跟踪。统计功能在命令退出时最多增加 1 秒等待；发送缓慢或离线时的数据可能被丢弃。

用 `export TREG_TELEMETRY=0`（或 `DO_NOT_TRACK=1`）可将其禁用。自托管的注册中心默认不启用统计；设置 `TREG_CLI_POSTHOG_KEY` 和可选的 `TREG_CLI_POSTHOG_HOST` 即可使用你自己的 PostHog 项目。随机 ID 存储在 CLI 配置文件旁边的 `analytics-id` 中。
