# Agent Worker Pool 服务设计文档

> 在 hiclaw 控制面之上，给公司内部用户提供一个浏览器版的 agent 终端。
> 用户打开网页 = 进到一个 agent TUI，等同于 `docker exec -it <worker> <agent-cmd>`。
> hiclaw 负责 worker 容器生命周期，本服务**纯消费**：从已存在的 agent worker 容器里挑一个，把 PTY 桥到浏览器。
>
> **当前默认 agent runtime**：Hermes（社区活跃、TUI 体验好）。`agent-cmd` 与发现 label 都走配置项，未来可换成其他 TUI 风格的 agent CLI。

---

## 0. 执行摘要（给领导）

### TL;DR

**agent-worker-pool 是什么**：内部用户从浏览器打开网页 → 一键进入跑在公司服务器里的 agent 终端（当前用 Hermes Agent），不用关心容器、SSH、配置。

**为什么做**：[ 填：当前内部使用 hermes/agent 的痛点 —— 例如：每人本地 `docker run`，环境分散；新员工上手成本高；公司无统一入口管理 LLM token 消费、审计 agent 行为；个人电脑性能不够跑大模型对话 ]

**核心价值**：

| 受众 | 价值 |
|------|------|
| 用户 | 浏览器一键进入，不再操心环境 / 镜像 / 配置 |
| 平台 | 统一接入点 → 统一鉴权、计费、审计 |
| 成本 | worker 复用 → 节省服务器与 token |

**规模与时间**：

| 阶段 | 周期 | 范围 |
|------|------|------|
| 1（demo） | 1 周 | 单机 + 5–10 个 worker + ~10 个内部试点用户 |
| 2（试点） | 1 周 | admin 页 + 监控 + 体验打磨 |
| 3（按需） | 视范围 | 自动伸缩 / ACL / HA |

**工程量**：阶段 1+2 ≈ **2 人周**。

### 极简架构图（一句话讲完）

```
                   HTTPS                docker.sock
   ┌──────────┐   WebSocket    ┌───────────────────┐    exec    ┌───────────────────┐
   │  浏览器   │ ─────────────▶ │ agent-worker-pool │ ─────────▶ │  Agent Worker 容器 │
   │ xterm.js │                 │   (WS↔PTY 桥)     │             │   (Hermes TUI)     │
   └──────────┘                 └───────────────────┘             └─────────┬─────────┘
                                                                            │ 调 LLM
                                                                            ▼
                                                                    ┌───────────────┐
                                                                    │   Higress     │
                                                                    │  (LLM 网关)    │
                                                                    └───────────────┘

  Worker 容器的创建 / 休眠 / 删除由 hiclaw 控制面统一负责，本服务零写权限。
```

---

## 1. 设计目标

| 目标 | 说明 |
|------|------|
| 体验透明 | 用户只看到浏览器里的 agent TUI，不感知容器/Matrix/hiclaw |
| 资源池化 | 多个用户共享一组 agent worker |
| 解耦清晰 | 本服务不调任何 hiclaw API；worker 增减由管理员通过 hiclaw 自己的前端做 |
| 复用 hiclaw | worker 创建、休眠、Higress consumer 全部继续用 hiclaw 现有能力，本服务不重做 |
| Runtime 可换 | `agent-cmd` 与发现 label 走配置；将来换其他 TUI agent 不动主代码 |

**非目标（一期不做）**：跨租户隔离、SSO 多协议、HA leader 选举、会话跨断线持久化、自动伸缩、移动端、K8s 模式。

---

## 2. 架构总览

```
┌──────────────────────── 浏览器 ────────────────────────┐
│   xterm.js + WS client                                  │
└─────────────┬───────────────────────────────────────────┘
              │ HTTPS + WSS
              ▼
┌────────────── agent-worker-pool（单二进制）───────────────┐
│   ┌─────────┐   ┌──────────┐   ┌──────────────────┐      │
│   │  BFF    │   │   WS     │   │   Scheduler      │      │
│   │  HTTP   │──▶│ Gateway  │   │ - 列容器(label)  │      │
│   │ /me/*   │   │ /ws/:sid │   │ - 选 worker      │      │
│   │ /sess/* │   │ PTY 桥   │   │ - 池占用记录     │      │
│   └─────────┘   └────┬─────┘   └────┬─────────────┘      │
│        │             │              │                    │
│        └─────────────┼──────────────┘                    │
│                      ▼                                    │
│            内存 map（会话索引 + 池占用）                   │
└──────────────────────┬────────────────────────────────────┘
                       │ docker.sock
                       │ (ps + exec + resize)
                       ▼
        ┌──────────────────────────────────────────┐
        │  Agent Worker 池（hiclaw 管理）           │
        │  hermes-1  hermes-2  hermes-3  ...        │
        │  label: hiclaw.runtime=hermes             │
        └──────────────────────────────────────────┘
                       ▲
                       │ 创建 / 休眠 / 删除
                       │
                  ┌─────────────┐
                  │  hiclaw 前端 │  ← 管理员
                  └─────────────┘
```

