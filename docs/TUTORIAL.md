# tools-registry — 动手教程

完整走一遍整个注册中心。每一步都给出**确切的命令**、**预期的输出**和**需要注意的地方**——因此本文可以独立阅读。把每条命令复制到你的终端里，跟着做即可。

这份同样的实操演练还有两个配套版本，由同一来源（`src/treg/web/tutorial.js`）生成：

- **在仪表盘中** → 在 `https://treg.to/` 登录，打开 **Help → Tutorial**。
- **独立版** → `https://treg.to/tutorial`。

### 两个聚焦的深入教程

这份主演练覆盖整个注册中心。有两个功能各自拥有详细的教程——完整的分步讲解请阅读它们（每篇都展示了确切的命令、真实输出以及测试方式）：

- **导入与 shell** → `https://treg.to/tutorial-import-shell.md` — 用一条命令（`treg upload clis`）把你机器上已有的 CLI 变成团队工具，并打开一个 shell（`treg shell`），让 `stripe`、`gh`、`gcloud` 等开箱即用，团队密钥自动注入。包含本地运行的**安全沙箱**（CLI 以一个受限用户运行，只能访问自己的 API，且不会把密钥留在磁盘上）。
- **团队访问控制** → `https://treg.to/tutorial-access.md` — 选择**每个成员可以使用哪些工具**，以及他们能否**在本地**运行 CLI——邀请时设置，随时可改。

---

## 概念（读一遍即可）

- **邮箱就是你的身份。** 你*就是*一个经过验证的邮箱。有三扇门可以证明它——**GitHub**、邮件发送的**一次性验证码**（OTP）或**邀请码**。首次证明某个邮箱时你会被*注册*；之后的每次证明都是*登录*。没有单独的注册流程（`treg register` 也已移除）。
- **代理 = 银行柜员。** 你是*通过*注册中心调用真实的上游 API。它把你的工具引用换成真实的机密，并在服务端注入。密钥永远不会落到你的机器上；你的令牌为这次调用授权。
- **令牌 = 一对（你, 组织）。** **User** 是一个身份；**Org** 是拥有资源的团队；**Membership** 以一个角色把两者关联起来。你的**身份令牌**（来自 `treg login`）在你所属的每个组织都可用——`treg org use <slug>` 选定当前组织。（Agent/CI 可以改用按组织颁发的令牌。）
- **邀请绑定到邮箱。** owner/admin 邀请的是一个*邮箱*。证明该邮箱（走任何一扇门）后，邀请就归你接受——无需验证码。验证码只是一条快速的带外捷径，并非必需。
- **工具与 skill。** **工具** = 一个上游 `base_url` + 一组凭据**绑定**（一个请求可携带多个）。**skill / bundle** = 一份配方（`SKILL.md`）+ 其机密 + 其工具，通过 `treg.json` 契约从文件夹注册。
- **使用工具的两种方式。** `treg call` 代理一个 HTTP **API**——机密在服务端注入，你的机器上什么都不会留下。`treg run` 在注入凭据的情况下运行一个厂商**命令行工具**（stripe、gh、gcloud 等），因此你无需持有密钥或登录就能使用 CLI。两个层级：`--local`（默认；在你的机器上运行——Linux 上密钥被隔离在专用的 `treg-run` 用户之下）和 `--server`（在注册中心服务器上运行，密钥永远不会到达你这里）。所有者需要先为每个工具选择加入；`treg runs` 是审计日志。

### 角色一览

| 操作 | viewer | member | admin | owner |
|---|:--:|:--:|:--:|:--:|
| 调用工具、查看清单 | ✅ | ✅ | ✅ | ✅ |
| 注册机密 / 工具 / skill | ❌ | ✅ | ✅ | ✅ |
| 编辑 / 删除自己的资源 | ❌ | ✅ | ✅ | ✅ |
| 编辑 / 删除组织内任何资源 | ❌ | ❌ | ✅ | ✅ |
| 邀请 / 移除成员 | ❌ | ❌ | ✅ | ✅ |
| 更改角色、删除组织 | ❌ | ❌ | ❌ | ✅ |

---

## 准备工作——在一台机器上模拟三个人

我们通过为每个用户指定各自的 `HOME`，在同一台笔记本上扮演三个用户，从而每个人都得到一个指向注册中心的、相互隔离的 `~/.treg/config.json`。在真实场景中，每个人都在自己的机器上，去掉 `HOME=` 前缀即可。

