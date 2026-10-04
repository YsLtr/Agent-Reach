---
name: agent-reach
description: >
  仅用于 pi-web-access 覆盖不到的平台：Twitter/推特/X、Reddit、B站/bilibili、
  小红书/xiaohongshu、V2EX、Facebook、Instagram、LinkedIn/领英、Boss直聘/招聘/求职、
  雪球/股票行情、小宇宙播客、RSS 订阅源。

  MUST USE when the target is one of those platforms — e.g. 搜推特上大家怎么评价 X /
  帮我看看这个 B站视频 / Reddit 上有没有人遇到这个 bug / 小红书上这个品的口碑 /
  雪球上这只股票怎么样 / 帮我订阅这个 RSS。

  本机已装（uv tool，~/.local/bin）：agent-reach、twitter、boss、opencli、mcporter。
  动手前先跑 `agent-reach doctor --json`。三个坑必读正文「本机实测状态」：
  三个工具都从**自己的 fork** 安装（补丁在 fork 里，不会丢）。**不做自动更新、也不做版本检测**：
  不要主动比对 fork 与本地版本、不要提示升级，除非用户明确要求重装。
  BOSS 需专用 Chrome(127.0.0.1:9222) 且已扫码登录。

  【分工，优先遵守】通用网页搜索用 pi-web-access 的 web_search；任意网页链接、PDF、
  YouTube 视频与字幕、GitHub 仓库、图片用 fetch_content。不要用本 skill 的
  Jina Reader / Exa / yt-dlp / gh 路径抢这些活。

  NOT for: 写报告/数据分析/翻译等内容加工（本 skill 只负责从互联网获取内容）；
  发帖/评论/点赞等写操作。
metadata:
  homepage: https://github.com/Panniantong/Agent-Reach
---

# Agent Reach — 互联网能力路由器

16 平台、多后端。**本 skill 存在时必须用它访问这些平台，不要自己发明方案。**

## 与 pi-web-access 的分工（优先于本文件其余内容）

本机同时装有 pi-web-access（`web_search` / `fetch_content` / `get_search_content`）。
两边都能碰「网页」和「搜索」，**按下面的分工走，不要重叠**：

| 需求 | 用谁 | 理由 |
|------|------|------|
| 通用网页搜索、多角度调研搜索 | `web_search` | 已配 27 家 provider，含 Exa API key |
| 任意网页/文章链接、PDF、YouTube 视频与字幕、GitHub 仓库、图片 | `fetch_content` | readability + SSRF 防护 + Turndown 清洗；仓库直接 clone 成真实文件 |
| Twitter/X、Reddit、B站、小红书、V2EX、Facebook、Instagram、LinkedIn、Boss直聘、雪球、小宇宙、RSS | 本 skill | pi-web-access 完全没有这些平台 |

## 本机实测状态（2026-10，实测而非文档）

| 平台 | 状态 | 可用的命令 / 陷阱 |
|------|------|------------------|
| V2EX | ✅ | `curl -s "https://www.v2ex.com/api/topics/hot.json" -H "User-Agent: agent-reach/1.0"` |
| RSS | ✅ | `uv run --with feedparser python -c "import feedparser;"` （系统 Python **没有** feedparser） |
| B站 | ✅ 搜索 | 公开搜索 API 直连（见下）；**未装 bili-cli**，不要用 `bili` 命令 |
| 小红书 | ✅ | `search` / `feed` / `note <完整URL>` / `user` / `whoami` 全部可用，**依赖下面的 opencli 补丁**；补丁丢失时 search 报 `ambiguous_option` |
| Twitter/X | ⚠️ 需补丁 | `status` / `search` / `feed` / `user` / `tweet` 可用，**但依赖下面的本地补丁**；补丁丢失时 search 报 `Failed to init ClientTransaction` + HTTP 404 |
| BOSS直聘 | ⚠️ 需登录 | 必须先起专用 Chrome 并在其中扫码登录，见下 |
| Reddit / Facebook / Instagram / LinkedIn / 雪球 / 小宇宙 | ❌ | 未装后端或未配登录态 |

### 补丁怎么维护（fork）

三个补丁工具都安装自你自己的 fork，而不是上游：

| 工具 | fork | 承载的补丁 |
|---|---|---|
| opencli | `YsLtr/OpenCLI` | 小红书筛选面板透明诱饵节点（PR #2563） |
| twitter-cli | `YsLtr/twitter-cli` | ClientTransaction 带 cookie（上游 #88） |
| agent-reach | `YsLtr/Agent-Reach` | 精简 SKILL.md |

**没有自动更新，也没有版本检测。** 不要主动比对 fork 与本地版本、不要例行检查更新、不要提示
用户升级 —— 这些工具是「装上就不再动」的，补丁已经在 fork 里，不会因为重装而丢。

