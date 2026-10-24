# VPS 自建 SearXNG 作为 DSH 搜索源（2026-10-11）

## 一、为什么要做

DSH 的 `web_search` 原先只挂了 `@godchen520/dsh-web-search-bing`，即单源抓 `cn.bing.com`。
自建 SearXNG 后是一次查询fan-out 到 google / bing / wikipedia 等引擎再合并排序，中文与英文结果都更全、更及时；仍然免密钥、不消耗 DeepSeek 搜索额度。

**为什么装在 VPS 而不是 .223**：SearXNG 的上游引擎（DuckDuckGo、Brave、Google 等）在国内被墙，必须放在香港 VPS 上跑；.223 只通过隧道取结果。

## 二、架构（没有新增任何公网端口）

```
DSH(.223) → http://127.0.0.1:8888
              │  systemd: searxng-tunnel.service（ssh -L）
              ▼
         VPS 127.0.0.1:8888  ← docker 容器 searxng（只绑回环）
```

- VPS 上 SearXNG **只监听 127.0.0.1:8888**，公网完全访问不到，安全组不用改。
- .223 用一条常驻 SSH 隧道把同一个回环端口映射到本地，DSH 插件打 `127.0.0.1:8888`。
- 隧道断了 systemd 5 秒重连；VPS 重启后容器 `restart: unless-stopped` 自动拉起。

## 三、关键文件与配置

| 位置 | 路径 | 说明 |
| --- | --- | --- |
| VPS | `/opt/searxng/docker-compose.yml` | 镜像 `searxng/searxng:latest`，端口 `127.0.0.1:8888:8080` |
| VPS | `/opt/searxng/searxng/settings.yml` | `search.formats: [html, json]`（**不开 json 就 403**）、`server.limiter: false`、`server.secret_key`（32 字节随机，存 `/opt/searxng/.secret`） |
| .223 | `/etc/systemd/system/searxng-tunnel.service` | `ssh -NT -L 127.0.0.1:8888:127.0.0.1:8888 vps`，Restart=always |
| .223 | `/root/dsh-plugins/dsh-searxng/` | 自写插件（`@local/dsh-searxng`），注册 provider id = `searxng` |
| .223 | `~/.dsh/profiles/web/node_modules/@local/dsh-searxng` | 软链到上面目录（bundle 安装） |

插件 patch（`/root/dsh-plugins/dsh-searxng/cordis.patch.yml`）做两件事：插入 `web-search-searxng` 行 + 把 `web.searchProvider` 改成 `searxng`。

## 四、踩过的坑（复现时注意）

1. **`cap_drop: ALL` 会让容器起不来**：`cap_drop ALL` 去掉了 `DAC_OVERRIDE`，entrypoint 以 root 往别的 uid 拥有的 `/etc/searxng` 写 `settings.yml` 时 `Permission denied`，容器无限重启（`"/etc/searxng/settings.yml" is not a valid file, exiting`）。解决：保留 `cap_add: [CHOWN, SETGID, SETUID]`、**不写 `cap_drop`**，并预先从镜像模板 `/usr/local/searxng/settings.template.yml` 落一份 `settings.yml` 到宿主机。
2. **`ui.default_locale: zh-CN` 不是合法值** → 每次请求 `searx.preferences.ValidationException: Invalid value: "zh-CN"` 直接 500。合法写法是 `zh-Hans`/`zh-Hant`，本项目干脆不设该项。
3. **`link:` 安装的插件解析不到 DSH 自带包**：`@local/dsh-searxng` 以 `link:` 装进 profile 后，Node 从真实路径 `/root/dsh-plugins/dsh-searxng` 向上找 `node_modules`，找不到 `@deepseek-ai/schemastery` / `@deepseek-ai/dsh-web` → 插件行 `failed to import`。解决：在插件目录里建软链
   `node_modules/@deepseek-ai/{schemastery,dsh-web} -> /opt/dsh/node_modules/@deepseek-ai/*`。
4. **两个 bundle 同时覆盖 `web` 行会打架**：bing 与 searxng 两个 bundle 都写了 `- id: web / config: searchProvider`，谁后应用谁赢。已把 bing bundle 关掉，只留 searxng 一个来源。

## 五、日常操作

