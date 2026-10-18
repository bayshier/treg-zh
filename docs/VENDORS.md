# 把你的 API 上架到 treg

treg 是面向 AI agent 的工具目录：约 2,600 个端点，覆盖约 40 家服务商，按*能力*（「查询某个域名的反向链接摘要」）而非服务商名称来发现。agent 搜索工具目录，把各家服务商在价格 / 实测成功率 / 速度上并排比较，然后通过 treg 的代理调用你的 API，凭据由服务端注入。

**上架能带来什么：**

- **可发现性**——每个连接到 treg 部署的 agent，在搜索你的 API 所完成的工作时都能找到你的端点，与你的竞争对手并列出现（并被放在一起比较）。
- **零摩擦的客户**——把自己的密钥连接到你 API 的团队会直接调用你；计费关系保留在你手中。treg 永远不会插在你和客户的账号之间。
- **按量计费的平台流量（可选）**——如果你的定价已公开且可被机器核验，treg 可以用*自己的*密钥对外提供你的端点，按次从团队的预充值余额中计量扣费。这意味着从未在你那里注册过的团队也会带来付费用量。见下文「平台资格」一节。

## 准入条件

以下条件必须全部满足。它们存在的原因是：每一条上架记录都经过*实测*，而不是照抄文档：

| 要求 | 原因 |
|---|---|
| **自助获取 API 密钥**——注册、拿到密钥，无需销售介入 | 整条上架流水线要在一次会话内跑完；需要销售放行的 API 无法验证 |
| **密钥通过 header 或 query 参数传递** | treg 以 header（首选）或 query 参数的形式注入凭据。不支持把密钥嵌在 URL 路径中 |
| **一个免费（或近乎免费）的探测端点**，对有效密钥返回 2xx，对无效密钥返回可区分的失败 | 用于连接时验证。我们会真的向你的 API 发送一个垃圾密钥，看着它被拒绝——**接受任何密钥的 API 无法上架** |
| **公开的定价页**，且 URL 稳定 | 工具目录中的成本带有出处（`source_url`、`checked`、`confidence`）；「联系销售」式的定价无法以数字形式上架 |
| **核心端点的文档准确**：参数名、类型、必填标志和示例*值* | 每个上架端点都会被真实调用测试；没有示例值的文档无法验证 |

强烈建议具备（每一项都直接改善你的上架效果）：

- **稳定 URL 上的 OpenAPI 规范**——让我们能够机器生成你*完整的*端点面，作为「扩展层」（数百条路由），而不仅是人工精选的 8–15 个核心端点。
- **机器可读的费率卡端点**（返回你当前各端点价格的 API 路由）——这是我们掌握的最强成本出处，也是获得平台资格的最快途径。
- **按成功计费**（出错免费）——让验证和重试变得便宜，也意味着失败的调用永远不会向 treg 用户收费。
- **有文档记录的速率限制**，包括按路由的限制，而不只是账户级的总限制。

## 需要提交什么

