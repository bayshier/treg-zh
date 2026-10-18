# Treg（面向工具的 OpenRouter）

> [!NOTE]
> **汉化说明**：本仓库是 [superdesigndev/treg](https://github.com/superdesigndev/treg) 的非官方中文汉化版，遵循上游的 Apache-2.0 许可证（见 [LICENSE](LICENSE)），与上游官方无关。已汉化面向用户的文档（README、USAGE、教程、插件文档）和全部 `skills/*/SKILL.md`；代码、CLI 输出、控制台界面以及 `docs/context/`、`docs/ideas/` 等内部开发文档保持英文。翻译基于上游当前版本，可能滞后于最新改动，功能与行为请以上游仓库为准。

![treg — 你的 agent 的工具目录](docs/assets/treg-hero.png)

**OpenRouter，只是面向 agent 工具而非模型。** 把 agent 指向一个 base URL、只用一个 token，
它就能完成任务：**一个横跨众多 provider、拥有数千端点的精选工具目录** — SEO 与外链、
社交与趋势、个人与企业信息增强、广告、抓取、图像与视频生成 —
**按次计费，低至一美分**，
无需在 provider 处注册。再加上你自己团队的 key、skill（技能）和 CLI，团队每位成员的
agent 都能调用，而凭据永远不会离开服务器。

**要的是任务，不是工具。** 你无需知道哪家厂商出售外链数据，也无需在他们那里持有账号。
搜索你想做的事，读一下价格，然后调用。

为 Superdesign 团队打造，线上服务见 [treg.to](https://treg.to) — 任何人都可以自托管。

## 它为何存在

agent 完成真实工作所需的工具，藏在没人会为单次运行而购买的订阅背后 —
Semrush $139/月、Moz $99/月、Crunchbase $99/月、Apollo $59/席位 — 藏在注册墙后，
或根本没有公开 API（仅限邀请、仅限合作伙伴、仅限应用审核）。treg 承载这些账号，
按每次调用几分之一美分计费。

## 两类工具，一个 token

- **工具目录** — treg 可以用自己的 key 提供服务的外部端点，或通过无需 provider key 的
  已验证公开路由提供服务的外部端点。自有 key 的调用使用团队的预付余额；匿名调用
  免费。无需在 provider 处开户。新验证账号在创建符合条件的团队时可一次性获得
  **$1.00 免费额度**。
- **你自己的工具** — 任何团队成员注册过的东西：付费 API 账号、OAuth 连接、
  厂商 CLI、一份 `SKILL.md`。**你自己的 key 永远优先于 treg 的 key，这些调用永远不会
  被计量。**

后半部分的词汇表：

- **tool（工具）** = 注册中心用组织的凭据替你调用的东西。分两类：
  - **endpoint（端点）** — 一个上游 `base_url` + 凭据**绑定**（每个绑定向请求注入一个
    secret；一个请求可以携带多个，例如 OAuth bearer *和* `developer-token` 头）。
  - **CLI** — 注入凭据后运行的厂商二进制程序（`stripe`、`gh`、`vercel` 等）。
- **skill / bundle** = 一份配方（`SKILL.md`）+ 其 secret + 其工具，一起注册。

**唯一的规则：** 代理只**转发、从不建模**上游，并且**在服务端注入认证**
— 因此它能扛住上游 API 的变化，调用方也永远不持有 key。

---

# 第 1 部分 · 使用注册中心

访问 [**treg.to**](https://treg.to)（托管于 Render）— 仪表盘、
登录以及下文的所有 URL 都在该域名下。

## 快速开始

与仪表盘的 **Getting started** 指南相同的流程：

```bash
# 1. install the CLI — also points it at the registry
curl -fsSL https://treg.to/install.sh | sh

# 2. sign in (GitHub default · --email for a one-time code · --token for agents/CI)
treg login

# 3. do something useful immediately — no key, nothing registered
treg catalog search "backlinks for a domain"     # find a tool by what it DOES
treg call tikhub.tiktok.user.profile --query uniqueId=tiktok
treg balance                                     # exactly what that cost

# (or `treg onboard` for the guided walkthrough)
```

Fish Audio 提供 S2.1 Pro 语音合成、公开声音发现和私有声音克隆。
语音以二进制 stdout 输出，因此请重定向到文件。已发现声音的 `_id` 或团队声音 id 即
TTS 的 `reference_id`；在 treg 的 Fish 账号上创建的声音是持久的团队资源：

```bash
treg call fishaudio.tts.s2-1-pro --method POST --header model=s2.1-pro \
  --data '{"text":"Hello from treg","format":"mp3"}' > speech.mp3
treg call fishaudio.voices.discover --query self=false --query licensed=false --query language=en
treg resources list --provider fishaudio --kind voice
```

使用你自己的 Fish key 时，请求仍是不受限、不计量的上游转发，账号边界由 Fish 掌控。

工具目录中工具的输入由 `treg catalog get <id>` 描述。标记为 `strict_query` 的工具会拒绝未声明或重复的查询参数、不支持的值以及请求体。

你的 token 在每次调用中标识你的身份（`X-Treg-Token` 头），且对所有工具通用。
看看团队共享了什么：`treg tool ls` · 检查凭据健康状况：`treg health`。

### 或者作为 Claude Code 插件安装

```
/plugin marketplace add superdesigndev/treg
/plugin install treg@treg
```

安装时无需 token，也无需配置。该 skill 以 `treg:treg` 加载，首次运行时，
它会引导你的 agent 完成其余步骤 — CLI、登录，然后 `treg mcp install` — 最终你会同时得到
命令行**和** treg 的工具。其他 agent：`npx skills add superdesigndev/treg -s treg`
（或参见 [Workflow skills](#workflow-skills) 了解其余方式）。
参见 [docs/CLAUDE-PLUGIN.md](docs/CLAUDE-PLUGIN.md)。MiniMax Code / MiniMax Agent 用户：同一
skill 也通过 MiniMax 插件市场分发（[docs/MINIMAX-PLUGIN.md](docs/MINIMAX-PLUGIN.md)）。

### Workflow skills

开箱即用的配方，你的 agent 可以端到端执行，每次调用都经过 treg：

| Skill | 你的 agent 能做什么 |
|---|---|
| [`treg`](skills/treg/SKILL.md) | 查找并调用工具目录中的任何工具，以及你自己的工具 |
| [`web-access`](skills/web-access/SKILL.md) | 无需本地爬虫即可读取实时网络：先用免费搜索和页面读取，再为社交、商店、评论、地图和论文安排合适的平台任务；某一家失败时，treg 会改用下一家厂商 |
| [`lead-signals`](skills/lead-signals/SKILL.md) | 从公开信号（投诉、职位变动、招聘、融资、工具采用）中发现有购买意向的买家，并据此监控某个话题、竞争对手或账号列表（[treg.to/leads-signals](https://treg.to/leads-signals)） |
| [`make-ugc`](skills/make-ugc/SKILL.md) | 制作 AI UGC 视频：热门开场钩子、出镜者、口播片段、字幕（[treg.to/ugc](https://treg.to/ugc)） |
| [`jev-memory`](skills/jev-memory/SKILL.md) | 给 Claude Code 一个记忆：一个 Claude Code mod，由 Jev 在每轮结束后评判每个 prompt 并保留你的持久偏好，可安装现成的，也可用一句 prompt 构建（[该 mod](examples/claude-code-mods/jev-memory)） |

`install.sh` 会为你安装除 `jev-memory` 之外的所有 skill，后者需要主动选择安装。若要将其添加到已有 agent
（使用 [skills.sh](https://skills.sh) CLI）：

```bash
npx skills add superdesigndev/treg                        # every public skill
npx skills add superdesigndev/treg --skill lead-signals   # just one
treg skill bootstrap                                      # same set, with the treg CLI you already have
```

工作流 skill 运行在 treg 上，所以请先完成设置（见上面的快速开始，或把你的 agent 指向
[treg.to/llms.txt](https://treg.to/llms.txt)）。

### Claude.ai 连接器

Claude Connectors Directory 的入口是 `https://treg.to/mcp/v2/`。它只暴露精选的
工具目录端点，不含图像、视频或音频生成，并将读调用与写调用分开，让 Claude 收到准确的安全信号。现有 `/mcp/` 入口仍然可用于工具目录端点、团队自有工具
和导入的 skill。边界与实现参见 [MCP 与 OAuth 架构](docs/context/architecture/mcp-oauth.md)，
发布门槛参见[提交手册](https://github.com/superdesigndev/treg-internal/blob/main/docs/distribution/CLAUDE-CONNECTOR-SUBMISSION.md)。

## 调用你没有 key 的工具

工具目录按端点**能做什么**分组：关键词与排名跟踪、外链与权威度、
AI 可见度、趋势与发现、发布到社交平台、个人与企业信息增强、广告
管理与创意、效果衡量。

```bash
treg catalog                                    # every platform, busiest first
treg catalog search "find a work email"         # by the job, not the vendor
treg catalog search --new                       # tools added in the last 30 days, newest first
treg catalog get hunter.people.email.find       # params, PRICE, example response
treg call hunter.people.email.find --query domain=reddit.com --query full_name="Alexis Ohanian"
```

**工具目录中的调用是如何被服务的** — 凭据阶梯，按顺序：

1. 你的团队为该 provider 注册了自己的工具 → 用那个工具、那把 key；
2. 你的团队为该 provider 存了 secret → 通过虚拟工具注入；
3. 两者皆无，且该端点有已验证的公开路由 → **无需 provider key**，免费；
4. 否则 → **treg 自己的 key**，计入团队的预付余额。

匿名价格假定调用方不发送 provider 凭据头。忠实转发会保留调用方的头，因此调用方自带的
provider key 可以使用该 key 自己的额度。你自己的凭据永远优先于 treg 的，所以接入一把
你已经在付费的 key，可以让这些调用不消耗余额，而不是重复计一份。treg 没有公布价格的
端点会被**拒绝**，而不是免费提供 — 系统会提示你改接自己的 key。当多个 provider
服务同一能力时，`treg catalog search` 会把它们连同价格并排展示；**选择权在你** —
treg 不会替你在 provider 之间默默挑选或故障转移。（当 treg 在某 provider 处的自有
账号用尽时，可能改用 treg 持有的中继账号来服务*同一个*端点，并会在响应中说明；
团队可以选择退出。）你可以主动选择的例外：`treg.<capability>` 路由端点，由 treg 替你
挑选 provider 并指明是谁。对于常见任务（抓取网页、web 搜索、Google 结果、查找邮箱），
请从路由端点开始，例如 `treg.web.extract` 或 `treg.web.search`，而不是某个你记得名字的
provider。在 `treg.people.email.find` 和 `treg.people.phone.find` 上，`X-Treg-Route-Verify: true`
还会在同一次调用中校验找到的地址或号码，作为单独计费的一项，并告知校验结果。

```bash
treg balance          # credit left, calls in flight, recent spend
treg topup            # add funds, or set up automatic top-ups
```

余额不足会返回 HTTP **402**，携带 `balance_micro`、`estimated_cost_micro` 和 `topup_url`，
agent 无需阅读文字说明即可据此行动。

**Enrich Arena** 位于 `/enrich-arena`，在仪表盘之外。比较各家厂商信息增强答案的成本与速度，
一键为最佳答案投票，或观看顺序瀑布。能力兼容的
异步 provider 也会参与；Arena 在内部处理提交与轮询。浏览是
公开的；提交需要登录，计费的尝试会消耗团队额度。参见
[Arena 指南](docs/context/interface/enrich-arena.md)。

**Web Arena** 在 `/web-arena` 比较 Web Search、Web Fetch 和 Sitemap provider。
Battle 用相同输入调用选定的 provider；Waterfall 按顺序逐一尝试。
实时排行榜将 provider 直调的命中率和响应时间，与经过核验的
Web Arena 质量结果合并在 30 天滚动窗口内。功能启用后这些页面
才会开放。参见
[Web Arena 指南](docs/context/interface/web-arena.md)。

## 共享与使用你自己的工具

零思考路径 — 把 treg 指向一个项目，它会自己找出可共享的内容：

```bash
treg scan     # read-only preview: the keys, skills & CLIs upload would register
treg upload   # register them (encrypted server-side); idempotent, --replace to update
```

`treg upload` 会扫描 `.env`（将 key 与约 80 个已知 provider 进行匹配）、每个 skill
子目录以及已安装的工具目录 CLI。有三类东西会进入注册中心 — 下面说明
各自如何共享和使用：

### 1. 端点（HTTP API）

**共享** — 一个可用已存 key 调用的上游 URL，或从 `.env` 批量导入：

```bash
treg secret add STRIPE_KEY --value sk_live_123
treg add stripe --base-url https://api.stripe.com --secret STRIPE_KEY

treg upload env --select openai,stripe,resend     # or straight from the .env
```

**使用** — agent 原生方式：构造**真实的**上游请求，并在前面加上代理前缀。
treg 按主机名解析工具，注入凭据，然后忠实转发其余一切
（你的 `X-Treg-Token` 会在上游看到之前被剥除）：

```
Real request:   GET https://api.intercom.io/conversations?per_page=5
Through treg:   GET https://treg.to/call/https://api.intercom.io/conversations?per_page=5
                    header:  X-Treg-Token: <your token>
```

或使用 CLI 简写 — 审计日志见 `treg calls`：

```bash
treg call intercom conversations --query per_page=5
treg call stripe v1/balance
```

### 2. CLI

**共享** — 自动的：`treg upload` 会检测已安装的工具目录 CLI（`stripe`、`gh`、`vercel` 等）
并注册它们；仅有配方（recipe）的工具目录 CLI skill（例如 `stripe-cli`）也会自动变为可运行。

**使用** — `treg run` 在**注入组织凭据**的情况下执行厂商 CLI，因此你永远
不需要持有 key 或登录：

```bash
treg run stripe -- get /v1/balance
treg run gh -- pr list
treg run --server agentmail-cli inboxes list   # runs on the registry server: the key never reaches you
```

`--local`（默认）在你的机器上运行；`--server` 在注册中心上运行并把输出流式传回。
对于整个会话，`treg shell start` 会打开一个子 shell，其中每个已注册的 CLI 都自动注入
— 正常使用 `stripe`、`gh` 等即可；`exit` 退出恢复。`treg runs` 是审计日志。

### 3. Skills

**共享** — 一个 skill 是一整项能力（`SKILL.md` 配方 + 其 secret + 其工具），
一起注册，让整个团队运行同一个 skill，并在一处维护：

```bash
treg upload skills --dir ~/.claude/skills --all   # register a folder of skills in one pass
```

**使用** — 把任何共享的 skill 拉取到你的 agent 中；它的 API 调用带着你的
token 经过 treg，因此 key 留在服务器上，永远不会进入 skill 本身：

```bash
treg skill install seo-blog-writer      # writes into ./.claude/skills/  (--all for the library)
```

### 手动注册 — 当启发式规则认不出某个工具时

```bash
# multi-credential tool (e.g. google-ads: OAuth bearer + a developer-token header)
treg tool add google-ads --base-url https://googleads.googleapis.com \
  --bind "secret=<oauth-id>,injector=oauth" \
  --bind "secret=<dev-id>,name=developer-token,format={secret}"

# one skill, step by step
treg skill init --dir ./my-skill          # drafts treg.json (guesses base_url, finds secrets)
treg skill add  --dir ./my-skill          # registers recipe + secrets + tool, atomically

# OAuth via the browser (mints the first token, treg holds it and auto-refreshes)
treg oauth connect gsc --client-secret client_secret.json \
  --scopes https://www.googleapis.com/auth/webmasters.readonly
```

每个命令的完整选项：[`USAGE.md`](USAGE.md)。

使用 treg.to 时，CLI 会向 PostHog 发送匿名的命令使用数据（不含参数或凭据）。
可用 `TREG_TELEMETRY=0` 或 `DO_NOT_TRACK=1` 禁用。
详见[分析详情](USAGE.md#anonymous-usage-analytics)。

## 团队

一个账号最多可拥有 10 个团队。以成员身份加入其他团队不计入该上限。

一切都以**组织**为范围：一个 token = 一份 `(user, org)` 成员关系，每个 secret、工具
和 skill 都属于当前激活的组织。角色：**owner / admin / member / viewer**。

```bash
treg org create "Acme"                        # make a team, become owner
treg org invite teammate@acme.com             # invite by email (pick role + tool access)
treg org join <code> --email you@acme.com     # accept an invite (creates you if new)
treg org ls | use <slug> | members            # switch orgs, see the roster
treg org access <member> --tools a,b          # per-member tool access (admin+)
```

## 深入了解

- **反馈：** `treg feedback submit friction "The pagination example is unclear."`
  在不包含隐私信息的前提下分享问题或建议。参见[反馈说明](https://treg.to/feedback.md)。
- **评价：** `treg review CALL_ID useful`
  在使用其结果后为受邀的工具目录调用评分；`not_sure` 也可以。略去隐私数据并继续任务。

- [`USAGE.md`](USAGE.md) — 完整的 `treg` CLI 参考。
- [`/llms.txt`](https://treg.to/llms.txt) — agent 接入文件：调用
  协议、发现、认证、CLI、skill。一次抓取就能让 agent 学会整个注册中心。
- **仪表盘**位于 [treg.to](https://treg.to) — 完整的 CRUD、引导式
  教程（Help → Tutorial），以及可复制的 agent 配置说明。
- **API** — CLI 做的一切都是纯 HTTP；交互式 OpenAPI 文档在 `/docs`。
  代理端点是 `/call/{...}`；所有端点都接受 `X-Treg-Token` 头。

---

# 第 2 部分 · 自托管与开发

## 本地运行

一条命令（需要 `tmux` + [`uv`](https://docs.astral.sh/uv/)；它会自行同步 venv）：

```bash
scripts/dev-local.sh up        # server on http://localhost:18790, dev-safe settings
```

它会在 tmux 中以热重载运行服务器，使用自己的 sqlite 数据库（`treg-dev.db`）和邮件 OTP 开发
模式（登录验证码直接显示在页面上 — 无需邮件发送服务）。日常操作：

```bash
scripts/dev-local.sh cli login   # sandboxed CLI: never touches your real ~/.treg/config.json
scripts/dev-local.sh logs        # server output          · status / restart / down
scripts/dev-local.sh reset       # wipe the dev DB + CLI sandbox for a fresh start
```

或者不通过 tmux，直接运行服务器：

```bash
bash scripts/build-dashboard.sh # Node 22.12+ and npm; build the Dashboard
uv sync                        # create the venv from uv.lock (pulls the server deps for dev)
uv run python -m treg upgrade  # prepare schema + run idempotent release tasks without serving
uv run python -m treg          # serve on 0.0.0.0:18790 (add --reload for dev)
uv run python -m treg keygen   # print a fresh Fernet key for TREG_SECRET_KEY
```

> **安装后要运行服务器（而非从源码）：** 基础包**只含 CLI**。要运行一个
> 注册中心，请安装 server extra — `pip install "tools-registry[server]"` — 它会加上 FastAPI、
> 数据库驱动和加密。只执行 `pip install tools-registry` 则只有用于连接现有注册中心的
> `treg` 命令。

官方托管服务见 `treg.to`。其生产拓扑和线上设置
维护在私有的[运维手册](https://github.com/superdesigndev/treg-internal/blob/main/docs/production/deploy.md)中。

## 配置

环境变量（前缀 `TREG_`，从 `.env` 读取）：


| 变量                                       | 默认值                          | 用途                                                                                                                                                                    |
| ----------------------------------------- | ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TREG_DATABASE_URL`                       | `sqlite+aiosqlite:///./treg.db` | 数据库 URL（开发用 SQLite，生产用 Postgres）                                                                                                                                  |
| `TREG_READ_DATABASE_URL`                  | *(空)*                       | 可选的 SQLite / PostgreSQL 只读数据源；为空则复用主库。需要调用方主动启用；现有查询不受影响。参见[只读数据源配置](docs/context/ops/deploy.md#optional-read-replica)。 |
| `TREG_SECRET_KEY`                         | *(空)*                       | 用于静态加密 secret 的 Fernet key；为空 → 会生成一把临时 key（重启后 secret 将无法恢复）                                                                       |
| `TREG_PUBLIC_URL`                         | `https://treg.to`  | treg 的公开基址，用于构造 OAuth 回调 URI                                                                                                                   |
| `TREG_SESSION_SECRET`                     | *(空)*                       | 为仪表盘会话 cookie 签名；缺省回退到 `TREG_SECRET_KEY`。生产环境请设置真实值                                                                              |
| `TREG_GITHUB_CLIENT_ID` / `_SECRET`       | *(空)*                       | GitHub OAuth 登录（回调 `<public_url>/auth/github/callback`）；为空则隐藏按钮                                                                                |
| `TREG_GOOGLE_CLIENT_ID` / `_SECRET`       | *(空)*                       | Google OAuth 登录（重定向 `<public_url>/auth/google/callback`）；为空则隐藏按钮                                                                                |
| `TREG_INSTAGRAM_CLIENT_ID` / `_SECRET`    | *(空)*                       | 用于 Instagram Login 直连的 Instagram App ID 和 secret（重定向 `<public_url>/oauth/callback`）                                                                           |
| `TREG_META_CLIENT_ID` / `_SECRET`         | *(空)*                       | 用于 Facebook Pages、Meta Ads 及可选 Instagram `page-tools` 的 Meta 应用凭据                                                                                     |
| `TREG_OAUTH_REVIEW_PENDING`               | `instagram-login,page-messages` | 以逗号分隔的注册中心审核 key，其能力必须保持受限；托管服务的审核状态另行私有维护。                                                   |
| `TREG_RESEND_API_KEY` / `TREG_EMAIL_FROM` | *(空)*                       | 通过 Resend 发送事务邮件（OTP 验证码 + 邀请）；发件人必须是经 Resend 验证的发送者                                                                                |
| `TREG_BLOCKED_EMAIL_DOMAINS`              | *(空)*                       | 以逗号分隔的邮箱域名，在所有注册/登录入口和团队创建处均被拒收（含子域名，不区分大小写）。为空则不拦截任何域名 — 代码中不附带任何名单 |
| `TREG_CLIENT_IP_HEADER`                   | *(空)*                       | 一个只有你的边缘层才能设置的头，用作每个按 IP 限制的客户端地址（Cloudflare 之后为 `cf-connecting-ip`） |
| `TREG_TRUSTED_PROXY_HOPS`                 | `1`                             | 没有该头时：取 `X-Forwarded-For` 中从右数第这几跳的条目（即离你最近的代理所追加的那个）；客户端直连时为 `0`。第一个条目由客户端写入，永远不被信任 |
| `TREG_REQUIRED_EXTENSIONS`                | *(空)*                       | 必须安装的以逗号分隔的策略扩展（`treg.extensions` 入口点），否则启动失败。treg.to 就以这种方式运行一个私有的反滥用策略扩展；它只能做限制（[策略扩展](docs/context/architecture/policy-extensions.md)） |
| `TREG_TURNSTILE_SITE_KEY` / `_SECRET`     | *(空)*                       | 每个邮箱登录表单上的 Cloudflare Turnstile；任一为空即关闭。site key 经 `/captcha.js` 提供，secret 只会送达 siteverify |
| `TREG_TURNSTILE_MODE`                     | `observe`                       | `observe` 记录每次结果但不做任何改变；`enforce` 在 token 缺失或无效时拒绝发放验证码，并在 Cloudflare 无法应答时不发放注册赠金 |
| `TREG_TURNSTILE_HOSTNAMES`                | *(空)*                       | 以逗号分隔的主机名，token 必须在这些主机上完成人机验证；为空 = `TREG_PUBLIC_URL` 的主机（及其已知别名） |
| `TREG_ADMIN_TOKEN`                        | *(空)*                       | 跨租户的**超级管理员** bearer；授权所有 `/admin/*` 端点。为空则禁用环境变量途径（仅 `is_superadmin` 用户可访问 `/admin`）。请设置得足够长并妥善保密。 |
| `TREG_EMAIL_DEV_MODE`                     | `false`                         | 为 true 时，`/auth/email/start` 会在响应中直接返回 OTP（无需邮件发送服务）— **仅限开发/本地**，切勿用于生产。                                                |
| `TREG_KV_URL`                             | *(空)*                       | 共享键值存储（Redis 协议），用于所有 worker 必须达成一致的计数器，目前是每团队的审核邀请预算。为空 = 进程内回退，单 worker 场景足够 |


本地开发不需要 `.env` — 每个设置都有可用的默认值（临时 key、sqlite）。

> **⚠️ 迁移或重新部署前请备份：** Fernet key（`TREG_SECRET_KEY`）和
> 数据库（生产用 Postgres；本地 sqlite 运行为 `treg.db`）。丢失 Fernet key 后，所有
> 已存 secret 都将无法恢复。

## 架构

**`/call` 的请求流程：** 解析工具（按 URL 主机 + 最长 `base_url` 前缀，或按名称）→
解密其 secret → 应用每个绑定的 injector → 流式转发到上游 → 以 fire-and-forget 方式
写入审计记录。基础设施转发层只流式传输字节，不含业务逻辑。调用应用
会缓冲需要结算或所有权凭证的响应，上限 8 MiB；更大的响应返回 502
且不计费，而不是返回被截断的成功。内联媒体的端点（Gemini 图像）
声明 `spooled_response`：其计量答案改为读入临时文件，依据其上报的
用量结算，然后整体转发。无需响应体凭证的已授权免费最终下载会完整流式传输，
自有 key 和自有工具的响应也是如此。

**模块地图**（`src/treg/`）：


| 模块                                       | 职责                                                                                                             |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `proxy.py`                                 | `relay()` — 整个产品浓缩在一个函数里：一个忠实的流式代理                                        |
| `injectors.py`                             | 认证形态的接缝：`env`、`cli_auth`、`secret_file`、`oauth` 把一个 secret 放进头/查询参数                |
| `oauth.py`                                 | token 新鲜度（single-flight 刷新）+ 连接流程（授权同意页 URL、授权码交换）                          |
| `health.py`                                | 凭据健康：刷新 OAuth、探测工具、通过 webhook 通知任何失效项的属主                              |
| `convert.py`                               | 把 skill 目录脚手架化为可注册的 bundle 清单                                                   |
| `api.py`                                   | API — 唯一的大脑；CLI 和 skill 只是它之上的瘦客户端                                                   |
| `cli.py`                                   | `treg` CLI                                                                                                   |
| `models.py`                                | SQLModel 表：`Org`、`User`、`Membership`、`Invite`、`Secret`、`Tool`、`Bundle`、`PendingOAuth`、`CallRecord` |
| `crypto.py` `config.py` `db.py` `audit.py` | Fernet 加密 + token · 配置 · 异步数据库 · 延迟写入的审计器                                         |


**4 种认证形态**（每个绑定的 `injector`）：`env`（纯字符串 / API key）· `secret_file`（一份
JSON token 文件，取其中一个字段）· `oauth`（JSON OAuth token，可刷新时自动刷新）·
`cli_auth`（从 CLI 钥匙串提取的凭据材料）。

**忠实转发契约：** 代理只改动**三样**东西，其余一切原样保留：

1. 逐跳传输头（每一跳重新推导），
2. treg 自己的控制头与边缘转发头（`x-treg-token`、`x-treg-org`、
`ngrok-skip-browser-warning`、`x-forwarded-*`、`via` 等）以及 treg 的会话 cookie — 全部剥除，
绝不泄漏到上游，
3. 注入的凭据。

**OAuth，获取第一个 token 的三种方式：** *手动上传*（放入一份 `token.json`）·
*自动刷新*（若 token 带 `refresh_token` + 客户端凭据，treg 会保持其新鲜，你无需
重新上传）· *托管连接流程*（`treg oauth connect` → 浏览器授权同意 → treg 自行捕获
token）。

**健康检查：** 可以为工具配置可选的探测（`{method, path, expect_status}`）；周期性运行
（按需或经 cron）会校验每个凭据、刷新 OAuth，并通过 webhook 通知失效项的属主。

深入设计见 [`docs/context/`](docs/context/README.md)（按子系统拆分的片段）。

## 测试

```bash
uv run --with pytest-xdist pytest -n auto -q   # daily local default (same shape as CI)
uv run --frozen python -m pytest -q            # serial: debugging one test, or order
```

覆盖范围：代理 walking-skeleton、所有 injector 形态、按用户的认证 + CRUD + 审计、skill 组装器、
URL 透传 + 忠实转发、OAuth 刷新 + 连接流程、健康检查、`treg run`/shell、
upload/scan、组织 + 邀请、仪表盘 API、CLI。

## 贡献与文档

```
treg/
├── src/treg/            # the package (api, cli, proxy, injectors, oauth, health, convert, models, …)
│   └── web/             # dashboard, landing, tutorial, llms.txt, skill.md, install.sh
├── tests/               # pytest suite (CI + local default: pytest-xdist -n auto)
├── docs/
│   ├── context/         # design fragments (codemap system) + generated index
│   └── ONBOARDING.md    # first-time bootstrap
├── USAGE.md             # full treg CLI reference
└── pyproject.toml
```

各子系统的设计文档是 `docs/context/` 中的**片段**，每个片段都标注其对应的 `src/treg/*`
源码。在这个仓库里和 AI agent 一起工作？`/tools-registry-context` skill 会为你正在
改动的部分加载正确的片段，并保持文档同步 — 推送前先运行
`/tools-registry-context sync`。

**路线图：** MCP 支持 · 更细的权限层级 · 静态密钥管理加固 · 可能的
Loopni 合并。

## 许可证

Apache 2.0 附加条款（[`LICENSE`](LICENSE)）：可自由使用 — 包括商业用途、
在你自己的组织内部（鼓励自托管你自己的注册中心）。限制是：未经书面
许可（`jason@superdesign.dev`），不得将代码作为竞争性的托管/受管注册中心服务再分发给
第三方。**在自己产品中使用托管版 treg.to API** —
通过 `X-Treg-Meta` 和 `usage/by-tag` 做透传计费 — 无需许可即可进行；
那是在调用我们的 API，而不是在再分发我们的软件。

### 固定的客户读取范围

对于受限的客户 agent，`treg org agent-new bot --pin customer=cust_A` 会强制归属，
并把调用/运行历史、归档结果以及共享 provider 异步所有权的范围限定到该 pin。
外来或未归属的 id 返回 404；未设置 pin 的操作者保留组织范围的视野和共享
余额。BYOK 账号访问和公开媒体 URL 保留其现有权限。多个 pin、迁移与重放行为
参见[多租户契约](docs/context/architecture/multi-tenancy.md#caller-tags-and-pinned-read-scopes)。