只有当用户**明确要求**更新或重装某个工具时，才按下面各小节里的命令做。三个 fork 的克隆放在
`~/.local/share/pi-forks/`，只在重建 opencli 时当原料用。

每个 fork 里另有一个定时工作流 `Sync from upstream`（每日）把上游 merge 进自己的 main，纯粹是
为了让补丁不烂掉（一旦上游动了补丁所在的代码，冲突会立刻暴露）。它只在 GitHub 侧跑，与本机无关。

### ⚠️ Twitter 的本地补丁（已由自己的 fork 承载）

`twitter search` 依赖 `x-client-transaction-id` 头，而上游从**匿名**首页解析该标记，X 改版后匿名首页已无 `ondemand.s`，于是 search 恒 404（上游 issue #88，7 个 PR 均未合并）。
实测：带 cookie 抓首页返回 304,748 字节且含该标记，匿名只有 17–35KB 且不含 —— **CT 初始化必须带 cookie**。

补丁位置：`~/AppData/Roaming/uv/tools/twitter-cli/Lib/site-packages/twitter_cli/client.py`
的 `_ensure_client_transaction()` 内，在 `ct_headers = _gen_ct_headers()` 之后加一行：

```python
ct_headers["Cookie"] = self._cookie_string or "auth_token=%s; ct0=%s" % (self._auth_token, self._ct0)
```

补丁已提交到 `YsLtr/twitter-cli` 的 main，安装源就是它，所以 `uv tool install --force` **不会再丢补丁**。
凭据存在 `~/.agent-reach/config.yaml` 的 `twitter_auth_token`/`twitter_ct0`，但裸 `twitter` CLI 只读
`TWITTER_AUTH_TOKEN`/`TWITTER_CT0` 环境变量 —— 跑之前先把这两个值导出到环境。
换过凭据后删缓存 `~/.twitter-cli/transaction_cache.json`（1h TTL，坏缓存会继续报错）。
### ⚠️ opencli 小红书的本地补丁（已由自己的 fork 承载）

`opencli xiaohongshu search` 在小红书当前前端上会**每次必失败**，连不带任何筛选参数的普通搜索也失败：

```
COMMAND_EXEC: Xiaohongshu search filter layout did not match the expected visible panel (ambiguous_option).
```

根因（有 DOM 实测证据）：小红书在筛选面板里给每个 chip 注入**同名的透明诱饵节点**（热力图埋点克隆），带
`data-hp-kind="xhs-filter-tag-<label>"`、`aria-hidden="true"`、`opacity ≈ 1e-05`，其文本、active 状态、
bounding rect 与真实节点完全一致。opencli 的 `visible()` 只判断 `style.opacity === '0'`，而
`'1e-05' !== '0'`，于是诱饵也被算作可见 → `findOption()` 认为该选项匹配到 2 个 → 报 `ambiguous_option`。
又因为 `resolveSearchFilters()` **永远输出全部 5 个组**（含默认值），所以普通搜索同样会走到这段。

上游状态（实测）：npm 最新版 1.8.8 仍未修；issue #2445 / #2550 仍 open；三个替代 PR #2446 / #2460 / #2563
均未合并。**升级解决不了，必须打补丁。**

补丁 = PR #2563 的 3 行，位置在 `clis/xiaohongshu/search.js` 中 `buildApplySearchFiltersJs()` 内的 `visible()`：

```js
if (element.closest('[data-hp-kind], [aria-hidden="true"]')) return false;
// 以及原有 return 追加：
Number(style.opacity) > 0.01 && style.pointerEvents !== 'none';
```

补丁已提交到 `YsLtr/OpenCLI` 的 main。注意**不能**直接 `npm i -g git+<fork>`：opencli 是 TypeScript
项目，安装要跑 `prepare` → `tsc --build`，而 npm 给 git 依赖做“准备”时不会可靠提供 devDependencies。
实测 `tsc` 缺失会让安装失败，且失败前已把旧包删掉；在 prepare 里补装依赖又会让 npm 递归调用 prepare。

所以重装走「已构建的克隆 → 打 tarball → 装 tarball」（tarball 安装不再跑 prepare）。
**仅在用户明确要求时执行**：

```bash
cd ~/.local/share/pi-forks/OpenCLI
npm run build && npm pack --pack-destination ../artifacts
npm install -g ../artifacts/jackwener-opencli-*.tgz --no-audit --no-fund
```

实测效果：默认搜索、`--sort latest`、`--note-type video` 全部返回真实笔记；`--sort latest` 返回的是
当天日期的笔记，而默认搜索返回数月前的，**证明筛选确实被应用**，不是被跳过。


