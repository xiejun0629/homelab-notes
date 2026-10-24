# 自建 AI 对话（Open WebUI）· 2026-10-14

## 是什么 / 为什么
- **是什么**：Open WebUI v0.12.0，一个自建的 ChatGPT 式网页界面（聊天、长对话、上传文件、多会话管理）。它自己不含模型，**模型来自你填的 API**——所以它是「聊天工具」，能不能解题取决于后面接的模型（接 DeepSeek 就是 DeepSeek 的能力）。
- **为什么自己搭**：第三方客户端是「你 ↔ 他家服务器 ↔ API」多一跳，长对话容易卡；自建的是「你 ↔ 家里 .223 ↔ API」直连，**流式输出的渲染在本地，历史对话存在自己机器上**，长对话不会因为对方服务端限流而卡。

## 访问方式
- 公网（推荐）：**https://example.com/** —— 正规 Let's Encrypt 证书，手机电脑都能开
- 主页第一张卡片：**「AI 对话 · Open WebUI · 自建」**
- 局域网：http://192.168.1.223:3080/
- 账号：`user@example.com`　密码 <REDACTED>（第一个账号自动是管理员；**注册已关闭**，别人进不来）

## 部署结构（都是自己的机器）
| 环节 | 位置 | 说明 |
|---|---|---|
| 应用 | `.223` docker 容器 `open-webui` | 端口 3080→8080，`restart unless-stopped`（重启自动拉起），数据在 `/opt/open-webui/data` |
| 镜像 | `ghcr.io/open-webui/open-webui:main`（6.37GB） | 从南大镜像 `ghcr.nju.edu.cn` 拉的，直连 ghcr.io 太慢 |
| 域名 | e8hk 面板加的 A 记录 `chat` → `203.0.113.10`（你操作的那步） | |
| 入口 | VPS `frps` 443 → `.164` NPM | frpc 的 customDomains 里加了 `example.com`（http + https 两处） |
| 反代 | `.164` NPM `proxy_host/14-chat.conf`（443）+ `15-chat-80.conf`（80：ACME 挑战 + 跳 HTTPS） | 转发到 `192.168.1.223:3080`，带 WebSocket、`proxy_buffering off`（流式输出必需） |
| 证书 | 原证书已扩展到 **7 个域名**（加了 chat.） | 到期 2027-01-12，`certbot renew --dry-run` 通过，全自动续期不受影响 |

### 为什么不挂在 /ai/ 这种子目录
Open WebUI 的前端资源全是绝对路径（`/_app/...`、`/static/...`），官方也不支持子目录部署（会和主页的 `/static/`、`/assets/` 打架）。所以用**独立子域名** example.com，最干净。

### 备用入口
VPS 上还留了一条隧道：本机 8443（NPM 里的 `13-ai.conf`）→ frps 远端端口 **20300**。隧道本身是通的（VPS 本机测 `https://127.0.0.1:20300/` = 200），但**云安全组只放行了 22/80/443/7500/8024**，外部进不来；哪天在控制台放行 20300，`https://example.com:20300/` 立刻可用。

## 已接上的 API（2026-10-14 完成并实测）
- **服务商**：DeepSeek 官方，地址 `https://api.deepseek.com/v1`，Key 已写入（`sk-a3ab2…e3a2`）
- **可用模型 2 个**：`deepseek-flash`（快）、`deepseek-v4-pro`（更强）
- **实测结果**（都是走公网 https://example.com/ 测的）：
  - 登录：✓ 拿到会话令牌
  - 流式对话：✓ `deepseek-flash` 一次回答收到 **381 个流式分片**，内容正常
  - 解题：✓ `deepseek-v4-pro` 问「长 8 宽 5 的长方形周长面积」→「周长 26 厘米，面积 40 平方厘米」
- **Key 存放位置**：`.223:/opt/open-webui/meta/api.env`（600 权限）+ 容器内部配置；**没有写进本文档、没有对外暴露**。
  （提醒：按你的要求对话记录会进 .223 本机的 git 备份，里面会出现这个 Key；仓库只在本机，没有推到任何外部平台。）
