本仓库的制作初衷是由于``lejianwen``于``20251022``不支持``多RELAY``，而我正好有阿里云的一台服务器CDT用完了，想着换一个中继不太好用，在ISSUEs发现了``wy41401``作者写的多RELAY，于是结合了一下

本仓库从官方仓库拉取镜像，结合lejianwen的：解决了使用API链接超时问题，可以强制登录后才能发起链接，支持客户端websocket，和wy41401的多地RELAY中继服务

|参考人员|仓库名|仓库链接|
|---|---|---|
|lejianwen|rustdesk-server|https://github.com/lejianwen/rustdesk-server|
|wy414012|rustdesk-server|https://github.com/wy414012/rustdesk-server|

## 功能特性

### 多中继服务器支持

支持配置多个中继服务器（RELAY_SERVERS），提供两种负载均衡策略：

#### 1. Round-Robin（轮询模式）- 默认
- **工作方式**：每次连接轮流使用不同的中继服务器
- **健康检查**：每 3 秒自动检测所有服务器健康状态
- **适用场景**：所有服务器性能相同，需要均匀分配流量

#### 2. Failover（故障转移模式）
- **工作方式**：始终优先使用第一个服务器，仅在故障时切换到备用服务器
- **健康检查**：按需检测（连接时实时检测），保持配置顺序
- **快速失败**：智能识别网络错误类型，快速切换
  - 端口关闭：~10ms 快速失败
  - 主机不可达：~100ms 快速失败
  - 网络超时：1000ms 超时
- **适用场景**：有主备服务器，希望主服务器承担所有流量

### 配置示例

```yaml
environment:
  # 配置多个中继服务器（逗号分隔）
  - RELAY_SERVERS=relay1.example.com:21117,relay2.example.com:21117

  # 选择负载均衡策略（可选，默认为 round-robin）
  - RELAY_STRATEGY=failover  # 或 round-robin
```

### 环境变量说明

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| RELAY_SERVERS | - | 多个中继服务器地址，用逗号分隔（如：server1:21117,server2:21117） |
| RELAY_STRATEGY | round-robin | 负载均衡策略：<br>- **round-robin**: 轮询模式，流量均匀分配<br>- **failover**: 故障转移模式，优先使用第一个服务器 |

### 两种策略对比

| 特性 | Round-Robin | Failover |
|------|-------------|----------|
| 健康检查频率 | 每 3 秒主动检测 | 按需检测（连接时） |
| 服务器选择 | 轮流使用所有健康服务器 | 始终优先使用第一个可用服务器 |
| 故障切换速度 | 最快 3 秒（下次健康检查） | 立即（10ms~1000ms） |
| 负载分配 | 均匀分配 | 主服务器承担所有流量 |
| 适用场景 | 负载均衡、多数据中心 | 主备架构、故障转移 |

## Docker Compose 配置示例

中国可以用``ghcr.nju.edu.cn``

### 基础配置（单中继服务器）

```yaml
version: '3'

networks:
  rustdesk-net:
    external: false

services:
  hbbs:
    container_name: hbbs
    ports:
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21118:21118
    image: ghcr.io/arkylin/rustdesk-server:latest
    command: hbbs -r <server[:21117]>
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    ports:
      - 21117:21117
      - 21119:21119
    image: ghcr.io/arkylin/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped
  api:
    container_name: api
    ports:
      - 21114:21114
    image: lejianwen/rustdesk-api
    environment:
      - TZ=Asia/Shanghai
      - RUSTDESK_API_RUSTDESK_ID_SERVER=<server[:21116]> #21116
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=<server[:21117]> #21117
      - RUSTDESK_API_RUSTDESK_API_SERVER=http://<server[:21114]> #21114
      #- RUSTDESK_API_RUSTDESK_KEY=<key>
    volumes:
      - /data/rustdesk/api:/app/data #将数据库挂载
      - /data/rustdesk/server:/app/conf/data #挂载key文件到api容器，可以不用使用 RUSTDESK_API_RUSTDESK_KEY
    networks:
      - rustdesk-net
    restart: unless-stopped
```

