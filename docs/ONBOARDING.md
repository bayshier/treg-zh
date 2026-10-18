# tools-registry——上手指南

这是 agent（或人）开始调用共享工具、并分享自己工具时遵循的入门流程。CLI 是 `https://treg.to` 上 API 的薄客户端；API 才是唯一的大脑。

## 1. 安装 CLI

```bash
curl -fsSL https://treg.to/install.sh | sh   # installs `treg`, points it at the registry
```

（想从克隆的仓库运行？`uv sync && uv run treg --help`。）

## 2. 登录

三种入口，同一个身份（你的邮箱）：

```bash
treg login                                  # GitHub OAuth (opens the browser)
treg login --email you@example.com          # email one-time code
treg login --token <TOKEN>                  # agents/CI: a per-org token from a team owner
```

你的 token 在每次调用中标识你的身份（`X-Treg-Token`）。初次使用？`treg onboard` 会用一个一次性演示团队带你走完整套流程。

## 3a. 使用别人分享的工具（消费者）

```bash
treg tool ls                                 # what's available
treg call posthog query/events --query limit=5
treg run gh -- pr list                       # vendor CLIs work too
```

你**不持有任何上游密钥**——注册中心会在服务端注入密钥。

## 3b. 分享你自己的工具（创建者）

让 treg 指向一个项目——它会找出 `.env`、skill 文件夹和已安装工具目录 CLI 中的服务商密钥，并注册你选中的部分：

```bash
treg scan          # read-only preview; nothing leaves the machine
treg upload        # register (encrypted server-side); idempotent, --replace to update
```

单个 skill 或复杂工具（多凭据、OAuth）请参见 [`USAGE.md`](../USAGE.md) 中的手动流程——`treg skill init` / `skill add`、`treg tool add --bind`、`treg oauth connect`。

## 4. 验证与观察

```bash
treg call <tool> <upstream-path>     # smoke-test
treg calls                           # audit: who called which tool, when, status
treg health                          # credential health across the org
```

## 备注

- 密钥只写不读；API 永远不会返回已存储的值。
- 工具可以引用其他成员上传的密钥（use-without-hold，用而不持）。
- 一切都以组织为范围：`treg org ls` / `treg org use <slug>` 用于选择活跃团队。