**边界**：
- BFF / WS Gateway / Scheduler = 同进程三个模块
- agent-worker-pool **不调 hiclaw API**，hiclaw **不调 agent-worker-pool**
- 所有 worker 容器跑在同一台宿主机的 docker 里，agent-worker-pool 通过 unix socket 直连

---

## 3. 数据通路

### 3.1 浏览器 ↔ WS Gateway

WebSocket 帧（JSON / text frame）：

| type | 方向 | 字段 | 用途 |
|------|------|------|------|
| `input` | →网关 | `data` (base64) | 键盘输入 |
| `output` | ←网关 | `data` (base64) | agent stdout/stderr（含 ANSI） |
| `resize` | →网关 | `cols`,`rows` | 终端尺寸 |
| `ping`/`pong` | 双向 | — | 心跳 |
| `closed` | ←网关 | `reason` | 会话结束 |

### 3.2 WS Gateway ↔ Agent 容器

Docker Engine HTTP API（unix socket）：

```
GET  /containers/json?filters=...      # 列 agent worker（label 过滤）
POST /containers/{name}/exec           # 创建 PTY exec, Cmd=[config.AgentCommand]
POST /exec/{id}/start  (Hijack)        # 双向 TCP 流
POST /exec/{id}/resize?h=&w=           # 调整尺寸
GET  /exec/{id}/json                   # 监控退出
```

`config.AgentCommand` 一期默认 `"hermes"`。

> 一期不存在「Scheduler ↔ hiclaw API」这条通路。

### 3.3 用户旅程图

**主路径（happy path）：**

```
  用户                浏览器              agent-worker-pool    docker            worker容器
  ────                ──────              ─────────────────    ──────            ──────────
   │                    │                     │                  │                  │
   │ 打开 URL          │                     │                  │                  │
   ├──────────────────▶│                     │                  │                  │
   │                    │ SSO 重定向          │                  │                  │
   │ ◀──────────────────┤                     │                  │                  │
   │                    │                     │                  │                  │
   │ 已登录, 进 /workspace                    │                  │                  │
   │                    │ GET /me/workers   ▶│                  │                  │
   │                    │                     │ docker ps      ▶│                  │
   │                    │ ◀ 返回池水位         │ ◀ 5个 worker    │                  │
   │                    │                     │                  │                  │
   │ 点[+ New Session] │                     │                  │                  │
   │ ──────────────────▶│ POST /sessions    ▶│                  │                  │
   │                    │                     │ Scheduler:      │                  │
   │                    │                     │  filter+score   │                  │
   │                    │                     │  → hermes-2     │                  │
   │                    │ ◀ {sid, wsUrl}      │                  │                  │
   │                    │                     │                  │                  │
   │                    │ 跳 /term/:sid       │                  │                  │
   │                    │ 建 WSS            ▶│                  │                  │
   │                    │                     │ exec create   ▶│                  │
   │                    │                     │                  │ exec start "hermes"
   │                    │                     │                  │ ──────────────▶│
   │                    │                     │                  │                  │ 渲染 TUI
   │                    │                     │ ◀ hijack stream │ ◀ stdout         │
   │                    │ ◀ output 帧          │                  │                  │
   │ ◀ xterm 渲染 logo  │                     │                  │                  │
   │                    │                     │                  │                  │
   │ 输入 "hi"         │                     │                  │                  │
   │ ──────────────────▶│ input 帧          ▶│                  │                  │
   │                    │                     │ pty write ────▶│ ─────────────▶│
   │                    │                     │                  │                  │ 调LLM
   │                    │ ◀ output (回复)      │ ◀ pty read       │ ◀                │
   │ ◀                  │                     │                  │                  │
   │                    │                     │                  │                  │
   │  ... 多轮交互 ...                                                                │
   │                    │                     │                  │                  │
   │ 点[End Session]   │                     │                  │                  │
   │ ──────────────────▶│ DELETE /sessions  ▶│                  │                  │
   │                    │                     │ kill exec     ▶│ ─SIGTERM─────▶│ 退出
   │                    │                     │ release pool   │                  │
   │                    │ ◀ 200 + closed帧    │                  │                  │
   │                    │ 跳回 /workspace     │                  │                  │
```