```bash
for u in tom bob alice; do
  mkdir -p ~/.treg-personas/$u
  HOME=~/.treg-personas/$u treg config --base-url https://treg.to
done
```

**注意：**在任何命令前加上 `HOME=~/.treg-personas/<name>` 前缀，即可扮演该用户。

---

# 第 1 部分——工具目录（无需密钥，无需设置）

这一部分之后的一切，都需要先有团队、机密和已注册的工具才能进行。工具目录则都不需要——它是从"已安装"到"拿到真实数据"的最短路径，也是 treg 现在的主打功能。

## 第 1 步——按它做什么来查找工具

Tom 刚装好 CLI。没有团队、没有密钥、什么都没注册——但他已经可以干活了。他不需要知道哪家厂商卖外链数据；他按**任务**搜索，每个搜索结果都显示价格。

```bash
treg catalog search "backlinks for a domain"
```

```
76 matches for "backlinks for a domain" — showing 25

  ENDPOINT                              PLATFORM   PROVIDER     COST
  dataforseo.web.backlinks.summary      web        DataForSEO   $0.024/call
  moz.links.summary                     web        Moz          $0.0025/call
  majestic.backlinks.list               web        Majestic     $0.010/call
```

> 通常有多家服务商以不同价格提供同一能力。treg 会把它们并排展示——**选择权在你**；它不会悄悄替你挑选，也不会替你自动切换。

## 第 2 步——花钱之前先看价格

每个收录的端点都带有价格、参数和示例响应。agent 应当先读这些内容，在*花掉*团队余额*之前*把成本告知人类。

```bash
treg catalog get tikhub.tiktok.user.profile
```

```
tikhub.tiktok.user.profile
Public TikTok profile by username (uniqueId) or secUid

  provider  TikHub (tikhub)
  call      GET /api/v1/tiktok/web/fetch_user_profile
  cost      $0.001/success
            charged on 2xx only
  verified  2026-07-28
```

> treg 没有公开定价的端点会被**拒绝**，绝不会免费提供服务——它会提示你改为接入自己的密钥。

## 第 3 步——直接调用，全程无需任何密钥

没有 TikHub 账号、没有注册、机器上也没有密钥。treg 持有凭据，在服务端注入，并把这次调用记到团队预付余额上。新验证账号在创建符合条件的团队时可获得**一次性 $1.00 免费额度**，按这个价格足够数百次调用。

```bash
treg call tikhub.tiktok.user.profile --query uniqueId=tiktok
```

> 如果你的团队已经拥有该服务商自己的密钥，它会自动优先——并且这些调用永远不会被计量。

## 第 4 步——看清到底花了多少

余额是一个只追加的账本，以整数微美元（百万分之一美元）计，因此花费不足一分的调用也会被精确记录，而不是被四舍五入抹掉。余额不足时返回 HTTP **402**，携带 agent 可以直接使用的数字。

```bash
treg balance
```

```
  Balance  $0.9990   (999000 micro-USD)

  RECENT
  settle   -$0.0010   tikhub.tiktok.user.profile
```

> 只有使用 treg 密钥的调用才消耗余额。你自己的密钥、你自己的工具和厂商 CLI 都不消耗。

# 第 2 部分——Tom 创立 Superdesign

## 第 5 步——Tom 登录（邮箱之门）

没有 `register` 这个命令。Tom 用一次性验证码证明自己的邮箱——由于是第一次，这个动作会同时**创建**他的账号和一个个人组织（因此永远不会出现空状态）。验证码会**通过邮件**发送给他；他查看收件箱并输入验证码。

```bash
HOME=~/.treg-personas/tom treg login --email tom@superdesign.dev
```
```
We sent a 6-digit code to tom@superdesign.dev.
Enter code: 429641
✓ Logged in as tom@superdesign.dev. Active org: tom-superdesign-dev
```

**注意：**去收件箱查收 6 位验证码，然后输入。Tom 现在持有一个**身份令牌**，在他加入的每个组织都有效——无需在多个令牌之间来回切换。（开发环境可以设置 `TREG_EMAIL_DEV_MODE=true`，把验证码直接打印出来而不是发邮件。）

## 第 6 步——Tom 创建团队

他的个人组织只属于他自己。现在他创建共享团队并成为其 **owner**；当前组织会切换到它，之后的一切都在该组织下进行。

```bash
HOME=~/.treg-personas/tom treg org create "Superdesign"
```
```json
{
  "org": "superdesign",
  "org_id": 2,
  "name": "Superdesign",
  "role": "owner",
  "token": "<per-org token — for agents/CI; a human doesn't need it>"
}
```

