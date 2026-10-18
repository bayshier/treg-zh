# Claude Code 插件，以及它在哪里上架

treg 的 Codex/ChatGPT 插件围绕 **MCP 连接器**构建，而那一条上架路径还卡在审核队列里。skill 并不需要 MCP：`src/treg/web/skill.md` 已经教会 agent 安装 CLI、登录并调用工具目录。因此它同时也以**仅含 skill 的 Claude Code 插件**形式发布，由本仓库作为自有 marketplace 提供——合入即上线，全程没有审核者介入。

## 安装

```
/plugin marketplace add superdesigndev/treg
/plugin install treg@treg
```

该 skill 以 `treg:treg` 加载，安装本身**不需要 token，也不需要任何配置**——这正是发布仅含 skill 的 manifest 的意义所在。

### 仅含 skill 的 manifest，首次运行时补全设置

这是两个不同的问题；把它们混为一谈，正是这里最容易犯的错误。

**manifest** 不声明任何 MCP server，因为 manifest 里声明的东西会在安装时注册，而那时还没有任何人登录——结果是五个常驻工具在每次调用时都返回 401。把 MCP server 排除在外，正是 `/plugin install` 得以零配置的原因。

**skill** 则在首次使用时补全设置，此时有人*在场*可以登录。它的 bootstrap 按固定顺序走三步：

```bash
curl -fsSL https://treg.to/install.sh | sh   # 1. the CLI (skipped if treg --version works)
treg login                                   # 2. sign in — the token
treg mcp install                             # 3. the tools, header-authed with that token
```

所以预期的最终状态是 CLI **加** skill **加** MCP 工具——由 skill 达成，而不是由 manifest 达成。

**这个顺序至关重要。** `cmd_mcp_install`（`src/treg/cli.py`）从配置读取 token，没有 token 时它会在*写入任何内容之前*就 `sys.exit`——遇到 401 时同样如此，并提示「什么都没写入」。若把第 3 步跑在第 2 步之前，它就是一个空操作，安静得足以让 agent 深信工具已经存在并继续往下走。`test_the_claude_bootstrap_sets_up_cli_login_and_mcp_in_order` 固定了这个顺序，也固定了重启说明：`claude mcp add` 要等 agent 重启后才生效。

有两个值得记住的后果：

- **`install.sh` 总是会运行 `treg skill bootstrap`**——没有任何开关能关掉它——因此第 1 步会把同一个 skill 的*第二份*副本放进 `~/.claude/skills/treg/`。无害，但与插件重复。bootstrap 会告诉 agent 向人类提及这件事，而不是默默删掉它；删除 `$HOME` 下的文件不是 skill 该做的决定。
- **`install.sh` 的 token 分支会一次性完成全部三步**（`… | sh -s -- --token <key>`）。对 CI 或拿到团队 token 的机器来说，这是正确的路径——但对插件用户不适用，他们在第 2 步之前还没有 token。

## 文件分布

```
.claude-plugin/plugin.json        the manifest Claude Code reads
.claude-plugin/marketplace.json   makes this repo its own single-plugin marketplace
skills/treg/SKILL.md              GENERATED — never edit by hand
skills.sh.json                    category grouping for skills.sh / Hermes' Skills Hub
```

`marketplace.json` 里的 `source: "./"` 让**插件根目录就是仓库根目录**，这正是 `skills/treg/` 位于顶层而非 `plugin/` 之下的原因。这同一个路径也是 `npx skills add` 所解析、`clawhub skill publish` 所接受的输入——一份生成产物，四个发布渠道。

一个值得记住的后果：Claude Code 在仓库根目录自动发现的任何东西都会发布给用户。只有 `skills/` 是有意提供的；根目录下的 `commands/`、`agents/` 或 `hooks/` 目录也会一并发布。`tests/test_plugin.py` 断言这些目录一个都不出现。

**绝不要直接编辑 `skills/treg/SKILL.md`。**请修改 `src/treg/web/skill.md` 然后重新生成：

```bash
uv run python scripts/build_plugin.py            # regenerate BOTH plugins
uv run python scripts/build_plugin.py --check    # fail if either is stale (also a test)
```

