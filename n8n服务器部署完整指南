# n8n 服务器部署完整指南（完善版）

## 🚨 重要提醒

**部署顺序很关键！** 必须先获取 SSL 证书，然后再配置完整的 Nginx，因为 Nginx 配置文件中会引用证书路径。如果证书文件不存在，Nginx 将无法启动。

## 1. 基础 Docker 部署方案

n8n 官方提供的基础 Docker 部署命令：

```bash
docker volume create n8n_data
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n

```

**⚠️ 限制说明：**

- ✅ **本地开发**：这种方式适用于本地开发环境
- ❌ **云端部署**：在云端服务器上会遇到安全问题，浏览器要求使用 HTTPS

**为什么需要 HTTPS？**

- 现代浏览器对非本地的 HTTP 连接有安全限制
- n8n 的某些功能（如 Webhook）需要 HTTPS 才能正常工作
- HTTPS 提供数据加密和身份验证

## 2. 域名准备

### 2.1 购买域名

- 在 namesilo、Cloudflare 或其他域名注册商购买域名（如：mydomain.com）

### 2.2 配置 DNS 解析

- 添加 A 记录：`n8n.mydomain.com` → 你的服务器 IP 地址
- 等待 DNS 传播生效（通常几分钟到几小时）

### 2.3 验证 DNS 解析

```bash
# 验证域名是否正确解析到服务器IP
dig n8n.mydomain.com
# 或者
nslookup n8n.mydomain.com

```

## 3. SSL 证书与 Nginx 的关系

### 🔐 理解证书的作用

**SSL 证书就像网站的"身份证"**：

- **加密功能**：确保浏览器和服务器之间数据传输的安全
- **身份验证**：证明网站的真实性，防止中间人攻击

### 🔄 Nginx 和证书的关系

**Nginx 是"门卫"，证书是"钥匙"**：

- Nginx 接收 HTTPS 请求时需要使用证书建立安全连接
- 证书文件必须在 Nginx 配置中的路径真实存在
- 如果证书文件不存在，Nginx 启动时会报错并拒绝启动

## 4. SSL 证书配置（关键步骤）

### 4.1 安装 Certbot

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install certbot python3-certbot-nginx

# CentOS/RHEL
sudo yum install certbot python3-certbot-nginx

```

### 4.2 创建临时 Nginx 配置

**为什么需要临时配置？**

- Certbot 需要通过 HTTP 访问你的域名来验证所有权
- 完整的 HTTPS 配置需要引用还不存在的证书文件
- 先用简单的 HTTP 配置让域名可访问，完成验证后再升级

```bash
sudo nano /etc/nginx/conf.d/n8n.mydomain.com.conf

```

**临时配置内容：**

```
server {
    listen 80;
    server_name n8n.mydomain.com;

    # Certbot 域名验证路径
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # 临时响应，证明服务器在线
    location / {
        return 200 "Server is ready for SSL setup";
        add_header Content-Type text/plain;
    }
}

```

### 4.3 应用临时配置

```bash
# 创建验证文件目录
sudo mkdir -p /var/www/certbot

# 测试配置语法
sudo nginx -t

# 如果测试通过，重载配置使其生效
sudo systemctl reload nginx

```

**这一步的作用：**

- 让临时配置立即生效
- 使域名可以通过 HTTP 访问
- 为 Certbot 验证做准备

### 4.4 获取 SSL 证书

```bash
sudo certbot --nginx -d n8n.mydomain.com

```

**Certbot 验证过程：**

1. **生成验证文件**：Certbot 在 `/var/www/certbot/.well-known/acme-challenge/` 下创建验证文件
2. **Let's Encrypt 验证**：通过访问 `http://n8n.mydomain.com/.well-known/acme-challenge/验证码` 确认域名所有权
3. **Nginx 响应验证**：你的临时配置处理这个请求，返回验证文件
4. **验证成功**：Let's Encrypt 确认你拥有该域名
5. **颁发证书**：证书保存到 `/etc/letsencrypt/live/n8n.mydomain.com/`

**成功后会看到：**

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/n8n.mydomain.com/fullchain.pem
Key is saved at: /etc/letsencrypt/live/n8n.mydomain.com/privkey.pem

```

## 5. 完整的 Nginx 反向代理配置

**现在证书文件已存在，可以配置完整的 HTTPS 反向代理了**

```bash
sudo nano /etc/nginx/conf.d/n8n.mydomain.com.conf

