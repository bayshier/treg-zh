---
name: web-access
description: 只要任务需要从实时网络获取内容就使用本 skill——搜索网页或新闻、读取 URL、映射或爬取网站、从页面提取结构化字段，或从平台获取数据（X、LinkedIn、Reddit、TikTok、Instagram、YouTube、小红书、Amazon、应用商店、Google Maps、学术论文）。它会为任务挑选合适的工具，优先走免费接口，某家服务商失败时让 treg 自动回退到下一家。不需要本地爬虫、浏览器或 cookie。
---

# 网页访问

一个 treg 密钥就能读取整个网络：免费的搜索与页面抓取打头阵，工具目录的其余端点在其后兜底，覆盖免费抓取够不到的页面和平台。本机不运行任何东西：不需要浏览器、cookie，也不需要一次性小号。treg 替你调用服务商；在路由任务中，当某家服务商返回空结果或被拦截时，会接着尝试下一家。

依赖 treg 运行。如果尚未设置（`treg --version` 失败），请阅读 https://treg.to/llms.txt 并按其中的指引完成安装和登录，然后再回到本文件。

## 判断任务

```
Know the URL?             → read it (free first)
Need to find pages?       → search (free) → read the 3–5 best results
A platform object?        → never scrape its HTML: use the platform's job
  (post, profile, video,     structured fields, fewer tokens, survives redesigns
   product, review, place)
A whole site?             → map it → read only the pages that matter
Need typed fields?        → structured extract with a JSON schema or an instruction
Need it recent or dated?  → news search, not web search
```

## 免费优先

网页搜索和页面读取通过 TinyFish 免费：

- `treg call treg.web.search --data '{"q":"…"}' --header 'X-Treg-Route-Prefer: tinyfish'`
- `treg call treg.web.extract --data '{"url":"…"}' --header 'X-Treg-Route-Prefer: tinyfish'`

两者都是路由端点：当 TinyFish 返回空结果或被拦截时，treg 会尝试下一家服务商（Crawl4AI、AnyAPI 等）。你只需为实际应答的服务商付费，外加它之前任何对未命中也计费的服务商（`treg catalog get` 会说明有多少家这样计费）；`X-Treg-Route-Max-Cost` 可为这条回退链设置上限。要在一次免费调用中读取最多十个 URL，可直接调用 `tinyfish.web.fetch` 并传入 `{"urls":[…]}`；该端点不会回退。

平台数据（社交、电商、评论、地图）并不免费：每次调用收费几美分甚至不足一美分。批量调用前请先向用户说明。

## 任务类型

在工具目录中搜索第二列里的词语。示例服务商只是当前由谁应答的举例；请以工具目录当前排名最高的为准。

| 需求 | 在工具目录中搜索 | 示例服务商 | 备注 |
| --- | --- | --- | --- |
| 网页搜索结果 | `treg.web.search` | TinyFish, Exa, Perplexity, Keenable | 加 TinyFish 请求头即免费 |
| 以 Markdown 读取页面 | `treg.web.extract` | TinyFish, Crawl4AI, AnyAPI | 免费优先；被拦截的页面自动回退 |
| 从页面提取类型化字段 | `treg.web.extract.structured` | | 发送 `url`，并附 `schema` 和/或 `instruction` |
| 网站上的所有 URL | `treg.web.map` | AnyAPI, Firecrawl, Tavily | 添加 `q` 可按主题对 URL 排序 |
| 网站的多个页面 | "crawl a website" | Firecrawl, Crawl4AI, Olostep | 只需要部分页面时先做映射 |
| 近期新闻 | `treg.web.search.news` | Serper, DataForSEO, Exa | 结果带日期 |
| 学术论文 | "search research publications" | TinyFish, Exa, Serper | |
| X、LinkedIn、Reddit、TikTok、Instagram、Threads、Facebook | "x search posts", "linkedin company posts", "reddit subreddit posts", "tiktok video comments", … | TikHub, ScrapeCreators, HarvestAPI, AnyAPI | 资料、帖子、评论、互动以字段形式返回 |
| YouTube、Bilibili | "youtube video comments", "youtube search videos" | Crawl4AI, TikHub, ScrapeCreators | |
| 小红书、抖音、微博 | "xiaohongshu user notes", "douyin search videos" | TikHub, JustOneAPI | |
| 电商平台 | "amazon product detail", "amazon search products", "walmart search", "ebay search" | JustOneAPI, SerpAPI, Crawl4AI | 价格、评分、评论以字段形式返回 |
| 评论 | "app store reviews", "google play app", "trustpilot reviews", "glassdoor reviews", "google maps reviews" | Crawl4AI, Bright Data, LiteScrape | |
| 地点 | "google maps search", "local results" | DataForSEO, SerpAPI | |
| 与 Google 展示一致的搜索结果 | "google serp organic" | DataForSEO, Serper | 排名、摘要、特色结果 |