### 多中继服务器配置（推荐）

#### 方式1：使用环境变量（推荐）

```yaml
version: '3'

networks:
  rustdesk-net:
    external: false

services:
  hbbs:
    container_name: hbbs
    ports:
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21118:21118
    image: ghcr.io/arkylin/rustdesk-server:latest
    environment:
      # 多中继服务器配置（逗号分隔）
      - RELAY_SERVERS=relay1.example.com:21117,relay2.example.com:21117
      # 负载均衡策略：round-robin（轮询）或 failover（故障转移）
      - RELAY_STRATEGY=failover
      # 可选：设置日志级别
      - RUST_LOG=info
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    ports:
      - 21117:21117
      - 21119:21119
    image: ghcr.io/arkylin/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped

  api:
    container_name: api
    ports:
      - 21114:21114
    image: lejianwen/rustdesk-api
    environment:
      - TZ=Asia/Shanghai
      - RUSTDESK_API_RUSTDESK_ID_SERVER=<server[:21116]>
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=<server[:21117]>
      - RUSTDESK_API_RUSTDESK_API_SERVER=http://<server[:21114]>
    volumes:
      - /data/rustdesk/api:/app/data
      - /data/rustdesk/server:/app/conf/data
    networks:
      - rustdesk-net
    restart: unless-stopped
```

#### 方式2：使用命令行参数

```yaml
services:
  hbbs:
    container_name: hbbs
    image: ghcr.io/arkylin/rustdesk-server:latest
    command: hbbs --relay-strategy=failover -r relay1.example.com:21117,relay2.example.com:21117
    # ... 其他配置
```

### 合体镜像配置（Combined Image）

合体镜像将 hbbs、hbbr 和 API 服务整合到单个容器中，简化部署流程。

```yaml
version: '3'

networks:
  rustdesk-net:
    external: false

services:
  rustdesk:
    container_name: rustdesk-combined
    ports:
      - 21114:21114  # API 服务
      - 21115:21115  # hbbs - NAT type test
      - 21116:21116  # hbbs - ID registration & heartbeat
      - 21116:21116/udp
      - 21117:21117  # hbbr - Relay
      - 21118:21118  # hbbs - WebSocket
      - 21119:21119  # hbbr - WebSocket
    image: ghcr.io/arkylin/rustdesk-server-combined:latest-amd64
    environment:
      # 日志级别（可选）
      # - RUST_LOG=debug

      # 负载均衡策略
      - RELAY_STRATEGY=failover

      # 多中继服务器配置
      - RELAY_SERVERS=relay1.example.com:21117,relay2.example.com:21117

      # 安全配置
      - ENCRYPTED_ONLY=1  # 仅允许加密连接
      - MUST_LOGIN=Y      # 强制登录后才能连接

      # 时区配置
      - TZ=Asia/Shanghai

      # API 服务配置
      - RUSTDESK_API_RUSTDESK_ID_SERVER=server.example.com:21116
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=server.example.com:21117
      - RUSTDESK_API_RUSTDESK_API_SERVER=https://server.example.com
      - RUSTDESK_API_KEY_FILE=/data/id_ed25519.pub

      # JWT 配置（请修改为您自己的随机密钥）
      - RUSTDESK_API_JWT_KEY=your-random-jwt-key-here-please-change-it
      - RUSTDESK_API_APP_TOKEN_EXPIRE=9000h

    volumes:
      - ./data/server:/data          # hbbs/hbbr 数据目录
      - ./data/api:/app/data          # API 数据库目录
    networks:
      - rustdesk-net
    restart: unless-stopped
```

**注意事项：**
- 合体镜像适用于中小规模部署，简化运维管理
- `RUSTDESK_API_JWT_KEY` 必须修改为您自己的随机密钥
- 如需使用中国镜像源，可使用 `ghcr.nju.edu.cn/arkylin/rustdesk-server-combined:latest-amd64`
- 确保 `./data/server` 和 `./data/api` 目录具有正确的读写权限
