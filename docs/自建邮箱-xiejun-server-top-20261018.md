# 自建邮箱 example.com（2026-10-18 完成）

## 总览
- 域名 `example.com` 的自建邮箱：
  - **收信**：本人家 VPS（香港 203.0.113.10，docker-mailserver v16.0.1）直接收，公网 25 入站已验证可达。
  - **发信**：VPS 通过 Brevo 免费中继（smtp-relay.brevo.com:587）投递，实测送达外部邮箱。
- 收信链路：外部 → MX `example.com` → VPS:25 → Dovecot（IMAP/SSL 993）。
- 发信链路：客户端 → VPS:587/465（SMTP AUTH）→ Brevo 中继 → 收件方。
- 域名认证：Brevo 已完成根域 `example.com` 与子域 `example.com` 认证；发信 DKIM 由 Brevo 用我们域名签名。
- 2026-10-18 mail-tester.com 实测：`dmarc=pass`、`dkim=pass (d=example.com, s=brevo2)`、`spf=pass`；SpamAssassin 判定 No/4.8/5.0（未达垃圾阈值），评分 4.7/10。

## 邮箱账号
| 邮箱地址 | 密码 | 说明 |
|---|---|---|
| user@example.com | <REDACTED> | 唯一账号（已验证可用于发信） |
| 其它任意地址@example.com | <REDACTED> | 不再转发（无 catch-all；未创建的地址直接被拒收，各账号收件箱独立） |

## 客户端设置（手机 / 电脑邮件 App）
- **IMAP**：`example.com`，端口 **993**，SSL/TLS
- **SMTP（发信）**：`example.com`，端口 **465**（SSL/TLS）或 **587**（STARTTLS）
- 用户名 = user@example.com；密码 <REDACTED>

## 服务端位置（VPS 203.0.113.10）
- `/opt/mail/compose.yaml`、`/opt/mail/mailserver.env`
  - 中继配置：`RELAY_HOST=smtp-relay.brevo.com` / `RELAY_PORT=587` / `RELAY_USER=user@example.com` / `RELAY_PASSWORD=<REDACTED> SMTP key>`
  - 备份：`/opt/mail/mailserver.env.bak-20261018-063233`
- 邮箱数据：`/opt/mail/mailstore.img`（8G loop ext4，挂载到 `/opt/mail/dms-data`）
- 邮件证书：`/opt/mail/certs/`（Let's Encrypt，HTTP-01 经 frp 隧道自动续期，cron `33 1,7,13,19 * * *`）
- 常用命令：
  - 加账号：`docker exec mailserver setup email add <addr> <password>`
  - 删账号：`docker exec mailserver setup email delete <addr>`
  - 列账号：`docker exec mailserver setup email list`
  - 重启：`cd /opt/mail && docker compose restart mailserver`

## Brevo（发信中继）
- 登录：`https://app.brevo.com`，账号 `user@example.com`，密码 <REDACTED>（手机已验证；Free 计划 300 封/天）
- SMTP 登录名：`user@example.com`；SMTP key 见 `/opt/mail/mailserver.env` 的 `RELAY_PASSWORD`
- 已认证域名：`example.com`、`example.com`；已验证发件人：`Xiejun Homelab <user@…>`、`Xiejun <1@…>`

## DNS 记录（Cloudflare 权威；日常在 E8.HK 域名解析面板管理）
| 类型 | 名称 | 值 |
|---|---|---|
| MX | @ | example.com（优先级 10） |
| A | mail | 203.0.113.10 |
| TXT | @ | `brevo-code:3d30698f50a05e5a2cb07600130380c5`（Brevo 域名认证） |
| CNAME | brevo1._domainkey | b1.user-server-top.dkim.brevo.com |
| CNAME | brevo2._domainkey | b2.user-server-top.dkim.brevo.com |
| TXT | _dmarc | `v=DMARC1; p=quarantine; pct=100; rua=mailto:user@example.com; adkim=r; aspf=r` |
| TXT | mail._domainkey | 本机 OpenDKIM 公钥（selector `mail`） |
| TXT | send | `brevo-code:3d30698f50a05e5a2cb07600130380c5`（子域认证） |
| CNAME | brevo1._domainkey.send / brevo2._domainkey.send | b1./b2.send-user-server-top.dkim.brevo.com |
| TXT | _dmarc.send | `v=DMARC1; p=none; rua=mailto:user@example.com` |