**注意：**个人组织在登录时自动创建；**团队则需要显式创建**，用 `org create`。

---

# 第 3 部分——Bob 通过邮箱之门加入

## 第 7 步——Tom 邀请 Bob

Tom 通过**邮箱**邀请一位队友。邀请绑定到该邮箱，Tom 会拿到一个一次性验证码，他*可以*转交——但 Bob 其实根本用不上它。

```bash
HOME=~/.treg-personas/tom treg org invite bob@superdesign.dev --role member
```
```json
{
  "code": "<one-time-invite-code>",
  "email": "bob@superdesign.dev",
  "role": "member",
  "org_id": 2,
  "expires_at": "2026-07-09T…"
}
```

**注意：**邀请处于待接受状态，发给 Bob 的邮箱，有效期 7 天（可用 `--expires-days` 修改）。

## 第 8 步——Bob 以自己的身份登录

切换到 Bob。他用同样的方式证明自己的邮箱——由于是第一次，这也会**创建**他的账号，连同他自己的身份令牌和个人组织。他从头到尾没有碰过邀请码。

```bash
HOME=~/.treg-personas/bob treg login --email bob@superdesign.dev
```
```
We sent a 6-digit code to bob@superdesign.dev.
Enter code: 512740
✓ Logged in as bob@superdesign.dev. Active org: bob-superdesign-dev
```

**注意：**与 Tom 相同的门——验证码落入 Bob 的收件箱，他输入即可。邮箱是身份——门（GitHub / 验证码）只是证明它的方式。

## 第 9 步——Bob 看到自己的邀请（无需验证码）

由于邀请绑定到 Bob 已证明的邮箱，他只需查询有哪些在等他。这就是完整的闭环：证明邮箱即可看到发给它的所有邀请。

```bash
HOME=~/.treg-personas/bob treg invites
```
```json
[
  {
    "id": 1,
    "org": "superdesign",
    "org_id": 2,
    "name": "Superdesign",
    "role": "member",
    "invited_by": "tom@superdesign.dev",
    "expires_at": "2026-07-09T…"
  }
]
```

**注意：**无需验证码，无需从 Tom 那里复制粘贴——已证明的邮箱本身就是凭证。

## 第 10 步——Bob 接受邀请

Bob 只需报出组织名即可加入 Superdesign。无需验证码——他已证明的身份就是凭证。当前组织切换为 Superdesign。

```bash
HOME=~/.treg-personas/bob treg accept superdesign
```
```json
{
  "org": "superdesign",
  "org_id": 2,
  "name": "Superdesign",
  "role": "member"
}
```

**注意：**Bob 现在是 Superdesign 的 **member**。

## 第 11 步——Bob 的两顶帽子

一个身份，两个成员身份——个人组织的 owner、Superdesign 的 member。同一个身份令牌在两者中都可用。

```bash
HOME=~/.treg-personas/bob treg org ls
```
```
  bob-superdesign-dev    bob@superdesign.dev    owner
* superdesign            Superdesign            member   (active)
```

**注意：**`*` 标记当前组织。随时可用 `treg org use <slug>` 切换。

---

# 第 4 部分——Alice 通过验证码之门加入

## 第 12 步——Tom 邀请 Alice 为 viewer

另一扇门：**验证码**。Tom 先把 Alice 邀请为 **viewer**——她可以读取和调用，但不能注册任何东西。

```bash
HOME=~/.treg-personas/tom treg org invite alice@superdesign.dev --role viewer
```
```json
{
  "code": "ZTeW5ss-cXiyvzeMs3em-…",
  "email": "alice@superdesign.dev",
  "role": "viewer",
  "org_id": 2,
  "expires_at": "2026-07-09T…"
}
```

**注意：**这次我们**保留验证码**——Alice 接下来直接使用它。

## 第 13 步——Alice 用验证码加入（无需先登录）

与 Bob 的对比：Alice **从不运行** `login`。验证码本身就证明了她的邮箱，因此 `join` 会创建她的账号、把她加入 Superdesign 并保存她的令牌——全部在一条命令中完成。

```bash
HOME=~/.treg-personas/alice treg org join ZTeW5ss-cXiyvzeMs3em-… --email alice@superdesign.dev
```
```json
{
  "org": "superdesign", "org_id": 2, "name": "Superdesign", "role": "viewer",
  "token": "<alice's superdesign token>",
  "personal": { "org": "alice-superdesign-dev", "org_id": 4, "role": "owner",
                "token": "<alice's personal token>" }
}
```

