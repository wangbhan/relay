# Hyvor Relay - 自托管部署

基于 [Hyvor Relay](https://github.com/hyvor/relay) 开源邮件 API，适配阿里云 ECS 国内环境部署。

## 项目说明

Hyvor Relay 是一个自托管的邮件发送服务，提供 REST API 和 SMTP 接口发送邮件，支持多项目、多域名、DKIM/SPF 验证、Webhook、邮件追踪等功能。

### 与原项目的区别

| 项目 | 说明 |
|---|---|
| `deploy/easy/Caddyfile` | 自定义 Caddy 配置，将内置 Caddy 改为监听 8080 端口，配合独立 Caddy 使用 |
| `DEPLOY_SERVER.md` | 阿里云 ECS 完整部署指南（含踩坑记录） |

其余文件与原项目一致，未做任何代码改动。

## 快速开始

### 文件结构

```
deploy/easy/
├── compose.yaml      # Docker Compose 配置
├── Caddyfile         # 自定义 Caddy 配置（监听 8080）
└── .env              # 环境变量（需自行创建）
```

### 1) 准备工作

详见 [DEPLOY_SERVER.md](./DEPLOY_SERVER.md) 第一至第三节，包括：

- DNS 记录配置（Porkbun）
- Auth0 OIDC 配置
- 阿里云 25 端口解封申请
- 安装独立 Caddy
- 安装 Clash 代理
- 绑定公网 IP 到网卡
- 释放 53 端口

### 2) 部署

```bash
cd deploy/easy

# 创建 .env（参考 DEPLOY_SERVER.md 2.3 节）
cp .env.example .env  # 然后修改实际值

# 启动
docker compose up -d
```

### 3) 验证

```bash
docker compose ps
docker compose logs -f relay
```

## 外部应用接入

| 字段 | 值 |
|---|---|
| SMTP 主机 | `mail.example.com` |
| SMTP 端口 | `465` |
| SMTP 用户名 | `relay` |
| SMTP 密码 | Relay API Key |
| 发件人邮箱 | `noreply@mail.example.com` |
| 使用 TLS | 开启 |

## 常见问题

**Q: 为什么不直接用项目根目录的 `quick-start.sh`？**
A: 项目根目录的 `quick-start.sh`、`check-config.sh`、`.env.example` 等文件使用了一套与实际代码不匹配的环境变量（如 `SMTP_HOST`、`RELAY_SECRET_KEY` 等），不要使用。请以 `deploy/easy/` 目录为准。

**Q: 为什么需要独立 Caddy？**
A: Relay 内置的 Caddy 与 FrankenPHP 编译在一起，无法从外部管理。独立 Caddy 统一管理 80/443 端口和 SSL 证书，Relay 内置 Caddy 退到 8080 端口。这样可以同时运行多个 Web 服务。

**Q: 邮件发不出去？**
A: 检查以下几点：
1. 阿里云 25 端口是否已解封（`telnet smtp.qq.com 25`）
2. `ip_count` 是否大于 0（`docker compose logs relay | grep ip_count`）
3. DKIM 域名是否已验证
4. 发件人邮箱是否使用已验证的域名

## 详细文档

完整的部署步骤和踩坑记录请参考 [DEPLOY_SERVER.md](./DEPLOY_SERVER.md)。