> 注意：E8.HK 面板「同一主机名同类型只允许一条记录」，根域原本的 SPF TXT（`v=spf1 mx include:spf.smtp2go.com ~all`）为了放 Brevo 验证记录已删除。发信全部走 Brevo 的 SPF（envelope 域为 Brevo），我们的域不再需要 SPF；DMARC 靠 DKIM 对齐通过。

## 已知限制 / 可选优化
- `.top` 顶级域在 SpamAssassin 有固定扣分（约 3.5 分，含 FROM_SUSPICIOUS_NTLD 等），代码层面无法消除；当前仍低于垃圾阈值。正式使用前建议实际给 Gmail/QQ 各发一封确认进收件箱。
- Brevo 事务邮件默认会改写正文（加追踪/List-Unsubscribe 等），导致本机 OpenDKIM 的签名失效；但 Brevo 用我们域名重签的 DKIM 有效，DMARC 通过，不影响投递。
- 可选：Brevo 品牌化子域 / 自定义 Return-Path（需再补 DNS 记录）；安装 webmail（如 SnappyMail）可在浏览器收发；把本机 SPF 加回去（需要 E8 面板支持一主机名多条 TXT）。
- 清理项（本次调试残留）：chromium 9232/9233 调试进程、SSH SOCKS 127.0.0.1:1080、`/root/cdp_probe.py`、`/root/brevo_signup*.py`、`/tmp/s2g` 等。

---
生成时间：2026-10-18 17:5x（Asia/Shanghai）

## 踩坑与经验（2026-10-18）
- 腾讯云 VPS **出站 TCP 25 被网络层封锁**（安全组放行无效），发信必须走 ESP 中继；**入站 25 正常**（check-host.net 实测）。
- ESP 选型：Brevo 免费版（300 封/天）最合适，Sender 只需邮箱验证即可；其他多数 ESP 要手机/验证码。
- Brevo 注册页 `https://onboarding.brevo.com/account/register` 是 React SPA，提交卡死根因是 `await reCAPTCHA`（Google 在国内不可达）；用 SSH SOCKS 到香港 VPS 后正常（`ssh -D 1080 vps` + chromium `--proxy-server=socks5://127.0.0.1:1080`）。
- Brevo 免费账号发信前需要手机验证（dashboard 顶部 Verify now）；SMTP key 也可以直接调用 `POST https://account-management-backend.brevo.com/api/keys/smtp`（body 带 `re_captcha_value`，reCAPTCHA sitekey `6LdI49cpAAAAAGpS6QLioPce37GoA9N2oujERXfH`）。
- Brevo 事务邮件**默认改写正文**（加追踪像素/List-Unsubscribe），会破坏本机 OpenDKIM 签名；解法=在 Brevo 完成**域名认证**，由 Brevo 用我们域名重签 DKIM（DMARC 靠 DKIM 对齐通过）。
- 认证域名必须与 **From 地址的域完全一致**：只认证 `example.com` 时，从 `@example.com` 发信不会用该签名；必须再认证根域。
- E8.HK 域名解析面板**同一主机名同类型只允许一条记录** → 根域原 SPF TXT 为放 Brevo 验证 TXT 被替换（发信走 Brevo 的 SPF，不影响；DMARC 由 DKIM 保障）。
- 验证工具：mail-tester.com（地址 `test-<id>@srv1.mail-tester.com`，报告 `https://www.mail-tester.com/check?id=<完整地址>`）。本次结果：`dmarc=pass`、`dkim=pass (d=example.com, s=brevo2)`、`spf=pass`，评分 4.7/10；扣分主要来自 `.top` 后缀固有声誉（约 3.5 分，无法消除）。
- 本机 DMS 仍会给经 Brevo 的邮件加 OpenDKIM 签名（selector `mail`），但正文被动过会显示 `dkim=fail`；不影响 DMARC（Brevo 签名有效）。若要日志干净可关闭 OpenDKIM 出站签名。
- 收信侧 DMS 实战要点：`OVERRIDE_HOSTNAME=example.com`、`NETWORK_INTERFACE=ens17`（host 网络）、`SSL_TYPE=manual` 手动证书；`/opt/mail/mailstore.img` 为 8G loop 挂载。