**异常分支：**

```
A. 池满 (T3)
   POST /sessions → Scheduler filter 后 eligible=∅ → 返回 429
   前端弹: "池子满了，请联系管理员通过 hiclaw 加 worker"
   管理员去 hiclaw 前端 → 创建新 worker → 5s 后 agent-worker-pool 巡检自动纳入
   用户重试 → 成功

B. 网络短抖
   WS read 失败 → 前端 backoff 重连 3 次
   server 端 exec 还活着 (60s 心跳窗口内) → 流恢复
   显示 "Reconnected"

C. 关 tab 后再回来
   关 tab → WS close → server 杀 exec → 释放 worker
   再开 /term/{sid} → BFF 查内存 → session 不存在 → 410 → 跳 /workspace
   (阶段3 加 dtach 才能恢复"上次的对话上下文")

D. Worker 被 hiclaw 删除
   管理员在 hiclaw 前端删 hermes-N
   5s 巡检发现少了 → 强杀该 worker 的 exec
   持有 session 的 WS 收到 closed{reason:"worker_removed"}
   前端跳 /workspace + 提示 "会话被管理员中断"
```

---

## 4. 模块说明

### 4.1 BFF

```
GET    /me                  当前用户身份
GET    /me/workers          池里全部 agent worker（信息展示用）
POST   /sessions            申请会话 → Scheduler 选 worker
GET    /sessions/{id}       会话状态
DELETE /sessions/{id}       主动结束
```

`POST /sessions` 返回 `{ sessionId, wsUrl }`。

### 4.2 WS Gateway

每个 WS 连接绑定一个 `docker exec`。一期单实例。

### 4.3 Scheduler（插件化）

**容器发现**：每 5s 调 `docker ps --filter label=<config.DiscoveryLabel> --filter status=running` 刷新池子。一期 `DiscoveryLabel=hiclaw.runtime=hermes`。新容器自动加入，消失的剔除（持有该 worker 的 session 强杀）。

**架构**：仿 K8s scheduler-framework，**Filter 链 + Scorer 链**，两条链都是 Go interface，编译期注册（不做动态加载）。

```go
type Request struct {
    UserID         string
    RequiredLabels map[string]string  // 阶段 2+ 用
    Priority       int                // 阶段 2+ 用
}

type Filter interface {
    Name() string
    Filter(ctx context.Context, req *Request, w *Worker) bool  // true = 留下
}

type Scorer interface {
    Name() string
    Score(ctx context.Context, req *Request, w *Worker) int    // 越高越优先
}

type Scheduler struct {
    filters []Filter
    scorers []Scorer
}

func (s *Scheduler) Pick(ctx, req, candidates) (*Worker, error) {
    eligible := candidates
    for _, f := range s.filters {
        eligible = applyFilter(f, ctx, req, eligible)
    }
    if len(eligible) == 0 { return nil, ErrNoCapacity }

    return pickHighestScore(s.scorers, ctx, req, eligible), nil
}
```

**一期内置插件**：

| 类型 | 插件 | 作用 |
|------|------|------|
| Filter | `NotOccupied` | 排除已被占用的 worker |
| Filter | `PerUserLimit` | 排除会让该 user 超过 N 个并发 session 的情况（一期 N=1） |
| Scorer | `SessionAffinity` | `+1000` 如果 worker 是该 user 上次用过的（软亲和） |
| Scorer | `Random` | `+rand(0..10)` 用于打散并列分数 |

> 这两个 Filter + 两个 Scorer 组合出来的行为，等价于早期版本里写死的 3 步算法（已有会话→复用 / 找空闲 / 满了 429），但代码上是数据驱动的。

**配置化（YAML）**：

```yaml
agent:
  command: hermes                          # exec 进 worker 时跑的 CLI
  discovery_label: hiclaw.runtime=hermes   # docker ps 过滤条件

scheduler:
  filters:
    - NotOccupied
    - PerUserLimit
  scorers:
    - {name: SessionAffinity, weight: 1000}
    - {name: Random, weight: 1}
```

**未来扩展点**（按需启用，不影响一期）：

