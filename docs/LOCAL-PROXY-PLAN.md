# 本地代理——在 agent 不知情的情况下截获它的调用（`treg serve`）

**状态：**已交付（P0–P5 + `treg serve`，分支 `feat/local-proxy`，2026-07-30）· **依赖：**一个小的服务端改动，即 `X-Treg-Error` 标记 · **v1 范围：**本地捕获，服务端注入。

## 它解决的问题

目前，只有当 agent 被明确告知使用 treg 时它才生效——通过 `treg call …`，或 `/call/` URL。一旦 agent 自己写了一个直接调用 `api.stripe.com` 的脚本，treg 就不可见，脚本也没有密钥可用。

本地代理填补这个空缺：agent 发起一次普通调用，我们在出口处截住它，由**服务端**添加凭据。agent 侧什么都不用改。

## 从 oneCLI 借鉴什么，不借鉴什么

他们的设计是可分离的两半，我们只要其中一半。

| 部分 | oneCLI | treg |
|---|---|---|
| 截获请求 | `HTTPS_PROXY` + 自有证书颁发机构 | **相同** |
| 应用凭据 | 在本地、于用户机器上解密真实密钥 | **在服务端**——密钥永不落地 |

他们的网关持有 `SECRET_ENCRYPTION_KEY`，并在笔记本上解密 secret（`crypto.rs`、`connect.rs`）。这正是 treg 要防止的局面，所以我们保留捕获机制，丢弃注入那一半。

**v1 只做服务端注入。**未来的「在 `treg-run` 下本地注入」模式是可能的——授权路径、run-proof 门禁和自有/共享区分规则都已存在——但它被刻意排除在范围之外。以后再决定毫无成本；现在做错，则会葬送产品的核心承诺。

## 工作原理

```
agent → api.stripe.com/v1/charges
   ↓  HTTPS_PROXY=127.0.0.1:<port>
local proxy   — is this host a registered tool?
   ├─ no  → blind tunnel, bytes untouched, we never see inside
   └─ yes → terminate TLS with a leaf cert we sign, read the request, re-address it
              ↓
        treg server  POST /call/https://api.stripe.com/v1/charges   (X-Treg-Token)
              ↓  server injects the credential — the existing relay
        Stripe → response streams all the way back to the agent
```

代理只携带成员的 **treg token**，别无其他。服务商密钥永远不会落到这台机器上。

由于调用落在常规的 `/call/` 路径上，已建成的一切原样适用：按成员的工具 ACL、项目范围、拒绝规则、每日上限、审计记录、OAuth 自动刷新。不需要再写第二个策略引擎——这正是 oneCLI 需要 2.4 万行 Rust 而我们不需要的原因。

## 硬性约束：CLI 必须保持轻量

`pip install tools-registry` 安装的是**纯 CLI**（`httpx` + `questionary`）。证书生成需要 `cryptography`，它是一个编译型依赖，目前放在 `[server]` extra 里。它绝不能进入基础安装。

**决策：新增第三个 extra——`[proxy]`，只含 `cryptography`。**代理本身使用标准库的 `asyncio` + `ssl` 和 `httpx`（已在基础依赖中）。缺少该 extra 时，代理会抛出 `ProxyDependencyError` 并给出确切的安装命令。`[server]` 已包含 `cryptography`，所以自托管用户天然具备。**已交付**——见 `pyproject.toml` 中的 `[project.optional-dependencies].proxy`。

## 组件

| 组件 | 位置 | 说明 |
|---|---|---|
| 代理服务器 | `src/treg/localproxy.py`（新增） | 得名于 `localrun.py`；**不是** `proxy.py`，后者是服务端中继 |
| CA + 叶子证书 | 同一模块 | ECDSA P-256，叶子证书按主机缓存在内存中 |
| CLI 标志 | `cli.py` 中的 `cmd_shell_start` | `treg shell start --proxy [--proxy-port] [--renew-ca]` |
| 守护进程 | `cli.py` 中的 `cmd_serve_*` | `treg serve start\|stop\|status\|env` — `eval "$(treg serve env)"` |
| Shell 接线 | `shell.py` | `start_session(extra_env=…, on_close=…)` 发布环境变量并停止代理 |
| 状态 | `~/.treg/proxy/` | `ca-key.pem` 0600 · `ca-cert.pem` 0644 · `ca-bundle.pem` 0644 · `proxy.json` 0600（**仅守护进程**——端口 + 代理 token，供另一个终端查找；`--proxy` 不写其中任何文件）· `serve.log`。若设置了 `TREG_CONFIG` 则遵循该变量 |