**注意：**一条命令创建了 Alice、给了她一个个人组织、让她成为 viewer——Tom 从未经手她的令牌。

## 第 14 步——viewer 角色的限制是动真格的

Alice 可以读取和调用，但 viewer **不能注册**任何东西。看看她如何被拦下。

```bash
HOME=~/.treg-personas/alice treg secret add testkey --value "nope"
```
```json
{
  "detail": "viewers can call and read, but cannot register"
}
```

**注意：**Alice 被授予的是*使用*权，而非*写入*权——角色门槛正在发挥作用。

---

# 第 5 部分——通过代理使用工具

## 第 15 步——Bob 注册一个机密

与 Alice 不同，**member** 可以注册。Bob 添加一个 API 密钥——在服务端加密存储，其值此后再也不会返回。

```bash
HOME=~/.treg-personas/bob treg secret add echo-key --value "sk-demo-secret-123"
```
```json
{
  "id": 1,
  "name": "echo-key",
  "kind": "env",
  "owner": "bob@superdesign.dev",
  "bundle_id": null
}
```

**注意：**机密是组织范围的（存在于 Superdesign 中）且归 Bob 所有。工具通过 `id` 绑定它。

## 第 16 步——Bob 注册一个工具

工具 = 一个上游 `base_url` + 凭据的注入方式。我们指向 postman-echo，以便*亲眼看到*注入效果。单个 `--secret` 默认注入 `Authorization` 请求头中的 `Bearer` 令牌。

```bash
HOME=~/.treg-personas/bob treg tool add echo --base-url https://postman-echo.com --secret 1
```
```json
{
  "id": 1, "name": "echo", "owner": "bob@superdesign.dev",
  "base_url": "https://postman-echo.com", "host": "postman-echo.com",
  "bindings": [
    { "secret_id": 1, "injector": "env", "location": "header",
      "name": "Authorization", "format": "Bearer {secret}", "secret_field": "access_token" }
  ]
}
```

**注意：**对于多凭据的上游，用 `--bind` 添加更多绑定——treg 在每次调用时应用每一个绑定。

## 第 17 步——Alice 调用它（没有任何密钥）

这正是 treg 的全部意义所在。Alice 是一个 **viewer**，她机器上**没有任何机密**。但当她调用时，上游看到的是 Bob 的密钥，在服务端注入。

```bash
HOME=~/.treg-personas/alice treg call echo /get
```
```json
{
  "args": {},
  "headers": {
    "host": "postman-echo.com",
    "authorization": "Bearer sk-demo-secret-123",
    "...": "..."
  },
  "url": "https://postman-echo.com/get"
}
```

**注意：**`authorization: Bearer sk-demo-secret-123`——这是 Bob 的机密，Alice 从未持有、看到或存储过它。

## 第 18 步——每次调用都有记录

代理为每次调用写一条审计记录。owner 可以查看组织的活动。

```bash
HOME=~/.treg-personas/tom treg calls --limit 5
```
```json
[
  {
    "id": 1,
    "user_email": "alice@superdesign.dev",
    "tool_name": "echo",
    "method": "GET",
    "path": "https://postman-echo.com/get",
    "status_code": 200,
    "created_at": "2026-07-02T…"
  }
]
```

**注意：**尽管 Alice 用的是 Bob 的机密，账本记录的是实际发起调用的**人**。

---

# 第 6 部分——调用形态与 skill

## 第 19 步——用完整 URL 调用（agent 原生）

agent 通常已经知道真实的上游 URL。不必用 `call <tool> <path>`，直接把**完整 URL** 交给 treg——它会按主机匹配到已注册的工具并注入密钥。无需任何 treg 专属知识。

```bash
HOME=~/.treg-personas/alice treg call https://postman-echo.com/get
```
```
# same echo response, with "authorization": "Bearer sk-demo-secret-123" injected.
# note: no tool name in the command — just the destination URL.
```

**注意：**treg 按**主机**解析工具，因此 agent 原生的完整 URL 形式开箱即用。

## 第 20 步——底层的原始 HTTP

`treg call` 只是语法糖。底层就是向 `<proxy>/call/<upstream-url>` 发送一个带令牌请求头的普通 HTTP 请求——任何语言、任何 agent、`curl` 都行。