## 上架操作手册

排序原则：所有自助渠道都先于需要排队的渠道上线。

### 1. 自有 marketplace——合入即上线

无需提交任何东西。合入 `main` 即完成发布；上面的安装命令随之生效。

把 `pyproject.toml` 和两份 manifest 里的 `version` 一起递增——`test_plugin_version_tracks_the_package` 把它们全部固定为同一个值。

### 2. ClawHub（clawhub.ai）——✅ 已上线，自助，基于 CLI

**已于 2026-08-14 发布：[clawhub.ai/superdesigndev](https://clawhub.ai/superdesigndev) · `treg@0.11.0` · 所有者 `@superdesigndev` · MIT-0。**验证方式是把它从注册中心装回一个临时目录——交付的 `SKILL.md` 与本仓库的 `skills/treg/SKILL.md` 逐字节一致。

发布者组织是用 `clawhub publisher create superdesigndev --display-name "Superdesign dev, Inc."` 创建的。用个人账号发布也完全可行——之后可以用 `--owner <handle> --migrate-owner` 迁移 skill——但公司账号看起来更像正式的产品上架，而不是 side project。

`https://clawhub.ai/submit` **返回 404**（Hermes 的 CLI 仍指向那里；它的 `--to clawhub` 只是个空壳）。实际可用的路径是 npm CLI：

```bash
npm i -g clawhub
clawhub login                     # device flow; opens a browser (--no-browser prints a URL + code)
clawhub whoami                    # confirm before publishing

clawhub skill publish "$PWD/skills/treg" \
  --slug treg --name "treg" --version 0.11.0 \
  --changelog "..." \
  --topics "api,seo,serp,backlinks,enrichment,scraping,agent-tools,web-search" \
  --source-repo superdesigndev/treg \
  --source-commit "$(git rev-parse HEAD)" \
  --source-ref main --source-path skills/treg \
  --dry-run                       # drop --dry-run to publish for real

# Publishing is TWO PHASES. The command above only SUBMITS — the version sits at
# `pending.publication`, invisible on every public surface, until a scan is attached to it:
clawhub scan --slug treg --version 0.11.0 --update    # took ~37 minutes; let it run
```

**不要跳过第二条命令，也不要在它运行期间断定发布失败了。**这是 ClawHub 最不显眼的一点，而且他们的文档里没写。`clawhub skill publish` 报告 `"status": "pending-publication"` 后即退出；之后不会有任何自动进展。`scan --update`（「把已发布的扫描结果写回所选版本」）不是一个便利功能——它正是完成发布的那一步。可以观察审核记录的变化：

```bash
curl -s -H "Authorization: Bearer $(clawhub token)" \
  https://clawhub.ai/api/v1/skills/treg/moderation
```

| | `scan --update` 之前 | 之后 |
|---|---|---|
| `legacyReason` | `pending.publication` | `scanner.llm.review` |
| `reasonCodes` | `[]` | `["review.llm_review"]` |
| 公开的 `GET /api/v1/skills/treg` | 403，不可见 | 返回该 skill |

注意 `package publish` 有一个能轮询到最终状态的 `--wait` 标志；**`skill publish` 没有**，而且对 skill 的发布尝试，`/api/v1/publish/attempts/<attemptId>` 会返回 404。所以对 skill 来说完全没有任何进度指示——运行 `scan --update`，然后等它跑完。

以上针对 clawhub CLI **v0.23.3** 验证——已发布的文档不如实际的 `--help` 完整，所以请直接阅读 CLI：

- **必须传绝对路径。**相对路径 `./skills/treg` 会失败并报 `Path must be a folder`，因为路径是相对 `--workdir` + `--dir`（默认值为 `skills`）解析的，于是它会去找 `skills/skills/treg`。`"$PWD/skills/treg"` 才是可靠的形式。
- **`--source-*` 系列标志值得填写。**它们把这条上架记录关联回一个 GitHub 仓库、commit 和路径。在一个因其自身审核历史而让 Hermes 把每个 ClawHub skill 都标为 `community` 信任级的注册中心上，这是唯一可用的来源信号。
- **`--categories` 接受的 slug 无法从公开 API 枚举**（`/api/v1/categories` 返回 404，上架记录返回的 `categories` 也是空的）。不要凭空编造一个——在能从真实上架记录里读到实际分类 slug 之前，先用自由格式的 `--topics`。
- `--dry-run --json` 会在不发布的前提下打印解析出的 slug、版本、文件数和指纹。`latestVersion: null` 表示该 slug 尚未被占用。
- 发布并非不可逆：`clawhub delete` 可以软删除一个 skill 或撤回某个版本，`clawhub hide` / `undelete` 也存在。所有权之后可以用 `--owner <handle> --migrate-owner` 转移，所以现在用个人账号发布，并不会把 skill 锁在将来 `superdesigndev` 组织的大门之外。

生成的 skill 已经满足的要求：frontmatter 含 `name` + `description` + 语义化版本的 `version`，且 `name` 与父目录名（`treg`）一致。发布后会运行一次安全扫描；被标记的版本会从公开工具目录中隐藏，但仍保留在你的控制台里。

⚠️ **ClawHub 强制每个 skill 都采用 MIT-0，不允许按 skill 覆盖。**本仓库是 Apache-2.0 外加托管服务限制条款。`SKILL.md` 是说明文字——它描述的是一个托管服务，而不是实现它——因此以宽松许可授权这一个文件，与「希望它被到处复制」是自洽的。这是深思熟虑的选择，不是疏忽；不要让它被引申为对服务端代码的许可声明。

在这里上架也会自动进入 Hermes 的 Skills Hub。注意，自 2026 年 2 月的「ClawHavoc」事件（341 个恶意 skill）之后，Hermes 把所有 ClawHub skill 硬编码为 `community` 信任级，所以这只是一个发现渠道，而不是信任信号。

#### 两种判定，只有一种决定可见性

不要把两者混淆——混淆的代价是一天时间：

- **Moderation**（`/api/v1/skills/treg/moderation`）才是决定 skill 隐藏或显示的机制。我们的记录是 `verdict: "clean"`、`isSuspicious: false`、`isMalwareBlocked: false`、`reasonCodes` 为空。阅读 CLI 源码可知：`isSuspicious` 只会在安装时打印一条可被 `--force` 绕过的警告；只有 `isMalwareBlocked` 会硬性拦截。
- **`clawscan`**（`clawhub scan`）是一次 LLM 审查，结果以**上架页上的 advisory（提示公告）**形式发布。它把 treg 评为 `suspicious`——而可见性没有因此发生任何变化。

v0.11.0 的 advisory 针对 `SKILL.md` 提出了四项发现。其中三项只是重申有意为之的产品决策；**有一项值得抛开 ClawHub、就其本身独立权衡**：

| id | 严重度 | 内容 |
|---|---|---|
| `PE3` | **HIGH** | `cli_auth` 会读取某个 CLI 钥匙串中的内容并转发——「扩大了 agent 的凭据访问面……却没有强而明确的同意边界」。建议的修复方式：在使用来自钥匙串的凭据之前按工具逐一征得同意（opt-in），并优先使用限定范围的服务 token。 |
| `SQP-1` | MEDIUM | frontmatter 里的 `description` 过于宽泛，可能导致 skill 在关联不大的任务中被选中 |
| `SQP-2` | MEDIUM | `curl … \| sh` 在没有校验和或签名的情况下执行远程代码 |
| `EA2` | MEDIUM | 「只询问一次、而非每条命令都询问」的指引会抑制逐命令审批 |

`EA2` 和 `SQP-2` 是 treg 有意接受的取舍。**`PE3` 是对一个真实功能的真实描述**，值得认真决策而不是一笔带过——`cli_auth` 的绑定方式见 `docs/context/architecture/`。静态分析与 VirusTotal 的结果均为无异常。

### 3. Skill Store（aiskillstore/marketplace）+ skills.sh

**不要向 `aiskillstore/marketplace` 发起 pull request。**它的 README 写得很明白——*「添加 skill 的 PR 会被直接关闭」*；这个仓库由平台自己的审核流水线写入，而不是由贡献者写入。请改走表单提交：

> <https://skillstore.io/submit>——只有一个必填字段，而且它接受**目录**，请填入：
> `https://github.com/superdesigndev/treg/tree/main/skills/treg`

GitHub 授权是可选的（只是多一层审核通知）；允许匿名提交。那里的审计是**仅报告性质**——风险发现不会阻止发布。这件事仍然值得做，因为 `aiskillstore/marketplace` 是 Hermes 的 `KNOWN_MARKETPLACES` 里仅有的两个仓库之一，所以这是进入 Hermes 的 `claude-marketplace` 来源最省事的途径。注意 `npx skillstore` 只支持安装——没有 CLI 提交途径。

**skills.sh / `npx skills`——始终带上 `-s treg`：**

```bash
npx skills add superdesigndev/treg -s treg     # ✅ just the treg skill
npx skills add superdesigndev/treg             # ❌ installs NINE skills
```

已在一台干净的机器上验证：不带 `-s` 的形式会扫描整个仓库，除了 `skills/treg/` 之外还会发现 `.agents/skills/*`，并把本仓库的**内部开发 skill**（`dev-local`、`tools-registry-context`、`write-provider-skill`、`add-oauth-provider`、`vendor-listing`，以及三个 Google 服务商 skill）连同 treg 一起装进用户的 agent。这等于把我们自己的仓库结构泄漏进了产品安装。`skills.sh.json` 阻止不了这一点——它只提供分类标签。

`-s treg` 把范围限定在单个 skill 上，我们发布的每一条安装命令都必须带上它。带上它之后，这一条命令依然能覆盖 Claude Code 之外的 70 多个 coding agent。

### 4. anthropics/claude-plugins-official——审核队列

在 <https://claude.ai/settings/plugins/submit> 提交。这是信任级别最高的上架位，也会同步给 Hermes。它是一个审核队列——和 MCP 渠道一样存在「耗时太久」的风险——这正是它被排在最后的原因：等到这份提交进入审核时，上面的渠道早已全部上线。

该目录中的条目使用 `git-subdir`，并同时固定 `ref` **和** `sha`，所以身处这个 monorepo 并不构成障碍。**打一个 release tag，把 tag 交给他们。**

## 第四扇门：treg.to 本身

`GET /.well-known/skills/index.json` + `/.well-known/skills/treg/SKILL.md`（位于 `src/treg/api.py`）使这台主机成为 agentskills.io 约定下的一等 skill 来源——不需要注册中心、不需要审核、不经过任何第三方。Hermes 直接读取它。由于它和 `/skill.md` 走同一个 `_serve_md`，`{BASE}` 会被模板替换成**实际提供服务的**主机，因此自托管的注册中心宣传的是自己，而不是 treg.to。

## 提交到任何渠道之前

- `uv run --frozen python -m pytest tests/test_plugin.py tests/test_skill_md.py -q`
- `claude plugin validate .`——按照 Claude Code 自身的解析方式校验 marketplace manifest，能发现 JSON schema 发现不了的结构错误。
- 从一个全新的 clone 真实安装一次，因为看起来正确的标记仍然可能出问题：

  ```bash
  git clone --depth 1 https://github.com/superdesigndev/treg /tmp/treg-check
  claude plugin marketplace add /tmp/treg-check
  claude plugin install treg@treg
  claude plugin details treg@treg     # then: uninstall + marketplace remove
  ```

  `details` 才是真正的检验。它必须报告 **Skills (1) treg** 和 **MCP servers (0)**——skill 数量为零是这套布局最担心的静默失败；而 agents/hooks 数量非零，则说明仓库根目录的什么东西泄漏进了插件。
- `claude plugin tag` 创建 `treg--v{version}` release tag，并校验 `plugin.json` 与所属 marketplace 条目是否一致——在把 tag 交给 Anthropic 之前值得运行一次。
- 确认文案仍然写明 treg 只**比较**各服务商、由调用方自行选择。treg 不做路由，也不做故障转移；落地页曾因这类说法被迫更正过一次，而商店上架页要更正难得多。`test_the_listing_does_not_promise_routing` 守着这条线。
- 确认一句话定位在三处完全一致（`test_one_product_one_sentence`）。
