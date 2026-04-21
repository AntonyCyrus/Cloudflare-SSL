## 通过Cloudflare DNS验证申请SSL证书并配置自动更新

### 步骤 1：获取 Cloudflare API Token
首先，您需要从 Cloudflare 获取一个 API token，该 token 允许 Certbot 使用 Cloudflare 的 DNS 来完成 DNS 验证。

登录到 Cloudflare Dashboard。
进入 "My Profile"（个人资料） > API Tokens。
点击 "Create Token"。
在模板中选择 "Zone DNS"（Zone DNS 编辑权限）。
为您的域名创建一个 API Token，确保它有足够的权限来管理 DNS 记录。
保存生成的 API Token。

### 步骤 2：安装 Certbot 和 Cloudflare 插件

```bash
sudo apt update
sudo apt install certbot python3-certbot-dns-cloudflare
```

### 步骤 3：配置 Cloudflare API Token
为了让 Certbot 使用 Cloudflare API 来进行 DNS 验证，您需要创建一个配置文件来存储 API Token。假设您将该文件保存在 /etc/letsencrypt/cloudflare.ini。

#### 创建配置文件 cloudflare.ini 

```bash
sudo nano /etc/letsencrypt/cloudflare.ini
```

#### 将以下内容添加到文件中

```bash
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
```

保存修改并关闭编辑器。对于 nano，按 CTRL + O 保存，然后按 CTRL + X 退出。

#### 修改文件权限以确保安全

```bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

### 步骤 4： 申请 SSL 证书
使用 Certbot 来申请 SSL 证书，并通过 Cloudflare DNS 验证。

##### 运行以下命令来申请证书：
请将 example.com 替换为您的域名。此命令会自动使用 Cloudflare 的 DNS API 来完成 DNS 记录的创建和验证。

```bash
sudo certbot certonly \
    --dns-cloudflare \
    --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
    -d example.com -d "*.example.com" \
    --agree-tos \
    --email yourname@domain.com \
    --non-interactive \
    -v
```
注意此处需要修改三处变量：根域名，通配符域名和Let’s Encrypt注册邮箱

运行完指令后等待申请完成。注意证书和密钥保存位置提示

Certificate is saved at: 

Key is saved at:         

### 步骤 5：配置自动续期
Certbot 默认会自动配置证书的续期，但是您需要确保 Certbot 正确设置了自动续期任务。

#### 验证 Certbot 自动续期是否正常工作
Certbot 会创建一个 cron 任务或 systemd 服务来自动续期证书。您可以检查续期配置

```bash
sudo certbot renew --dry-run
```

返还结果
Congratulations, all simulated renewals succeeded: 
意味着自动续期成功

#### 手动验证续期状态
运行以下命令，查看当前证书的状态和续期计划：

```bash
sudo certbot certificates
```

#### 添加 Cron 任务（如果需要）
如果系统没有自动添加续期任务，您可以手动添加一个 cron 任务，每天检查证书并在必要时续期。

编辑

```bash
sudo crontab -e
```

添加以下行以确保证书每月自动续期

```bash
0 0 * * * certbot renew --quiet && systemctl reload nginx
```

保存修改并关闭编辑器。对于 nano，按 CTRL + O 保存，然后按 CTRL + X 退出。
