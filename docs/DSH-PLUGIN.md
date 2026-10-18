# DeepSeek Harness bundle

[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（`dsh`）是一个把「一切皆插件」作为立身之本的 coding agent。它是同一个 skill 的第四个橱窗，也是唯一一个能在单次零配置安装中同时交付连接器**和** CLI 路径的渠道。

## 安装

```bash
dsh plugin --profile <name> add github:superdesigndev/treg
dsh --profile <name>
```

## 为什么没有 manifest

dsh 不读取任何 `plugin.json`。它安装的是一个 **`package.json` 中声明了 `dsh.bundle` 的 npm 包**，指向一个 `cordis.patch.yml` 配置层，并把该 bundle（捆绑包）追加到 profile 的 `dsh.profile.bundles` 列表中。各层按列表顺序叠加，因此用户可以在自己的 profile patch 中覆盖我们提供的任何一行，而无需改动这个包（[publish.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.md)）。

```
package.json                  the bundle manifest — `dsh.bundle.patch`
dsh/cordis.patch.yml          the layer: the skill row + the MCP row
dsh/index.js                  a ctx.skills provider over the tree below
dsh/skills/treg/SKILL.md      GENERATED — never edit by hand
```

## 在有 token 之前，MCP 行保持禁用

Claude Code 插件完全不声明任何连接器，因为 manifest 里声明的任何东西都会在安装时注册，而那时还没有任何人登录：结果是五个常驻工具在每次调用时都返回 401（[CLAUDE-PLUGIN.md](CLAUDE-PLUGIN.md)）。dsh 能以更好的方式表达这一取舍，因为每一行都可以携带一个在启动时求值的 `disabled` 表达式：

```yaml
- id: treg-mcp
  name: '@deepseek-ai/dsh-mcp-client'
  disabled: !!js !process.env.TREG_TOKEN
```

因此，环境里设置了 `TREG_TOKEN` 的用户在启动时就能得到 `mcp__treg__catalog_search` 等工具，其他用户则得到一次干净的安装和一个引导他们完成 `treg login` 的 skill。这一行是**在启动时**求值的，所以之后再导出 token 需要重启 `dsh`——bootstrap 明确说明了这一点，因为让人干等本会话永远不会出现的工具，正是要避免的失败模式。

`treg mcp install` 被刻意排除在这条路径之外：它写入的是 Claude Code、Cursor 和 opencode 的配置（`mcp_install.py`），而 dsh profile 哪个都不是。`treg mcp install` 在检测到 `~/.dsh` 时会打印相应的提示，dsh 的 bootstrap 也会引导用户避开它。

## `dsh/index.js` 的两个约束

**不能有构建步骤。**`dsh plugin add github:...` 拉取的是源码，而不是构建产物。TypeScript 包需要一个 `prepare` 脚本，而 pnpm ≥10 在用户于自己 profile 的 `pnpm-workspace.yaml` 中添加 `allowBuilds` 条目之前，拒绝为 git 依赖运行这类脚本——也就是说，文档中的安装命令对所有人来说首次运行都会失败。手写的纯 ESM 则完全绕开了这个问题。

**不能依赖 `@deepseek-ai/*`。**最直接的实现是用 `customSkillDirs` 挂载内置的 `@deepseek-ai/dsh-skill-filesystem`，但一个树外 bundle 依赖内置包时会安装出*第二份*副本，并逐渐与宿主的副本产生偏差。provider 的 `list`/`get` 契约才是稳定的接缝，因此 `index.js` 直接实现该契约——这也是 `dsh-skill-badge` 为其自带 skill 采用的形态。`tests/test_plugin.py` 固定了这两个约束。

skill 的 `description` 在运行时从 `SKILL.md` 中解析出来，而不是在 JS 里重复声明，因此不存在会腐化的第二份副本。

## 发现渠道

插件通过 [`dsh-plugin`](https://github.com/topics/dsh-plugin) 这个 GitHub topic 被发现，生态目录（awesome-dsh-plugin、dsh.tools）正是从这里同步的。本仓库已设置该 topic；向那些目录提交上架 PR 则是人工的那一半。