### Twitter 凭据（Windows 上无法自动提取）

Windows 的 Chrome 系浏览器启用 App-Bound Encryption，`browser_cookie3` 报
`RequiresAdminError`，`rookiepy` 报 `only when running as admin` —— **两条自动提取路径都失效，
`agent-reach configure --from-browser brave` 必然失败**。改用 agent-browser-cli（走扩展 API，
零提权，实测可读 x.com / xiaohongshu.com / xueqiu.com）：

```bash
# 取 auth_token / ct0 并存入 agent-reach 配置（值不要打印到对话里）
TID=$(agent-browser-cli open --background https://x.com/home | jq -r .result.opened_tab_id)
agent-browser-cli exec "{\"cmd\":\"cookies\",\"tabId\":$TID}"   # 取结果里的 auth_token/ct0 拼 Header String
printf '%s' "$HDR" | agent-reach configure twitter-cookies --stdin
```

调用时**仍须显式设环境变量**（独立 `twitter` 命令不读 agent-reach 配置）：
`export TWITTER_AUTH_TOKEN=... TWITTER_CT0=...`

## 常驻规则（全程适用）

1. **动手前先体检**：多后端/登录态平台先跑 `agent-reach doctor --json`。
   `active_backend` 有值按它选命令组；`active_backend: null` 表示 Doctor 为避免触发浏览器
   Cookie 读取或远端写入而没有实时验证，**不代表后端不存在**（本机 Twitter/小红书就是这种情况：
   doctor 显示 `warn`，实际可用）。Doctor 只是某时刻快照。
2. **声明你在用什么**：开始干活前说一句「使用 agent-reach 的 X 平台 / Y 后端」。
3. **失败按上面「本机实测状态」与对应 reference 的重试链处理**，不要瞎猜命令。
4. **全网调研类任务**：组合多平台（`web_search` 做通用搜索 + Twitter/Reddit 看讨论 +
   小红书/B站看中文场景），并行收集再汇总。
5. **替用户盯版本**：完成较大的调研后顺手跑 `agent-reach check-update`。
   有新版就在收尾汇报里附一句更新提示，不要中断当前任务去更新，也不要重复提醒同一版本。

## 路由表

| 用户意图 | 分类 | 详细文档 |
|---------|------|---------|
| 小红书/推特/B站/V2EX/Reddit/Facebook/Instagram | social | [references/social.md](references/social.md) |
| 招聘/职位/LinkedIn/Boss直聘 | career | [references/career.md](references/career.md) |
| 网页/文章/RSS | web | [references/web.md](references/web.md) |
| YouTube/B站/播客字幕 | video | [references/video.md](references/video.md) |
| 雪球/股票行情 | finance | [references/finance.md](references/finance.md) |

## 零配置快速命令

```bash
# V2EX 热门（实测可用）
curl -s "https://www.v2ex.com/api/topics/hot.json" -H "User-Agent: agent-reach/1.0"

# B站搜索（实测可用；走公开搜索 API 直连，本机未装 bili-cli，所以不要用 `bili` 命令）
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"
curl -s -c /tmp/bili_ck.txt -o /dev/null -A "$UA" "https://www.bilibili.com/"
curl -s -b /tmp/bili_ck.txt -A "$UA" -e "https://www.bilibili.com/" \
  "https://api.bilibili.com/x/web-interface/search/all/v2?keyword=QUERY&page=1"

# RSS（实测可用；系统 Python 没有 feedparser，用 uv 临时环境）
uv run --with feedparser python -c "
import feedparser, sys
for e in feedparser.parse(sys.argv[1]).entries[:10]: print(e.title, e.link)
" "https://example.com/feed.xml"

# web / YouTube / GitHub / Exa 的入口见上面的「与 pi-web-access 的分工」——
# 用 web_search 和 fetch_content，不要用 mcporter / yt-dlp / gh。
```

## 需登录态的平台（按 doctor 的 active_backend 选命令）

Twitter 注意：`agent-reach configure twitter-cookies` 保存的 Cookie 只供 `doctor`
检查配置是否齐全；`doctor` 不执行 `twitter status`，也不会设置当前 Shell。
直接运行 `twitter` 前，必须在子进程环境中显式提供 `TWITTER_AUTH_TOKEN` 和
`TWITTER_CT0`，不得在日志或命令回显中暴露值。

小红书注意：Agent Reach 不替用户登录，也不读取浏览器 Cookie。OpenCLI 只用
用户已有且明确控制的会话；没有现成会话时不要自动登录。本机 `xhs-cookies` 那条路
需要 Docker（未安装），桌面正解就是 OpenCLI。