### 我们设置的环境

```
HTTPS_PROXY / HTTP_PROXY = http://treg:<local-token>@127.0.0.1:<port>
NODE_USE_ENV_PROXY       = 1      # Node's built-in fetch IGNORES HTTPS_PROXY without this
NODE_EXTRA_CA_CERTS      = ~/.treg/proxy/ca-bundle.pem
SSL_CERT_FILE            = "
REQUESTS_CA_BUNDLE       = "
CURL_CA_BUNDLE           = "
GIT_SSL_CAINFO           = "
DENO_CERT                = "
AWS_CA_BUNDLE            = "
NO_PROXY                 = localhost,127.0.0.1,<treg host>
```

`NODE_USE_ENV_PROXY` 是不加就会让你多调试一天的那个：自 Node 18 起，内置 `fetch` 会静默忽略代理环境变量并直接出网。oneCLI 的集成文档里也带着同一个开关。

这个 bundle 是**系统 CA + 我们的 CA**，因此 agent 依然信任真实的互联网。绝不要替换系统根证书；只能在其后追加。

## 不可妥协的原则

1. **CA 私钥按机器生成。**绝不随包分发、绝不共享、绝不提交。把同一个 CA 分发给所有用户，等于让任何人都能向每个用户冒充任何网站。
2. **v1 绝不触碰系统信任存储。**通过环境变量限定作用范围，意味着只有 agent 的进程树信任我们——浏览器不信任，操作系统也不信任。oneCLI 也是这么做的；他们的仓库里找不到任何 `add-trusted-cert`。
3. **只走白名单。**只有当主机是已注册的工具时才截获。其余一切都是我们无法读取的盲隧道。这也保证了我们不会破坏（或读取）agent 自身对 `api.anthropic.com` / `api.openai.com` 的调用。
4. **只绑定 `127.0.0.1`**——绝不绑定 `0.0.0.0`。
5. **代理认证。**每个会话生成一个随机 token，放在代理 URL 中携带，这样本机的其他进程就无法悄悄借我们消耗该成员的配额。
6. **代理自己的 httpx 客户端要设 `trust_env=False`。**否则它会从自己的环境里读到 `HTTPS_PROXY` 并和自己对话——第一个请求就陷入无限循环。
7. **绝不记录请求体、header 或 token。**只记录主机和状态。

## 各阶段——每一阶段都可独立测试

**P0 · 骨架（不截获）。——已完成。**监听、认证，双向盲隧道转发一切。
*验证：*`curl -x http://treg:<token>@127.0.0.1:PORT https://example.com` 的表现与没有代理时完全一致（已验证：200，`ssl_verify_result=0`）；没有 token 则返回 407。

**P1 · 证书。——已完成。**生成并持久化 CA、构建 bundle、签署并缓存叶子证书（`ensure_ca`、`build_bundle`、`CertAuthority.leaf_pem` / `context_for`），外加 `proxy_env()`——即 P4 将要发布的那份环境变量。
*验证：*单元测试签署一张叶子证书并与之完成真实的 TLS 握手；`curl --cacert
ca-bundle.pem` 接受它，而只带系统根证书的 curl 拒绝它。

**P2 · 截获并转发。——已完成。**对白名单内的主机：终止 TLS（`_intercept`）、读取请求、重新寻址到 `{base}/call/https://{host}{path}`（`_treg_url`）、把调用方的 transport 和 `x-treg-*` header 换成我们自己的（`_forward_headers`），并把响应流式回传（`_write_response`）。尊重 keep-alive，多个调用可以共享一次握手。
*验证：*`test_the_agent_calls_the_vendor_and_treg_injects_the_key` 让一个普通的 `httpx.Client`——对 treg 毫无感知、也没有密钥——穿过代理打到**真实的** FastAPI 应用上；凭据到达了上游，因为是服务端注入的。

**P3 · 白名单 + 错误。——已完成。**启动时从 `GET /tools` 拉取主机列表，并定时刷新——`ProxyConfig.hosts` 是接缝，目前由调用方填充。把 treg 的失败映射为可读的信息：`_resolve_call` 返回 404 意味着「该主机或路径没有注册工具」；403 意味着「你没有访问权限」。（treg 不可达的情况已经处理——返回一个指明 treg 问题的 502，并说明服务商调用**没有**发出，这样 agent 就不会怪罪一个健康的服务商并盲目重试。）

**P4 · 接线 UX。——已完成。**`treg shell start` 生成一个 token、启动代理，并把 `handle.env(treg_host)` 合并进子 shell 环境（`start_session` 已经构建该字典），在退出时停止代理。横幅会说明哪些主机被截获。没有独立命令——见决策 1。

