# Cloudflare Tunnel 内网穿透完整教程

## 📖 目录
1. [什么是 Cloudflare Tunnel](#什么是-cloudflare-tunnel)
2. [前置准备](#前置准备)
3. [创建 Cloudflare Tunnel](#创建-cloudflare-tunnel)
4. [部署 cloudflared 容器](#部署-cloudflared-容器)
5. [配置服务路由](#配置服务路由)
6. [特殊应用配置](#特殊应用配置)
7. [常见问题](#常见问题)
8. [日常管理](#日常管理)

---

## 什么是 Cloudflare Tunnel

Cloudflare Tunnel（原名 Argo Tunnel）是 Cloudflare 提供的内网穿透服务：

### ✅ 优势
- **免费**：HTTP/HTTPS 服务完全免费
- **无需公网 IP**：不需要购买服务器或固定 IP
- **自动 HTTPS**：自动配置 SSL 证书
- **高性能**：利用 Cloudflare 全球 CDN 加速
- **安全**：不暴露源服务器 IP，防 DDoS

### 🎯 适用场景
- 家庭 NAS（群晖、威联通）外网访问
- Home Assistant 智能家居远程控制
- Emby/Jellyfin/Plex 媒体服务器分享
- 开发测试环境临时公网访问
- 内网服务统一域名管理

### ⚠️ 限制
- **通用 TCP/UDP 直连需付费**（Cloudflare Spectrum）
- **HTTP/HTTPS 服务免费**，非 HTTP 服务需客户端配合（Access）

---

## 前置准备

### 1. 域名要求
- ✅ 拥有一个域名（如 `smallcherry.cn`）
- ✅ 域名已托管到 Cloudflare（免费账号即可）

**域名托管到 Cloudflare 步骤：**
1. 注册 [Cloudflare 账号](https://dash.cloudflare.com/sign-up)
2. 添加域名到 Cloudflare
3. 到域名注册商修改 DNS 服务器为 Cloudflare 提供的地址
4. 等待 DNS 生效（通常 10-60 分钟）

### 2. 环境要求
- ✅ 一台能联网的内网机器（群晖/Linux/Windows/Mac）
- ✅ 机器已安装 Docker 和 Docker Compose
- ✅ 机器能访问内网中需要映射的服务

---

## 创建 Cloudflare Tunnel

### 步骤 1：登录 Cloudflare Zero Trust

1. 访问 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 左侧菜单选择 **Zero Trust**（如果首次使用需要创建团队）
3. 左侧菜单展开 **Networks** → 点击 **Tunnels**

### 步骤 2：创建新隧道

1. 点击右上角 **Create a tunnel** 按钮
2. 选择 **Cloudflared**（推荐方式）
3. 点击 **Select Cloudflared**

### 步骤 3：命名隧道

1. 在 **Name your tunnel** 页面输入隧道名称（如 `home-tunnel`）
2. 点击 **Save tunnel**

### 步骤 4：获取 Token

进入 **Install and run a connector** 页面：

1. 选择操作系统：**Docker**
2. 复制显示的命令，格式如下：
   ```bash
   docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token eyJhIjoiXXXXXX...
   ```
3. **重要**：复制并保存这个 Token（`eyJhIjo...` 这一长串），后面会用到

---

## 部署 cloudflared 容器

### 方式 1：使用 docker-compose（推荐）

#### 1. 创建配置文件

在你的工作目录（如 `/volume1/docker/cloudflared/`）创建 `docker-compose.yml` 文件：

```yaml
version: '3.8'

services:
  cloudflared:
    # Cloudflare 官方镜像
    image: cloudflare/cloudflared:latest
    
    # 容器名称
    container_name: cloudflared
    
    # 自动重启（开机自启）
    restart: unless-stopped
    
    # 使用宿主机网络模式（解决容器无法访问宿主机服务的问题）
    network_mode: host
    
    # 启动命令：运行隧道，禁用自动更新，使用你的 Token
    command: tunnel --no-autoupdate run --token 你的Token
```

**⚠️ 重要**：将 `你的Token` 替换为上一步复制的实际 Token！

#### 2. 启动容器

```bash
# 进入配置文件目录
cd /volume1/docker/cloudflared/

# 启动容器
docker-compose up -d

# 查看日志（确认连接成功）
docker-compose logs -f cloudflared
```

#### 3. 验证连接状态

查看日志，如果看到类似以下内容说明成功：
```
Connection registered connIndex=0
Connection registered connIndex=1
Connection registered connIndex=2
Connection registered connIndex=3
```

### 方式 2：使用 docker run 命令

如果你不想用 docker-compose，也可以直接运行命令：

```bash
docker run -d --name cloudflared \
  --restart unless-stopped \
  --network host \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate run --token 你的Token
```

---

## 配置服务路由

### 在 Cloudflare Web 界面配置路由（推荐）

#### 步骤 1：进入隧道配置

1. 回到 Cloudflare Zero Trust 的 **Networks** → **Tunnels**
2. 找到你创建的隧道，点击隧道名称进入详情
3. 选择 **Public Hostname** 标签页

#### 步骤 2：添加路由规则

点击 **Add a public hostname** 按钮，填写以下信息：

| 字段 | 说明 | 示例 |
|------|------|------|
| **Subdomain** | 子域名 | `emby` |
| **Domain** | 你的域名（下拉选择） | `smallcherry.cn` |
| **Path** | 路径（可选，一般留空） | 留空或填 `/` |
| **Type** | 服务类型 | 选择 `HTTP` |
| **URL** | 内网服务地址 | `http://192.168.1.21:8096` |

完整访问地址：`https://emby.smallcherry.cn`

#### 示例配置：

**示例 1：群晖 Emby 服务**
- Subdomain: `emby`
- Domain: `smallcherry.cn`
- Type: `HTTP`
- URL: `http://192.168.1.21:8096`

**示例 2：局域网内其他机器的服务**
- Subdomain: `home`
- Domain: `smallcherry.cn`
- Type: `HTTP`
- URL: `http://192.168.1.62:8123`（虚拟机 IP）

**示例 3：群晖 DSM 管理界面**
- Subdomain: `nas`
- Domain: `smallcherry.cn`
- Type: `HTTP`
- URL: `http://192.168.1.21:5000`

#### 步骤 3：保存并测试

1. 点击 **Save** 保存配置
2. 等待 1-2 分钟 DNS 生效
3. 在浏览器访问 `https://your-subdomain.your-domain.com`

### 注意事项

✅ **IP 地址填写规则：**
- 如果服务在**运行 cloudflared 的同一台机器**上：
  - 可以填 `http://localhost:端口` 或 `http://127.0.0.1:端口`
  - 也可以填机器的局域网 IP
- 如果服务在**局域网内其他机器**上：
  - 填写目标机器的局域网 IP，如 `http://192.168.1.100:端口`
  - 确保运行 cloudflared 的机器能访问到目标机器

✅ **端口说明：**
- Cloudflare 会自动提供 HTTPS（443 端口）
- 外网访问：`https://subdomain.domain.com`（不需要加端口号）
- 内网服务地址必须包含端口号（除非是 80 端口）

---

## 特殊应用配置

### Home Assistant 配置

Home Assistant 有安全验证机制，通过反向代理访问需要额外配置。

#### 方法 1：Cloudflare 配置 Host Header（推荐）

在添加路由规则时：

1. 填写基本信息（Subdomain、Domain、URL）
2. 展开 **Additional application settings**
3. 展开 **HTTP Settings**
4. 在 **Host Header** 字段填写：`192.168.1.62:8123`（你的 HA 实际地址）
5. 点击 **Save**

#### 方法 2：修改 Home Assistant 配置

编辑 Home Assistant 的 `configuration.yaml` 文件，添加：

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.1.21      # 运行 cloudflared 的机器 IP
    - 192.168.1.0/24    # 整个局域网段
    - 127.0.0.1
    - ::1
```

保存后重启 Home Assistant。

### 群晖 DSM 配置

群晖管理界面默认可直接访问，无需额外配置。

**建议配置：**
- Subdomain: `nas`
- URL: `http://192.168.1.21:5000`（DSM HTTP 端口）
- 或: `https://192.168.1.21:5001`（DSM HTTPS 端口，推荐）

### Emby/Jellyfin/Plex 配置

这些媒体服务器通常无需额外配置，直接添加路由即可。

**注意**：如果遇到访问问题，检查应用的网络设置，确保允许外部访问。

---

## 常见问题

### 🔴 502 Bad Gateway

**原因：**
- 后端服务未运行
- IP 地址或端口填写错误
- 防火墙阻止访问
- 容器网络隔离问题

**解决方法：**

1. **检查服务是否运行**：
   ```bash
   # 测试能否访问
   curl http://192.168.1.21:8096
   
   # 检查端口监听
   netstat -tuln | grep 8096
   ```

2. **检查 cloudflared 日志**：
   ```bash
   docker logs cloudflared --tail 50
   ```

3. **如果使用 Docker 部署，添加 `network_mode: host`**：
   ```yaml
   services:
     cloudflared:
       network_mode: host
   ```

### 🔴 400 Bad Request

**原因：**
- 应用验证 HTTP Host 头失败（常见于 Home Assistant、Nextcloud）

**解决方法：**
- 在 Cloudflare 配置 Host Header（见上方 Home Assistant 配置）
- 或在应用中配置信任代理

### 🔴 容器一直重启

**原因：**
- Token 错误或过期
- 系统不支持某些 Docker 配置参数

**解决方法：**

1. **查看日志定位问题**：
   ```bash
   docker logs cloudflared
   ```

2. **简化配置**，移除可能不兼容的参数：
   ```yaml
   version: '3.8'
   services:
     cloudflared:
       image: cloudflare/cloudflared:latest
       container_name: cloudflared
       restart: unless-stopped
       network_mode: host
       command: tunnel --no-autoupdate run --token 你的Token
   ```

3. **重新生成 Token**：
   - 在 Cloudflare 后台删除旧隧道
   - 创建新隧道获取新 Token

### 🔴 DNS 解析失败

**原因：**
- DNS 未生效
- Cloudflare 域名配置错误

**解决方法：**

1. **检查 DNS 记录**：
   - 进入 Cloudflare Dashboard → DNS
   - 确认子域名已创建（类型为 CNAME，指向隧道）
   - Cloudflare Tunnel 通常会自动创建，如果没有可以手动添加

2. **等待 DNS 生效**（通常 1-5 分钟）

3. **测试 DNS 解析**：
   ```bash
   nslookup emby.smallcherry.cn
   dig emby.smallcherry.cn
   ```

---

## 日常管理

### 查看容器状态

```bash
# 查看运行状态
docker ps | grep cloudflared

# 查看日志
docker logs cloudflared -f

# 使用 docker-compose
docker-compose logs -f cloudflared
```

### 重启容器

```bash
# 直接重启
docker restart cloudflared

# 使用 docker-compose
docker-compose restart
```

### 停止容器

```bash
# 停止容器
docker stop cloudflared

# 使用 docker-compose
docker-compose down
```

### 更新容器

```bash
# 使用 docker-compose
docker-compose pull
docker-compose up -d

# 或者手动
docker pull cloudflare/cloudflared:latest
docker-compose up -d
```

### 添加新的服务路由

1. 进入 Cloudflare Zero Trust → Networks → Tunnels
2. 点击隧道名称 → Public Hostname 标签
3. 点击 **Add a public hostname**
4. 填写新的服务信息并保存
5. **无需重启容器**，立即生效

### 修改或删除路由

1. 进入隧道的 Public Hostname 页面
2. 找到要修改的路由，点击右侧的 **Edit** 或 **Delete**
3. 修改后保存，立即生效

### 查看隧道连接状态

在 Cloudflare Zero Trust → Networks → Tunnels 页面：
- **Healthy**（绿色）：连接正常
- **Degraded**（黄色）：部分连接异常
- **Down**（红色）：隧道离线

---

## 架构示意图

```
┌─────────────────────────────────────────────────────────────────┐
│                          外网用户                                │
│                    https://emby.smallcherry.cn                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Cloudflare CDN（全球加速）                    │
│              • 自动 HTTPS 证书                                   │
│              • 隐藏源 IP                                         │
│              • DDoS 防护                                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│            Cloudflare Tunnel（云端路由配置）                     │
│              emby.smallcherry.cn → 192.168.1.21:8096            │
│              home.smallcherry.cn → 192.168.1.62:8123            │
│              nas.smallcherry.cn  → 192.168.1.21:5000            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  内网：cloudflared 容器                          │
│                  运行在群晖 192.168.1.21                         │
│              （通过 docker-compose 管理）                        │
└──────────────────────────┬──────────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │  Emby    │      │   NAS    │      │   HA     │
   │  :8096   │      │  :5000   │      │  :8123   │
   │ 本机服务 │      │ 本机服务 │      │ 虚拟机   │
   └──────────┘      └──────────┘      └──────────┘
   192.168.1.21      192.168.1.21      192.168.1.62
```

---

## 安全建议

### 1. Token 安全
- ⚠️ **不要泄露 Token**，任何人有 Token 就能运行你的隧道
- ⚠️ Token 泄露后立即在 Cloudflare 后台删除隧道并重新创建

### 2. 应用安全
- ✅ 所有公开服务**必须设置强密码**
- ✅ 启用**两步验证（2FA）**
- ✅ 定期更新应用版本修复安全漏洞

### 3. Cloudflare 安全选项
在 Cloudflare Zero Trust 的 Access 中可以配置：
- **访问策略**：限制特定 IP 或邮箱才能访问
- **身份验证**：强制通过 Google/GitHub 等登录
- **地理限制**：只允许特定国家访问

### 4. 日志监控
定期检查 cloudflared 日志，留意异常访问：
```bash
docker logs cloudflared --tail 100
```

---

## 成本说明

### ✅ 完全免费的部分
- Cloudflare Tunnel 基础服务
- HTTP/HTTPS 流量（无限制）
- 自动 SSL 证书
- CDN 加速
- DDoS 防护（基础版）

### 💰 需要付费的场景
- **Cloudflare Spectrum**：公网直连 TCP/UDP 服务（非 HTTP），约 $10/月起
- **Cloudflare Teams**：企业级访问控制和审计，约 $7/用户/月

### 对比 frp（传统内网穿透）

| 项目 | Cloudflare Tunnel | frp |
|------|-------------------|-----|
| **成本** | 免费 | 需购买公网 IP 服务器（约 $5-10/月） |
| **配置难度** | 简单（Web 界面） | 中等（需配置服务端+客户端） |
| **性能** | 全球 CDN 加速 | 取决于服务器位置和带宽 |
| **HTTPS** | 自动配置 | 需手动配置证书 |
| **安全性** | Cloudflare 防护 | 需自己加固 |
| **适用场景** | HTTP/HTTPS 为主 | 任意 TCP/UDP |

---

## 总结

### ✅ Cloudflare Tunnel 的最佳实践

1. **容器管理**：使用 `docker-compose.yml` 管理容器
2. **路由配置**：使用 Cloudflare Web 界面配置（实时生效，易管理）
3. **安全加固**：启用应用密码、两步验证、Cloudflare Access
4. **定期维护**：更新容器镜像、检查日志、监控连接状态

### 📌 关键要点

- 一个隧道可以映射**无限个服务**
- 路由配置在 Cloudflare 后台修改，**无需重启容器**
- 支持同一局域网内**任意机器的服务**
- 完全**免费**，适合个人和家庭使用

---

## 附录：常用命令速查表

```bash
# === 容器管理 ===
# 启动容器
docker-compose up -d

# 停止容器
docker-compose down

# 重启容器
docker-compose restart

# 查看日志
docker-compose logs -f cloudflared

# 查看状态
docker ps | grep cloudflared

# === 容器更新 ===
# 拉取最新镜像
docker-compose pull

# 重新创建容器
docker-compose up -d

# === 故障排查 ===
# 查看最近 50 行日志
docker logs cloudflared --tail 50

# 实时查看日志
docker logs cloudflared -f

# 测试内网服务连通性
curl http://192.168.1.21:8096

# 检查端口监听
netstat -tuln | grep 8096

# DNS 测试
nslookup emby.smallcherry.cn
```

---

## 相关链接

- [Cloudflare Tunnel 官方文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
- [Cloudflare Dashboard](https://dash.cloudflare.com/)
- [Docker Hub - cloudflared](https://hub.docker.com/r/cloudflare/cloudflared)

---

**文档版本**：v1.0  
**最后更新**：2025-10-02  
**适用范围**：个人/家庭内网穿透场景

---

🎉 **恭喜你完成配置！现在你可以随时随地安全访问家中的服务了！**