- 换 Key / 加第二个服务商：重跑 `set-api.sh`，或在网页「设置 → 管理员 → 连接」里改。
- 注意：v0.12.0 的接口路径变了，写配置要用 `POST /openai/config/update`（旧教程里的 `/api/v1/openai/config` 会返回 `Method Not Allowed`）。
## 日常维护
```bash
docker logs --tail 50 open-webui          # 看日志
docker restart open-webui                 # 重启
bash /opt/open-webui/deploy.sh            # 重建容器（数据不会丢，改 env 后用它）
curl -s localhost:3080/api/version        # 看版本
```
升级：
```bash
docker pull ghcr.nju.edu.cn/open-webui/open-webui:main
docker tag ghcr.nju.edu.cn/open-webui/open-webui:main ghcr.io/open-webui/open-webui:main
bash /opt/open-webui/deploy.sh
```

## 踩坑记录
1. **容器里访问 huggingface.co 会「Network is unreachable」**（被墙），Open WebUI 启动时要下嵌入模型，直接起不来 → 必须加 `HF_ENDPOINT=https://hf-mirror.com`（已写进 deploy.sh，第一次启动 48 秒就绪）。
2. `ghcr.io` 直连拉 6.37GB 极慢（十几分钟没动静）；`docker.m.daocloud.io` 不支持 ghcr 路径；**南大镜像 `ghcr.nju.edu.cn/open-webui/open-webui:main` 90 秒拉完**。
3. VPS 云安全组只放行 22/80/443/7500/8024 —— 所以新服务不能靠「随便开个端口」，要么开安全组，要么用 443 + 子域名（本次选后者）。
4. 管理员账号用 `WEBUI_ADMIN_EMAIL` / `WEBUI_ADMIN_PASSWORD` 环境变量在首次启动时自动创建，顺带自动关闭注册，**不用手工点注册**。

## 登录邮箱已改（2026-10-14）
- 管理员邮箱从 `user@example.com` 改为 **`user@example.com`**（用户要求用自己常用邮箱）。
- v0.12.0 没有改邮箱的接口，直接改的数据库：`docker exec open-webui python3 -c "import sqlite3;c=sqlite3.connect('/app/backend/data/webui.db');c.execute(\"UPDATE user SET email='user@example.com' WHERE email='user@example.com'\");c.commit()"`，然后 `docker restart open-webui`。
- `deploy.sh` 与 `set-api.sh` 里的默认邮箱也一并改了，以后重建容器保持一致。
- 实测：新邮箱登录 ✓（admin / 站主），旧邮箱返回 400 ✓。密码 <REDACTED>。

## 手机上当 App 用（PWA，2026-10-14）
Open WebUI 本身**就是 PWA**（官方文档：https://docs.openwebui.com/getting-started/open-webui-as-app ），所以手机上不用装应用商店的软件，直接「添加到主屏幕」即可获得接近原生 App 的体验：桌面有图标、点开全屏无浏览器边框、登录状态保留、还能收到「回答完了」的通知。

**安卓（Chrome / Edge）**：打开 https://example.com/ → 右上角 ⋮ →「安装应用 / 添加到主屏幕」→ 桌面出现「AI 对话」图标
**iPhone（Safari）**：打开同一个网址 → 底部「分享」→「添加到主屏幕」→ 图标名「AI 对话」