**P5 · 文档。——已完成。**`docs/context/architecture/local-proxy.md` fragment（新子系统，所以需要一份），外加 `/llms.txt` 和控制台的 agent 说明。

## 测试

- **单元测试，无网络：**CA + 叶子证书生成、白名单匹配、URL 重写、header 过滤、`NO_PROXY` 解析。
- **集成测试：**让代理对现有的 ASGI 测试应用运行；用配置了代理并信任 bundle 的 `httpx` 驱动它。断言落到 `/call/` 上的请求逐字节保真。
- **循环防护：**一条显式测试，验证代理自己的客户端忽略 `HTTPS_PROXY`。
- **盲隧道：**断言未注册的主机永远不会被解密（不会为它生成叶子证书）。

## 已知限制——要在文档里明说

- **证书固定（certificate pinning）。**只接受自己那把精确证书的工具会拒绝连接。需要一个按主机的「永不截获」清单。
- **远程 MCP server 不在覆盖范围内。**本地（stdio）MCP server 会继承我们的环境变量并被截获；托管型的则是在别人的机器上发起调用。
- **非 HTTPS 的流量不覆盖：**SSH、数据库线路协议。WebSocket 应当走隧道而非截获——中继是请求/响应式的。
- **延迟 + 硬依赖。**每一次被截获的调用都要经 treg 走一个来回；treg 不可用时这些调用随之失败。这就是密钥永不落地的代价。
- **大体积上传会流经我们。**留意 Render 上的请求超时。
- **部分服务商 CLI 仍然需要 `treg run`。**`gh` 在发起任何网络调用之前就会拒绝运行，所以根本没有可截获的东西。代理扩大了覆盖面；它并不取代 `treg run`。

## 刻意排除在 v1 之外

在 `treg-run` 下本地注入 · 安装到系统信任存储 · Windows · WebSocket 截获 · 按 agent 的代理 token（每个会话一个已足够）· 路由之外的请求/响应检查。

## 决策（2026-07-30 与 Unclecode 敲定）

1. **命令形态——先做 `treg shell start --proxy`，之后补上 `treg serve`。**最初的决策是只做 shell 模式，因为这样代理随会话启动和消亡，其 token 永远不需要写盘。实地测试后，Unclecode 提出还要守护进程，于是两者都交付了：`--proxy` 用于子 shell，`treg serve start|stop|status|env` 用于后台服务，其他终端用 `eval "$(treg serve env)"` 指向它。守护进程的代价恰恰是第一个决策想避免的东西——一个存有端口和代理 token 的 `proxy.json`（0600）——USAGE 里写明了这一点，让使用者知情后再做选择。`_start_proxy_handle` 是唯一的共享代码路径。
2. **CA 有效期——2 年**，在最后 30 天内（`_RENEW_WITHIN_DAYS`）自动重新生成，另支持显式续期。不是 oneCLI 的 10 年：一把能冒充任何网站的钥匙在笔记本上躺十年，太久了。
3. **默认端口——18791**，紧挨开发服务器的 18790。
4. **`[proxy]` extra——要**，只含 `cryptography`。把它并入基础依赖的方案被否决：它是编译型依赖，而 `pip install tools-registry` 必须保持为轻量 CLI。

## 构建日志

- **P0 + P1 交付**（2026-07-30）——`src/treg/localproxy.py`、`tests/test_localproxy.py`（34 个测试）。用真实的 `curl` 验证：`https://example.com` 和 `https://api.github.com` 原样穿过隧道，证书验证完好；CA 签发的叶子证书在指定 `ca-bundle.pem` 时被 curl/OpenSSL 接受，而在只用系统根证书时被**拒绝**——这就是不可妥协原则 #2 成立的证明。
- `_is_self` 循环防护和纯 `http://` 转发路径（在上游看到之前剥掉 `Proxy-Authorization`）是 P0 草图之外补充的——`HTTP_PROXY` 与 `HTTPS_PROXY` 一同设置，http 调用方不能就此坏掉。
- **P2 交付**（2026-07-30）——新增 21 个测试，文件内共 55 个。超出草图的部分：
  - **边缘 WAF 重试。**Render 的边缘节点会对看起来像注入的请求体返回 403。CLI 的 `_RegistryClient` 已经会把这类请求体以 base64 编码、带上 `X-Treg-Body-Encoding` 重发；被截获的调用也必须这样做，否则一条合法的 SQL 查询会无声无息地失败。最大 1 MiB 的请求体会被缓冲，重试因此才成为可能；更大的请求体流式传输，无法重试。
  - **`_CALLER_MUST_NOT_SET`。**自行设置 `X-Treg-Token` 的 agent 不能借我们的代理消耗另一位成员的配额。调用方的 `x-treg-*` header 会被丢弃；能到达 treg 的身份只有代理自己的。
  - **`intercepts()` 在缺少凭据时拒绝工作。**没有 CA 或没有 treg token 时，我们会在终止 TLS 之后无法完成调用——这比不截获更糟，因为 agent 自己的请求会因一个它看不见的原因而失败。
  - **`_body_chunks`。**当响应已被 transport 加载过时，`aiter_raw()` 会抛出 `StreamConsumed`，导致响应体被无声丢弃。两种形态都已处理。
  - **分帧是重新推导的，而不是照抄**——treg 给了 `Content-Length` 就用它，没给就用 chunked——因为我们这一跳必须自洽，即使进入 treg 的那一跳分帧方式不同。