```bash
ATOK=$(python3 -c "import json;print(json.load(open('/Users/you/.treg-personas/alice/.treg/config.json'))['token'])")
curl -s -H "X-Treg-Token: $ATOK" \
  "https://treg.to/call/https://postman-echo.com/get"
```
```
# the postman-echo JSON again, "authorization": "Bearer sk-demo-secret-123" injected —
# just curl, no secret on the client.
```

**注意：**整个产品浓缩成一行：在任何上游 URL 前加上代理前缀，带上你的令牌，treg 就会换入真实凭据。

## 第 21 步——起草一个 skill 的注册

整个 skill 文件夹（配方 + 凭据文件）可以通过 `treg.json` 契约一次性注册。`skill init` 会扫描 `SKILL.md` 和 `.secret/` 目录并起草——猜测基础 URL 并找出机密。文件里不写入任何值，只有引用。

```bash
HOME=~/.treg-personas/bob treg skill init --dir /tmp/skills/echo-svc
```
```
wrote /tmp/skills/echo-svc/treg.json
  auto: base_url=https://postman-echo.com | secrets=['echo-svc']
  review / fill:
    - base_url — heuristic guess, verify
    - health / examples — optional
```

**注意：**它读取了配方并正确猜出了 `base_url` + 找到了机密——修正它标记的任何内容，然后注册。

## 第 22 步——上传整个 skill

一条命令把这个文件夹变成一个可用的工具：配方、机密（值从 `.secret/` 加载，绝不会来自 json）和工具——全部以 **bundle** 的形式原子化创建。

```bash
HOME=~/.treg-personas/bob treg skill add --dir /tmp/skills/echo-svc
```
```json
{
  "id": 1, "name": "echo-svc", "owner": "bob@superdesign.dev",
  "recipe": "# echo-svc\n…the SKILL.md…",
  "tools":   [{ "id": 2, "name": "echo-svc", "base_url": "https://postman-echo.com", "bundle_id": 1 }],
  "secrets": [{ "id": 2, "name": "echo-svc", "kind": "env", "bundle_id": 1 }]
}
```

**注意：**所有内容共享一个 `bundle_id`，因此 skill 也可以作为一个整体删除。

---

# 第 6b 部分——导入：神奇的批量入口

把上面的一切，一次应用于你的**整个环境**。这是填充团队注册中心最快的方式。

## 第 22a 步——把整个 `.env` 变成工具

`treg upload env` 读取你的 `.env`，把每个变量与 treg 认识的服务商匹配，并把你选中的注册为可直接调用的工具。检测只读取**变量名**；只有你确认的密钥才会加载其值。配置类变量（`*_HOST`、`*_MODEL`、`*_PROJECT_ID`）和你应用自己的机密（`SECRET_KEY`、`SESSION_SECRET`、`DATABASE_URL`、`*_WEBHOOK_SECRET`）会被自动排除。

```bash
HOME=~/.treg-personas/bob treg upload env --select openai,stripe,resend
```
```text
Scanned .env: 6 key(s) to register, 1 OAuth, 4 other.
  ✓ openai         https://api.openai.com/v1   [Authorization: Bearer {secret}]
  ✓ stripe         https://api.stripe.com/v1    [Authorization: Bearer {secret}]
  ✓ resend         https://api.resend.com       [Authorization: Bearer {secret}]

Registered 3/3 tools.
```

**注意：**`CLIENT_ID`+`CLIENT_SECRET` 成对出现时会被识别为 **OAuth**，并引导你使用 `treg oauth connect`，而不是生成一个无法工作的 bearer 密钥。其他认证形态也会被处理——API 密钥请求头（`x-api-key`）、查询参数密钥（`?apiKey=`）或 Basic 对（base64 的 `id:secret`）。

## 第 22b 步——导入整个 skill 文件夹

把 `treg upload skills` 指向一个 skills 目录。对其中每一个，它会使用已有的 `treg.json`，或根据 skill 的脚本**生成一个**（基础 URL + 它读取的环境变量）——把 API 类 skill 注册为工具，把知识类 skill 注册为纯配方 bundle。整个团队库一次导入到位。

```bash
HOME=~/.treg-personas/bob treg upload skills --dir ~/.claude/skills --all
```
```text
Scanned ~/.claude/skills: 5 API-tool skill(s), 23 recipe-only.
  ✓ render          (tool)   [wrote treg.json]
  ✓ intercom        (tool)
  ✓ seo-blog-writer (recipe)
  …
Imported 27/28 skills.
```

