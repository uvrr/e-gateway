# Emby 网关系统架构文档

## 项目概述

Emby 网关是一个统一的媒体聚合系统，为多个 Emby 服务器提供统一入口，实现用户认证、媒体聚合、智能播放路由和进度同步等功能。

### 技术栈

**前端**:

- Angular 15+ (TypeScript)
- RxJS (响应式编程)
- Angular Material (UI 组件库)
- SCSS (样式)

**后端**:

- Go 1.21+ (主要语言)
- Gin (Web 框架)
- GORM (ORM)
- JWT (认证)

**数据库**:

- SQLite (默认，适合小规模部署)
- PostgreSQL (推荐生产环境)
- MySQL (可选)

**部署方式**:

- Docker (容器化部署)
- Kubernetes (集群部署)
- 二进制文件 (直接部署)

### 架构特点

- **前后端分离**: 前端 Angular SPA，后端 Go RESTful API
- **多数据库支持**: 通过 GORM 抽象层支持多种数据库
- **容器化**: 提供 Docker 镜像和 K8s 部署配置
- **可扩展**: 支持水平扩展和负载均衡

## 基于 spec-workflow 的项目管理

本项目使用 spec-workflow 进行规范化管理，文档位于 `.spec-workflow/specs/emby-gateway/`

## 系统总体架构

     3→```
     4→┌──────────────────────┐         ┌──────────────────────────┐
     5→│          前端应用层          │         │      多个 Emby 服务器     │
     6→│ (Web / App / TV / Mini App) │         │ (Server A / B / C / ...) │
     7→└───────────────┬──────────────┘         └─────────────┬────────────┘
     8→                │                                        │
     9→                ▼                                        │
    10→      ┌────────────────────┐                              │
    11→      │    Emby 网关系统   │   登录、鉴权、多服务器路由    │
    12→      │ (独立系统 + 统一入口) │◀──────────────────────────────┘
    13→      ├────────────────────┤
    14→      │● 用户统一鉴权      │
    15→      │● 多服务器账号管理  │
    16→      │● 影视信息聚合读取  │
    17→      │● 播放代理与转发    │
    18→      │● 播放进度同步      │
    19→      │● 缓存 + 日志跟踪    │
    20→      └────────────────────┘

## 1. 系统总体架构

```
┌──────────────────────────────┐         ┌──────────────────────────┐
│          前端应用层          │         │      多个 Emby 服务器     │
│ (Web / App / TV / Mini App) │         │ (Server A / B / C / ...) │
└───────────────┬──────────────┘         └─────────────┬────────────┘
                │                                        │
                ▼                                        │
      ┌────────────────────┐                              │
      │    Emby 网关系统   │   登录、鉴权、多服务器路由    │
      │ (独立系统 + 统一入口) │◀──────────────────────────────┘
      ├────────────────────┤
      │● 用户统一鉴权      │
      │● 多服务器账号管理  │
      │● 影视信息聚合读取  │
      │● 播放代理与转发    │
      │● 播放进度同步      │
      │● 缓存 + 日志跟踪    │
      └────────────────────┘
```

## 2. 架构模块拆分

### 2.1 网关核心模块

| 模块 | 功能描述 |
|------|-----------|
| **Auth 模块** | 前端 → 网关统一登录，网关使用多个 Emby 账号登录多个服务器并保存 Token |
| **Server Registry（服务器注册表）** | 保存多个 Emby Server 的连接信息、登录令牌、可用状态 |
| **Media Aggregator（媒体聚合器）** | 向多个 Emby Server 拉取媒体列表、搜索、多源合并去重 |
| **Playback Proxy（播放代理）** | 前端播放请求 → 选择最佳服务器 → 返回播放 URL（HLS/DASH/MP4） |
| **Progress Broadcaster（进度广播器）** | 播放进度回传 → 同步到所有拥有该 Item 的 Emby Server |
| **Cache + 限流模块** | 提升网关性能，避免多服务器压力 |
| **日志 & 追踪模块** | API 请求链路跟踪、错误追踪，便于多服务器环境排查 |

### 2.2 数据流核心逻辑

#### 用户登录流程

```
前端 → Emby 网关 → 网关使用多个 Server 的账号登录 → 保存 Token → 返回网关 Token
```

#### 拉取影视信息流程

```
前端 → /media/search
      → 网关调用多个 Emby Server /Items/Search
      → 网关合并去重（依据名称 / TMDB ID / IMDB ID）
      → 返回统一格式的媒体数据
```

#### 播放流程

```
前端 → /media/{id}/play
      → 网关根据媒体 ID 查找在哪些服务器存在
      → 选择最佳服务器（可按：距离、网络延迟、服务器负载、原始码率）
      → 返回该 Emby Server 的真实播放地址（HLS/DASH/MP4）
