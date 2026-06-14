# 部署文档：自托管网站流量分析系统

## 架构概览

本项目使用 [Umami](https://umami.is/) 作为流量统计工具，部署在自己的服务器上，数据完全自有。

```
访客浏览器
    │
    │ HTTPS (443端口)
    ▼
Nginx Stream 层（SNI路由）
    │  根据域名把流量分发到不同后端
    ├──► 其他网站 ──► localhost:8443 ──► Nginx HTTP ──► 业务服务
    │
    └──► analytics.你的域名.com ──► localhost:8445 ──► Nginx HTTP ──► Umami (3000端口)
                                                                          │
                                                                          ▼
                                                                       MySQL 数据库
```

**关键概念**：

- **SNI 路由**：HTTPS 握手时客户端会先发送目标域名（SNI），Nginx Stream 层读取这个域名后把连接转发给对应的后端，实现"一个 443 端口跑多个 HTTPS 站点"。
- **PROXY Protocol**：Stream 层做 TCP 转发时，原始的访客 IP 会丢失（后端只能看到 127.0.0.1）。开启 PROXY Protocol 后，Stream 层会在 TCP 连接前附上真实 IP，后端 Nginx 就能读取到，用于地理位置统计。
- **PM2**：Node.js 进程管理工具，负责在后台运行 Umami，并在服务器重启后自动拉起。

---

## 前置条件

| 项目 | 要求 |
|------|------|
| 服务器系统 | Ubuntu 20.04 / 22.04 |
| 内存 | 最低 1GB（构建时需要，建议 2GB） |
| 磁盘 | 最低 5GB 可用空间 |
| Node.js | v18 或以上（推荐 v20） |
| MySQL | v5.7 或以上 |
| Nginx | 已安装并运行 |
| 域名 | 已有一个子域名指向服务器 IP（如 analytics.你的域名.com） |

---

## 第一步：准备 MySQL 数据库

```bash
# 登录 MySQL（替换 YOUR_PASSWORD 为你的 root 密码）
mysql -u root -pYOUR_PASSWORD

# 在 MySQL 中执行：
CREATE DATABASE umami CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
exit
```

---

## 第二步：下载并构建 Umami

```bash
# 克隆 Umami（使用 v2.13.2，这是最后一个支持 MySQL 的版本）
git clone https://github.com/umami-software/umami.git
cd umami
git checkout v2.13.2

# 安装依赖
npm install --legacy-peer-deps

# 配置环境变量
cp .env.example .env
nano .env   # 填入数据库连接信息（见下方说明）
```

`.env` 文件内容：

```env
DATABASE_URL=mysql://root:YOUR_DB_PASSWORD@localhost:3306/umami
APP_SECRET=在这里填入一串随机字符（至少32位）
SKIP_DB_CHECK=1
```

> **生成随机 APP_SECRET 的方法**：`openssl rand -hex 32`

```bash
# 构建（服务器内存不足 2GB 时需要加内存限制参数）
npm run build-db
npm run build-tracker
npm run build-geo
NODE_OPTIONS='--max-old-space-size=768' npm run build-app

# 初始化数据库表结构
npm run update-db
```

---

## 第三步：用 PM2 启动 Umami

```bash
# 安装 PM2
npm install -g pm2

# 启动 Umami（在 umami 目录下执行）
pm2 start npm --name umami -- start

# 保存进程列表，设置开机自启
pm2 save
pm2 startup   # 按照输出的提示执行那条命令
```

验证是否启动成功：

```bash
pm2 status          # 应该看到 umami 状态为 online
curl http://localhost:3000   # 应该返回 HTML 内容
```

---

## 第四步：配置 Nginx

### 4.1 配置 Stream 层（SNI 路由）

编辑 `/etc/nginx/nginx.conf`，在 `http { }` 块**外面**添加 stream 块：

```nginx
stream {
    map_hash_bucket_size 64;
    map $ssl_preread_server_name $backend_443 {
        你的其他域名.com        127.0.0.1:8443;
        analytics.你的域名.com  127.0.0.1:8445;
        default                 127.0.0.1:8444;
    }
    server {
        listen 443;
        proxy_pass $backend_443;
        proxy_protocol on;   # 传递真实访客 IP
        ssl_preread on;      # 读取 SNI 但不解密
    }
}
```

> **注意**：如果你的 nginx.conf 里已经有 stream 块，只需在 map 里新增一行 `analytics.你的域名.com 127.0.0.1:8445;` 并加上 `proxy_protocol on;` 即可。

### 4.2 创建 analytics 站点配置

先用 Certbot 申请 SSL 证书（需要临时用 80 端口，确保 analytics 域名的 DNS 已生效）：

```bash
# 临时在 Nginx HTTP 块中添加一个监听 80 端口的 server，让 Certbot 验证域名
# 或者先创建一个简单配置文件
cat > /etc/nginx/sites-available/analytics-temp << 'EOF'
server {
    listen 80;
    server_name analytics.你的域名.com;
    location / { return 200 'ok'; }
}
EOF
ln -s /etc/nginx/sites-available/analytics-temp /etc/nginx/sites-enabled/analytics-temp
nginx -t && systemctl reload nginx

# 申请证书
certbot certonly --nginx -d analytics.你的域名.com

# 删除临时配置
rm /etc/nginx/sites-enabled/analytics-temp
```

然后创建正式配置（参考本项目 `nginx/analytics.conf`）：

```bash
cat > /etc/nginx/sites-available/analytics << 'EOF'
server {
    listen 8445 ssl proxy_protocol;
    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;
    server_name analytics.你的域名.com;

    ssl_certificate /etc/letsencrypt/live/analytics.你的域名.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/analytics.你的域名.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
EOF

ln -s /etc/nginx/sites-available/analytics /etc/nginx/sites-enabled/analytics
nginx -t && systemctl reload nginx
```

### 4.3 更新已有站点配置支持 PROXY Protocol

如果你已有其他站点监听 8443 端口，需要在这些配置里加上 proxy_protocol 支持，否则访问会报错：

```nginx
# 原来：
listen 8443 ssl;

# 改为：
listen 8443 ssl proxy_protocol;
set_real_ip_from 127.0.0.1;
real_ip_header proxy_protocol;
```

修改后 `nginx -t && systemctl reload nginx`。

---

## 第五步：在目标网站嵌入追踪脚本

1. 访问 `https://analytics.你的域名.com`，默认账号 `admin` / 密码 `umami`（登录后立即修改）
2. 进入 **Settings → Websites → Add website**，填入要统计的网站域名
3. 点击 **Get tracking code**，复制生成的 `<script>` 标签
4. 将脚本粘贴到目标网站所有页面的 `<head>` 标签内

```html
<script defer src="https://analytics.你的域名.com/script.js"
        data-website-id="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"></script>
```

---

## 常见问题

**Q：地区显示"未知"**

原因：Stream 层做 TCP 代理时丢失了访客真实 IP。确认以下两点：
- `nginx.conf` 的 stream server 块中有 `proxy_protocol on;`
- analytics 的 Nginx 配置中 listen 指令包含 `proxy_protocol`，且有 `real_ip_header proxy_protocol;`

**Q：构建时报 JavaScript heap out of memory**

服务器内存不足。构建时加大 Node 堆内存限制：
```bash
NODE_OPTIONS='--max-old-space-size=768' npm run build-app
```

**Q：为什么使用 v2.13.2 而不是最新版**

Umami v3 移除了 MySQL 支持，仅支持 PostgreSQL。v2.13.2 是最后一个同时支持 MySQL 和 PostgreSQL 的版本。如果你的服务器有 PostgreSQL，可以使用最新版。

**Q：Umami 进程挂了怎么办**

```bash
pm2 restart umami   # 重启
pm2 logs umami      # 查看日志排查原因
```