### 为此做的改造（都在 `.164:/opt/web/npm-data/` 与 NPM 配置里）
| 项目 | 说明 |
|---|---|
| `/opt/web/npm-data/chat-pwa/manifest.json` | 自定义 PWA 清单：名字「AI 对话」、独立窗口模式、中文、竖屏、4 个图标 |
| `/opt/web/npm-data/chat-pwa/sw.js` | 极简 Service Worker（**故意不做任何缓存**）。Chrome 要求「有带 fetch 的 SW」才给「安装应用」，否则只能建快捷方式 |
| `nginx/proxy_host/14-chat.conf` | 新增 `location = /manifest.json`、`location = /sw.js`；用 `sub_filter` 在 `</head>` 前注入：注册 SW + `apple-touch-icon` + `apple-mobile-web-app-title=AI 对话` |
| `deploy.sh` | `WEBUI_URL` 修正为 https://example.com；新增 `ENABLE_API_KEYS=True` |
| 数据库 config 表 | `auth.enable_api_keys` 从 false 改成 true（**持久化配置会盖掉环境变量**，只加环境变量没用） |

备份：`.164:/root/chat-proxy-bak-pwa-*`；`.223:/opt/open-webui/deploy.sh.bak-*`

### 也能用「原生 App」的备选方案
Open WebUI 对外提供 **OpenAI 兼容接口**，所以任何 OpenAI 兼容的手机 App 都能接：
- 地址（Base URL）：`https://example.com/api`
- API Key：存在 `.223:/opt/open-webui/meta/mobile-api-key.txt`（600，没写进本文档、没进 git）
- 模型：`deepseek-flash` / `deepseek-v4-pro`
- 安卓应用商店里有第三方客户端「Open Mobile UI」（专门连自建 Open WebUI）；也可以用 Chatbox 之类通用客户端
- 实测：用这个 Key 调 `/api/models` 返回两个模型 ✓，调 `/api/chat/completions` 让模型「只回复两个字：收到」→ 真的回了「收到」✓

### 验证记录
- 公网 `/manifest.json` 200 且 short_name=「AI 对话」；`/sw.js` 200（application/javascript）
- NPM 访问日志里能看到 `HeadlessChrome … GET /sw.js 200`，说明浏览器确实注册了 Service Worker ✓
- 手机尺寸（390×844）截图渲染正常：`手机上的AI对话-20261014.png`

## 家人账号（2026-10-17）

- 账号：登录名 **`user@example.com`**、密码 <REDACTED>、姓名「家人」、角色 **user（非管理员）**，用户 id `71b704d4-34e1-42fc-9c64-0c2ef8533ab2`。
  - 为什么不是纯「1」：Open WebUI 拿「邮箱」当登录名，登录框是 `type="email"`（浏览器不认没有 @ 的东西），后端 Pydantic 也要求域名带点 —— 实测 `1`、`1@local`、`1@home`、`1@1` 全部被拒，`1@a.b`/`user@example.com` 这类才通过。
- **各用各的 Key 怎么实现的**：v0.12.0 已**不支持**「用户自带连接」（`openai.py` 里 `/openai/config` 仅管理员可用，全文没有 connections 字样），所以用「**全局两条连接 + 模型访问授权**」：
  - 连接 0 = 管理员 Key `sk-a3ab…` → 模型 id `deepseek-flash` / `deepseek-v4-pro`
  - 连接 1 = 家人 Key（`sk-074f…`，存 `.223:/opt/open-webui/meta/family-api.env`，600、不进 git）→ `api_configs["1"].prefix_id = "family"` → 模型 id `family.deepseek-flash` / `family.deepseek-v4-pro`
  - 前缀就是路由关键：模型 id 带 `family.` 才会走连接 1，发给 DeepSeek 前由 `strip_provider_model_prefix`（`utils/model_ids.py`）去掉前缀
  - 普通用户**默认看不到任何模型**（v0.12 要显式授权），用 `POST /api/v1/models/model/access/update`（`access_grants: [{principal_type:"user", principal_id:<家人id>, permission:"read"}]`）给两条 `family.*` 建条目并授权 → 家人登录只看到「DeepSeek-V4.1-Flash（家人号）」「DeepSeek-V4-Pro（家人号）」
