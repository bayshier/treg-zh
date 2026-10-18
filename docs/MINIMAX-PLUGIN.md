# MiniMax 插件及其提交方式

treg 的第五个橱窗：**MiniMax Plugin Marketplace**，同时面向 **MiniMax Code**（桌面端）和 **MiniMax Agent**（云端）。与 Claude Code、Cursor 插件一样，它是**仅含 skill** 的，但理由比图方便更硬：MiniMax 拒绝任何含有凭据的包，而 treg 的 MCP 是按用户做 bearer 认证的。带认证的 MCP 必须走他们的 **App/Connector** 计划（OAuth MCP server、PKCE、由 MiniMax 注册的 client）——见下文「后续：以 App 形式接入」。

规则的出处：MiniMax 的 *Plugin Development and Submission Guide*（飞书 wiki `QlTLwbAGNiACwLkWI85cFxHzn4L`，最后更新于 2026-08-17）。那是一个需要飞书登录的 JS 渲染页面；重要的规则已编码在 `scripts/minimax_plugin.py` 中。

## 文件分布

```
plugins/minimax/
├── .minimax-plugin/plugin.json   the manifest (schemaVersion 1)
├── icon.png                      512×512, the ▚ mark — same file as plugin/assets/icon.png
└── skills/treg/SKILL.md          GENERATED — never edit by hand
scripts/minimax_plugin.py         validator (--check) and ZIP builder (--zip OUT)
```

`plugins/minimax/` 是**插件根目录**（与 Cursor 的根目录 `plugins/treg/` 平级；每个商店都需要自己的根目录，因为 manifest 和生成的 bootstrap 各不相同）：`.minimax-plugin/plugin.json` 直接位于其中，这正是 MiniMax 对 ZIP 根目录或 GitHub 子目录的要求。manifest 的 `name` 为 `treg`；skill 在运行时以 `treg:treg` 暴露。

**绝不要直接编辑 `plugins/minimax/skills/treg/SKILL.md`。**请修改 `src/treg/web/skill.md` 然后重新生成：

```bash
uv run python scripts/build_plugin.py            # regenerate ALL plugins
uv run python scripts/build_plugin.py --check    # fail if any is stale (also a test)
python3 scripts/minimax_plugin.py --check  # MiniMax package rules (also a test)
```

### 为什么它的 bootstrap 与 Claude Code 的不同

Claude/Cursor 的 bootstrap 有三步：安装 CLI → `treg login` → `treg mcp install`。MiniMax 的只有**两步**，并明确说明不要执行第 3 步。`treg mcp install` 写入的是 Claude Code、Cursor 和 opencode 的配置；没有 MiniMax 目标，因此在 MiniMax Code 上它是一个静默的空操作，agent 会毫无察觉地直接略过。skill 里的所有事情都通过 CLI 完成。

### 他们的校验器强制执行什么（以及我们的预检查）

- `.minimax-plugin/plugin.json` 必须位于根目录，ZIP 中**不得有包裹目录**。
- `name`：小写，以字母开头，≤80 个字符，字符集为 `[a-z0-9._-]`。`version`：SemVer，**只要包内容变化就必须递增**（仅地区/投放目标变化可以保持不变）。我们的版本跟随 `pyproject.toml`——由 `test_plugin_version_tracks_the_package` 固定。
- `category` 从固定列表中选择（Office、Studio、Design & Sites、Code、Business、Sales、Productivity、Science & Healthcare、Education、Other）。我们使用 **Productivity**。
- `exampleQueries`：0–3 条，非空，≤4,096 字符，必须是*生产环境*内容。
- `apps` / `mcpServers` / `skills`：相对路径；**未使用的字段保持为空数组**。
- `icon`：相对路径，小写的 `.png/.jpg/.jpeg/.webp`，正方形。
- skill：`skills/<name>/SKILL.md`，YAML frontmatter 的 `name` == 目录名且 `description` 非空；正文必须是可执行的指令，而非营销文案。
- 路径仅限 ASCII 字符，使用 `/` 分隔符。不允许符号链接、硬链接、LFS 指针、子模块、安装脚本、原生二进制文件、可执行位。UTF-8 无 BOM。JSON 必须是对象，不允许重复键。
- 限制：ZIP ≤64 MiB，条目 ≤2,048 个，文件 ≤1,024 个，单文件 ≤16 MiB，路径 ≤512 B / 单段 ≤128 B / 深度 ≤16。
- **任何地方都不得出现 token、密钥、secret 或个人数据。**

