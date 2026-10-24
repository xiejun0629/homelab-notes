# Vaultwarden 自建密码 <REDACTED>）

## 访问地址
https://example.com

## 部署信息
- 主机：.164（nas），Docker 容器 `vaultwarden`，镜像 `vaultwarden/server:latest`（版本 1.37.4）
- 端口：宿主机 8091 -> 容器 80
- 数据目录：`/opt/web/vaultwarden-data`（已加入 sync.sh 每日备份）
- 环境变量：DOMAIN=https://example.com、SIGNUPS_ALLOWED=true（注册完成后关闭）、INVITATIONS_ALLOWED=true、ADMIN_TOKEN 存于 `/root/.vaultwarden-admin-token`（600）、TZ=Asia/Shanghai
- 管理面板：https://example.com/admin （用 ADMIN_TOKEN 登录）

## 网络链路
- Cloudflare：A 记录 `vault` -> 203.0.113.10（DNS only，灰云）
- frpc（.164）：frpc.toml 的 web-https / web-http 代理已加入 example.com
- NPM：`/opt/web/npm-data/nginx/proxy_host/23-vault.conf`（80 ACME + 443 反代 8091，WebSocket 支持，client_max_body_size 525M）
- 证书：与 example.com 共用 Let's Encrypt 多域名证书（10 个 SAN：apex/blog/chat/dsh/git/ledger/memory/photo/video/vault），有效期至 2027-01-22；renew_hook 已配 docker exec npm nginx -s reload

## 已验证
- https://example.com/ 200，API /api/config domain 正确，Web Vault 登录/注册页正常渲染
- 所有原有域名 HTTPS 正常（blog 200、memory 200、git 200、chat 200 等）

## 待办
1. 用户注册第一个账号（邮箱 + 主密码；建议密码短语，如"咖啡-火车-葡萄-雨伞-27"，写纸质备份）
2. 注册后把 SIGNUPS_ALLOWED 改为 false 并重启容器（仅邀请制）
3. 把现有重复密码（NPM/Gitea/Memos/PhotoPrism/Open WebUI/博客后台/.155 Windows/路由器）逐个改成随机强密码 <REDACTED>
4. 重要服务开 TOTP 二次验证（验证码也可存 Vaultwarden）
5. 博客后台可加 TOTP（后续可做）
6. 每周确认备份包含 vaultwarden-data（已接入 sync.sh）

## 进度更新（2026-10-24 18:5x）
- 用户已注册：`user@example.com`（用户 id 6cedde2d…，2026-10-24 18:42 创建）
- `SIGNUPS_ALLOWED=false` 已生效（容器已重建，仅邀请注册）；`INVITATIONS_ALLOWED=true`
- ⚠️ 主密码已在聊天中暴露，建议用户尽快在 Web Vault「安全设置 → 修改主密码」更换，之后不再外发
- 待办：主密码更换、服务密码迁移（NPM/Gitea/Memos/PhotoPrism/Open WebUI/博客后台/.155/路由器）、TOTP 2FA、博客后台 TOTP