- 双 Key 隔离验证：① 把连接 1 的 Key 换成假的 → 家人的模型立刻 "Model not found"、管理员模型照常 200；② 家人拿管理员的模型 id 提问 → `400 Model not found`；③ 恢复真 Key 后家人两个模型都能正常对话（返回「收到」）。家人那把 Key 当前余额 ¥10.00。
- 运维：`/opt/open-webui/set-api.sh` 已改成**一次写两条连接**（家人 Key 从 `meta/family-api.env` 读，文件在就写 2 条、不在就退回 1 条），重跑不会把家人那条冲掉；跑完会分别打印管理员与家人各自能看到的模型。
- 已知默认行为：管理员账号在后台能看到全部会话（含家人的）；会话是分账号隔离的，家人之间互相看不到。

## 联网搜索（2026-10-17）

### 结论
两个账号（管理员 `user@example.com`、家人 `user@example.com`）现在都能联网搜索。在聊天输入框左边的「**+**」里打开「**联网搜索**」开关，之后每次提问会先搜网页、再带着网页内容回答，答案里带 `[1] [2]` 这样的引用角标；不开就是普通对话。

### 之前为什么没有
不是坏了，是**默认关着**：数据库里 `web.search.enable=false`、`web.search.engine=""`，前端就不显示这个开关。

### 这次的改动
1. **搜索引擎**：用香港 VPS 上已有的 SearXNG（`127.0.0.1:8888`）。因为 Open WebUI 跑在 .223 的 docker 容器里，新建了一条 SSH 端口转发把容器能访问的 `172.17.0.1:8888` 接到 VPS：
   - `/etc/systemd/system/searxng-tunnel-docker.service`（`ssh -NT -L 172.17.0.1:8888:127.0.0.1:8888 vps`，已 enable --now）
2. **配置**（DB `config` 表，重启不丢）：
   - `web.search.enable = true`、`web.search.engine = "searxng"`、`web.search.searxng_query_url = "http://172.17.0.1:8888/search"`、`web.search.result_count = 4`
   - `models.default_params = {"function_calling":"legacy"}` —— **这条是关键**：新版默认走「模型原生工具调用」，DeepSeek 没有内置 web 工具，模型只会答「我无法联网」；切到 legacy 后由 Open WebUI 自己执行搜索并把结果塞进上下文。
   - `web.search.bypass_web_loader = true` —— 只吃搜索结果的摘要，**不去抓原网页**。原因：.223 在国内，抓 polymarket 这类站点会超时重试 3 次、每次 30 秒，一次提问从 7 秒拖到 115 秒。

### 实测
| 账号 | 模型 | 耗时 | 结果 |
|---|---|---|---|
| 管理员 | <REDACTED> | 6.9s | 「今天上海多云到阴，最高 28℃、最低 22℃…」+ 引用 1 条 |
| 家人 | family.deepseek-flash | 14.6s | 同上（风的细节更全）+ 引用 1 条 |

### 运维
- 搜索后端挂了：`systemctl status searxng-tunnel-docker`（.223），VPS 上 `docker ps | grep searxng`
- 想关掉联网：DB 里 `web.search.enable=false`（前端开关就消失）或让用户自己不开那个开关
- 想让速度再快一点：`web.search.bypass_embedding_and_retrieval=true`（跳过向量化，上下文略粗）
- 改配置必须 `docker exec -i open-webui python3 -`（**不带 `-i` 的 heredoc 会被静默吞掉**），写完 `docker restart open-webui`

### 已知小毛病
模型自己不知道真实日期（它以为现在是 2026 年 5 月，会问「需要我查一下最新数据吗」）。开着联网搜索时以搜到的网页为准，不受影响；如果要它不开搜索也知道今天几号，得加一个在每次对话前注入当天日期的小过滤器（需要的话再做）。

## 每次对话都带上当天日期（2026-10-17）

### 结论
模型现在**每次对话都知道今天是几号**（不用开联网搜索）。实测：
- 管理员问「今天是几月几号？星期几」→ 0.6 秒，「今天是 2026年10月17日，星期六。」
- 家人问「今天是几号？现在几点」→ 0.9 秒，「今天是 2026年10月17日，星期六，现在是北京时间 11:04」
- 流式（网页界面走的方式）也正常；和联网搜索同时用也正常。