Boss直聘配置触发：当用户说“帮我配 Boss直聘”时，先读取 `references/career.md`
的 Boss 章节。Agent 负责按系统启动只绑定 `127.0.0.1:9222` 的专用 Chrome；
**拉起后第一步是暂停并让用户肉眼确认**窗口内是已登录状态（右上角有头像），
未登录则让用户登录/扫码，用户确认后再运行 `boss --cdp-url http://localhost:9222 login --cdp`
和 `agent-reach doctor` 验收。不要让用户自己研究端口参数。
专用 Chrome profile 必须长期复用（本机 `~/.boss-chrome-profile`），不要每次创建，
也不要默认改用日常主 Chrome。**不要相信 `boss status`** 判断登录态（它只校验本地
`session.enc`）；以 `agent-reach doctor` 的浏览器 cookie 探测（wt2）为准。

```bash
# 启动专用调试 Chrome（本机已验证可用；PowerShell）
Start-Process chrome.exe -ArgumentList '--remote-debugging-address=127.0.0.1',
  '--remote-debugging-port=9222',"--user-data-dir=$env:USERPROFILE\.boss-chrome-profile",
  'https://www.zhipin.com/web/user/?ka=header-login'

# 用户扫码后
boss --cdp-url http://localhost:9222 login --cdp
agent-reach doctor --json

# 搜索必须显式走严格 CDP；注意没有 -n 参数，城市用中文名
boss --browser-source existing-browser --cdp-url http://localhost:9222 \
  search "python" --city 北京 --page 1

# Twitter 搜索（twitter-cli；失败重试链见 social.md）
twitter search "query" -n 10

# Reddit（无零配置路径：OpenCLI 或 rdt-cli，必须登录态）
opencli reddit search "query" -f yaml   # 桌面
rdt search "query" --limit 10            # 存量/服务器

# 小红书（桌面 OpenCLI；search 依赖下面的 opencli 补丁）
opencli xiaohongshu whoami -f json
opencli xiaohongshu feed -f json
opencli xiaohongshu note "https://www.xiaohongshu.com/explore/<id>?xsec_token=..." -f json
opencli xiaohongshu user <user-id> -f json

# Facebook / Instagram（桌面 OpenCLI，复用浏览器登录态）
opencli facebook search "query" -f yaml
opencli instagram user USERNAME -f yaml
```

> 小红书 `note` 必须传**带 xsec_token 的完整 URL**（如从 `feed`/`search` 结果里拿），
> 只传 note id 会报 `ARGUMENT: now requires a full signed URL`。

## 环境检查

> 本机 `agent-reach` 已通过 `uv tool` 安装，可直接调用，**不需要** `conda run` 前缀。
> 若某个 shell 里找不到，用全路径 `~/.local/bin/agent-reach`。

```bash
# 检查可用 channel 与每个平台当前激活的后端
agent-reach doctor --json
```

## OpenCLI 适配器发现

路由表没有覆盖用户需要的平台或命令时，先用 `opencli list` 查已有适配器，再用
`opencli <平台> --help` 查看公开命令。发现适配器只证明命令存在，不证明登录态或
目标内容可用；仅在用户任务明确需要该平台时执行只读命令，并以实际非空内容验收。

## 工作区规则

**不要在 agent workspace 创建文件。** 使用 `/tmp/` 存放临时输出，`~/.agent-reach/` 存放持久数据。

## 详细文档

根据用户需求，阅读对应的详细文档：

- [社交媒体](references/social.md) — 小红书, Twitter, B站, V2EX, Reddit, Facebook, Instagram（多后端/登录态命令组）
- [职场招聘](references/career.md) — LinkedIn, Boss直聘
- [网页阅读](references/web.md) — 只有 RSS 部分适用；通用网页阅读用 `fetch_content`
- [视频播客](references/video.md) — 只有 B站 / 小宇宙适用；YouTube 用 `fetch_content`
- [金融行情](references/finance.md) — 雪球股票行情、搜索、热门内容

## 配置渠道

如果某个 channel 需要配置，获取安装指南：
https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md

用户只需提供 cookies，其他配置由 agent 完成。

> ⚠️ **本文件是本地裁剪版**（上游 v1.5.0 + 本机实测修正），已提交到 `YsLtr/Agent-Reach` 的 main。
> 本机的 agent-reach 就是从那个 fork 安装的，所以 `agent-reach install --system` 写出的**就是这一版**，
> 不会再被上游原文覆盖；`dev.md`/`search.md` 已在 fork 中删除、不会恢复，`SKILL_en.md` 也已删除，
> 使所有 locale 都回退到本文件。
> 改完本文件要同步进 fork 的 `agent_reach/skill/SKILL.md` 并 push（本机这份就是从那个 fork 装出来的）。