随时可以重新运行——它会跳过已注册的（或用 `--replace` 更新）。队友随后可用 `treg skill install <name>`（或 `--all`）拉取其中任何一个，命令会把配方写入他们的 `.claude/skills/`。不带 `env`/`skills` 的裸 `treg upload` 会对当前目录**两者都做**。

> **非交互场景的安全保护：**在 agent/CI 中运行（无 TTY）时，没有 `--all` 或 `--select` 的话 upload 会拒绝执行，因此凭据绝不会在无人值守的情况下被意外注册。

---

# 第 6c 部分——运行 CLI 工具

到目前为止，每个工具都是你 `call` 的 HTTP **API**。许多服务商还提供**命令行工具**（stripe、gh、gcloud 等）。`treg run` 会在注入组织凭据的情况下执行该 CLI——队友*无需*持有密钥或登录即可使用。

## 第 22c 步——为一个 CLI 工具开启 `run`

因为运行会把凭据交给一台机器，所有者需要先为每个工具选择加入（仪表盘的 **⌘ run** 开关就是同一个开关）。这里 Tom 为 `stripe` 工具开启它（其 id 来自 `tool ls`）。

```bash
HOME=~/.treg-personas/tom treg tool update 4 --local-run on
```
```json
{
  "id": 4, "name": "stripe",
  "base_url": "https://api.stripe.com/v1",
  "cli": { "enabled": true }
}
```

**注意：**默认关闭——run 比代理调用能力更强，因此按工具选择加入，且只有 owner/admin 才能切换。

## 第 22d 步——运行厂商 CLI（无需登录，磁盘上不留任何东西）

现在 Bob *通过* treg 运行 Stripe 的真实 CLI。`--` 之后的一切都原样交给厂商工具。treg 仅为这一次运行注入凭据；Bob 从未登录过 Stripe，也没有存储过它的密钥。默认层级是 `--local`——在 Bob 自己的机器上运行，Linux 上密钥被隔离在专用的 `treg-run` 用户之下（用 `sudo treg setup-local-run` 安装一次；macOS 上是尽力而为）。加上 `--server` 则改为在注册中心上运行（适用于工具目录已知的 CLI），这样密钥完全不会到达 Bob 的机器。

```bash
HOME=~/.treg-personas/bob treg run stripe -- get /v1/balance
```
```
{
  "object": "balance",
  "available": [{ "amount": 0, "currency": "usd" }],
  …
}
# Stripe's own CLI ran with the org key injected — Bob never logged in or held the key.
```

**注意：**CLI 用 `treg run <tool> -- <args>`；HTTP API 用 `treg call <tool> <path>`——同一凭据，两种用法。

## 第 22e 步——每次运行都有记录

与代理调用一样，CLI 运行也会被审计。`--server` 运行会连同退出码和时长记录在运行账本中；`--local` 运行则把审计痕迹留在调用记录旁边（`treg calls`）。owner 用 `treg runs` 查看服务器运行。

```bash
HOME=~/.treg-personas/tom treg runs --limit 5
```
```json
[
  {
    "id": 1,
    "user_email": "bob@superdesign.dev",
    "bundle_name": "stripe",
    "argv": ["get", "/v1/balance"],
    "exit_code": 0,
    "duration_ms": 812,
    "created_at": "2026-07-02T…"
  }
]
```

**注意：**记忆口诀——`treg call` → `treg calls`；`treg run` → `treg runs`。两个动词，两本账。

---

# 第 7 部分——组织管理

## 第 23 步——查看团队

owner 列出所有人及其角色。角色变更通过 `user_id` 引用成员。

```bash
HOME=~/.treg-personas/tom treg org members
```
```json
[
  { "user_id": 1, "email": "tom@superdesign.dev",   "role": "owner"  },
  { "user_id": 2, "email": "bob@superdesign.dev",   "role": "member" },
  { "user_id": 3, "email": "alice@superdesign.dev", "role": "viewer" }
]
```

**注意：**一条命令看到完整名册。

## 第 24 步——把 Bob 提升为 admin

只有 owner 能变更角色。我们把 Bob 设为 **admin**——他可以邀请/管理，但转移和删除仍仅限 owner。"最后一名 owner" 保护机制会防止组织变得群龙无首。

```bash
HOME=~/.treg-personas/tom treg org set-role 2 admin
```
```json
{
  "user_id": 2,
  "role": "admin",
  "org_id": 2
}
```

**注意：**一个原语（`set-role`）覆盖了提升、降级和所有权转移。

## 第 25 步——admin 权限实战

作为普通 member 时 Bob 不能邀请；成为 **admin** 后就可以了。他邀请一位新队友。