```

**完整配置内容：**

```
# HTTP 服务块 - 处理 80 端口请求
server {
    listen 80;
    server_name n8n.mydomain.com;

    # 保留 Certbot 续订证书时需要的路径
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # 将所有其他 HTTP 请求永久重定向到 HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS 服务块 - 处理 443 端口请求
server {
    listen 443 ssl http2;
    server_name n8n.mydomain.com;

    # SSL 证书配置（路径由 Certbot 生成，现在文件已存在）
    ssl_certificate /etc/letsencrypt/live/n8n.mydomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.mydomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # 反向代理到 n8n 容器
    location / {
        # 转发到本地 5678 端口的 n8n 服务
        proxy_pass http://127.0.0.1:5678;

        # WebSocket 支持和协议识别的必要头信息
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_cache_bypass $http_upgrade;
    }
}

```

### 5.1 验证和应用完整配置

```bash
# 测试完整配置语法
sudo nginx -t

# 如果测试通过，重载配置
sudo systemctl reload nginx

```

## 6. 启动 n8n Docker 容器

**重要的环境变量配置：**

```bash
# 创建持久化数据卷
docker volume create n8n_data

# 启动容器（后台运行）
docker run -dit --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_HOST="n8n.mydomain.com" \
  -e WEBHOOK_URL="https://n8n.mydomain.com/" \
  -e N8N_PROTOCOL="https" \
  docker.n8n.io/n8nio/n8n

```

### 6.1 环境变量说明

- **`N8N_HOST`**：告诉 n8n 它的主机名，用于生成正确的链接
- **`WEBHOOK_URL`**：设置 Webhook 的完整 HTTPS URL，确保外部服务能正确回调
- **`N8N_PROTOCOL`**：明确指定使用 HTTPS 协议，避免协议混乱

### 6.2 容器管理命令

```bash
# 查看容器状态
docker ps

# 查看容器日志
docker logs n8n

# 停止容器
docker stop n8n

# 重启容器
docker restart n8n

# 删除容器（数据会保留在数据卷中）
docker rm -f n8n

```

## 7. 验证部署

### 7.1 基础验证

```bash
# 检查 Nginx 状态
sudo systemctl status nginx

# 检查端口监听
sudo netstat -tlnp | grep -E ':(80|443|5678)'

# 检查容器状态
docker ps | grep n8n

```

### 7.2 功能验证

1. **HTTPS 访问**：访问 `https://n8n.mydomain.com`
2. **证书验证**：确保浏览器地址栏显示安全锁图标
3. **HTTP 重定向**：访问 `http://n8n.mydomain.com` 应自动跳转到 HTTPS
4. **应用功能**：验证 n8n 界面可以正常加载和使用

## 8. 故障排除

### 8.1 常见问题

**问题1：Nginx 启动失败**

```bash
# 检查错误日志
sudo tail -f /var/log/nginx/error.log

# 常见原因：证书文件不存在或路径错误

```

**问题2：无法访问网站**

```bash
# 检查防火墙
sudo ufw status
sudo ufw allow 80
sudo ufw allow 443

# 检查 DNS 解析
dig n8n.mydomain.com

```

**问题3：n8n 容器无法启动**

```bash
# 查看详细日志
docker logs n8n -f

# 检查端口占用
sudo lsof -i :5678

```

### 8.2 调试技巧

```bash
# 测试代理转发
curl -H "Host: n8n.mydomain.com" http://127.0.0.1:5678

# 测试 SSL 证书
openssl s_client -connect n8n.mydomain.com:443 -servername n8n.mydomain.com

```

## 9. 维护和安全

### 9.1 证书自动续期

```bash
# 测试自动续期
sudo certbot renew --dry-run

# Certbot 会自动创建 cron 任务，通常无需手动干预
# 查看自动续期任务
sudo systemctl list-timers | grep certbot

```

### 9.2 数据备份

```bash
# 备份 n8n 数据
docker run --rm -v n8n_data:/source -v $(pwd):/backup ubuntu tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz -C /source .

# 恢复数据
docker run --rm -v n8n_data:/target -v $(pwd):/backup ubuntu tar xzf /backup/n8n-backup-YYYYMMDD.tar.gz -C /target

```

### 9.3 安全建议

- **定期更新**：保持 Nginx、Docker 和 n8n 镜像的最新版本
- **访问控制**：考虑添加 IP 白名单或 HTTP 认证
- **监控日志**：定期查看访问日志，发现异常访问
- **防火墙**：只开放必要的端口（80、443、22）

```bash
# 设置基础防火墙规则
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443

```

## 10. 总结

**关键要点回顾：**

1. **部署顺序至关重要**：域名 → 临时配置 → 证书获取 → 完整配置 → 容器启动
2. **理解证书原理**：Nginx 需要证书文件存在才能处理 HTTPS 请求
3. **环境变量配置**：正确的环境变量确保 n8n 生成正确的 HTTPS 链接
4. **安全考虑**：使用 HTTPS、定期备份、保持更新

**部署成功标志：**

- ✅ 可以通过 HTTPS 正常访问 n8n
- ✅ HTTP 请求自动重定向到 HTTPS
- ✅ 浏览器显示安全锁图标
- ✅ Webhook 功能正常工作

恭喜！现在你可以安全地使用 n8n 进行自动化工作流开发了！🎉
