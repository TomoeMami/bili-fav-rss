# bilibili 收藏夹 → 个人播客 RSS：Agent 构建规格

> **配套用法**：把本文件全文 + `README.md` 中的「推荐 prompt」一起发给 AI Agent。
>
> 本文是一份**可直接发送给 AI Agent** 的自包含构建规格：只描述产品目标、架构、自有接口契约、RSS 契约与验收标准。
> **本文故意不包含**任何第三方平台的接口地址、参数名、请求头细节、签名算法、响应字段清单或错误码——
> 这些实现细节由 Agent 自行检索、调研、验证，并对合规性负责。
>
> 目标平台：**bilibili**（非官方、无关联）。
> 适用场景：**仅个人、仅局域网、仅非商业使用**。不建议公开部署，也不建议把本文当作公开 API 文档发布。

---

## 0. 合规边界（硬约束，Agent 必须遵守并写入实现说明）

**允许**
- 个人自用、只在家庭局域网访问、非商业。
- 只处理**公开**收藏夹（或你自己有权访问的收藏夹）。
- **只使用你自己的账号**，只播放/转封装你账号本来就有权播放的音频。

**禁止**
- 公网部署、监听 `0.0.0.0`、多用户共享、代挂、公开服务。
- 把音频/视频落盘长期保存、镜像、转存、再分发。
- 绕过会员/付费内容保护、去除广告、批量抓取、导出他人数据。
- 规避或对抗平台的人机验证、风控、访问控制；不得实现“反-反爬”。
- 把调研得到的接口/字段/签名细节写进 README、Issue、commit message 或任何公开位置。

**失败处理**
- 遇到 403/412/验证码/风控/接口变更：**停止并报告**，不要尝试绕过。
- 若无法在不绕过保护的前提下完成：输出阻塞报告，不要猜测或硬编码。

> 非官方项目，与任何平台无关联。平台条款、著作权与当地法律风险由使用者自行承担。**发布本文档前请阅读 §15。**

---

## 1. 产品目标

实现一个局域网内的个人播客 RSS 服务：

1. 浏览器打开局域网地址，用环境变量中的密码登录。
2. 在网页内绑定**自己的账号**（扫码授权或粘贴自己的会话凭据），可解绑、可查看状态。
3. 输入公开收藏夹页面 URL（形如 `https://space.bilibili.com/<uid>/favlist?fid=<fid>`），生成一个**带 token 的订阅地址**。
4. 播客客户端（AntennaPod / Pocket Casts / 小宇宙等）订阅该地址后：
   - 条目按收藏时间倒序，最新在上；
   - 每条 `enclosure` 指向本服务的音频路径；
   - 支持 `HEAD`、`Range`/`206`、seek；
   - 音频 URL 稳定（上游直链短时有效，由服务端在请求时重新获取，不暴露给客户端）。

### 非目标

- 不做转码、不落盘、不做离线库、不做视频下载。
- 不做多用户/公开分享/公网访问。
- 不引入前端框架或构建步骤；一个静态 HTML + 原生 JS 即可。
- 不引入 npm 依赖（可选：二维码渲染库需为 MIT 并保留许可证）。

---

## 2. 验收标准（Definition of Done）

| # | 验收项 |
| --- | --- |
| A1 | `node server/test.mjs` 全部通过（见 §10） |
| A2 | 未登录 `/api/me` 返回 `authed:false`；错误密码 401；正确密码 200 + HttpOnly session cookie |
| A3 | 无 token/错 token 访问 `/rss`、`/audio/...` → 401；正确 token 可访问 |
| A4 | `/rss` 返回 `application/rss+xml`，`feedparser` 解析无错误，`<guid>` 为稳定不变的视频 ID |
| A5 | enclosure 形如 `http://<lan-ip>:<port>/audio/<videoId>/<partId>.m4a?token=<token>` |
| A6 | 音频 `HEAD` → 200 + `Content-Length` + `Accept-Ranges: bytes`；`Range: bytes=0-1023` → 206 |
| A7 | `ffprobe` 能直接读音频 URL，时长与视频信息一致 |
| A8 | 重置 token 后旧 URL 立即 401、新 URL 可用；`guid` 不变（客户端不会重复剧集） |
| A9 | 仓库/日志无真实凭据；`data/`、`.env` 已 gitignore |
| A10 | 默认只绑 `127.0.0.1` 或指定局域网 IP，不监听 `0.0.0.0` |

---

## 3. 技术约束