### 做法
用 Open WebUI 的「过滤器（Functions / Filter）」功能，在每次请求发给模型之前，往系统提示里插一句真实时间：

- 函数名：`当天日期注入`，id `current_date`，类型 `filter`，已 `is_active=true` + `is_global=true`（对所有人、所有模型生效）
- 插入的文字：「【当前真实日期时间】今天是 YYYY年MM月DD日（星期X），现在是 HH:MM（北京时间）。用户说「今天／现在／最近／最新」时以这个日期为准；如果你的训练数据有截止日期、而问题涉及此日期之后的事件，请直接说明或建议开启联网搜索，不要凭猜测回答。」
- 源码备份：`.223:/opt/open-webui/filters/current_date_filter.py`（已纳入 git 备份）；数据库里存的是同一份代码

### 运维
- 想临时关掉：管理员设置 → Functions → 「当天日期注入」关掉开关（不用删）
- 想改措辞：在同一个页面里编辑保存，或 `POST /api/v1/functions/id/current_date/update`
- 它是 `inlet` 过滤器，**异常已被 try/except 包住**，出问题只会不注入、不会让对话报错

### 顺带调优的搜索参数（配合上面的日期注入）
- `web.search.result_count`：4 → **6**（多给几条摘要）
- `web.search.bypass_embedding_and_retrieval` = **true**：跳过向量检索，把 6 条搜索摘要**原样全部**给模型。之前只喂 4 条再经过向量筛选，天气这类问题经常把有用的那条摘要筛掉，模型只好说「查不到」；改完实测连问 3 次都答对（22～28℃、多云到阴、风力），耗时 5～7 秒。

---

## 联网搜索怎么用（含家人账号）· 2026-10-17 实测

**开关位置**：输入框左下角「＋」右边的**滑块图标（扩展功能）**，点开后第一项「联网搜索」，第二项「代码解释器」。

**顺序很重要**：先点「新对话」→ 再打开「联网搜索」→ 然后提问。
我先在旧对话里打开开关、再点「新对话」，开关被复位，那一问就变成「我没有联网能力」；按正确顺序重来即正常。

**家人账号同样有**（`user@example.com` / `<REDACTED>`），权限与服务端配置都正常：
- 家人号问「今天上海天气怎么样」→ 回答底部显示「检索到 16 个引用来源」，内容：多云到阴、22–28℃、东南风4~5级，来源 m.sohu.com / weather.cma.cn / weather.com.cn。
- 家人号回答里带「日期：2026年10月17日，星期六（北京时间 11:39）」→ 当天日期注入已生效。
- 服务端 A/B：`features.web_search=true` 时，`params:{}` 与 `params:{"function_calling":"legacy"}` 两种都返回 sources（≥1），都能联网。

**截图**：`家人号-联网搜索开关-20261017.png`、`家人号-联网搜索回答-20261017.png`、`家人号-当天日期回答-20261017.png`。

**验证方法备忘**：UI 层验证要用 CDP 真实点击（chromium `--remote-debugging-port=9222`，`/root/venv` 里装了 websocket-client，连接需 `suppress_origin=True`）；自建代理不支持 WebSocket upgrade，经代理的界面 socket.io 会 400、无法流式聊天，所以 UI 验证必须直连 `https://example.com/`。

---

## 联网搜索的精准性：现在的架构与本次调优 · 2026-10-17

### 现在用的是什么搜索引擎
不是直接买某家的搜索 API，而是**自建元搜索引擎 SearXNG**（VPS 香港，docker `searxng/searxng:latest`，版本 2026.10.9）：

