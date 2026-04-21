## 通过 Cloudflare DNS 验证申请 SSL 证书并配置自动续期（适用于 Debian 13 VPS）

本文介绍如何在 Debian 13 VPS 上，使用 Cloudflare DNS API + Certbot 申请 Let’s Encrypt 证书，并配置自动续期。

本方案适用于以下场景：
- 域名的权威 DNS 托管在 Cloudflare
- 需要申请根域名证书，例如 example.com
- 需要申请通配符证书，例如 *.example.com

注意：
- 通配符证书必须使用 DNS 验证，不能使用 HTTP 验证
- 建议后续始终使用固定的 --cert-name`，这样证书路径不会因为重复申请而变成 `-0001`、`-0002
- Web 服务请始终引用 /etc/letsencrypt/live/证书名/ 下的路径，而不要引用 archive 目录

---

### 步骤 1：获取 Cloudflare API Token

首先，您需要从 Cloudflare 获取一个 API Token，用于让 Certbot 自动写入 _acme-challenge TXT 记录完成 DNS 验证。

操作步骤：
1. 登录 Cloudflare Dashboard
2. 进入 My Profile > API Tokens
3. 点击 Create Token
4. 选择 Edit zone DNS 模板，或自行创建最小权限 Token
5. 为目标域名所在 Zone 授予 DNS 编辑权限
6. 保存生成的 API Token

建议：
- 使用 API Token，不要使用 Global API Key
- Token 仅授予所需域名的最小权限
- 请像密码一样妥善保管该 Token

---

### 步骤 2：安装 Certbot 和 Cloudflare 插件

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-dns-cloudflare
```

---

### 步骤 3：配置 Cloudflare API Token

创建 Cloudflare 凭据文件：

```bash
sudo mkdir -p /etc/letsencrypt
sudo nano /etc/letsencrypt/cloudflare.ini
```

写入以下内容：

```bash
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
```

保存后，修改文件权限：

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

---

### 步骤 4：首次申请证书

请将以下命令中的域名和邮箱替换为您自己的信息。

这里特别说明：
- --cert-name example.com 用于固定证书名
- 以后即使重复执行，也会尽量更新同一个证书 lineage，而不是创建 example.com-0001
- --dns-cloudflare-propagation-seconds 30 用于增加 DNS 生效等待时间，降低验证失败概率

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  --cert-name example.com \
  -d example.com \
  -d "*.example.com" \
  --agree-tos \
  --email yourname@example.com \
  --non-interactive
```

说明：
- `example.com`：根域名
- `*.example.com`：通配符域名
- `yourname@example.com`：Let’s Encrypt 注册邮箱
- `--cert-name example.com`：固定证书名称，后续服务配置引用这个路径即可

申请完成后，证书通常位于以下固定路径：

```bash
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
```

请让您的 Nginx、Apache、Postfix、Dovecot 等服务始终引用上面的路径，而不是引用带版本号的文件。

---

### 步骤 5：查看当前证书状态

可以使用以下命令查看当前 Certbot 管理的证书：

```bash
sudo certbot certificates
```

这条命令可以帮助您确认：
- 当前证书名称（Certificate Name）
- 证书包含的域名
- 到期时间
- 实际使用的证书路径

---

### 步骤 6：以后不要重复“新建申请”，而是优先使用续期

首次申请成功后，后续自动续期应使用：

```bash
sudo certbot renew --dry-run
```

如果返回类似以下内容，则说明自动续期流程正常：

```bash
Congratulations, all simulated renewals succeeded
```

正式续期时使用：

```bash
sudo certbot renew
```

说明：
- renew 会针对已有证书进行续期
- 它不会像反复手动执行新的 certonly 一样，轻易产生新的 -0001`、`-0002
- 所以生产环境建议把“重复申请”改为“续期”

---

### 步骤 7：如果需要修改已有证书中的域名，继续使用同一个 --cert-name

如果您后续想调整该证书包含的域名，不要新建另一个名字，而应继续指定同一个 `--cert-name`。

例如，仍然维护 example.com 这张证书：

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  --cert-name example.com \
  -d example.com \
  -d "*.example.com" \
  --non-interactive
```

如果以后要新增或减少域名，也建议继续基于同一个 --cert-name 调整，而不是放任 Certbot 自动生成新名字。

---

### 步骤 8：配置服务使用固定证书路径

请确保您的服务配置使用如下固定路径：

```bash
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
```

只要您一直使用同一个 `--cert-name example.com`，那么即使证书续期，服务配置文件也无需修改。

---

### 步骤 9：配置自动续期后的服务重载

Certbot 在 Debian 系统上通常已经自带自动续期机制，但您仍然应该验证一次：

```bash
systemctl list-timers | grep certbot
systemctl status certbot.timer
sudo certbot renew --dry-run
```

如果您的服务在证书更新后需要重载，推荐使用 deploy hook，而不是直接手写简单 cron。

以 Nginx 为例：

```bash
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

写入以下内容：

```bash
#!/bin/sh
systemctl reload nginx
```

保存后赋予执行权限：

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

这样当证书实际续期成功后，Certbot 会自动执行该脚本并重载 Nginx。

