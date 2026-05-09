# Hyvor Relay 阿里云部署完整指南（含踩坑记录）

> 适用于：阿里云 ECS + Ubuntu + Docker + Porkbun 域名
> 项目：[Hyvor Relay](https://github.com/hyvor/relay) - 开源自托管邮件 API

---

## 一、部署前准备

### 1.1 服务器要求

- Ubuntu/Debian，已安装 Docker 和 Docker Compose
- 至少 1 vCPU、1GB 内存、20GB 存储
- **公网 IP**（本文示例：`YOUR_SERVER_IP`）

### 1.2 域名（Porkbun）

假设域名为 `example.com`，需要在 Porkbun 添加以下 DNS 记录：

| Type | Host     | Answer                         | 说明      |
| ---- | -------- | ------------------------------ | --------- |
| A    | `relay`  | `服务器公网IP`                 | Web 访问  |
| A    | `mail`   | `服务器公网IP`                 | 邮件 EHLO |
| MX   | *(留空)* | `mail.example.com`             | 接收邮件  |
| TXT  | *(留空)* | `v=spf1 ip4:服务器公网IP ~all` | SPF 验证  |

> Host 只填 `relay`，Porkbun 自动补全为 `relay.example.com`。
> PTR 反向 DNS 需在阿里云控制台 → 弹性 IP 中配置，指向 `mail.example.com`。

### 1.3 Auth0 OIDC 配置

Relay 不内置用户系统，必须用 OIDC 登录。

1. 注册 [auth0.com](https://auth0.com)（免费，无需绑卡）

2. Applications → Create Application → **Regular Web Applications**

3. Settings 中填写：

   | 字段                  | 值                                            |
   | --------------------- | --------------------------------------------- |
   | Allowed Callback URLs | `https://relay.example.com/api/oidc/callback` |
   | Allowed Logout URLs   | `https://relay.example.com`                   |
   | Allowed Web Origins   | `https://relay.example.com`                   |

4. 记录 **Domain**、**Client ID**、**Client Secret**

### 1.4 阿里云 25 端口解封

**安全组开放 25 端口 ≠ 25 端口解封！** 阿里云在网络层面默认封锁 25 出站。

1. 阿里云控制台 → 搜索「安全管控」→ **25端口解封申请**
2. 类型选 **VPC公网IP**，填写服务器 IP 和 `mail.example.com`
3. 等待审核通过（1-2 个工作日）

> 验证是否解封：`telnet smtp.qq.com 25`，看到 `220` 响应即成功。

---

## 二、服务器部署

### 2.1 创建项目目录和配置文件

```bash
mkdir -p ~/relay/config && cd ~/relay
```

### 2.2 创建 docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:18
    container_name: hyvor-relay-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: hyvor_relay
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d hyvor_relay"]
      interval: 5s
      timeout: 5s
      retries: 5
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql

  relay:
    image: hyvor/relay:latest
    container_name: hyvor-relay
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    network_mode: "host"
    env_file:
      - .env
    environment:
      - HTTPS_PROXY=http://127.0.0.1:7890
      - HTTP_PROXY=http://127.0.0.1:7890
      - GO_SYMFONY_URL=http://localhost:8080
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile

volumes:
  postgres_data:
```

### 2.3 创建 .env 文件

```bash
# 生成密钥
openssl rand -base64 32   # APP_SECRET
openssl rand -hex 32      # POSTGRES_PASSWORD
```

```env
APP_SECRET=你生成的密钥
POSTGRES_PASSWORD=你生成的数据库密码
DATABASE_URL="postgresql://postgres:${POSTGRES_PASSWORD}@127.0.0.1:5432/hyvor_relay?serverVersion=16&charset=utf8"
WEB_URL=https://relay.example.com
INSTANCE_DOMAIN=mail.example.com
OIDC_ISSUER_URL=https://你的Auth0 Domain/
OIDC_CLIENT_ID=你的ClientID
OIDC_CLIENT_SECRET=你的ClientSecret
APP_ENV=prod
LOG_LEVEL=info
```

> ⚠️ `WEB_URL` 只填根地址 `https://relay.example.com`，**不要**带 `/api/oidc/callback` 路径！

---

## 三、启动前必须解决的问题

### 3.1 配置独立 Caddy（统一管理所有服务）

Relay 内置的 Caddy 与 FrankenPHP 编译在一起，无法从外部管理。因此采用**独立 Caddy + Relay 内部 8080 端口**的架构：

- 独立 Caddy 监听 80/443 → 统一管理 SSL 和反向代理
- Relay 内置 Caddy 监听 8080 → 只处理 Relay 应用

#### 3.1.1 安装独立 Caddy

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

#### 3.1.2 配置独立 Caddy

编辑 `/etc/caddy/Caddyfile`：

```
# Relay 服务
relay.example.com {
    reverse_proxy 127.0.0.1:8080
}

# 其他服务示例（按需添加）
# app.example.com {
#     reverse_proxy 127.0.0.1:3000
# }
```

```bash
sudo systemctl restart caddy
```

#### 3.1.3 Relay 使用自定义 Caddyfile

项目已包含自定义 Caddyfile（`deploy/easy/Caddyfile`），将 Relay 内置 Caddy 改为监听 8080 端口。需要在 compose.yaml 中挂载并配置 Go Worker 地址（见 2.2 节）。

> **为什么不能用 Relay 内置 Caddy 管理所有服务？**
> Relay 的 Caddy 与 FrankenPHP 编译在一起，是 PHP 运行时的一部分，不是独立进程，无法从外部管理。只能通过挂载自定义 Caddyfile 修改其配置。

### 3.2 释放 53 端口（DNS）

Ubuntu 的 `systemd-resolved` 默认占用 53 端口：

```bash
sudo ss -tlnp | grep :53
# 如果是 systemd-resolved，执行：
sudo mkdir -p /etc/systemd/resolved.conf.d
cat <<EOF | sudo tee /etc/systemd/resolved.conf.d/noport53.conf
[Resolve]
DNSStubListener=no
EOF
sudo systemctl restart systemd-resolved
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

### 3.3 安装代理（解决 cloudflare-dns.com 被墙）

国内服务器无法访问 `cloudflare-dns.com`，导致 DKIM 域名验证失败。需要安装 Clash 或其他代理：

```bash
# 安装 Clash（按你熟悉的方式）
# 确保 HTTP 代理运行在 127.0.0.1:7890
# 推荐clash for linux - github：https://github.com/nelvko/clash-for-linux-install

# 验证代理可用：
curl -x http://127.0.0.1:7890 https://cloudflare-dns.com/dns-query?name=example.com&type=A -H "Accept: application/dns-json"
# 应返回 JSON
```

代理配置已写入 compose.yaml 的 `environment` 部分。

### 3.4 绑定公网 IP 到网卡

阿里云 ECS 通过 NAT 映射公网 IP，网卡上只有内网 IP。Relay 检测不到公网 IP 会导致 `ip_count=0`，邮件发不出去。

```bash
# 添加 IP 别名（重启丢失，见 3.5 持久化）
ip addr add YOUR_SERVER_IP/32 dev eth0
```

### 3.5 IP 别名持久化

```bash
cat <<'EOF' | sudo tee /etc/netplan/99-public-ip.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - YOUR_SERVER_IP/32
EOF
sudo netplan apply
```

### 3.6 开放防火墙端口（阿里云ECS的安全组管理）

```bash
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 25/tcp    # SMTP
sudo ufw allow 587/tcp   # SMTP Submission
sudo ufw allow 465/tcp   # SMTPS（stunnel 用）
```

---

## 四、启动部署

```bash
cd ~/relay
docker compose up -d
```

首次启动会自动：拉取镜像、初始化数据库、运行迁移、启动所有服务、申请 SSL 证书。**（其中如果.env文件错误过需要删除数据库挂载的卷之后再重新启动，不然域名地址会是之前文件的值）**

验证：

```bash
docker compose ps
curl https://relay.example.com/api/health
```

---

## 五、Relay 控制台配置

### 5.1 登录

访问 `https://relay.example.com`，通过 Auth0 登录。

### 5.2 创建项目

左上角「只读系统」下拉 → **Create new project**

### 5.3 添加发送域名

项目 → Domains → 添加 `mail.example.com` → 在 Porkbun 添加 DKIM TXT 记录 → 验证

### 5.4 生成 API Key

项目 → API → Create API Key → 勾选 `sends.send` → 复制（只显示一次）

---

## 六、SMTP TLS 配置（stunnel）

部分客户端只支持隐式 TLS（端口 465），但 Relay 只支持 STARTTLS（端口 587）。用 stunnel 做转换。

### 6.1 获取 Let's Encrypt 证书（通过cerbot）

```bash
certbot certonly --manual --preferred-challenges dns -d mail.example.com --agree-tos --register-unsafely-without-email
```

按提示在 Porkbun 添加 `_acme-challenge.mail` 的 TXT 记录，等几分钟按回车。

### 6.2 安装配置 stunnel

```bash
apt install -y stunnel4

cat > /etc/stunnel/stunnel.conf << 'EOF'
[smtps]
accept = 465
connect = 127.0.0.1:587
cert = /etc/letsencrypt/live/mail.example.com/fullchain.pem
key = /etc/letsencrypt/live/mail.example.com/privkey.pem
EOF

sed -i 's/ENABLED=0/ENABLED=1/' /etc/default/stunnel4
systemctl restart stunnel4
```

### 6.3 手动生成邮件 TLS 证书

```bash
docker exec hyvor-relay php bin/console tls:generate-mail-certificate
```

日志中出现 `Certificate downloaded successfully` 即成功。如果提示 DNS challenge，在 Porkbun 添加对应 TXT 记录。

---

## 七、外部应用 SMTP 配置

| 字段        | 值                                                        |
| ----------- | --------------------------------------------------------- |
| SMTP 主机   | `mail.example.com`                                        |
| SMTP 端口   | `465`                                                     |
| SMTP 用户名 | `relay`（随便填，不校验）                                 |
| SMTP 密码   | **Relay API Key**（项目 → API 中生成的）                  |
| 发件人邮箱  | `noreply@mail.example.com`（必须是 Relay 中验证过的域名） |
| 使用 TLS    | 开启                                                      |

> ⚠️ 发件人邮箱**必须**使用在 Relay 项目中注册并验证过的域名，不能用 gmail.com 等外部域名。

---

## 八、踩坑总结

### 坑 1：WEB_URL 配置错误

| 错误                                                 | 正确                                |
| ---------------------------------------------------- | ----------------------------------- |
| `WEB_URL=http://relay.example.com/api/oidc/callback` | `WEB_URL=https://relay.example.com` |

`WEB_URL` 是 Web 根地址，回调路径是 Relay 内部自动处理的。

### 坑 2：项目中存在两套配置

| 官方部署（正确）            | 第三方文件（不要用）                                         |
| --------------------------- | ------------------------------------------------------------ |
| `deploy/easy/` 目录下的文件 | `quick-start.sh`、`check-config.sh`、`hyvor-relay-deploy-guide.md`、`.env.example` |

第三方文件的环境变量与实际代码不匹配（如 `SMTP_HOST`、`RELAY_SECRET_KEY` 等变量不存在）。

### 坑 3：80/53 端口被占用

- **80 端口**：Nginx/Apache 占用 → 停掉，Relay 内置 Caddy 不需要 Nginx
- **53 端口**：`systemd-resolved` 占用 → 禁用 stub listener（见 3.2）

### 坑 4：cloudflare-dns.com 被墙

国内服务器无法访问，导致 DKIM 验证失败。解决：安装 Clash 代理 + compose.yaml 中配置 `HTTPS_PROXY`。

### 坑 5：ip_count=0（阿里云 NAT）

阿里云公网 IP 不在网卡上，Relay 检测不到 → `ip addr add` 绑定到网卡。每次容器重启会运行 `management:init` 重新检测，所以必须绑定到网卡上才能持久生效。

### 坑 6：SMTP TLS 模式不匹配

- Relay 支持 **STARTTLS**（先明文再升级）
- 部分客户端只支持**隐式 TLS**（直接加密）
- 解决：stunnel 在 465 端口提供隐式 TLS，转发到 Relay 的 587 STARTTLS

### 坑 7：自签名证书 SAN 缺失

生成证书时必须加 `-addext "subjectAltName=DNS:mail.example.com"`，否则客户端报错 `x509: certificate relies on legacy Common Name field`。建议直接用 Let's Encrypt 证书。

### 坑 8：25 端口出站封锁

**安全组开放 ≠ 端口解封**。阿里云在网络层面封锁 25 出站，必须在「安全管控」中单独申请解封。

### 坑 9：发件域名未注册

发件人邮箱必须是 Relay 项目中已注册并验证过的域名，不能用 gmail.com、qq.com 等。

### 坑 10：旧域名残留

改 `.env` 不会更新数据库中的域名记录。需要在 Relay 控制台删除旧域名，重新添加。或重置数据库：

```bash
docker compose down
docker volume rm easy_postgres_data
docker compose up -d
```

---

## 九、日常维护

### 更新版本

```bash
docker compose pull relay
docker compose up -d
```

### 数据库备份

```bash
docker exec hyvor-relay-postgres pg_dump -U postgres hyvor_relay > backup_$(date +%Y%m%d).sql
```

### 查看日志

```bash
docker compose logs -f relay
docker compose logs -f relay | grep -i "error\|fail"
```

### 重启

```bash
docker compose restart
```

> 重启后需要重新绑定公网 IP（如果没做持久化）：`ip addr add YOUR_SERVER_IP/32 dev eth0`

---

## 十、架构图

```
用户浏览器 ──HTTPS──→ 独立 Caddy (:80/:443, SSL) ──proxy──→ Relay 内置 Caddy (:8080)
其他应用   ──HTTPS──→ 独立 Caddy (:80/:443, SSL) ──proxy──→ 其他服务 (:3000/...)

              Docker 容器 (network_mode: host)
              ┌────────────────────────────┐
              │  Caddy/FrankenPHP (:8080)  │ ← 仅处理 Relay 应用
              │    ├── /api/* → PHP 后端    │
              │    └── /* → 静态前端文件    │
              │                            │
              │  Go Worker                 │ ← 邮件发送/DNS 服务
              │  DNS Server (:53)          │
              │  SMTP Server (:25, :587)   │ ← STARTTLS
              │  Messenger Workers x3      │ ← 异步任务队列
              │  Supervisor                │ ← 进程管理
              └────────────────────────────┘
                          ↓
              PostgreSQL (127.0.0.1:5432)

外部应用 ──TLS:465──→ stunnel ──plain:587──→ Relay SMTP
```
