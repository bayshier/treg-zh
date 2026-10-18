# 参与 tools-registry 贡献

感谢你的关注！

## 环境搭建

```bash
git clone https://github.com/superdesigndev/treg
cd treg
bash scripts/build-dashboard.sh # Node 22.12+ and npm; compile the browser app
uv sync                     # install deps (uv >= 0.12, pinned in pyproject - https://docs.astral.sh/uv/)
uv run --with pytest-xdist pytest -n auto -q   # daily local default (same shape as CI)
```

开发时不需要 `.env` — 每个设置都有可用的默认值（临时加密 key、本地
sqlite）。用于持久化 / 真实部署的 `TREG_*` 配置项在 README 的 **配置**
一节（以及 `docs/context/ops/deploy.md`）中有说明。

一条命令即可启动的本地技术栈在 `scripts/dev-local.sh`（`up` / `logs` / `cli` / `reset`）。
它会在 :18790 启动 Python、在 :5173 启动 Vite；打开 `http://localhost:18790/app`。
`TREG_DEV_DB=/absolute/path/to/dev.db` 可以选用一个独立的本地数据库，而不会重置另一个。

仪表盘源码在 `frontend/src/`：`.vue` 页面、组件和对话框，功能用例
在 `state/`，类型化 JSON 客户端在 `api.ts`。`frontend/index.html` 只是文档入口。
运行 `npm --prefix frontend test` 做传输层测试，运行 `npm --prefix frontend run test:e2e`
针对一次性 SQLite 服务器做浏览器流程测试。先用
`cd frontend && npx playwright install chromium` 安装 Chromium。

在 `uv build` 之前，先运行 `bash scripts/build-dashboard.sh`。wheel 和 sdist 会包含构建出的
资源；安装已发布的包不需要 Node 运行时。可编辑（editable）的 Python 安装不
需要前端构建，因此 CLI 和后台 worker 的开发保持独立。

## 项目结构

- `src/treg/` — 应用本体（FastAPI API、代理、CLI、运行器）。`docs/context/` 中的**设计文档**
  解释每个子系统并标注其覆盖的源码文件 — 改代码前先读对应片段。
- `tests/` — 测试套件（pytest）。新行为必须有测试。
- `docs/context/` — 按子系统拆分的设计片段（「它如何工作」的唯一事实来源）。

## 进行改动

1. 从 `main` 分出分支。
2. 与周边代码风格保持一致 — 带类型注解的朴素 Python，没有充分理由不引入新依赖。
   提交信息遵循 Conventional Commits（`feat(scope): …`、`fix: …`、`docs: …`）。
3. 添加或更新测试；运行 `uv run --with pytest-xdist pytest -n auto -q`（全部通过）。
   串行模式 `uv run --frozen python -m pytest -q` 用于调试单个测试或执行顺序。
   针对 Postgres 测试时，设置 `TREG_TEST_DB_URL`；每个 xdist worker 会依据它创建自己的数据库。
   测试必须以任意顺序通过。要验证这一点，可用几个种子打乱顺序（pytest-randomly 不是
   项目依赖，因此 CI 的顺序保持固定）：
   `uv run --with pytest-xdist --with pytest-randomly pytest -n auto -q -p randomly --randomly-seed=<n>`。
   测试改动的任何东西都要恢复：`monkeypatch.chdir`、`monkeypatch.setenv`、`tmp_path`，
   仓库文件用 `Path(__file__).parents[1]` 解析，而不是用工作目录。
4. 如果改动了某个子系统，请在同一个 PR 中更新它在 `docs/context/` 里的片段。
5. 发起 PR。CI 会运行测试 + secret 扫描；由维护者评审。

## 可以做些什么

路线图在 [`README.md`](README.md) 末尾（MCP 支持、更细的权限层级、
密钥管理加固）。bug 报告和小修复随时欢迎；较大的功能
请先开一个 issue，让我们在你投入之前先就方案达成一致。

## 报告 bug / 安全问题

一般 bug → 提 GitHub issue。**安全漏洞 → 参见 [SECURITY.md](SECURITY.md)**（私下
报告，切勿开公开 issue）。

## 行为准则

保持友善和建设性。不容忍骚扰或人身攻击；越线的评论或贡献者可能会被维护者移除。
