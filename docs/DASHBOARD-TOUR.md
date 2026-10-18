# tools-registry——控制台导览

你在 Web UI 里能做的所有事情，每一步配一张截图。交互版（Ledger 风格，带上一页/下一页和主题切换）托管在 `https://treg.to/dashboard-tour/`（Help → Dashboard tour）；CLI 走查见 [`docs/TUTORIAL.md`](TUTORIAL.md) / `https://treg.to/tutorial`。

截图由 `docs/dash-tour/capture.py`（Playwright）生成——UI 有改动后请重新运行。

---

## 登录

### 1 · 三种进入方式
登录界面提供三个独立的入口——任选**其一**：**Continue with GitHub**（真实 OAuth）、**Email me a sign-in code**（对任意邮箱有效），或可折叠的**粘贴组织 token**（面向 agent 与 CLI）。

![](../src/treg/web/tour/img/01-signin.webp)

> **注意：**本导览使用邮箱验证码入口。邮件服务器尚未接入，因此在**开发模式下验证码直接显示在页面上**，而不是通过邮件发送。

### 2 · 邮箱验证码
输入邮箱并点击 **Email me a sign-in code**——页面会显示 `dev code 619565` 和一个对应的输入框。填入这 6 位数字并点击 **Sign in**。首次登录还会同时为你完成注册（并创建一个个人组织）。

![](../src/treg/web/tour/img/02-email-code.webp)

### 3 · 你的控制台
你会落在自己**当前激活的组织**上。新账户只有一个个人组织，没有任何工具。顶栏有组织切换器、主题切换、头像和 Sign out；左侧导航有 Tools、Organizations、Activity、Tutorial。

![](../src/treg/web/tour/img/03-dashboard-empty.webp)

---

## 搭建团队

### 4 · 创建团队
点击 **+ New team** 并为它命名——你会成为它的 **owner**，它也随之变为激活状态。团队就是你共享工具的地方。

![](../src/treg/web/tour/img/04-create-team.webp)

> **注意：**个人组织在登录时自动创建；团队始终需要显式创建。

### 5 · 你就是 owner
在 **Organizations** 页面，你激活的团队会出现一个 **Manage** 面板：成员、邀请和一个危险操作区。

![](../src/treg/web/tour/img/05-manage-panel.webp)

---

## 邀请成员加入

### 6 · 邀请队友
输入邮箱，选择角色（viewer / member / admin），点击 **Invite**。treg 会生成一个**一次性代码**并内联显示；对方也可以不用代码直接接受。下方的 **Pending invites** 列出尚未处理的邀请（可 Revoke）。

![](../src/treg/web/tour/img/06-invite.webp)

### 7 · 通过横幅接受
被邀请人登录时，邀请会以**横幅**形式出现——点击一下 **Accept** 即可加入，无需复制代码。

![](../src/treg/web/tour/img/08-invite-banner.webp)

> **注意：**邀请绑定的是邮箱，所以只要证明拥有该邮箱（从任意入口）就足够了——代码只是一个带外快捷方式。

### 8 · ……或凭代码加入
通过其他渠道拿到了代码？打开 **⤷ Join by code** 并粘贴（必须与你的邮箱匹配）。

![](../src/treg/web/tour/img/17-join-by-code.webp)

---

## 注册资源

### 9 · 添加 secret
**member 及以上**角色可以注册。打开 **⚿ Secrets** 并添加一个凭据（名称、值、类型）。值会在服务端加密，之后永远不会再显示。

![](../src/treg/web/tour/img/09-secrets.webp)

### 10 · 注册工具
**+ Add tool**：一个工具等于一个上游 **base URL** 加一个 **binding（绑定）**——即使用哪个 secret、放在哪里（header/query）、字段名，以及形如 `Bearer {secret}` 的格式。

![](../src/treg/web/tour/img/10-add-tool.webp)

> **注意：**`{secret}` 会在调用时被替换成真实凭据——调用方从头到尾都不持有它。