- **Node.js ≥ 20**，ESM，零 npm 依赖；一个进程提供 HTTP 服务 + 静态页。
- 数据不写入代码目录：默认 `DATA_DIR=./data`，生产用 `/var/lib/<service>`。
- 必须能在无网络环境用 mock 跑测试；真实网络测试允许跳过并标记。
- 所有面向用户的文案用中文；错误信息不得包含凭据/token。

---

## 4. 架构

```text
浏览器（局域网）
  └─ 密码登录 → 管理账号绑定 / 订阅 token
播客客户端
  └─ GET /rss?url=<收藏夹页面URL>&token=<token>
        └─ 服务端实时：取收藏夹 → 取视频信息 → 取音频信息 → 生成 RSS
  └─ GET /audio/<videoId>/<partId>.m4a?token=<token>
        └─ 服务端实时：取新的音频直链 → 转发 Range/HEAD → 客户端
```

### 组件职责

| 组件 | 职责 |
| --- | --- |
| `server/server.mjs` | HTTP 服务、登录/session、token、路由、静态页、CSRF、限流 |
| `server/account.mjs` | 账号授权：扫码/凭据、状态、解绑；只保存不透明凭据 |
| `server/source.mjs` | 平台数据访问：收藏夹、视频信息、音频信息（细节由 Agent 自行调研） |
| `server/feed.mjs` | RSS 组装 + 音频中继（HEAD/Range/流式转发） |
| `server/store.mjs` | `state.json` 原子读写（0600） |
| `server/test.mjs` | 集成测试（临时子进程 + 临时数据目录） |
| `server/public/index.html` | 登录、绑定、订阅地址 UI |

**为什么两层鉴权**：播客客户端不会做网页登录，也无法携带自定义请求头。所以浏览器 UI 用密码，订阅地址用 token。

---

## 5. 本服务自有接口契约（可完整按此实现）

> 以下均为**本服务自己设计**的 HTTP 接口，不涉及任何第三方平台细节。

| 方法 | 路径 | 鉴权 | 说明 |
| --- | --- | --- | --- |
| GET | `/` | 无 | 登录/管理页（静态 HTML） |
| GET | `/health` | 无 | `{ok:true}` |
| POST | `/api/login` | 无 | body `{password}` → 200 + session cookie |
| POST | `/api/logout` | 无 | 清除 session |
| GET | `/api/me` | 可选 | `{authed, account:{bound,name,id,savedAt}, feedToken?, publicBase?}` |
| POST | `/api/account/qr` | 登录 | 开始扫码授权 → `{url, key}`（前端渲染二维码） |
| GET | `/api/account/qr/poll?key=` | 登录 | `{status: waiting\|scanned\|ok\|expired}`；ok 后保存凭据 |
| POST | `/api/account/credential` | 登录 | body `{credential}`；手动粘贴自己的会话凭据 |
| GET | `/api/account/status` | 登录 | 查询当前账号是否有效 |
| POST | `/api/account/logout` | 登录 | 解除绑定 |
| POST | `/api/token/regenerate` | 登录 | 重置订阅 token |
| GET | `/rss` | token | 生成播客 RSS |
| GET/HEAD | `/audio/<videoId>/<partId>.m4a` | token | 音频中继 |
| GET | `/favicon.ico` | 无 | 204 |

### 鉴权细节

- 密码：环境变量 `ADMIN_PASSWORD`；常量时间比较（SHA-256 后 `timingSafeEqual`）。
- session：24 字节随机 id，内存 Map（TTL 7 天），`HttpOnly; SameSite=Lax; Path=/`，HTTPS 时加 `Secure`。
- feed token：24 字节随机 base64url，持久化到 `state.json`；`FEED_TOKEN` 环境变量优先。
- token 校验：常量时间比较；失败返回 401 文本。
- `/api/*` 的 POST：若带 `Origin`，host 必须等于请求 `Host`，否则 403。
- `/rss` 限流：每 token+IP 每分钟 20 次；超限 429。

### 配置

| 变量 | 说明 |
| --- | --- |
| `ADMIN_PASSWORD` | 必填，网页登录密码 |
| `PORT` | 默认 8080 |
| `HOST` | 默认 `127.0.0.1`；局域网模式填具体 LAN IP |
| `DATA_DIR` | 默认 `./data` |
| `PUBLIC_BASE_URL` | RSS 中使用的绝对地址；不填用请求 origin |
| `FEED_TOKEN` | 可选，固定订阅 token |
| `COOKIE_SECURE` | `1` = Secure cookie（仅 HTTPS） |