- **`treg serve` 交付**（2026-07-30，实地测试之后）——`cmd_serve_start/_stop/_status/_env` + `localproxy.py` 中的状态辅助函数（`write_state`、`read_state`、`running`、`pid_alive`）。测试发现了两个真实问题：
  - `pid_alive(0)` 曾返回 **True**，因为 `os.kill(0, …)` 会给调用者所在的整个进程组发信号。状态文件里一个被截断的 pid 本可能被当作「正在运行」，而 `treg serve stop` 本可能把信号发给用户的终端。现在非正数 pid 会在调用前被拒绝。
  - 分离的子进程死掉时，父进程原本只会说「去看日志」。现在它会打印子进程的最后一行日志——端口冲突这类问题因此能一句话读懂，而不是一场寻宝游戏。
- 端到端实地验证：`serve start` → `eval "$(treg serve env)"` → 对被截获主机的普通 `curl` 到达服务商时已由服务端注入 `Authorization: Bearer …`；未截获的主机直接出网；`--unset` 清除了 shell 环境；第二次 `start` 被拒绝；`stop` 移除了状态文件；带死 pid 的过期文件在下次调用时被清理。

- **Agent hook 交付**（2026-07-30）——`treg serve hook [--install] [--agent]`。Unclecode 问 oneCLI 是如何避开 eval 的；读他们的代码发现它并没有避开：`onecli run` 是父进程，容器通过 `-e HTTPS_PROXY=…` 拿到变量，而他们的 Claude 插件把 **`BASH_ENV`** 设为一个生成的 `env.sh`（`plugins/claude/hooks/session-start.mjs:236`）。我们现在采用第三种，复用 `agents.py` 里的 harness 注册表。用最朴素的 `BASH_ENV=… bash -c 'curl …'` 验证过：代理运行时带凭据，`serve stop` 之后回归普通。
- **hook 当天就被 `treg <command>` 取代。**Unclecode 否决了全局安装器，他是对的：把 `BASH_ENV` 写进 `~/.claude/settings.json` 会捕获这台机器上该 agent 的每一个会话，永久生效，不管你那天想不想用 treg——而且不留一条轻松改用自己的个人密钥的退路。启动器（`cmd_with`）给出了同样「永远不用 eval」的结果，却没有那种波及面：treg 只是它所运行那一条命令的父进程。`treg claude` 使用团队访问权限；裸的 `claude` 不受影响。任何配置文件都不写入。`treg serve hook`、`env.sh` 和三个配置安装器已删除——不要再加回来。
- **下一步，已提出但未实现：**`treg claude` 还可以把团队的 skill 预加载进会话，而不是让 agent 自己下载并安装。
- **实测后修正了计划：**`NODE_USE_ENV_PROXY` 从 **Node 24** 才存在。在开发机（Node 23.11）上，`node --use-env-proxy` 报「bad option」，穿过代理的普通 `fetch()` 到达时没有凭据。对 Node 24+ 设置这个变量仍然是对的，但计划里关于它的那一行读起来像「Node 现在就能用」，而这只在 24 上成立。已在 fragment、USAGE 和 /llms.txt 中说明。
- **`examples/proxy-demo/`**——一个零依赖的 Node 应用，带一个小 Web UI：点一个按钮，它会调用 `api.openai.com`（已注册）或 `example.com`（未注册）。`node server.js` 得到 401；`treg node server.js` 得到真实答案，代码相同。由于上面提到的 Node 注意事项，它手工实现了 CONNECT；其 README 解释了为什么真实的应用不需要这么做。