```

#### 播放进度同步流程（关键）

```
播放器 → /progress/report
       → 网关接收进度
       → 网关查找所有含该 Item 的服务器
       → 对每个服务器调用：POST /Sessions/Playing/Progress
```

## 3. API 设计（网关侧）

emby 连接使用 <https://github.com/MediaBrowser/Emby.ApiClients/blob/master/Clients/Go>
采用 User Authentication 进行认证
相关api可查询 <https://swagger.emby.media/?staticview=true>

### 3.1 登录相关

#### POST /auth/login

```
请求：{ username, password }
返回：{ token }
```

网关会自动登录多个 Emby Server，保存多个 token。

### 3.2 媒体相关

#### GET /media/search?q=xxx

返回：聚合媒体列表（合并去重）

#### GET /media/{itemId}

返回：统一格式的元数据（来自多个服务器）

#### POST /media/{itemId}/play

返回：

```
{
  "server": "emby-A",
  "streamUrl": "https://emby-a.xxx/Video/.../main.m3u8"
}
```

### 3.3 播放进度同步

#### POST /progress/report

```
请求：{ itemId, positionTicks, isPaused, isFinished }
```

网关：

1. 查找拥有 itemId 的所有服务器
2. 逐个调用：`POST /Sessions/Playing/Progress`（Emby API）

## 4. 网关内部数据库结构

### 4.1 servers

```
{
  id: string,
  name: string,
  url: string,
  username: string,
  password: string,
  token: string,
  lastSync: datetime,
  isOnline: boolean
}
```

### 4.2 media_index

```
用于记录跨服务器媒体的统一 ID 映射。

{
  gatewayItemId: string,
  serverItems: [
    { serverId: "A", itemId: "123" },
    { serverId: "B", itemId: "889" }
  ],
  tmdbId: "",
  imdbId: "",
  name: "",
  year: 2023
}
```

## 5. 核心组件示例代码（TypeScript 伪代码）

### 5.1 服务器登录并缓存 Token

```ts
async function loginToServer(server) {
  const res = await fetch(server.url + "/Users/AuthenticateByName", {
    method: "POST",
    body: JSON.stringify({ Username: server.username, Pw: server.password })
  });

  const data = await res.json();
  server.token = data.AccessToken;
  db.saveServer(server);
}
```

### 5.2 聚合搜索媒体

```ts
async function searchAllServers(query) {
  const results = [];
  for (const server of servers) {
    const r = await fetch(server.url + `/Items?SearchTerm=${query}`, {
      headers: { 'X-Emby-Token': server.token }
    });
    results.push(...(await r.json()).Items);
  }

  return mergeMedia(results); // 按 tmdbId/imdbId/name 去重
}
```

### 5.3 播放进度广播

```ts
async function broadcastProgress(gatewayItemId, progress) {
  const mapping = db.findMediaMapping(gatewayItemId);

  for (const item of mapping.serverItems) {
    await fetch(item.serverUrl + `/Sessions/Playing/Progress`, {
      method: "POST",
      headers: { 'X-Emby-Token': item.token },
      body: JSON.stringify(progress)
    });
  }
}
```

## 6. 网关与 Emby Server 状态检测

- 每 60s 执行一次健康检查 `/System/Info/Public`
- server.isOnline = false → 从路由中剔除
- 自动重连登录

## 7. 部署架构

### 7.1 部署拓扑

```
前端（Angular SPA）
      ↓
Nginx / Traefik / Cloudflare
      ↓
Emby 网关集群（Go 服务）
      ↓
数据库（SQLite/PostgreSQL/MySQL）
      ↓
多个 Emby 服务器
```

### 7.2 Docker 部署

**Docker Compose 示例**:

```yaml
version: '3.8'
services:
  gateway:
    image: emby-gateway:latest
    ports:
      - "8080:8080"
    environment:
      - DB_TYPE=postgres
      - DB_HOST=postgres
      - DB_PORT=5432
    volumes:
      - ./config:/app/config
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=emby_gateway
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data

  frontend:
    image: emby-gateway-frontend:latest
    ports:
      - "80:80"
    depends_on:
      - gateway

volumes:
  pgdata:
```

### 7.3 Kubernetes 部署

**部署清单示例**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emby-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: emby-gateway
  template:
    metadata:
      labels:
        app: emby-gateway
    spec:
      containers:
      - name: gateway
        image: emby-gateway:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_TYPE
          value: "postgres"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: emby-gateway-service
spec:
  selector:
    app: emby-gateway
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### 7.4 二进制部署

**配置文件 (config.yaml)**:

```yaml
server:
  port: 8080
  mode: release

