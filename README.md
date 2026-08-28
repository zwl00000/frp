# FRP + 公网云服务器 + 华为云 SWR 内网穿透部署记录

> 目的：在有公网 IP 的云服务器部署 `frps`，在内网主机部署 `frpc`，将内网 SSH/其他服务映射到公网；FRP Docker 镜像统一同步到华为云 SWR，方便后续机器直接拉取。



---

## 1. 整体结构

```text
外部电脑
   ↓
你的公网IP:你的映射端口
   ↓
公网云服务器 frps
   ↓
内网主机 frpc
   ↓
127.0.0.1:22
```

端口关系：

```text
你的SSH管理端口      = 公网云服务器自身 SSH
你的FRP控制端口      = frpc → frps
你的映射端口         = 外部访问内网服务
你的Dashboard端口    = FRP 管理页面
22                   = 内网主机 SSH 服务
```

## 2. 创建公网云服务器

以 Azure Ubuntu 为例：

```text
系统：Ubuntu Server 24.04 LTS x64
认证：SSH public key
公网：绑定公网 IPv4
```

安全组 / NSG 至少开放：

```text
你的SSH管理端口
你的FRP控制端口
你的FRP映射端口范围
你的Dashboard端口（如需公网访问）
```

SSH 示例：

```bash
chmod 600 你的私钥路径
ssh -p 你的SSH管理端口 -i 你的私钥路径 你的云服务器用户名@你的公网IP
```

也可写入 `~/.ssh/config`：

```sshconfig
Host 你的云服务器别名
    HostName 你的公网IP
    User 你的云服务器用户名
    Port 你的SSH管理端口
    IdentityFile 你的私钥路径
```

以后直接：

```bash
ssh 你的云服务器别名
```

## 3. 安装 Docker

检查：

```bash
docker --version
docker compose version
```

普通用户如果没有 Docker 权限：

```bash
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

## 4. 将 FRP 官方镜像同步到华为云 SWR

拉取官方镜像：

```bash
docker pull ghcr.io/fatedier/frps:v0.70.1
docker pull ghcr.io/fatedier/frpc:v0.70.1
```

登录 SWR：

```bash
docker login 你的SWR仓库地址
```

重新打标签：

```bash
docker tag ghcr.io/fatedier/frps:v0.70.1 你的SWR仓库地址/你的SWR组织/frps:v0.70.1
docker tag ghcr.io/fatedier/frpc:v0.70.1 你的SWR仓库地址/你的SWR组织/frpc:v0.70.1
```

上传：

```bash
docker push 你的SWR仓库地址/你的SWR组织/frps:v0.70.1
docker push 你的SWR仓库地址/你的SWR组织/frpc:v0.70.1
```

说明：

```text
私有镜像：每台新机器首次拉取前需要 docker login
公开镜像：可以直接 docker pull
```

## 5. 公网云服务器部署 frps

创建目录：

```bash
mkdir -p 你的FRPS目录/log
cd 你的FRPS目录
openssl rand -hex 32
```

`frps.toml`：

```toml
bindPort = 你的FRP控制端口

auth.method = "token"
auth.token = "你的FRP_TOKEN"

transport.tls.force = true

allowPorts = [
  { start = 你的映射端口范围起点, end = 你的映射端口范围终点 }
]

webServer.addr = "0.0.0.0"
webServer.port = 你的Dashboard端口
webServer.user = "你的Dashboard用户名"
webServer.password = "你的Dashboard密码"

log.to = "/frp/log/frps.log"
log.level = "warn"
```

`docker-compose.yml`：

```yaml
services:
  frps:
    image: 你的SWR仓库地址/你的SWR组织/frps:v0.70.1
    container_name: frps
    restart: always
    network_mode: host
    ulimits:
      nofile:
        soft: 100000
        hard: 100000
    volumes:
      - ./frps.toml:/etc/frp/frps.toml:ro
      - ./log:/frp/log
    command: -c /etc/frp/frps.toml
```

启动：

```bash
docker compose pull
docker compose up -d
docker ps
sudo ss -lntp | grep 你的FRP控制端口
```

## 6. 内网主机部署 frpc

推荐目录：

```text
你的FRP目录/
├── old-cloud/
└── new-cloud/
```

创建目录：

```bash
mkdir -p 你的FRPC目录/log
cd 你的FRPC目录
```

如果 SWR 是私有仓库，先登录，再拉镜像：

```bash
docker pull 你的SWR仓库地址/你的SWR组织/frpc:v0.70.1
```

`frpc.toml`：

```toml
serverAddr = "你的公网IP"
serverPort = 你的FRP控制端口

auth.method = "token"
auth.token = "你的FRP_TOKEN"

log.to = "/frp/log/frpc.log"
log.level = "warn"

transport.tls.enable = true
transport.heartbeatInterval = 10

[[proxies]]
name = "你的代理名称"
type = "tcp"

localIP = "127.0.0.1"
localPort = 22
remotePort = 你的映射端口
```

`docker-compose.yml`：

```yaml
services:
  frpc:
    image: 你的SWR仓库地址/你的SWR组织/frpc:v0.70.1
    container_name: 你的frpc容器名
    restart: always
    network_mode: host
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml:ro
      - ./log:/frp/log
    command: -c /etc/frp/frpc.toml
```

启动：

```bash
docker compose pull
docker compose up -d
docker ps
```

## 7. 验证连接

公网云服务器：

```bash
sudo ss -lntp | grep -E '你的FRP控制端口|你的映射端口'
```

外部电脑：

```bash
nc -vz -w 5 你的公网IP 你的映射端口
ssh -p 你的映射端口 你的内网用户名@你的公网IP
```

## 8. Dashboard

如果 Dashboard 绑定公网：

```text
http://你的公网IP:你的Dashboard端口
```

Dashboard 可查看：

```text
Clients      已连接的 frpc 数量
Proxies      已注册的代理数量
Connections  当前连接数量
Traffic      流量
```

如果开代理/VPN后 Dashboard 出现 `502 Bad Gateway`，但直连正常，可将“你的公网IP”加入浏览器或代理软件的 `DIRECT / No proxy`。

## 9. 新增一台内网机器

后续新增机器只需要：

1. 安装/检查 Docker。
2. 拉取 `frpc` 镜像。
3. 创建新的 `frpc.toml`。
4. 使用相同的公网 IP、FRP 控制端口和 Token。
5. 为该机器分配一个新的 `remotePort`。
6. 创建 `docker-compose.yml`。
7. `docker compose up -d`。
8. 在 Dashboard 检查 Client / Proxy 是否 `Online`。

> 新增机器时，不需要重新部署公网服务器 `frps`，也不需要重新上传 SWR 镜像。

## 10. 常见故障

| 现象 | 检查 |
|---|---|
| Docker `permission denied` | Docker 用户组权限 |
| SWR `You may not login yet` | 当前机器未登录私有 SWR |
| `docker tag requires 2 arguments` | 命令换行错误，建议一行执行 |
| 控制端口监听，但映射端口没有 | 检查 frpc、Token、NSG、安全组、`allowPorts` |
| `Connection refused` | 目标端口没有程序监听 |
| `Connection timed out` | 防火墙、NSG、路由或端口未放行 |
| Dashboard `401` | 正常，表示正在等待用户名密码 |
| Dashboard 代理访问 `502` | 对公网 IP 设置 `DIRECT` |
| `docker logs` 为空 | 日志可能写入挂载的日志文件 |