### 状态文件

`DATA_DIR/state.json`，权限 **0600**：

```json
{
  "feedToken": "<random base64url>",
  "accountCredential": "<opaque string, 仅本机>",
  "accountName": "<string|null>",
  "accountId": "<string|number|null>",
  "savedAt": 1700000000000
}
```

原子写（临时文件 + rename）；启动时无 token 则生成；不得把凭据写进日志。

---

## 6. RSS 契约（公开标准，可完整实现）

`GET /rss` 参数：`url`（收藏夹页面 URL，需 encode）或 `uid`+`fid`；`token` 必填；`limit`（1–50，默认 20）；`order=mtime|pubtime`；`exact=1`（可选精确长度）。

### channel

| 字段 | 值 |
| --- | --- |
| `<title>` | 收藏夹标题 |
| `<link>` | 收藏夹页面 URL |
| `<atom:link rel="self">` | 当前 feed 绝对 URL（建议含 token） |
| `<description>` | 「公开收藏夹『X』的个人播客订阅源」 |
| `<language>` | `zh-cn` |
| `<itunes:author>` | 收藏夹作者名 |
| `<itunes:explicit>` | `false` |
| `<itunes:category text="Society &amp; Culture"/>` | 固定即可 |
| `<itunes:image href>` | 收藏夹封面（https，仅引用） |
| `<lastBuildDate>` | 当前时间 RFC 822 |
| `<generator>` | 自定义字符串 |

### item（按收藏时间倒序）

| 字段 | 值 |
| --- | --- |
| `<title>` | 视频标题（XML 转义） |
| `<link>` | 视频页面 URL |
| `<guid isPermaLink="false">` | 稳定的视频 ID（不要用签名 URL） |
| `<pubDate>` | 收藏时间（回退发布时间），RFC 822，带 `+0800` |
| `<itunes:duration>` | `H:MM:SS` |
| `<itunes:explicit>` | `false` |
| `<itunes:image href>` | 视频封面（https，仅引用） |
| `<description>` | 视频简介纯文本（XML 转义） |
| `<enclosure url length type>` | `url = <PUBLIC_BASE_URL>/audio/<videoId>/<partId>.m4a?token=<token>`；`type="audio/mp4"`；`length` 为字节数 |

**length**
- 默认用「音频码率 × 时长 / 8」估算（误差通常 < 0.1%，可接受）。
- `exact=1` 时用一次小 `Range` 请求从响应头取真实总字节数；失败则回退估算。
- 估算值不要宣称精确；客户端会使用实际 `Content-Length`。

**响应头**：`Content-Type: application/rss+xml; charset=utf-8`，`Cache-Control: public, max-age=300`；可加 `X-Feed-Items`。

---

## 7. 音频中继行为契约

`GET|HEAD /audio/<videoId>/<partId>.m4a?token=<token>`：

1. 每次请求时，用**当前账号会话**向平台获取一个新的可播放音频地址；内存缓存建议 30 分钟。
2. 永远不把上游地址写进 RSS、日志或响应；客户端只看到本服务的稳定路径。
3. `HEAD`：若上游 HEAD 不可靠，发一次极小 `Range` 请求（如 `bytes=0-0`），从响应头取总长度，返回 200 + `Content-Length` + `Accept-Ranges: bytes`，丢弃 body。
4. `GET`：透传上游状态码（200/206）、`Content-Length`、`Content-Range`；`Content-Type` 固定 `audio/mp4`；`Accept-Ranges: bytes`；流式转发，不整段缓冲。
5. 客户端断开时取消上游 body，释放连接。
6. 上游地址失效/403 时换候选地址或重新获取；全部失败 → 502 文本（不含内部 URL）。
7. 支持 `Range`：客户端 seek 应得到 206。

> 上游直链短时有效（通常数小时），且可能要求“正常网页播放时的请求上下文”。
> 实现者必须自行研究并遵守平台条款；若需要绕过来自平台的新增保护才能播放，**停止**，不要实现。

---

## 8. 账号授权（抽象）

- 提供两种方式（至少一种）：
  1. **扫码授权**：服务端向平台发起授权、展示二维码、轮询结果；成功后保存平台返回的会话凭据。
  2. **手动凭据**：用户粘贴自己的会话凭据；保存前先调用平台“当前账号”接口验证，无效则 400。