```bash
HOME=~/.treg-personas/bob treg org invite dana@superdesign.dev --role member
```
```json
{
  "code": "<one-time-code>",
  "email": "dana@superdesign.dev",
  "role": "member",
  "org_id": 2,
  "expires_at": "2026-07-09T…"
}
```

**注意：**Bob 不必是 owner 也能管理团队。

## 第 26 步——查看待处理的邀请

admin 可以看到组织中所有仍未处理的邀请。已接受、已撤销和已过期的会被过滤掉。

```bash
HOME=~/.treg-personas/bob treg org invites
```
```json
[
  {
    "id": 3, "email": "dana@superdesign.dev", "role": "member",
    "invited_by": "bob@superdesign.dev", "expires_at": "2026-07-09T…"
  }
]
```

**注意：**只有 Dana 显示——Bob 和 Alice 的邀请已被接受，因此从列表中消失了。

## 第 27 步——撤销一个邀请

计划有变——Bob 在 Dana 使用之前作废了她的邀请。这会硬删除该验证码，使其永远无法被接受。

```bash
HOME=~/.treg-personas/bob treg org revoke 3
```
```json
{
  "revoked_invite": 3
}
```

**注意：**加入时：过期 → `410`；已撤销 / 已使用 / 未知 → `404 invalid or already-used invite`。

## 第 28 步——从 viewer 一侧看角色门槛

Alice 是 viewer。她可以调用工具，但不能邀请——那需要 admin 及以上。她被拒绝了。

```bash
HOME=~/.treg-personas/alice treg org invite eve@superdesign.dev --role member
```
```json
{
  "detail": "admin role in this org is required"
}
```

**注意：**角色被干净利落地执行：**owner** > **admin** > **member** > **viewer**。

---

# 第 8 部分——超级管理员

## 第 29 步——成为平台运营者

超级管理员位于组织*之上*——可以读取并管理每个租户。两种授权方式：平台 bearer 令牌 `TREG_ADMIN_TOKEN`，或被标记为 `is_superadmin` 的用户。我们使用 bearer，从 `.env` 读取，因此它不会出现在屏幕上。

```bash
treg admin login --token "$(grep -E '^TREG_ADMIN_TOKEN=' .env | cut -d= -f2-)"
```
```
admin token saved
```

**注意：**由 `require_superadmin` 把关，与组织角色相互独立：普通令牌 → 403，无令牌 → 401。

## 第 30 步——整个平台尽收眼底

一次调用即可获得所有租户的汇总——这是任何单个组织 owner 都看不到的全景。另有 `admin orgs / users / tools / health` 用于跨租户清单。

```bash
treg admin stats
```
```json
{
  "totals": { "users": 3, "orgs": 4, "tools": 2, "secrets": 2, "calls": 1 },
  "...recent-activity + distributions...": "..."
}
```

**注意：**可直接用于门户的 JSON：按 injector/主机的分布、凭据健康度汇总、调用量和增长计数。

## 第 31 步——所有租户的每个组织

跨租户可见性：Superdesign 及其成员 + 工具，外加每个人的个人组织。

```bash
treg admin orgs
```
```json
[
  { "id": 2, "slug": "superdesign", "name": "Superdesign", "members": 3, "tools": 2 },
  { "id": 1, "slug": "tom-superdesign-dev", "members": 1, "tools": 0 },
  { "id": 3, "slug": "bob-superdesign-dev", "members": 1, "tools": 0 },
  { "id": 4, "slug": "alice-superdesign-dev", "members": 1, "tools": 0 }
]
```

**注意：**这正是日后客服控制台或计费门户所依托的接缝——同样的 JSON，只是换了个渲染。

## 第 32 步——授予真实用户超级管理员

环境变量里的 bearer 用于引导；然后你提升具名用户，让他们用自己的身份令牌访问 `/admin/*`——无需传递共享密钥。

```bash
treg admin grant 1
```
```json
{
  "user_id": 1,
  "is_superadmin": true
}
```

**注意：**授权后，Tom 的普通身份令牌即可用于 `admin` 命令——仪表盘的 Admin 面板也会为他点亮。

---

# 第 9 部分——仪表盘

## 第 33 步——同一个注册中心，在浏览器里

打开 **treg.to**，用**邮箱验证码**之门登录（就是你在终端里用的那个）：输入邮箱 → 点击 **Email me a sign-in code** → **去收件箱**查收 6 位验证码 → 粘贴并 **Sign in**。你会落在团队组织页——Tools 显示 `echo` 工具，Activity 显示那次调用，而且（因为 Tom 现在是超级管理员）还会出现一个 **Admin** 面板。

