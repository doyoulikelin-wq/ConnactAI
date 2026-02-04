# 阿里云轻量应用服务器部署指南

## 前置准备

### 1. 购买阿里云轻量应用服务器

访问：https://www.aliyun.com/product/swas

**推荐配置**：
- **地域**：香港（无需备案）或离用户最近的节点
- **镜像**：Ubuntu 22.04 LTS
- **套餐**：2核4G内存 60GB存储 200Mbps带宽
- **预计费用**：¥400-600/年

### 2. 配置防火墙规则

在阿里云控制台 → 防火墙，开放端口：
- `22`：SSH 登录
- `80`：HTTP
- `443`：HTTPS（可选）

### 3. 获取服务器信息

```bash
# 记录以下信息
服务器公网 IP: ___________
SSH 密码/密钥: ___________
```

---

## 快速部署（方案A：手动部署）

### Step 1: SSH 登录服务器

```bash
# 本地终端执行
ssh root@你的服务器IP
# 输入密码
```

### Step 2: 一键部署脚本

复制以下脚本并在服务器执行：

```bash
# ========== 自动化部署脚本 ==========
#!/bin/bash
set -e

echo "🚀 开始部署 Connact.ai..."

# 1. 更新系统
echo "📦 更新系统包..."
apt update && apt upgrade -y

# 2. 安装依赖
echo "📦 安装 Python 和工具..."
apt install -y python3.10 python3.10-venv python3-pip nginx supervisor git

# 3. 创建应用用户（安全起见，不用 root 运行）
if ! id "connact" &>/dev/null; then
    useradd -m -s /bin/bash connact
    echo "✅ 创建用户 connact"
fi

# 4. 克隆代码
echo "📥 克隆 GitHub 仓库..."
cd /home/connact
if [ -d "Connact.ai" ]; then
    echo "⚠️  目录已存在，更新代码..."
    cd Connact.ai
    sudo -u connact git pull
else
    sudo -u connact git clone https://github.com/doyoulikelin-wq/Connact.ai.git
    cd Connact.ai
fi

# 5. 创建虚拟环境并安装依赖
echo "📦 安装 Python 依赖..."
sudo -u connact python3 -m venv venv
sudo -u connact ./venv/bin/pip install --upgrade pip
sudo -u connact ./venv/bin/pip install -r requirements.txt
sudo -u connact ./venv/bin/pip install gunicorn

# 6. 配置环境变量
echo "⚙️  配置环境变量..."
if [ ! -f ".env" ]; then
    cat > .env << 'EOF'
# API Keys (请填写你的密钥)
GEMINI_API_KEY=your_gemini_api_key_here
# OPENAI_API_KEY=your_openai_key_here  # 可选

# Flask 配置
SECRET_KEY=$(openssl rand -hex 32)
FLASK_ENV=production

# 邀请码（可选）
INVITE_CODE=beta2026

# Google OAuth（可选）
# GOOGLE_CLIENT_ID=your_google_client_id
# GOOGLE_CLIENT_SECRET=your_google_client_secret

# 数据库
DATA_DIR=/home/connact/Connact.ai/data
DB_PATH=/home/connact/Connact.ai/data/app.db
EOF
    chown connact:connact .env
    echo "⚠️  请编辑 /home/connact/Connact.ai/.env 填入真实的 API Key!"
    echo "   运行：nano /home/connact/Connact.ai/.env"
fi

# 7. 创建数据目录
mkdir -p /home/connact/Connact.ai/data
chown -R connact:connact /home/connact/Connact.ai/data

# 8. 配置 Supervisor（进程守护）
echo "⚙️  配置 Supervisor..."
cat > /etc/supervisor/conf.d/connact.conf << EOF
[program:connact]
command=/home/connact/Connact.ai/venv/bin/gunicorn -w 4 -b 127.0.0.1:5000 --timeout 120 app:app
directory=/home/connact/Connact.ai
user=connact
autostart=true
autorestart=true
stderr_logfile=/var/log/connact.err.log
stdout_logfile=/var/log/connact.out.log
environment=PATH="/home/connact/Connact.ai/venv/bin"
EOF

# 9. 配置 Nginx（反向代理）
echo "⚙️  配置 Nginx..."
cat > /etc/nginx/sites-available/connact << 'EOF'
server {
    listen 80;
    server_name _;  # 改为你的域名，或保持 _ 使用 IP 访问

    client_max_body_size 16M;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 120s;
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;
    }

    # 静态文件缓存（如果有）
    location /static {
        alias /home/connact/Connact.ai/static;
        expires 30d;
    }
}
EOF

# 启用站点
ln -sf /etc/nginx/sites-available/connact /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default  # 删除默认站点

# 10. 测试配置
echo "🧪 测试 Nginx 配置..."
nginx -t

# 11. 启动服务
echo "🚀 启动服务..."
supervisorctl reread
supervisorctl update
supervisorctl restart connact
systemctl restart nginx

echo ""
echo "✅ 部署完成！"
echo ""
echo "📋 下一步操作："
echo "1. 编辑环境变量：nano /home/connact/Connact.ai/.env"
echo "2. 填入真实的 GEMINI_API_KEY"
echo "3. 重启应用：supervisorctl restart connact"
echo ""
echo "🌐 访问地址："
echo "   http://$(curl -s ifconfig.me)"
echo ""
echo "📊 查看日志："
echo "   tail -f /var/log/connact.out.log  # 应用日志"
echo "   tail -f /var/log/connact.err.log  # 错误日志"
echo "   tail -f /var/log/nginx/access.log # Nginx 访问日志"
echo ""
echo "🔧 常用命令："
echo "   supervisorctl status             # 查看服务状态"
echo "   supervisorctl restart connact    # 重启应用"
echo "   systemctl restart nginx          # 重启 Nginx"
```