- 只保存**不透明凭据字符串**，不要解析/拆分/记录其内容；`0600` 存 `state.json`。
- 提供解绑：清空凭据并保存。
- 凭据过期时：`/rss` 回退匿名模式（若平台允许），不要直接 500。
- 严禁实现账号共享、代挂、多用户。

---

## 9. 安全要求

- 默认只绑 `127.0.0.1`；局域网模式绑具体 LAN IP；**禁止 `0.0.0.0`**。
- 静态文件白名单（`/`、`/index.html`、可选二维码库、`/favicon.ico`），禁止路径穿越。
- 响应头：`X-Content-Type-Options: nosniff`、`Referrer-Policy: no-referrer`。
- 日志只记录 method/path/status/耗时；**不记录 query 中的 token、Cookie、密码、凭据**。
- `state.json` 与 env 文件权限 0600；`.gitignore` 覆盖 `data/`、`.env`。
- 发布前扫描 git 历史，确保从未提交过真实凭据。
- 若使用二维码库，必须为 MIT（或兼容）并保留版权声明。

---

## 10. 测试计划（必须实现）

`server/test.mjs`：`spawn` 临时服务（随机端口 + 临时 `DATA_DIR` + 固定密码），用 `fetch` 跑用例，最后清理。

| # | 用例 | 断言 |
| --- | --- | --- |
| 1 | `/health` | 200 |
| 2 | 未登录 `/api/me` | `authed:false` |
| 3 | 错误密码 | 401 |
| 4 | 正确密码 | 200 + HttpOnly cookie |
| 5 | 登录后 `/api/me` | `authed:true` 且有 `feedToken` |
| 6 | `/rss` 无 token | 401 |
| 7 | `/rss` 错 token | 401 |
| 8 | `/rss` 正确 token | 200、`application/rss+xml`、条目数 ≤ limit |
| 9 | feed 排序 | `pubDate` 非递增 |
| 10 | enclosure | 含 `/audio/` 与 `token=`，`type="audio/mp4"`，`length` 为正整数 |
| 11 | guid | 稳定视频 ID、无签名参数 |
| 12 | 音频 HEAD | 200、`Content-Length` > 0、`Accept-Ranges: bytes` |
| 13 | 音频 Range | 206、`Content-Range: bytes 0-1023/<total>`、body 1024B |
| 14 | 音频无 token | 401 |
| 15 | 重置 token | 旧 401、新 200、guid 集合不变 |
| 16 | 扫码开始 | 200，返回可渲染的二维码内容 |
| 17 | 扫码轮询（未扫） | `waiting` 或接口错误被容忍 |
| 18 | 无效凭据 | 400，状态未被写入 |
| 19 | 登出 | 200；随后 `authed:false` |
| 20 | 静态页 | `/` 200，含登录 UI |
| 21 | 路径穿越 | 不返回任意文件 |
| 22 | 仓库扫描 | 无凭据文件进入版本控制 |
| 23 | feedparser | RSS 可被 feedparser 解析 |
| 24 | ffprobe | 音频 URL 可读、时长误差 < 2s |

测试数据：不要硬编码他人 UID/收藏夹；由实现者填入自己的公开收藏夹，或使用 mock/fixture。

---

## 11. 部署（局域网）

```bash
cd server
ADMIN_PASSWORD='换成强密码' \
HOST=192.168.x.x \
PORT=8080 \
DATA_DIR=/var/lib/<service> \
PUBLIC_BASE_URL=http://192.168.x.x:8080 \
COOKIE_SECURE=0 \
node server.mjs
```

- 浏览器打开 `http://192.168.x.x:8080`。
- 路由器里给机器做 **DHCP 保留**，否则 IP 变化后绑定失败。
- 不要端口映射到公网；不要 `0.0.0.0`。

systemd（可选）：

```ini
[Unit]
Description=personal podcast RSS
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=<user>
WorkingDirectory=/path/to/server
EnvironmentFile=/etc/<service>/env
ExecStart=/usr/bin/node server.mjs
Restart=always
RestartSec=3
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/var/lib/<service>
StateDirectory=<service>

[Install]
WantedBy=multi-user.target
```

HTTPS 仅在客户端强制要求时使用（反向代理 + 本地 CA 或个人 VPN/tailnet）；开启后 env 设 `COOKIE_SECURE=1`。

---

## 12. 失败与边界