```bash
open https://treg.to/
```
```
# Sign in with email → land on Superdesign
#   Tools    → the echo tool (Copy a snippet · Try it live)
#   Activity → Alice's GET echo · 200
#   Admin    → cross-tenant stats + orgs (super-admin only)
```

**注意：**仪表盘目前只支持读取 + 调用；创建/邀请/注册仍在 CLI 中完成（仪表盘的写入界面属于第二阶段）。

---

# 第 10 部分——清理

## 第 34 步——删除工具

Bob（工具的创建者，也是 admin）删除该工具。绑定的机密保留——只删除工具。member 不能删除队友的资源。

```bash
HOME=~/.treg-personas/bob treg tool rm 1
```
```json
{
  "deleted": 1
}
```

**注意：**删除顺序很重要：先删除工具（或其绑定），再删除它使用的机密。

## 第 35 步——删除组织（完全级联）

收官之作。删除组织仅限 owner，且需要**按名称确认**——你必须输入 slug，而且它必须是你的当前组织。级联删除会移除所有成员关系、工具、机密、bundle、邀请和审计记录。

```bash
HOME=~/.treg-personas/tom treg org delete superdesign
```
```json
{
  "deleted_org": 2
}
```

**注意：**Bob 和 Alice 保留他们的个人组织——他们始终是独立的租户。这就是完整的生命周期：登录 → 团队 → 邀请（两扇门）→ 角色 → 工具 → 代理调用 → 审计 → 管理 → 拆除。 🏁

---

## 附录 A——四种认证形态

| 形态 | 使用者 | 方式 | 请求头 |
|---|---|---|---|
| **身份令牌** | CLI 上的人类用户 | `treg login`（GitHub）或 `treg login --email`（OTP） | `X-Treg-Token: <identity>` + `X-Treg-Org: <slug>` |
| **按组织令牌** | agent / CI | 在 `org create` / `org join` 时固化到组织中 | `X-Treg-Token: <org-token>`（组织隐含其中） |
| **会话 cookie** | 仪表盘 | GitHub 或邮箱验证码登录会设置一个签名的 HttpOnly cookie | cookie + `X-Treg-Org: <slug>` |
| **Admin bearer** | 平台运营者 | `TREG_ADMIN_TOKEN`，或被标记为 `is_superadmin` 的用户 | `/admin/*` 上的 `X-Treg-Token: <admin>` |

## 附录 B——命令参考

```
treg login [--email you@x.com | --token <t>]   # sign in (GitHub default; email OTP; or a raw token)
treg logout                                     # clear your credentials
treg invites                                    # invites addressed to you (code-free)
treg accept <org-slug>                          # accept one addressed to you (no code)

treg org create "<Name>"                        # make a team, become owner
treg org ls | org use <slug>                    # list / switch active org
treg org invite <email> [--role viewer|member|admin] [--expires-days N]
treg org join <code> --email you@x.com          # the code door (creates you if new)
treg org members | set-role <user_id> <role>    # roster / change a role (owner)
treg org invites | revoke <invite_id>           # list / kill pending invites (admin+)
treg org leave | delete <slug>                  # self-remove / delete (owner, confirm-by-name)

treg secret add <name> (--value V | --file F | --dir D) [--kind env|oauth|...]
treg tool add <name> --base-url URL (--secret ID | --bind '...' | --binding '<json>')
treg secret ls | rm ID | update ID …            # tool ls | rm | update likewise
treg call <tool> <path>  |  call <full-url>     # proxy a call (named or agent-native)
treg calls [--limit N]                          # the call audit log

treg run <tool> -- <cli args>                   # run a vendor CLI with the key injected (--local default; --server = on the registry)
treg runs [--limit N]                           # the CLI-run audit log
treg tool update <id> --local-run on|off        # owner opt-in: allow/forbid `treg run` for a tool
sudo treg setup-local-run                       # one-time: isolate local runs under the treg-run user (Linux)

treg skill init --dir D | skill add --dir D | skill ls | skill rm ID
treg admin login --token T | stats | orgs | users | tools | health | grant/revoke ID | …

Global: --org <slug> runs any single command in that org instead of the active one.
```

`register` 命令已退役——`login` 即注册即登录。多用户演示请使用**邀请路径**（`org invite` → `login`/`invites`/`accept`，或 `org join <code>`）。