**执行方式**：

```bash
# 方法1：直接粘贴执行
# 将上述脚本复制，在服务器终端粘贴运行

# 方法2：保存为文件执行
nano deploy.sh
# 粘贴脚本内容，Ctrl+X 保存
chmod +x deploy.sh
./deploy.sh
```

### Step 3: 配置 API Key

```bash
# 编辑环境变量
nano /home/connact/Connact.ai/.env

# 修改以下行：
GEMINI_API_KEY=实际的密钥

# 保存后重启（Ctrl+X, Y, Enter）
supervisorctl restart connact
```

### Step 4: 验证部署

```bash
# 查看服务状态
supervisorctl status

# 应该看到：
# connact    RUNNING   pid 1234, uptime 0:00:05

# 查看日志
tail -f /var/log/connact.out.log

# 访问网站
curl http://localhost
```

### Step 5: 浏览器访问

打开浏览器，访问：`http://你的服务器IP`

---

## 更新代码（日常维护）

当你在本地 push 新代码后，在服务器执行：

```bash
# SSH 登录服务器
ssh root@你的服务器IP

# 切换到应用目录
cd /home/connact/Connact.ai

# 更新代码
sudo -u connact git pull

# 如果有新依赖
sudo -u connact ./venv/bin/pip install -r requirements.txt

# 重启应用
supervisorctl restart connact

# 查看日志确认
tail -f /var/log/connact.out.log
```

**或创建快捷脚本**：

```bash
# /home/connact/update.sh
#!/bin/bash
cd /home/connact/Connact.ai
sudo -u connact git pull
sudo -u connact ./venv/bin/pip install -r requirements.txt
supervisorctl restart connact
echo "✅ 更新完成！"
tail -n 20 /var/log/connact.out.log
```

使用：
```bash
chmod +x /home/connact/update.sh
/home/connact/update.sh
```

---

## 配置域名（可选）

### 1. 购买域名

在阿里云、腾讯云、Cloudflare 等购买域名

### 2. 添加 DNS 记录

```
类型: A
主机记录: @（或 www）
记录值: 你的服务器IP
TTL: 600
```

### 3. 更新 Nginx 配置

```bash
nano /etc/nginx/sites-available/connact

# 修改 server_name
server_name connact.ai www.connact.ai;  # 改为你的域名
```

### 4. 配置 HTTPS（推荐）

```bash
# 安装 Certbot
apt install -y certbot python3-certbot-nginx

# 自动配置 SSL 证书
certbot --nginx -d connact.ai -d www.connact.ai

# 自动续期
certbot renew --dry-run
```

---

## 故障排查

### 问题1：502 Bad Gateway

```bash
# 检查应用是否运行
supervisorctl status

# 如果 FATAL，查看错误日志
tail -f /var/log/connact.err.log

# 常见原因：端口被占用、环境变量错误、依赖缺失
```