| 插件 | 类型 | 启用条件 |
|------|------|----------|
| `LeastConnection` | Scorer | 1 worker 跑多 session 时（解除 1:1 限制后） |
| `LabelMatch` | Filter | 异构池（如有些 worker 带 GPU） |
| `Capacity` | Scorer | 接入容器 CPU/mem 监控后 |
| `Tenant` | Filter | 引入多租户后 |
| `Priority` | Filter+Scorer | admin 角色抢占空闲 worker |
| `RoundRobin` | Scorer | 想要"绝对均衡"的轮询场景 |

新增插件 = 实现一个 interface + 在 `init()` 里注册一行，不动 Scheduler 主循环。

「池占用」状态只在内存里，不动 docker、不调 hiclaw。

### 4.4 内存数据模型

```go
type Session struct {
    ID         string
    UserID     string
    WorkerName string
    ExecID     string
    StartedAt  time.Time
    LastSeen   time.Time
}

type Store struct {
    sessions       map[string]*Session  // sid → session
    userActive     map[string]string    // userId → sid
    workerOccupied map[string]string    // workerName → sid
    pool           map[string]bool      // workerName → discovered
}
```

心跳 30s 续约，60s 丢心跳 → 强杀 exec + 释放占用。

进程重启 → 所有 session 丢失，用户重连时 BFF 返回 410，前端跳回 /workspace 重新创建。一期可接受。

### 4.5 容器 label 约定

```
hiclaw.runtime=<runtime-name>      # 必需，用于发现；一期值=hermes
hiclaw.worker.name=<unique>         # 可选，调试用
```

发现过滤由配置项 `agent.discovery_label` 决定，换 runtime 时改配置即可。

**待确认**：hiclaw 创建 hermes worker 时是否已带这些 label。如果没有，可降级为按容器名前缀（如 `hermes-`）发现，但 label 更干净。

---

## 5. 鉴权与权限（一期最小）

```
浏览器 → 公司 IdP (OIDC/Cookie) → BFF
BFF 拿到 userId 后写 Cookie
任何已认证用户都能用池里的任何 worker
```

不与 hiclaw 共享认证体系。Human CR / accessibleWorkers 一期忽略；阶段 2 可缓存为 ACL。

---

## 6. 前端（3 个页面）

```
/workspace          工作台：my sessions + [+ New Session]
/term/:sessionId    终端：xterm.js 全屏 + header（worker 名、计时、结束按钮）
/admin              管理：池水位（几个 worker、几个被占）
                    不提供"加 worker"按钮 —— 跳转到 hiclaw 前端做
```

技术栈：xterm.js（+ addon-fit / addon-web-links）、TanStack Query、shadcn/ui。

**前端边界**：

```
✗ 不直连 docker.sock
✗ 不直接调 hiclaw API
前端只认 BFF 给的 sessionId 和 wsUrl
```

会话恢复：刷新页面 → URL 里 sessionId → BFF 验内存 → 在则重连同 PTY；不在则跳回 /workspace。

---

## 7. 关键设计决策

| 决策点 | 一期选择 | 理由 |
|--------|---------|------|
| 调 hiclaw API？ | **否，纯消费** | 解耦清晰；worker 增减由管理员通过 hiclaw 前端做 |
| Session 跨断线持久？ | **否**，关浏览器 = exec 退出 | 简单；阶段 2 加 dtach |
| 一个 worker 跑多个 session？ | **否**，1 worker = 1 session | 避免 workspace 互踩 |
| 多租户隔离？ | **否**，所有用户共享一个池 | 内部公司用 |
| K8s 后端？ | **否**，只支持本机 docker.sock | 砍一半复杂度 |
| 自动伸缩？ | **否**，池大小由管理员决定 | agent-worker-pool 不写 docker、只读 |
| 持久化（Redis/Postgres）？ | **否**，全内存 | 重启丢 session 可接受 |
| 自定义 JWT / 多协议 SSO？ | **否**，直接用公司 IdP | 不重造鉴权 |
| 多 agent runtime？ | **否**，一期单 runtime（hermes） | `agent.command` 配置可换；多 runtime 要多池要扩展 |

---

## 8. 落地路线

| 阶段 | 周期 | 目标 | 验收（建议） |
|------|------|------|-------------|
| 1 | 1 周 | 单二进制，跑通浏览器→xterm→WS→docker exec→hermes 链路；按 label 发现池 | 能在浏览器里看到 hermes TUI logo；输入回显正常；resize 跟随窗口 |
| 2 | 1 周 | admin 页 + Prometheus 指标 + 引导跳 hiclaw 前端 | 20 并发会话稳定 1 周；P95 冷启动 < 5s；至少 10 个真实用户用过 1 次以上 |
| 3 | 按需 | 调 hiclaw API（自动伸缩）/ ACL / Redis / HA / session 跨断线持久 | 视后续需求拆任务 |