```bash
# VPS：看容器 / 重启
ssh vps 'docker ps --filter name=searxng; docker compose -f /opt/searxng/docker-compose.yml restart'

# .223：看隧道
systemctl status searxng-tunnel --no-pager
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8888/healthz   # 期望 200

# .223：直接问 SearXNG
curl -s 'http://127.0.0.1:8888/search?q=测试&format=json' | head -c 300
```

**回到 Bing**（一条命令）：`plugin_manager set_bundle @local/dsh-searxng enabled=false` + `set_bundle @godchen520/dsh-web-search-bing enabled=true`。

## 六、实测

- `healthz` 200；单次搜索 1.2–2.5 秒；每次 20 条原始结果，工具层取前 8 条。
- 出结果引擎：`google cse`（主力，中英文均正常）、bing、wikipedia。
- `brave`、`duckduckgo` 在机房 IP 上持续 unresponsive（被反爬挡），属正常现象，不影响可用性。
- 验证方式：停掉隧道 → `web_search` 报 `SearXNG request failed (is the tunnel up?): http://127.0.0.1:8888`，证明选源确实是自建实例；恢复隧道后立即正常。

## 七、还没做的（可选项）

- 网页版已做，见第八节。
- 引擎调优：可按需在 `settings.yml` 里关掉持续失败或拖慢响应的引擎。

## 八、公网网页版（手机/浏览器直接用）

**地址 `https://example.com`　用户名 `user`　密码 `<REDACTED>`**

链路（没有新增任何公网端口，也没有新开证书——复用了通配符证书 `*.example.com`）：

```
手机/浏览器 → 公网 443 → VPS frps（按 SNI 分流）
            → .164:443 NPM（Basic Auth 拦在门口）
            → 192.168.1.223:8888（.223 上 SSH 隧道的 LAN 绑定）
            → VPS 容器 searxng
```

| 位置 | 改了什么 |
| --- | --- |
| .164 | `/opt/frp-vps/frpc.toml`：https + http 两个 `customDomains` 白名单各加一行 `example.com`（备份 `frpc.toml.bak-20261011-160614`），随后 `systemctl restart frpc-vps` |
| .164 | `/opt/web/npm-data/nginx/proxy_host/11-searxng.conf`：443 server 块 + Basic Auth + 反代到 `192.168.1.223:8888`；`location = /robots.txt` 免密返回 `Disallow: /`（`X-Robots-Tag: noindex` 由 SearXNG 自己下发） |
| .164 | `/opt/web/npm-data/access/searxng.htpasswd`：apr1 哈希，用户 `user` |
| .164 | 源文件留档 `/root/searxng-web/`（含一键脚本 `apply-on-164.sh`） |
| .164 | 主页 Homer `/opt/web/homer-assets/config.yml`：在「我的服务」里新增**第一个**入口 `搜索 → https://example.com/`（副标题 `SearXNG 聚合搜索`），副标题文案改为「单页十三个入口」；备份在 `/opt/web/homer-backups/` |
| .223 | `searxng-tunnel.service` 增加 `-L 192.168.1.223:8888:127.0.0.1:8888`，让 .164 的 NPM 能取到隧道 |
| DNS | `example.com` → A 记录 → `203.0.113.10`（DNS only / 灰云）✅ 2026-10-11 已生效 |

验证结果（**真实 DNS 链路**，未用 --resolve）：无密码 401、错误密码 401、正确密码 200；TLS 严格校验通过（Let's Encrypt 通配符 `*.example.com`，有效期至 2026-12-31）；页面标题 `SearXNG`，CSS/JS/favicon 全部 200；`POST /search` 200；公网搜索 20 条结果约 3 秒；整链路 0.68 秒（DNS 0.17s + TLS 0.37s）；其它站点（主页/con/dsh）不受 frpc 重启影响。

注意：`http://example.com`（80 端口）会被 frpc 的 http 代理把 Host 改写成 `example.com`，落到主页而**不是**跳转 —— 请一律用 `https://` 访问（手机浏览器直接收藏 https 地址）。

**回退**：删掉 `11-searxng.conf` → `docker exec npm nginx -s reload`；再 `systemctl restart frpc-vps` 即恢复原状（frpc.toml 用备份还原）。DSH 的搜索不受影响（它走 `127.0.0.1:8888`，不经过 NPM）。