---

### 步骤 10：如果系统确实没有自动续期任务，再考虑手动添加

一般不建议在未检查现有机制前就手动添加 cron。

若确认系统没有可用的自动续期机制，再手动添加：

```bash
sudo crontab -e
```

添加以下内容：

```bash
0 3 * * * certbot renew --quiet
```

说明：
- 续期成功后的服务重载，优先通过 deploy hook 处理
- 不建议简单写成 certbot renew --quiet && systemctl reload nginx
- 因为 deploy hook 只会在证书真正更新后执行，逻辑更准确

---

### 常见问题 1：为什么重复运行申请命令后，证书目录变成了 `example.com-0001`？

原因是：
- Certbot 会把证书按 certificate lineage 管理
- 如果它判断这是一次新的证书申请，而不是对已有证书的更新，就可能创建新名字
- 这会导致路径变成：
  - /etc/letsencrypt/live/example.com-0001/
  - /etc/letsencrypt/live/example.com-0002/

解决方法：
1. 首次申请时就显式使用固定的 --cert-name
2. 服务始终引用 /etc/letsencrypt/live/固定证书名/
3. 后续优先使用 certbot renew
4. 如需重新申请或调整域名，也继续使用同一个 --cert-name

---

### 常见问题 2：如何查看当前有哪些证书名？

```bash
sudo certbot certificates
```

如果看到多个名称，例如：

```bash
Certificate Name: example.com
Certificate Name: example.com-0001
```

说明此前已经重复创建过多个 lineage。

建议：
- 选定一个长期使用的证书名，例如 example.com
- 后续所有申请和变更都使用 --cert-name example.com
- 同时将业务服务统一改为引用：

```bash
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
```

---

### 推荐的最终使用方式

首次申请：

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  --cert-name example.com \
  -d example.com \
  -d "*.example.com" \
  --agree-tos \
  --email yourname@example.com \
  --non-interactive
```

检查证书：

```bash
sudo certbot certificates
```

测试自动续期：

```bash
sudo certbot renew --dry-run
```

以后正式续期：

```bash
sudo certbot renew
```

服务配置固定引用：

```bash
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
```

---

【重要说明】

建议在首次申请时添加：

```bash
--cert-name example.com
```

作用：
- 固定证书名称，避免生成 example.com-0001 等新目录
- 服务配置可长期使用：
```bash
/etc/letsencrypt/live/example.com/
```

注意：
- 所有 -d 参数会被视为“完整域名集合”
- 每次执行都会覆盖当前证书中的域名列表（不是增量）

因此：
- 每次执行 certonly 时，请写全所有需要的域名
- 日常只需使用：
```bash
certbot renew
```

### 总结

在 Debian 13 VPS 上，使用 Cloudflare DNS 插件申请 Let’s Encrypt 通配符证书是可行的。

推荐实践如下：
- 使用 certbot + python3-certbot-dns-cloudflare
- 使用 Cloudflare API Token
- 首次申请时固定 --cert-name
- 服务始终引用 /etc/letsencrypt/live/固定证书名/
- 后续优先使用 certbot renew
- 通过 deploy hook 在续期成功后重载服务


### 卸载与还原说明

根据需求不同，可以选择以下三种方式：

---

### 一、仅停止使用（推荐）

适用于：不再使用证书，但保留环境

```bash
sudo systemctl stop certbot.timer
sudo systemctl disable certbot.timer
```

如果配置过自动重载脚本：

```bash
sudo rm -rf /etc/letsencrypt/renewal-hooks/
```

同时请修改服务配置（如 Nginx），移除证书路径引用：

```bash
/etc/letsencrypt/live/example.com/fullchain.pem
/etc/letsencrypt/live/example.com/privkey.pem
```

---

### 二、删除证书（保留 Certbot）

适用于：清空证书，重新申请

查看已有证书：

```bash
sudo certbot certificates
```

删除指定证书：

```bash
sudo certbot delete --cert-name example.com
```

删除 Cloudflare API 凭据（建议）：

```bash
sudo rm -f /etc/letsencrypt/cloudflare.ini
```

---

### 三、完全卸载（彻底还原）

适用于：恢复为未安装状态

卸载软件：

```bash
sudo apt remove --purge -y certbot python3-certbot-dns-cloudflare
sudo apt autoremove -y
```

删除所有数据：

```bash
sudo rm -rf /etc/letsencrypt
sudo rm -rf /var/lib/letsencrypt
sudo rm -rf /var/log/letsencrypt
```

删除定时任务：

```bash
sudo rm -f /etc/cron.d/certbot
sudo systemctl daemon-reexec
```

检查是否清理完成：

```bash
systemctl list-timers | grep certbot
```

无输出则说明已清理干净

---

### 补充说明

- DNS 验证产生的 _acme-challenge TXT 记录通常会自动删除
- 如 Cloudflare 中仍存在，可手动删除（非必须）

---

### 总结

- 停用：关闭 timer + 修改服务配置
- 重置：删除证书
- 彻底：卸载 + 删除目录

这样可以避免重复申请后出现 -0001`、`-0002 导致的证书路径变化问题。