manifest 不携带 `license` 键——他们的 schema 没有定义这个键，未知键是校验器上的风险。许可改为在这里声明：`SKILL.md` 是关于一个托管服务的说明文字，按本仓库的 Apache-2.0 发布（见 `LICENSE`）。

## 提交操作手册

接受两种来源，走同一条流水线。**优先用 GitHub**——没有产物需要管理，更新只需换一个新的 `Ref`。

**提交**（表单入口在指南里：「Open the Plugin submission form」；需要飞书账号——使用团队保存的那个账号，之后查询状态必须用*同一个*账号）：

| 字段 | 值 |
|---|---|
| Action | Submit a new Plugin |
| Region | **US**（需要的话也选 CN——两者是相互独立的维度；从 CN 可以访问 treg.to，但计费走 Stripe） |
| Delivery target | **Desktop (MiniMax Code) + Cloud (MiniMax Agent)**——skill 由 CLI 驱动，所以只有当 Agent 沙箱能运行 `install.sh` 时云端才有意义；两个都选，他们可以停用其中一个 |
| Source | GitHub |
| Repo | `https://github.com/superdesigndev/treg` |
| Ref | `main`（或 release tag，例如 `v0.12.1`） |
| Plugin subdirectory | `plugins/minimax` |
| Email | jason@superdesign.dev |

ZIP 备选方案——确定性构建，无包裹目录：

```bash
python3 scripts/minimax_plugin.py --zip treg-minimax-$(python3 -c 'import tomllib;print(tomllib.load(open("pyproject.toml","rb"))["project"]["version"])').zip
```

**提交之后：**保存 `submission_id`。流水线：源码冻结 → 脚本校验（失败会以飞书消息返回）→ 向 CN/US marketplace 的 `main` 发起 Release MR → 人工审查 MR diff、Markdown 报告、图标、CI → 所有者合入 → 流水线打包 → 通过飞书消息告知状态 "Publishing" / "Published" / "Publishing failed"。查询状态需要提交时所用的飞书账号 + `submission_id` + 邮箱。

**更新：**`name` 保持 `treg`，递增 `version`（release 时的版本递增已经覆盖这一点），提供持续维护的证明（GitHub 仓库本身就是证明），用新的 Ref 重新提交。无需变更摘要——他们会对比 MR 的 diff。

## 已知的审核风险

1. **skill 中的 `curl … | sh`。**包本身不含安装脚本，但 skill *会让 agent 去运行一个*。ClawHub 的 LLM 审查标记过同一行（`SQP-2`，见 `docs/CLAUDE-PLUGIN.md`）。如果审核者有异议，退路是 `pip install treg` / `uv tool install treg`——CLI 已上架 PyPI——以及在 `MINIMAX_BOOTSTRAP` 里做一行替换。
2. **云端目标。**在 MiniMax Agent 上，skill 只有在沙箱具备可出网的 shell、并且能为 `treg login` 打开浏览器时才能工作（如果不具备，就引导他们改用设备码登录）。如果云端没有通过他们的验证，就改为仅桌面端重新提交——这只是商店目录元数据的变更，不需要递增版本。
3. **描述与实现。**他们的规则：说明解决什么问题，而不是怎么实现。manifest 的描述只写能力；保持这个写法。

## 后续：以 App 形式接入

要让五个 MCP 工具原生进入 MiniMax，treg 需要加入他们的 **App/Connector** 流程：在构建*之前*联系 MiniMax，提供测试 + 生产的 Streamable-HTTP MCP 端点、带发现元数据（discovery metadata）的 OAuth issuer、authorization/token/registration/revocation 端点、scopes、DCR/PKCE（S256）支持、refresh token、回调白名单需求、测试账号、速率限制和错误码。MiniMax 随后会分配一个 `provider`，插件随之获得 `treg.app.json`（`{"schemaVersion":1,"provider":"treg"}`）。treg 的 MCP 目前用静态 bearer token 认证，他们的指南要求在开发*之前*就此事与他们沟通——所以这是一个产品决策（在 `/mcp/` 前面加一个 OAuth AS），而不是打包问题。