| 场景 | 行为 |
| --- | --- |
| 收藏夹私密/不存在 | 400/404，提示不可访问 |
| 视频失效 | 跳过，不进入 feed |
| 无可用音频（番剧/OGV/课程等） | 跳过该条，不整体失败 |
| 多 P 视频 | 取第一 P（可在实现说明中写明） |
| 上游直链失效/403 | 换候选或重新获取；HEAD 用 Range 探测代替 |
| 403/412/验证码/风控 | 停止并返回 502，不绕过 |
| 会话过期 | 回退匿名（如平台允许）；状态显示未登录 |
| 接口结构变化 | 返回明确错误；不猜测、不硬编码 |
| 客户端中止下载 | 取消上游 body |
| 请求过多 | `/rss` 429；feed 缓存 5 分钟 |

---

## 13. Agent 需要自行调研补全的细节（本文故意不提供）

> 目标平台为 **bilibili**；以下问题的**答案不在本文档中**（本文不提供任何 bilibili 接口地址、参数、签名算法或字段清单）。Agent 应优先使用平台官方公开 API/文档；其次参考公开可得的资料；
> 如需实测，只能使用实现者**自己的账号**、单条、低频，并遵守平台条款。不得批量采集，不得整理成公开 API 文档。

1. 收藏夹页面 URL 的构成，`uid` 与 `fid` 的关系，旧版/新版 fid 如何兼容。
2. 当前可用的收藏夹列表读取方式：分页、排序参数、失效/私密条目如何识别，返回哪些字段。
3. 视频信息读取方式：如何拿分 P、时长、封面、简介。
4. 音频信息读取方式：
   - 匿名与登录状态下的差异；
   - 如何选择 AAC 音轨、如何过滤不兼容音轨；
   - 直链有效期、是否绑定请求上下文、HEAD 行为；
   - 失败/403 时的表现。
5. 账号授权：扫码流程或会话凭据的获取方式；如何验证登录态、如何撤销。
6. 平台的错误码/风控表现（403/412/验证码）以及“停止而不是绕过”的判定。
7. 平台用户协议与当地法律对“个人使用/自动化访问”的限制；是否存在官方 API 可替代。
8. 如何构造 mock/fixture：在没有平台网络时仍能覆盖 §10 的测试。

**约束**：调研结果只用于实现；不要写进 README/Issue/commit message；不要发布成公开文档。

---

## 14. Agent 执行清单

1. 读 §0 合规边界，在实现说明与 UI 中写入「仅个人局域网、非官方、不公开部署」。
2. 初始化 `server/`（ESM、零依赖），实现 `store/account/source/feed/server` 模块。
3. 实现网页：登录、绑定（扫码/凭据）、订阅地址、token 重置。
4. 实现 `/rss` 与 `/audio`，遵守 §6/§7。
5. 实现 §9 安全要求与 §5 鉴权/限流。
6. 写 `server/test.mjs`，覆盖 §10；无网络时 mock 并标记跳过。
7. 用**自己的**公开收藏夹做端到端：登录 → feed → feedparser → HEAD/Range → ffprobe → seek。
8. 检查仓库无凭据、无 demo 内容、`data/` 已 gitignore。
9. 输出：运行命令、测试结果、已知限制、合规声明。

**完成定义**：§2 的 A1–A10 全部满足；`node server/test.mjs` 通过；能用自己的播客客户端订阅并播放。

---

## 15. 发布前合规检查（若公开本文档）

- [ ] 本文不含任何第三方接口地址、参数名、签名算法、响应字段清单、错误码。
- [ ] 仓库不含真实凭据、token、内网地址、他人 UID、收藏夹快照、封面、视频标题。
- [ ] 不含下载/镜像/转存代码，不提供公网部署模板。
- [ ] 默认监听 `127.0.0.1`/具体 LAN IP。
- [ ] 名称/图标不使用平台官方 logo；对平台名仅作描述性提及，README 顶部声明非官方、无关联。
- [ ] 许可证只覆盖自己的代码；NOTICE 声明不授予任何平台内容/接口权利。
- [ ] 收到下架/律师函时：停止分发、保留记录、不要公开对抗。

---

## 附录：占位符

- `<uid>`：用户 UID；`<fid>`：收藏夹页面里的 fid；`<videoId>`：视频 ID；`<partId>`：分 P ID。
- `<token>`：订阅 token；`<LAN-IP>`：本机局域网 IP。
- 本文所有示例均为占位符；实现时不要写入真实账号/收藏夹。
- 本文与 bilibili 官方无关联，仅用于描述个人自用工具的构建规格。