```
Open WebUI(.223) --web.search.engine=searxng--> http://172.17.0.1:8888/search
   --SSH 隧道(searxng-tunnel-docker.service)--> VPS 127.0.0.1:8888
   --SearXNG 并发请求多个上游引擎(google cse / yahoo / 360search ...)--> 去重+加权排序 --> 标题+URL+摘要(JSON)
   --> Open WebUI 取前 N 条塞给 DeepSeek，并把当天日期注入 system 消息
```

SearXNG 自己不做索引，它把多个搜索引擎的结果聚合起来：好处是**免费、可自建、不依赖单一来源**；代价是受各上游反爬限制，且拿到的是**摘要（约 100 字/条）**而不是全文。

### 本次做的调整
- SearXNG（VPS `/opt/searxng/searxng/settings.yml`，已备份 `/root/searxng-settings.yml.bak-*`）：
  - 改成**完整默认配置 + 覆盖**（此前的部分覆盖会让引擎丢字段），3512 行；
  - 启用 **yahoo、360search**（权重 1.5），保留 **google cse**；
  - 停用 **bing**（这个版本 bing 引擎返回与查询无关的热门内容）、**duckduckgo/baidu**（CAPTCHA 拦）、**brave/wikipedia/wikidata/sogou**（限流/崩溃）；
  - 超时收紧为 6s/8s，`formats` 含 json。
- Open WebUI（`.223` 数据库 `config` 表）：
  - `web.search.result_count`：6 → **10**（给模型的候选结果更多，召回更好）；
  - `web.search.bypass_web_loader=true`（只用摘要，快）；
  - `web.search.bypass_embedding_and_retrieval=true`。

### 实测（家人账号 + DeepSeek-V4.1-Flash + 联网搜索）
| 问题 | 用时 | 结果 |
|---|---|---|
| 顺丰控股 2025 前三季度营收 / Q3 单季 | **6.4s** | 前三季度 2,252.61 亿（+8.89%）、Q3 单季约 784 亿（+8.2%）、件量 43.1 亿票 ✓ |
| 今天上海天气 | **6.8s** | 阴天为主、早晨 22℃、白天 27–28℃、明天小雨升至 31℃ ✓ |
| 对照：开启「抓取网页正文」 | 28.6s / 70.8s | 更全（784.03 亿、单季净利 25.7 亿、件量 43.1 亿票），但部分网页超时拖慢 → 保持摘要模式为默认 |

### 关于 Google 限流
测试期间高频查询触发了 Google 的 "unusual traffic" 临时封禁（`google cse` 被 SearXNG 暂停），此时自动切到 **yahoo + 360search** 兜底，上面两组实测就是在只有兜底引擎时跑的。正常使用（一小时几次）不会触发；若要更稳可接搜索 API（见下）。

### 还能怎么提高精准性（按性价比排序）
1. **接一个搜索 API**（最有效）：Tavily / Bocha（博查，中文）/ Brave，都有免费额度，返回的正文摘要 1–2KB（比 100 字摘要准很多），速度约 2–4s。需要注册拿一个 Key。
2. 打开「抓取网页正文」：准，但每次 20–70s，不适合手机日常用。
3. 调中文引擎权重 / 增加中文引擎：受上游反爬限制，收益有限（bing/baidu 在这个版本/这个 IP 上都不可用）。

---

# 2026-10-17 更新：搜索已换成博查（Bocha）API

> 上面「还能怎么提高精准性」的第 1 条已落地：SearXNG 保留为回退，日常搜索走博查。

## 为什么换
- SearXNG 受上游反爬 / Google 限流影响大，摘要只有约 100 字；博查返回 **1–2KB 长摘要**，中文更准，单次搜索 0.3–0.5s。

## 怎么换的
只改数据库配置，不动代码：
- 文件：`/opt/open-webui/data/webui.db` 的 `config` 表（已备份 `webui.db.bak-20261017-bocha`，同目录）
- 改动：`web.search.engine` = `bocha`；`web.search.bocha_search_api_key` = 博查 Key（其余不变：`result_count=10`、`bypass_web_loader=true`）
- 生效：`docker restart open-webui`，17s 后 `/health` 正常