---

## 9. 可观测性（一期最小集）

```
agent_worker_pool_workers_total{state}      # state=available|occupied
agent_worker_pool_session_active
agent_worker_pool_session_wait_seconds
agent_worker_pool_cold_start_seconds
```

日志：BFF 访问日志、WS 生命周期、调度决策 → 标准输出 → 公司日志栈。

---

## 10. 关键风险与对策

| 风险 | 对策 |
|------|------|
| WS 被反代超时 | nginx `proxy_read_timeout 86400; proxy_buffering off;` + 30s ping/pong |
| docker.sock 是 root 权限 | agent-worker-pool 跑在专机；最小用户运行 |
| agent 进程泄漏 | 60s 心跳超时强杀 exec |
| 一个用户占满池 | 每用户 1 个 session（内存表唯一约束） |
| 池打满 | 429 + 前端提示「联系管理员」 |
| 容器消失（hiclaw 重建/删除） | 5s 巡检剔除；持有该 worker 的 session 强杀 |
| 浏览器 Cmd+W 误关 | beforeunload 警告 |
| hiclaw worker 镜像/CLI 升级破坏兼容 | 锁版本 + 升级前联调；阶段 2 加冒烟测试 |
| LLM 成本超预期 | 沿用 hiclaw 现有 token 预算上限；阶段 2 上指标看板 |

---

## 11. 与 hiclaw 的边界

| 能力 | 谁负责 | 接口 |
|------|--------|------|
| 创建 agent worker 容器 | **hiclaw** | hiclaw 前端（管理员触发） |
| Worker 休眠 / 唤醒 / 删除 | **hiclaw** | hiclaw 前端 |
| LLM token / Higress consumer | **hiclaw** | 自动 provision |
| Worker 容器打 label | **hiclaw**（待确认） | 见 §4.5 |
| 容器发现 + 会话路由 + WS↔PTY 桥 | **agent-worker-pool** | 本服务 |
| 浏览器前端 | **agent-worker-pool** | 本服务 |

> agent-worker-pool 对 hiclaw 的依赖**仅限于：hiclaw 把符合 label 约定的 agent worker 容器跑在同一台宿主机的 docker 里**。除此之外两边零耦合。

---

## 12. 实现参考（Portainer）

WS↔docker exec 这条核心数据通路，Go 生态已有成熟开源参考 —— **Portainer**。直接对照抄即可，不需要从零摸索。

### 12.1 关键文件

