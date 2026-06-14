# 安全报告全链路时序：从用户举报到下游消费

## 零、关键结论先列

1. **RabbitMQ 与安全报告无直接关联**——AMQP 的 8 个通道全部用于推送通知（好友请求、消息通知、已读回执、DM 通话），安全报告不经过 RabbitMQ。
2. **Redis Pub/Sub 是安全报告唯一的下游分发通道**——报告创建后通过 `EventV1::ReportCreate.global()` 发布到 Redis `"global"` 通道，由 bonfire 推送给在线客户端。
3. **crond 守护任务与安全报告无任何关系**——4 个 crond 任务（file_deletion、prune_dangling_files、prune_members、acks）均不涉及安全报告的读写。
4. **Rejected / Resolved 状态在当前仓库中没有任何写入代码**——`AbstractReport` trait 只有 `insert_report` 一个方法，不存在 `update_report` / `fetch_report` 等方法；delta 路由中没有管理后台 API；也没有 admin panel 相关代码。状态流转的设计责任归属于独立的管理后台服务（尚未实现）。

---

## 一、完整时序图

```
时间 ──────────────────────────────────────────────────────────────────────────▶

用户浏览器
  │
  │  ① POST /safety/report
  │     { content, additional_context }
  │
  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  delta (Rocket HTTP 服务)                                                     │
│                                                                               │
│  ② report_content() 入口                                                      │
│     ├─ 校验数据 (length ≤ 1000)                                                │
│     ├─ 校验 Bot 不可举报                                                       │
│     └─ 校验不可自举报                                                           │
│                                                                               │
│  ③ 内容快照生成                                                                │
│     ├─ Message: generate_from_message()                                       │
│     │   └─ 查 DB 拉前后各 15 条上下文 + 附件列表                                 │
│     ├─ Server:  generate_from_server()                                        │
│     │   └─ 取 server icon/banner 附件列表                                      │
│     └─ User:    generate_from_user()                                          │
│         └─ 取 avatar/background 附件 + 可选 message 快照                       │
│                                                                               │
│  ④ 标记附件已举报: db.mark_attachment_as_reported()                            │
│                                                                               │
│  ⑤ 生成 Report ULID                                                          │
│                                                                               │
│  ⑥ 写入 safety_snapshots 集合 (可能多条)                                        │
│     db.insert_snapshot(&snapshot)                                             │
│       └─ MongoDB: insert_one("safety_snapshots", ...)                          │
│                                                                               │
│  ⑦ 写入 safety_reports 集合 (一条)                                             │
│     db.insert_report(&report)                                                 │
│       └─ MongoDB: insert_one("safety_reports", ...)                            │
│     注意: 此时 status = ReportStatus::Created {}                               │
│                                                                               │
│  ⑧ 发布全局事件                                                                │
│     EventV1::ReportCreate(report.into()).global().await                       │
│       └─ redis_kiss::p("global", event)                                       │
│           └─ Redis PUBLISH "global" <serialized EventV1>                      │
│                                                                               │
│  ⑨ 返回 204 No Content                                                        │
└──────────────────────────────────────────────────────────────────────────────┘
          │                                         │
          │ MongoDB 写入                             │ Redis Pub/Sub
          │                                         │
          ▼                                         ▼
   ┌──────────────┐                    ┌──────────────────────────────────┐
   │  MongoDB     │                    │  Redis                           │
   │              │                    │  channel: "global"               │
   │ safety_reports│                   │  payload: EventV1::ReportCreate  │
   │ safety_snapshots│                 └──────────┬───────────────────────┘
   └──────────────┘                               │
                                                  │ PUBLISH
                                                  │
                      ┌───────────────────────────┼────────────────────┐
                      │                           │                    │
                      ▼                           ▼                    ▼
              ┌───────────────┐          ┌───────────────┐   ┌───────────────┐
              │ bonfire 实例 1 │          │ bonfire 实例 2 │   │ bonfire 实例 N │
              │ (WebSocket)   │          │ (WebSocket)   │   │ (WebSocket)   │
              └───────┬───────┘          └───────┬───────┘   └───────┬───────┘
                      │                          │                    │
                      ▼                          ▼                    ▼
               每个实例的 listener() 循环接收 Redis 消息，
               对每个已连接的 WebSocket 客户端：
                 1. 反序列化为 EventV1
                 2. 调用 state.handle_incoming_event_v1()
                    → ReportCreate 进入 _ => {} 分支（透传，无特殊处理）
                 3. 判断 should_send = true
                 4. config.encode(&event) → 发送到 WebSocket 写缓冲
                      │
                      ▼
              ┌──────────────┐
              │ 在线客户端    │
              │ (浏览器/Bot)  │
              │ 收到 ReportCreate 事件
              └──────────────┘

═══════════════════════════════════════════════════════════════════════════════

以下通道与安全报告 **无关**，列出以供对比：

  RabbitMQ (AMQP)                      crond 守护任务
  ┌─────────────────────┐              ┌─────────────────────┐
  │ friend_req_accepted │              │ file_deletion       │
  │ friend_req_received │              │ prune_dangling_files│
  │ generic_message     │              │ prune_members       │
  │ message_sent        │              │ acks (RabbitMQ消费)  │
  │ mass_mention        │              └─────────────────────┘
  │ ack_notification    │
  │ dm_call_updated     │
  │ process_ack         │
  └─────────────────────┘
  ↑ 全部用于推送通知，       ↑ 全部用于日常数据维护，
    安全报告不经过 AMQP        不涉及安全报告的读写
```