## 实测（家人号 + DeepSeek-V4.1-Flash + 联网搜索）
| 问题 | 用时 | 结果 |
|---|---|---|
| 顺丰控股 2026 前三季度营收 / 归母净利 | **8.8s** | 指出三季报尚未披露、给出 2026 半年报（营收 1,555.06 亿 +5.89%、归母净利 55.02 亿 -4.11%），还识别出检索结果里"1,890.1 亿/-5.1%"是 **2023 年三季报**数据并提醒别误用 ✓ |
| 今天上海天气 | **4.6s** | 阴、23–27℃、东南风<3 级，带 3 条引用 ✓ |

## 费用
- 博查按**成功调用**计费 **¥0.036/次**；账户余额 **19.57 元**（测试约 12 次消耗 0.43 元）。
- 按每天 3–10 次估算：每月 100–300 次 = **¥3.6–10.8/月**（免费额度/充值包另算，余额查 `GET https://api.bochaai.com/v1/fund/remaining`）。

## 回滚（一分钟内可恢复）
1. `web.search.engine` 改回 `searxng`（隧道和 VPS 上的 SearXNG 都还留着，`web.search.searxng_query_url`=`http://172.17.0.1:8888/search`）
2. `docker restart open-webui`

## DSH 也接了博查
给 DSH agent 增加了 `mcp__bocha__search` 工具（走 MCP，不改 Open WebUI 的搜索设置），详见 `DSH接入博查搜索-20261017.md`。

---

# 2026-10-17 更新（二）：两个账号都走博查 + 真实成本

## 两个账号都接上了
Open WebUI 只有两个用户：`站主`（user@example.com，管理员）和 `家人`（user@example.com）。搜索配置是**全局的**（`config` 表 `web.search.*`），所以两个账号天然都走博查；为确认没有按用户生效的开关，又分别实测了一遍：

| 账号 | 模型 | 问题 | 用时 | 结果 |
|---|---|---|---|---|
| 站主（管理员） | <REDACTED> | 顺丰控股 2026 前三季度 | **5.1s** | 回答带 [1][3] 引用，流式 `sources` 的 `source.docs[].metadata.source` 全是 `https://bocha.cn/share/...` ✓ |
| 家人 | family.deepseek-flash | 今天上海天气 | **5.7s** | 回答带引用，来源同上 ✓ |

（家人号此前已在真实页面验证：一次提问显示「检索到 11 个引用来源」，正文带 `bocha.cn`、`i.ifeng.com` 内联引用。）

## 真实成本：一次联网提问 ≈ 2 次博查调用 ≈ ¥0.07
- 实测：余额 18.55 → 问一次「上海今天天气」→ 18.48，**差 ¥0.07 ≈ 2 次调用**（7.96s，回答带 [3][4] 引用）。
- 之前按「1 次/问题」估的偏低，月度估算相应翻倍：

| 使用强度 | 每月调用 | 每月费用 |
|---|---|---|
| 每天 3 问 | ≈180 次 | ≈ **¥6.5/月** |
| 每天 5 问 | ≈300 次 | ≈ **¥10.8/月** |
| 每天 10 问 | ≈600 次 | ≈ **¥21.6/月** |

- 当前余额 **18.48 元**（2026-10-17 16:00），按每天 3–5 问大约够 **3.5–5.5 个月**。
- 余额与消耗已上监控台：每 15 分钟采样，余额 <¥5 手机 ntfy 提醒、<¥2 更严重告警，见 `监控台-博查余额-20261017.md`。

## DSH 侧默认仍用免费搜索
为省钱，这个 agent 自己的 `web_search` 工具继续走 **免费 SearXNG**；只有需要更准的中文资料时才手动调 `mcp__bocha__search`。所以你在 DSH 里的日常搜索**不花钱**，花钱的只有 Open WebUI 里打开「联网搜索」的提问。