### 11 · 多凭据工具
点击 **+ binding** 可以再加一行——每次调用会应用所有绑定（例如一个 OAuth bearer *加上*一个 developer-token header）。

![](../src/treg/web/tour/img/11-multi-binding.webp)

### 12 · 编辑工具
**✎** 按钮会重新打开预填好的构建器（base URL + 绑定）→ 保存时发送 PATCH。名称保持固定，不可修改；删除是 **✕**（内联确认）。仍被某个工具绑定的 secret 受保护，无法删除。

![](../src/treg/web/tour/img/12-edit-tool.webp)

### 13 · 注册 skill（捆绑包）
一个 **skill** = 一份配方 + 它的 secret + 它的一个或多个工具，以原子方式注册。点击 **+ Skill** → 粘贴 payload（secret 值直接内联；CLI 的 `treg.json` 则改为从文件加载）。binding 通过 `local_name` 引用 secret；JSON 会先经过校验。

![](../src/treg/web/tour/img/13-skill.webp)

### 14 · 你的工具
已注册的工具以卡片呈现——主机、注入器、所有者、健康状态徽标。每张卡片可以：复制片段（Copy）、Try it、编辑或删除。

![](../src/treg/web/tour/img/14-tools-list.webp)

---

## 使用工具

### 15 · Try it——密钥由服务端注入
**Try it** → 输入上游路径 → Send。调用经由代理完成，凭据由服务端注入；响应里会出现 `authorization: Bearer …`，而你从未提供过任何密钥。

![](../src/treg/web/tour/img/15-try-it.webp)

> **注意：**这就是整个产品的核心——通过 treg 调用真实 API，secret 由服务端添加。即便是 **viewer** 也能这么做。

### 16 · 复制代码片段
**Copy for…** 会给出针对 Claude Code、CLI、Python、Node 或 cURL 的现成片段——指向代理并携带你的 token，不内联任何密钥。

![](../src/treg/web/tour/img/16-copy.webp)

---

## 角色

### 17 · viewer 角色
以 **viewer** 身份登录后，你能看到工具并可以 **Copy** / **Try it**——但注册相关的控件（Secrets、+ Skill、+ Add tool）和 Manage 面板都不见了。这是只使用、不改动凭据的角色。

![](../src/treg/web/tour/img/18-viewer-tools.webp)

> **注意：**角色从高到低是 owner > admin > member > viewer。UI 会隐藏你的角色做不到的事；服务端也会强制执行同样的限制。

### 18 · 管理团队
作为 owner，Manage 面板列出所有成员：通过下拉框更改角色（仅限 owner）、**Remove** 成员（admin 及以上），以及一个**危险操作区**，内含 Leave（带最后一位 owner 保护）和 Delete（输入确认）。

![](../src/treg/web/tour/img/20-danger-zone.webp)

---

## 超级管理员

### 19 · 平台管控
超级管理员会得到一个 **Admin** 导航，拥有跨租户的操作范围：平台统计、所有组织（暂停 / 删除），以及所有用户（授予/撤销超级管理员、暂停、删除）。你自己的那一行有防自锁保护。

![](../src/treg/web/tour/img/22-admin-users.webp)

> **注意：**Admin 导航只对超级管理员可见。

---

## 收尾

### 20 · Activity——审计日志
**Activity** 列出组织内的每一次代理调用——谁、调了什么、什么时候、状态如何。即使有人用的是队友的密钥，账本里记下的仍是真实的调用者。

![](../src/treg/web/tour/img/23-activity.webp)

### 21 · 应用内教程
在 **Help → Tutorial** 下，控制台还内置了完整的交互式 CLI 走查——终端与浏览器操作汇集在同一处文档中。

![](../src/treg/web/tour/img/24-help-tutorial.webp)

> 这就是整个 UI：登录 → 搭建团队 → 邀请 → 注册 → 调用 → 管理 → 清理收尾。 🏁