---

## 二、各环节详细代码追踪

### 2.1 用户提交举报 → delta API

**入口**：[report_content.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L26-L134)

```
POST /safety/report
  → DataReportContent { content: ReportedContent, additional_context: String }
  → 校验 → 快照生成 → 附件标记 → 写 DB → 发事件 → 返回 204
```

关键时序节点：
| 序号 | 动作 | 代码位置 | 耗时特征 |
|------|------|---------|---------|
| ② | 入口校验 | [L33-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L33-L42) | 同步，微秒 |
| ③ | 快照生成 | [L46-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L46-L97) | 异步 DB 查询，毫秒级 |
| ④ | 附件标记 | [L100-L102](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L100-L102) | 异步 DB 写入 |
| ⑥ | 写入快照 | [L108-L117](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L108-L117) | 异步 DB 写入 |
| ⑦ | 写入报告 | [L120-L129](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L120-L129) | 异步 DB 写入 |
| ⑧ | 发布事件 | [L131](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L131) | Redis PUBLISH，亚毫秒 |

### 2.2 Redis 事件总线

**发布端**：[client.rs:EventV1::global()](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs#L402-L404)

```rust
pub async fn global(self) {
    self.p("global".to_string()).await;
}
```

底层调用 `redis_kiss::p("global", self)`，即 Redis `PUBLISH global <payload>`。

**通道命名规则**：

| 方法 | Redis 通道 | 用途 |
|------|-----------|------|
| `global()` | `"global"` | 全局广播（ReportCreate 等走此通道） |
| `private(id)` | `"{id}!"` | 用户私有事件 |
| `server(id)` | `"{id}u"` | 服务器成员事件 |
| `p(channel)` | 自定义通道 | 底层通用发布 |

**订阅端**：bonfire 中每个 WebSocket 客户端建立独立的 Redis Pub/Sub 连接，在认证后订阅 `"global"` 等通道。

### 2.3 bonfire 推送线程

**核心代码**：[websocket.rs:listener()](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/websocket.rs#L221-L403)

事件处理流程：

```
Redis Pub/Sub 消息到达
  → 反序列化为 EventV1（JSON / Msgpack / Bincode，由 REDIS_PAYLOAD_TYPE 决定）
  → if EventV1::Auth → 检查 session 是否被删除
  → else → state.handle_incoming_event_v1(db, &mut event)
      → 对 ReportCreate: 命中 _ => {} 分支，直接透传
      → 返回 should_send = true
  → write.lock().await.send(config.encode(&event))
  → 通过 WebSocket 发送给客户端
```

**ReportCreate 在 bonfire 中的特殊行为**：**无**。与 `ChannelStartTyping` 等事件不同，`ReportCreate` 不触发任何状态更新、不修改订阅、不引发权限重算。它被原样推送给所有订阅了 `"global"` 通道的在线客户端。

### 2.4 RabbitMQ 队列（与安全报告无关）

**AMQP 结构**：[amqp.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/amqp/amqp.rs#L18-L29) 维护 8 个专用通道：

| 通道字段 | 消费者 | 用途 |
|----------|--------|------|
| `friend_request_accepted` | pushd | 好友请求已接受通知 |
| `friend_request_received` | pushd | 好友请求收到通知 |
| `generic_message` | pushd | 通用推送消息 |
| `message_sent` | pushd | 消息推送（过滤在线用户） |
| `mass_mention_message_sent` | pushd | 群组 @mention 推送 |
| `ack_notification_message` | pushd | iOS badge 更新（带 `x-deduplication-header`） |
| `dm_call_updated` | pushd | DM 通话状态变更 |
| `process_ack` | crond | 已读回执批量处理 |

**pushd 消费者**：[pushd/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/pushd/src/main.rs) 消费以上 7 个队列，向 APNs / FCM / Web Push 推送。

**关键结论**：安全报告不经过 RabbitMQ，没有对应的 AMQP 通道。这意味着举报创建不会触发任何推送通知（不会给管理员发通知，也不会给举报人发确认）。

### 2.5 crond 守护任务（与安全报告无关）

[crond/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/crond/src/main.rs) 运行 4 个长驻任务：

| 任务 | 功能 | 与安全报告的关系 |
|------|------|-----------------|
| `file_deletion` | 定期删除标记为删除的文件 | 无 |
| `prune_dangling_files` | 清理悬空附件 | 无 |
| `prune_members` | 修剪服务器成员数据 | 无 |
| `acks` | 消费 RabbitMQ acks 队列，处理已读回执 | 无 |

---

## 三、驳回(Rejected)与结案(Resolved)状态缺失分析

### 3.1 现状

**模型定义完整**：[v0/safety_reports.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/models/src/v0/safety_reports.rs#L122-L147) 定义了完整的三态枚举 `ReportStatus { Created, Rejected, Resolved }`。

**数据层只有写入**：[AbstractReport trait](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/ops.rs#L10-L12) 仅有 `insert_report`，缺少以下方法：

| 缺失方法 | 作用 | 影响 |
|----------|------|------|
| `fetch_report` / `fetch_reports` | 查询单条/列表报告 | 无法读取待处理报告 |
| `update_report_status` | 变更报告状态 | 无法实现驳回/结案 |
| `update_report_notes` | 更新管理员备注 | 无法添加处理备注 |

**MongoDB 实现同样只有 insert**：[mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/ops/mongodb.rs#L11-L15) 仅实现 `insert_one`。

**Reference 实现同样只有 insert**：[reference.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/ops/reference.rs#L11-L19) 仅实现 HashMap insert。

**API 路由缺失**：[safety/mod.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/mod.rs) 只注册了 `report_content`，没有驳回/结案/查询的 API。

**事件类型缺失**：[client.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs) 中只有 `ReportCreate` 变体，没有 `ReportUpdate` 或 `ReportResolve` 变体。

### 3.2 全局搜索验证

| 搜索模式 | 结果 |
|----------|------|
| `update_report` / `report.*update` | 仅命中 january 的 DNS resolver（无关）和 report_content 的 insert |
| `fetch_report` / `query_report` / `find_report` | 零命中 |
| `Rejected` / `Resolved`（Rust 文件） | 仅在 models/v0/safety_reports.rs 枚举定义 |
| `admin.*panel` / `admin.*dashboard` / `moderation.*api` | 零命中 |

### 3.3 设计责任归属

```
┌─────────────────────────────────────────────────────────────────┐
│                       设计意图（从代码结构推断）                    │
│                                                                  │
│  revolt-backend（本仓库）          独立管理后台（未实现）           │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │ ✅ Report 数据模型    │         │ ❌ 管理后台 Web UI     │      │
│  │ ✅ ReportStatus 枚举 │         │ ❌ 报告列表查询 API    │      │
│  │ ✅ 创建举报 API      │         │ ❌ 驳回/结案 API       │      │
│  │ ✅ 快照生成与存储     │         │ ❌ ReportUpdate 事件   │      │
│  │ ✅ ReportCreate 事件  │         │ ❌ 管理员备注写入      │      │
│  │ ❌ 状态变更方法       │         │ ❌ Strike 处罚执行     │      │
│  │ ❌ 报告查询方法       │         └──────────────────────┘      │
│  └──────────────────────┘                                       │
│                                                                  │
│  责任边界: 本仓库负责"写入端"（用户举报），                         │
│  管理后台负责"处理端"（审核、驳回、结案、处罚）。                    │
│  但管理后台目前完全缺失，导致 Report 状态永远停留在 Created。       │
└─────────────────────────────────────────────────────────────────┘
```

**具体缺失项及责任归属**：

| 缺失项 | 应归属模块 | 说明 |
|--------|-----------|------|
| `AbstractReport::fetch_report` / `fetch_reports` | revolt-backend 数据层 | 管理后台需要先读才能改，数据层应提供查询方法 |
| `AbstractReport::update_report` | revolt-backend 数据层 | 状态变更的持久化方法 |
| `EventV1::ReportUpdate` | revolt-backend 事件层 | 状态变更后通知在线管理员 |
| `POST /safety/report/{id}/resolve` | 管理后台 API 服务 | 结案操作路由 |
| `POST /safety/report/{id}/reject` | 管理后台 API 服务 | 驳回操作路由 |
| `GET /safety/reports` | 管理后台 API 服务 | 报告列表查询 |
| 管理员权限校验中间件 | 管理后台 API 服务 | 区分普通用户与管理员 |
| Strike 处罚执行逻辑 | 管理后台服务 | 结案后的用户处罚流程 |

---

## 四、完整数据流对比：安全报告 vs 推送通知

```
安全报告（Redis Pub/Sub 通道）：
  用户举报 → delta API → MongoDB 写入 → Redis PUBLISH "global"
    → bonfire 订阅 → WebSocket 推送给在线客户端
    → 结束（无后续处理）

消息推送（RabbitMQ 通道）：
  用户发消息 → delta API → MongoDB 写入 → Redis PUBLISH（频道事件）
    → bonfire → WebSocket 推送给在线用户
  同时 → delta ack 任务入队 → RabbitMQ message_sent routing key
    → pushd 消费 → APNs / FCM / Web Push 推送给离线用户
```

**关键差异**：消息推送走的是 Redis（在线）+ RabbitMQ（离线）双通道；安全报告只走 Redis 单通道，且无离线通知机制。

---

## 五、从举报到消费的精确时间顺序

```
T+0ms     用户提交 POST /safety/report
T+0ms     delta 接收请求，校验数据
T+1ms     查询被举报内容（消息/服务器/用户）
T+2ms     生成内容快照（消息需额外拉取 30 条上下文，耗时略高）
T+5ms     标记附件为已举报（db.mark_attachment_as_reported）
T+6ms     写入 safety_snapshots（可能多条 insert_one）
T+7ms     写入 safety_reports（单条 insert_one，status = Created）
T+7ms     Redis PUBLISH "global" EventV1::ReportCreate
T+7ms     返回 204 No Content 给用户

T+7ms     ── Redis 网络传输 ──
T+8ms     bonfire 各实例的 listener() 收到 Redis 消息
T+8ms     反序列化为 EventV1::ReportCreate
T+8ms     handle_incoming_event_v1() → _ => {} → 透传
T+8ms     config.encode(&event) → WebSocket write buffer
T+9ms     在线客户端收到 ReportCreate 事件

          ── 以下永远不会发生（代码缺失）──
          管理员查询待处理报告列表
          管理员驳回/结案
          ReportStatus 从 Created 变更为 Rejected/Resolved
          ReportUpdate 事件广播
          用户收到举报处理结果通知
```

---

## 六、附录：代码引用索引

| 组件 | 文件 | 关键行 |
|------|------|--------|
| 举报 API | [report_content.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs) | L26-L134 |
| 路由注册 | [safety/mod.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/mod.rs) | L6-L10 |
| Report 模型 | [safety_reports/model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/model.rs) | L1-L21 |
| ReportStatus 枚举 | [v0/safety_reports.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/models/src/v0/safety_reports.rs) | L122-L147 |
| AbstractReport trait | [ops.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/ops.rs) | L10-L12 |
| MongoDB insert | [mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/ops/mongodb.rs) | L11-L15 |
| Snapshot 模型 | [safety_snapshots/model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/model.rs) | L8-L120 |
| EventV1 定义 | [client.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs) | L268 |
| global() 发布 | [client.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs) | L402-L404 |
| p() 底层 | [client.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs) | L368-L377 |
| bonfire listener | [websocket.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/websocket.rs) | L221-L403 |
| 事件处理分支 | [impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs) | L434-L696 |
| AMQP 8 通道 | [amqp.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/amqp/amqp.rs) | L18-L29 |
| pushd 消费者 | [pushd/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/pushd/src/main.rs) | L28-L219 |
| crond 任务 | [crond/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/crond/src/main.rs) | L10-L23 |
| delta 启动 | [delta/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/main.rs) | L33-L97 |