## 查找端点

工具目录每周都在变化，所以绝不要依赖记忆中的端点 id。

1. `treg catalog search "<用平实语言描述的任务>"`，而不是搜索服务商名称。
2. 有路由型 `treg.*` 端点时优先使用：先尝试团队自己的密钥，再按顺序尝试各服务商，且大多数未命中是免费的。
3. `treg catalog get <id>`：查看价格、路由任务背后的各服务商，以及每家的实测**命中率**（真的找到了结果），而不仅是可用（返回 200），还有速度。
4. 先做一次小规模调用。查看 `_treg.served_by`、`_treg.tried` 和报告的扣费，然后再放量。
5. 都不合适：`treg catalog request "<缺少什么>"`。

## 控制路由任务

选项通过请求头传递，绝不放在请求体中：

| 请求头 | 用途 |
| --- | --- |
| `X-Treg-Route-Prefer: <vendor>` | 优先尝试该服务商（免费的 TinyFish，或上次成功的那家） |
| `X-Treg-Route-Exclude: <vendor>` | 跳过曾给出糟糕结果的服务商 |
| `X-Treg-Route-Max-Cost: <usd>` | 限制单次调用费用；不设上限时，一次未命中最多可能消耗 $1 的服务商调用 |
| `X-Treg-Route-Waterfall: 0` | 当只需要某一家服务商的答案时，在首次未命中即停止 |

批量调用：务必带上 `Max-Cost`。

## 省钱省 token 的习惯

- 以 Markdown 读取页面，并在导航栏和页脚进入上下文之前将其剔除。
- 平台任务返回的是字段；抓取同样内容的整页要多花数倍的 token。
- 先搜索，再只读取需要的结果。绝不要为回答一个问题而读完整个网站。
- 交付的每条事实旁都保留来源 URL。

## 配方

- **调研一家公司：**网页搜索 → 读取其官网、定价和文档页面 → 搜索最近 90 天的新闻 → 读取其 LinkedIn 公司主页。
- **把文档站读入上下文：**用 `q` 设定主题来映射站点 → 读取排名靠前的页面。
- **监视页面变化：**读取页面，保存文本的哈希，下次运行时再次读取并报告差异。用用户的 agent 进行调度（Claude Code 用 `/schedule` 或 `/loop`，其他 agent 用 cron）。
- **社交聆听：**搜索 X、Reddit 和 LinkedIn 上的帖子 → 读取其中最佳帖子的评论 → 按主题附带链接进行总结。
- **跨平台比较商品：**在各电商平台搜索该商品 → 对匹配结果取商品详情 → 汇总成一张包含价格、评分和评论数的表格。
- **收集评论：**应用用 App Store 和 Google Play，公司用 Trustpilot 和 Glassdoor，地点用 Google Maps → 引述评论中的规律，并附上数量。
- **视频转笔记：**YouTube 搜索 → 视频详情和评论 → 总结。

## 检查安装

```
treg --version && treg balance
treg call treg.web.search --data '{"q":"treg web access"}' --header 'X-Treg-Route-Prefer: tinyfish'
```

返回结果带有 `_treg.served_by` 且没有扣费，说明免费搜索工作正常。如有问题：`treg feedback "<发生了什么>"` 可以直接反馈给团队。

## 不可跨越的红线

只访问公开数据。绝不以用户身份登录、索要用户的 cookie 或绕过付费墙。不要批量收集私人个体的个人数据。当服务商拒绝返回某个页面时，使用回退或告知用户；不要反复轰炸它。