| 文件 | 作用 |
|------|------|
| [`api/http/handler/websocket/exec.go`](https://github.com/portainer/portainer/blob/develop/api/http/handler/websocket/exec.go) | HTTP 入口：参数校验 → upgrade → 调 hijack |
| [`api/ws/hijack.go`](https://github.com/portainer/portainer/blob/develop/api/ws/hijack.go) | 编排层：发 `/exec/{id}/start` + 起两个 goroutine pump |
| [`api/ws/stream.go`](https://github.com/portainer/portainer/blob/develop/api/ws/stream.go) | 两个 pump 实体：`StreamFromWebsocketToWriter` + `WriteReaderToWebSocket` |
| [`api/ws/resize.go`](https://github.com/portainer/portainer/blob/develop/api/ws/resize.go) | resize 帧处理 |

### 12.2 核心模式（30 行）

```go
errorChan := make(chan error, 2)
go StreamFromWebsocketToWriter(ws, conn, errorChan)   // WS read → PTY stdin
go WriteReaderToWebSocket(ws, &mu, conn, errorChan)   // PTY stdout → WS write
err := <-errorChan                                     // 任一方挂掉就结束
```

任一 pump 出错 → errorChan → 主流程返回 → `defer conn.Close()` → 另一 pump 阻塞的 Read 立即 EOF → 自然退出。**这条 defer 是避免 goroutine 泄漏的关键。**

### 12.3 与 agent-worker-pool 的差异

| 维度 | Portainer | agent-worker-pool |
|------|-----------|-------------------|
| 与 docker 通信 | 自己拼 HTTP `/exec/{id}/start` + 手工 TCP hijack | 用 Docker SDK `ContainerExecAttach`，直接拿 `types.HijackedResponse{Conn, Reader}` |
| 帧编码 | 直接转发裸字节（text/binary） | 一期同样裸字节 binary frame；控制帧（resize/ping/closed）走 JSON |

### 12.4 落到 T6 的实现路径

1. 抄 `stream.go` 两个函数，改帧解析（input/resize/pong 分支）
2. 抄 `hijack.go` 的 errorChan 编排，**跳过** sendHTTPRequest（SDK 已做）
3. resize 用 SDK 的 `ContainerExecResize` 替代 Portainer 自己拼的 HTTP

T6 估时从 2 天压到 ~1 天。

---

## 13. 资源与成本估算

> 数字基于「试点 ≈10 个并发用户、每用户独占一个 worker」估算。正式上线前用真实数据校准。

### 13.1 服务器

| 用途 | 规格 | 数量 | 备注 |
|------|------|------|------|
| Pool 宿主机 | [ 填：8C16G / 200G SSD ] | 1 台 | 跑 agent-worker-pool 二进制 + 5–10 个 hermes worker 容器 |
| 反向代理 | [ 填：2C4G ] | 1 台 | nginx + SSL；可与公司现有网关复用 |

试点期单机即可；阶段 3 走 HA / 多机时再扩。

### 13.2 LLM 成本

- LLM token 走 hiclaw 现有 Higress consumer，**本服务不引入新 LLM 渠道**
- 月度预算口径：[ 填：沿用 hiclaw 现有月度预算，初期不增加 ]
- 风险：用户增长 → token 消耗超预期；阶段 2 上 Prometheus 实时监控

### 13.3 工程人力

| 阶段 | 估时 | 角色 |
|------|------|------|
| 阶段 1（demo） | ~1 周 × 1 人 | Go 后端 + 前端简化版 |
| 阶段 2（试点） | ~1 周 × 1 人 | 监控 / admin 页 / 体验打磨 |
| 阶段 3（按需） | 视范围 | 自动伸缩 / ACL / HA |

阶段 1+2 合计 ≈ **2 人周**（不含设计、评审、试点用户支持）。

### 13.4 长期运维

| 项 | 方案 |
|----|------|
| 部署 | 复用公司现有部署平台 |
| 监控 | 复用现有 Prometheus + 日志栈 |
| On-call | [ 填：由谁负责 ] |
| SLO | [ 填：例如 99.5% 可用 / P95 冷启动 < 5s / 单会话稳定 30 分钟无中断 ] |

---

## 14. 关键依赖项（需领导拉通）

| # | 事项 | 说明 | 责任方 | 状态 |
|---|------|------|--------|------|
| 1 | 服务器资源申请 | Pool 宿主机 1 台 + 反代 1 台（见 §13.1） | IT / 运维 | [ ] |
| 2 | 域名 + SSL 证书 | `agent-pool.<company>.com` | 网络 / 安全 | [ ] |
| 3 | 公司 IdP 接入 | OIDC 或 SAML 任一即可，需要接口人 | 身份组 | [ ] |
| 4 | hiclaw 团队配合 | 确认 §4.5 容器 label；提供 hermes worker 镜像 SLA | hiclaw 团队 | [ ] |
| 5 | LLM 预算上限 | 复用 hiclaw 现有；明确试点期消耗预期 | 财务 / hiclaw 团队 | [ ] |
| 6 | 安全审查 | docker.sock 暴露范围、SSO 接入流程 | 安全团队 | [ ] |
| 7 | 试点用户招募 | [ 填：哪个团队、N 个人、试点周期 ] | 业务方 | [ ] |

> 1–6 任一未拉通都会卡阶段 1 上线。建议在 kickoff 会议上就并行启动。

---

## 附录：后续可能的扩展点

> 一期 YAGNI，列出来只是为了不被「以后做不到」的担忧绑架。

- **自动伸缩**：scheduler 池水位低 → 调 hiclaw API 创建 → 引入 hiclaw 客户端 + token
- **ACL**：BFF 启动时同步 hiclaw Human CR，缓存 user→accessibleWorkers
- **Session 跨断线持久**：agent 进程外包一层 dtach
- **K8s 后端**：docker.sock → SPDY exec stream
- **多租户**：hiclaw 加 Tenant CRD；池按 tenant 拆
- **HA**：scheduler 单点变多点，引入 Redis 共享池占用状态
- **多 agent runtime 同池共存**：Channel 抽象（`webterm | http | matrix`），调度器按 runtime 分流