database:
  type: sqlite  # sqlite, postgres, mysql
  path: ./data/emby_gateway.db
  # host: localhost
  # port: 5432
  # name: emby_gateway
  # user: admin
  # password: secret

jwt:
  secret: your-secret-key
  expire: 24h

emby:
  servers:
    - name: Server-A
      url: https://emby-a.example.com
      username: admin
      password: password
    - name: Server-B
      url: https://emby-b.example.com
      username: admin
      password: password

cache:
  enabled: true
  ttl: 300s

log:
  level: info
  format: json
```

**启动命令**:

```bash
# 使用默认配置
./emby-gateway

# 指定配置文件
./emby-gateway --config /path/to/config.yaml

# 指定端口
./emby-gateway --port 8080
```

### 7.5 部署特性

- **水平扩展**: 支持多实例部署，通过负载均衡分发请求
- **数据持久化**: 支持 SQLite（单机）、PostgreSQL/MySQL（集群）
- **健康检查**: 提供 `/health` 和 `/ready` 端点
- **优雅关闭**: 支持 SIGTERM 信号处理
- **日志收集**: 结构化日志，支持 ELK/Loki 集成
- **监控指标**: Prometheus metrics 端点

## 8. 项目结构

### 8.1 后端结构 (Go)

```
backend/
├── cmd/
│   └── gateway/
│       └── main.go              # 入口文件
├── internal/
│   ├── api/                     # API 层
│   │   ├── handler/            # HTTP 处理器
│   │   ├── middleware/         # 中间件
│   │   └── router/             # 路由配置
│   ├── service/                # 业务逻辑层
│   │   ├── auth/               # 认证服务
│   │   ├── media/              # 媒体聚合服务
│   │   ├── playback/           # 播放代理服务
│   │   └── progress/           # 进度同步服务
│   ├── repository/             # 数据访问层
│   │   ├── server/             # 服务器仓储
│   │   └── media/              # 媒体索引仓储
│   ├── model/                  # 数据模型
│   ├── config/                 # 配置管理
│   └── pkg/                    # 工具包
│       ├── emby/               # Emby API 客户端
│       ├── cache/              # 缓存
│       └── logger/             # 日志
├── migrations/                 # 数据库迁移
├── config.yaml                 # 配置文件
├── Dockerfile
└── go.mod
```

### 8.2 前端结构 (Angular)

```
frontend/
├── src/
│   ├── app/
│   │   ├── core/               # 核心模块
│   │   │   ├── auth/          # 认证
│   │   │   ├── interceptors/  # HTTP 拦截器
│   │   │   └── services/      # 核心服务
│   │   ├── shared/             # 共享模块
│   │   │   ├── components/    # 共享组件
│   │   │   ├── pipes/         # 管道
│   │   │   └── directives/    # 指令
│   │   ├── features/           # 功能模块
│   │   │   ├── media/         # 媒体浏览
│   │   │   ├── player/        # 播放器
│   │   │   └── settings/      # 设置
│   │   ├── app.component.ts
│   │   └── app.module.ts
│   ├── assets/                 # 静态资源
│   ├── environments/           # 环境配置
│   └── styles/                 # 全局样式
├── angular.json
├── package.json
├── Dockerfile
```

## 9. 开发路线图

### Phase 1: 后端基础 (2-3周)

- [ ] 项目初始化和配置管理
- [ ] 数据库模型和迁移
- [ ] Emby API 客户端封装
- [ ] 认证和 JWT 实现

### Phase 2: 核心功能 (3-4周)

- [ ] 服务器注册和管理
- [ ] 媒体聚合和搜索
- [ ] 播放代理和路由
- [ ] 进度同步机制

### Phase 3: 前端开发 (3-4周)

- [ ] Angular 项目搭建
- [ ] 认证和路由守卫
- [ ] 媒体浏览界面
- [ ] 播放器集成

### Phase 4: 优化和部署 (2-3周)

- [ ] 缓存优化
- [ ] 性能测试
- [ ] Docker 镜像构建
- [ ] K8s 部署配置

## 10. 特点总结

- **完整隔离**: 前端永远不知道真实 Emby Server
- **多源整合**: 超越 Emby 官方界面
- **播放进度共享**: 多服务器同时学习同用户记录
- **支持 Emby 协议**: TV / App / Web 播放
- **前端统一入口**: 单一访问点
- **前后端分离**: Angular + Go 独立部署
- **多数据库支持**: SQLite/PostgreSQL/MySQL
- **容器化部署**: Docker/K8s 支持
- **水平扩展**: 支持集群部署

### 可使用emby测试环境

测试服务器地址： <http://192.168.3.171:8096>
账号：ssr
密码：123456