最快的路径：把控制台 Catalog 页面上 **"List as vendor"** 按钮给出的提示词粘贴给你的 coding agent——它会按照 [`/vendor-listing`](https://treg.to/vendor-listing) 上的托管说明执行，并替你发起 PR。或者你也可以直接在这个仓库开 issue 或 PR，内容包含：

0. **在 PR/issue 描述中附上联系邮箱**（例如 `Contact: partnerships@yourapi.com`）——我们用它来安排用于实测的测试凭据；缺少联系邮箱的 PR 无法验证，也无法合入

1. `service` slug（小写）、显示名称，以及一句话概括 agent 能用它*做什么*
2. `base_url`、认证机制（header 名称 + 格式，或 query 参数名）
3. 探测端点路径 + 无效密钥的确切行为（状态码，或发生翻转的 JSON 字段）
4. 定价页 URL + 计费模式；如有费率卡端点也一并附上
5. 文档 URL；如已发布 OpenAPI 规范也请附上其 URL
6. 你的 8–15 个核心端点，每个都附带示例参数值和一个低成本测试目标
7. 一份测试凭据（或少量额度赠送），在我们联系后私下提供——**绝不要贴在 issue/PR 里**——以便我们进行实测验证
8. **一份自查核验台账**：提交之前，用你自己的账号对每一条 `test_request` 做真实调用，并按端点记录 HTTP 状态、你的 YAML 所声称的成本，以及你的计量系统实际收取的成本（charge 字段、费率卡端点或余额变动），并注明日期。文档会漂移，计量系统不会——曾有一个真实提交，按文档誊写的价格比计量系统实际收取的低了 5 倍。故意打偏（deliberate-miss）的测试目标必须如实标注，并附上在真实目标上观测过一次的命中价格。
9. **一张完整端点面地图**：每一个有文档记载的操作，标注为已收录或附原因排除——包括免费的计数/预览路由和你最便宜的操作档位，这些正是 agent（和审核者）最先寻找的东西。

下面的示例展示了一条上架记录的完整形态。

## 实現示例——「Acme SEO」

一个虚构的服务商：Acme SEO 在 `https://api.acmeseo.example/v1` 出售反向链接数据，密钥放在 `X-Api-Key` header 中，`/account` 路由免费，每次成功请求 $0.002，定价见 `https://acmeseo.example/pricing`。

### 1. 注册中心条目——`src/treg/oauth_providers.py`

团队如何连接密钥，以及 treg 如何验证它是真实的：

```python
ACMESEO = OAuthProvider(
    service="acmeseo",
    display_name="Acme SEO",
    auth_kind="key",
    token_label="API key",
    token_placeholder="your Acme SEO API key",
    token_header="X-Api-Key",
    token_format="{secret}",
    setup_url="https://acmeseo.example/dashboard/api-keys",
    setup_action_label="Get your Acme SEO API key",
    setup_steps=(
        "Sign in to Acme SEO and open Settings → API Keys.",
        "Copy your API key.",
    ),
    setup_note="Backlink queries are billed per successful request; the account check is free.",
    auth_uri="", token_uri="",
    scopes={},
    client_id_setting="", client_secret_setting="",
    category="SEO",
    summary="Backlink profiles, referring domains and anchor texts for any domain or URL.",
    base_url="https://api.acmeseo.example/v1",
    docs_url="https://docs.acmeseo.example",
    probe_path="/account",  # free — a bad key gets a 401 here
)
```

……把它加入 `REGISTRY` 元组，再在 `src/treg/web/logos/acmeseo.svg` 放一个中性字母标（lettermark），并在两个测试清单里补上条目（`test_every_provider_is_registered`、`test_key_providers`）。

如果你的 API 即使对无效密钥也返回 HTTP 200，条目则改为指定用于区分两者的响应体字段（`token_verify_field`、`token_ok_field`/`token_ok_value`，或 `token_reject_field`）——把确切行为告诉我们，我们来选对的那个。

### 2. 核心工具目录文件——`src/treg/catalog/acmeseo.yaml`

agent 能做什么，附带价格且可验证：

```yaml
provider: acmeseo                  # must equal the registry entry's `service`
source:
  docs: https://docs.acmeseo.example
  openapi: https://api.acmeseo.example/v1/openapi.json
  curated: 2026-08-12
limits: "60 requests/minute per key"
pricing_url: https://acmeseo.example/pricing
endpoints:
  - id: acmeseo.web.backlinks.summary
    capability: web.backlinks.summary     # the cross-provider join key, from capabilities.yaml
    platform: web
    scope: any_account                     # data about anything, not the caller's own account
    method: GET
    path: /backlinks/summary
    name: "Backlink summary"
    summary: "Aggregate backlink profile of a domain or URL"   # vendor's wording, verbatim
    input:
      queryParams:
        target: {type: string, required: true, note: "domain or full URL", example: "moz.com"}
        mode:   {type: string, required: false, note: "one of domain | url", example: "domain"}
    test_request:                          # the CHEAP call verification replays
      queryParams: {target: "moz.com", mode: "domain"}
    cost:
      type: per_success                    # errors free
      value: 0.002
      currency: USD
      source: docs
      source_url: https://acmeseo.example/pricing
      checked: 2026-08-12
      confidence: documented
    docs_url: https://docs.acmeseo.example/backlinks-summary
  # …7–14 more endpoints, same shape
```

`capability` 字段是把你放上比较货架的关键：请求 `web.backlinks.summary` 的 agent 会看到实现该能力的每一家服务商，价格并排呈现。如果你的 API 所做的工作在现有分类体系中没有对应条目，就在文件的 `proposed_capabilities:` 下提议一个。

每个端点都能力优先：写下这个字段之前，先在 `capabilities.yaml` 和其他服务商文件里搜索这项工作，复用已有的 id，而不是造一个近似重复。只有在没有任何条目覆盖这项工作时才提议新能力；如果另一家服务商已经有做这件事但尚未映射的端点，就在 PR 里点名那个端点，让两者挂到同一个能力之下。重叠正是意义所在：共享同一能力的端点，正是 agent 会放在一起比较并在其间路由的对象。

### 3. 合入之前我们会运行什么

你的自查核验台账能加快这一步，但永远替代不了它：服务商 PR 中的每一项声明——价格、探测行为、证据——都会用我们掌控的凭据，通过一次**独立的**真实调用重新核验。`verified:` 戳记和示例响应由我们添加，依据是我们亲眼观测到的成功调用。台账与它自己的 YAML 相互矛盾，PR 会被退回；价格与我们的计量不符，则以实际观测到的扣费为准修正。

```bash
# 1. Live bogus-key test: a garbage key POSTed to /connections/token must come back rejected
# 2. Live verification of every endpoint's test_request (with the test credential, from env)
TREG_CATALOG_CRED='<key>' uv run python scripts/catalog_verify.py acmeseo.yaml
# 3. Schema + referential integrity
uv run python scripts/catalog_validate.py     # must exit 0
# 4. The full test suite
uv run --frozen python -m pytest -q
```

每个端点行还带有 `added:`，即它进入工具目录的 UTC 日期。发起 PR 之前请运行 `uv run python scripts/catalog_added.py`：它会为每一个缺失日期的行写入今天的日期。已有工具的日期永远不会改变。

通过验证的端点会获得 `verified:` 日期，以及一条经过截断和 PII 清洗的真实示例响应，提交到 `src/treg/catalog/examples/`——这个示例正是在多家服务商之间做选择的 agent 最有说服力的依据，因此通过验证是上架记录最重要的资产。

## 平台资格——用 treg 自己的密钥获得付费调用

除了可发现性之外，treg 还能用**自己的**密钥对外提供你的端点，按次从调用团队的预充值余额中扣费。一个端点符合资格的条件是：

- 其成本可以按美元机器计算（有具体的 `value`，或可换算的 credit/单位费率），
- 价格的 `confidence` 为 `verified`（观测到实际计费，或读取自你在线的费率卡）或 `documented`（从你公开的定价誊写），并且
- 它不是自有账号类或账号管理类路由。

符合资格是必要条件而非充分条件——treg 还要在你那里持有一个有余额的账号，并把你的服务加入白名单，这是按服务商逐个做出的运营决策。最快达成的方式：发布一个费率卡端点、按成功计费，并给我们一笔用于验证的额度赠送。

## 哪些情况会被拒绝

过往审核中记录在案的拒绝原因，免得有人白费一轮功夫：

- API 接受任何密钥（无法验证连接）
- 密钥嵌在 URL 路径中
- 需要销售放行 / 仅限企业客户的接入
- 法律或关停风险；已废弃或被吸收合并的产品
- 只有 UI、没有 API 的产品

## 保持上架记录的健康

- **价格变了？**告诉我们，或者直接对 `cost` 块提 PR——价格超过 90 天未更新会被标记。
- **路由变了？**如果你发布了 OpenAPI，重新摄取会按规范重新生成你的扩展层；人工精选的核心文件靠手工更新，因此那里的破坏性变更需要提前告知。
- **成功率是公开的。**treg 汇总所有调用者在每个端点上观测到的成功率和延迟，并展示在你的价格旁边。可靠性就是你的排名。