### 问题2：应用启动失败

```bash
# 手动测试应用
cd /home/connact/Connact.ai
source venv/bin/activate
python app.py

# 查看具体错误信息
```

### 问题3：Git pull 失败

```bash
# 检查 GitHub 连通性
ping github.com

# 如果被墙，配置代理或使用 Gitee 镜像
```

### 问题4：数据库权限问题

```bash
# 确保目录权限正确
chown -R connact:connact /home/connact/Connact.ai/data
chmod -R 755 /home/connact/Connact.ai/data
```

---

## 性能优化

### 1. 增加 Gunicorn Worker 数量

```bash
# 编辑 Supervisor 配置
nano /etc/supervisor/conf.d/connact.conf

# 修改 command 行
command=/home/connact/Connact.ai/venv/bin/gunicorn -w 8 -b 127.0.0.1:5000 app:app
# w = 2 * CPU核心数 + 1

# 重启
supervisorctl reread && supervisorctl restart connact
```

### 2. 启用 Nginx Gzip 压缩

```bash
# 编辑 Nginx 主配置
nano /etc/nginx/nginx.conf

# 在 http 块中添加
gzip on;
gzip_vary on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
gzip_min_length 1000;

# 重启 Nginx
systemctl restart nginx
```

### 3. 配置日志轮转

```bash
# 防止日志文件过大
nano /etc/logrotate.d/connact

# 添加
/var/log/connact.*.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
    create 0644 connact connact
}
```

---

## 安全加固

### 1. 禁用 root SSH 登录

```bash
# 创建新用户
adduser admin
usermod -aG sudo admin

# 修改 SSH 配置
nano /etc/ssh/sshd_config

# 改为
PermitRootLogin no
PasswordAuthentication yes  # 或使用密钥

# 重启 SSH
systemctl restart sshd
```

### 2. 配置防火墙

```bash
# 使用 ufw
ufw allow 22
ufw allow 80
ufw allow 443
ufw enable
```

### 3. 定期备份

```bash
# 创建备份脚本
nano /root/backup.sh

#!/bin/bash
DATE=$(date +%Y%m%d)
tar -czf /root/backup-$DATE.tar.gz /home/connact/Connact.ai/data
find /root/backup-*.tar.gz -mtime +7 -delete

# 添加到 crontab
crontab -e
# 每天凌晨2点备份
0 2 * * * /root/backup.sh
```

---

## 监控与告警（可选）

### 使用阿里云监控

1. 登录阿里云控制台
2. 云监控 → 主机监控
3. 配置告警规则（CPU、内存、磁盘）

### 使用开源监控工具

```bash
# 安装 Netdata（实时监控面板）
bash <(curl -Ss https://my-netdata.io/kickstart.sh)

# 访问：http://你的IP:19999
```

---

## 成本估算

| 项目 | 费用 |
|------|------|
| 阿里云轻量服务器（香港） | ¥400-600/年 |
| 域名（.ai） | ¥200-500/年 |
| SSL 证书 | ¥0（Let's Encrypt 免费） |
| **总计** | **¥600-1100/年** |

---

## 对比 Render

| 特性 | 阿里云（香港） | Render |
|------|----------------|--------|
| 速度（大陆访问） | 30-50ms ⚡ | 200-300ms |
| 冷启动 | ❌ 无 | ✅ 有（免费版） |
| 可控性 | ✅ 完全控制 | ⚠️ 受限 |
| 成本 | ¥600/年 | $0-84/年 |
| 配置难度 | ⭐⭐ | ⭐ |

---

## 常见问题

**Q: 是否需要备案？**  
A: 香港节点不需要，大陆节点需要（15-20天）

**Q: 如何切换到大陆节点？**  
A: 先备案域名，然后重新购买大陆服务器，迁移数据

**Q: 数据库如何迁移？**  
A: 定期备份 `/home/connact/Connact.ai/data/app.db`

**Q: 如何扩容？**  
A: 阿里云控制台 → 升级配置（无缝升级）

---

## 联系支持

- 阿里云工单：https://workorder.console.aliyun.com/
- GitHub Issues: https://github.com/doyoulikelin-wq/Connact.ai/issues

---

**部署完成后别忘了**：
1. ⭐ Star 项目：https://github.com/doyoulikelin-wq/Connact.ai
2. 📝 在 README 更新线上地址