## 网页版邮箱 Webmail（2026-10-23 新增）
- 地址：**https://example.com/**（复用 example.com 证书 + frp 443 隧道，无需新 DNS）
- 栈：`/opt/mail-web/`（`djmaze/snappymail` + `caddy:2-alpine`），Caddy 用 `/opt/mail/certs` 终止 TLS，frpc 代理 `webmail-https`（type=https，localPort 8443，customDomains=["example.com"]，见 `/opt/frp/acme-frpc.toml`）
- 登录：账号 `user@example.com`，密码 <REDACTED>；后台管理 `https://example.com/?admin`（密码见 VPS `/opt/mail/credentials.txt`）
- 多账号：设置 → 账号 → 添加一个账户；已绑定 **126 邮箱** `user@example.com`（IMAP imap.126.com:993 SSL / SMTP smtp.126.com:465 SSL，密码用 126 授权码）
- 兼容性补丁：网易 126/163 的 IMAP 要求客户端发送 RFC2971 客户端 ID，否则 SELECT 报 `Unsafe Login`。SnappyMail 默认不发，已 patch `/opt/mail-web/patches/ImapClient.php`（登录成功后发送 `ID ("name" "SnappyMail" ...)`）并通过 compose bind mount 挂进容器；**镜像升级到新版本时若路径 2.38.2 变化需同步更新挂载路径**。
- 证书续期：acme.sh 的 reloadcmd 已改为同时 `docker compose restart mailserver` 和 `docker compose restart webmail-tls`
- 主页入口：Homer 第二项「邮箱」（subtitle「SnappyMail · 网页邮箱」）；副标题已改「单页十六个入口」
- 验证：webmail 登录 ✅；1@ 收发 ✅；126 账号切换后收件箱正常加载 ✅；126 → 1@ 外网投递实测 SPF/DKIM/DMARC 全 pass ✅

### 追加：yeah.net 邮箱也已绑定（2026-10-23）
- 账号 `user@example.com`，已在网页邮箱「设置 → 账号」中绑定并验证收件箱正常（切换账号可见）
- 服务地址：IMAP `imap.yeah.net:993` SSL / SMTP `smtp.yeah.net:465` SSL，密码用 yeah.net 授权码（见 VPS `/opt/mail/credentials.txt`）
- 网易 yeah.net 在开启 IMAP/SMTP 时直接生成授权码（若已绑定安全手机无需短信）；本账号安全手机已绑定
- 与 126 一样依赖 SnappyMail 的 IMAP ID 补丁

### 追加：SnappyMail 管理后台密码 <REDACTED>）
- 管理后台 `https://example.com/?admin` 密码已从自动生成的初始密码改为 **`<REDACTED>`**（与邮箱账号一致，方便记忆），登录验证通过、默认密码警告已消失
- 后台用户名：`admin`

## 告警通知渠道改为邮箱（2026-10-23）
- 原手机告警走 ntfy；现改为**邮箱推送**，收件地址默认 `user@example.com`（手机邮件 App / 网页邮箱均可收）。
- 监控台（.164 `/opt/dashboard/notify.py`）：新增 `email` 通道类型（SMTP 465 SSL，账号 1@，密码 <REDACTED>.json` 的 channels 已切换为 email；原 ntfy 配置备份为 `notify.py.bak-email-*` / `notify.json.bak-email-*`（`send_ntfy` 代码保留，以后想加回改配置即可）。
- Gatus（.223 `/opt/gatus/config/config.yaml`）：alerting 从 `ntfy` 切换为 `email`（type: email, host example.com:587, from/to 1@，STARTTLS+AUTH）；配置备份 `config.yaml.bak-email-*`。已用临时失败 endpoint 实测收到 Gatus 告警邮件，测试 endpoint 已移除。
- 验证：`notify.py --test` 邮件已收到（主题「⚠️ 告警推送测试」）；Gatus 告警邮件已收到（主题「🧪 测试/邮件告警…」）。
- 改进：Windows .155「按需关机」节点在 CLI `--check` 里不再误报离线（已补 optional 标记）。
- 注意：邮件告警依赖自家邮件服务器，若 VPS/邮件服务整体宕机则收不到告警；如需更强保障可再加一个备用渠道（ntfy/webhook）。

### 追加：告警改为「邮箱 + ntfy」双通道（2026-10-23）
- 按用户要求，两个渠道同时发送：邮件（user@example.com）和 ntfy（topic `<REDACTED_TOPIC>`）。
- 监控台 notify.json channels = [email, ntfy]，`notify.py --test` 返回「已发送到 2 个通道」；ntfy 侧实测收到「⚠️ 告警推送测试」。
- Gatus `alerting` 同时配置 email + ntfy，16 个 endpoint 的 alerts 均含两种类型；临时故障项实测两个渠道都收到（ntfy 主题收到「Gatus: 🧪 测试/告警双通道自测」），测试项已移除。
- ntfy.sh 免费版消息缓存约 12 小时，可用 `curl "https://ntfy.sh/<topic>/json?poll=1&since=15m"` 核对历史。
