# 安全报告全链路时序：从用户举报到下游消费

## 零、关键结论先列

1. **RabbitMQ 与安全报告无直接关联**——AMQP 的 8 个通道全部用于推送通知（好友请求、消息通知、已读回执、DM 通话），安全报告不经过 RabbitMQ。
2. **Redis `"global"` 通道实际上没有任何订阅者**——ReportCreate 通过 `global()` 发布到 Redis `"global"` 通道，但 bonfire 客户端从不订阅该通道。该通道在代码中只有发布逻辑，没有订阅逻辑，ReportCreate 事件发布后即被丢弃，没有实际消费者。
3. **crond 守护任务与安全报告无任何关系**——4 个 crond 任务（file_deletion、prune_dangling_files、prune_members、acks）均不涉及安全报告的读写。
4. **Rejected / Resolved 状态在当前仓库中没有任何写入代码**——`AbstractReport` trait 只有 `insert_report` 一个方法，不存在 `update_report` / `fetch_report` 等方法；delta 路由中没有管理后台 API；也没有 admin panel 相关代码。状态流转的设计责任归属于独立的管理后台服务（尚未实现）。
5. **举报入库没有任何事务保障**——`report_content()` 中包含 4 类独立写入操作（附件标记、快照写入、报告写入、Redis 事件发布），彼此独立执行，无 MongoDB 事务包裹，无任何回滚机制。
6. **失败时会产生多种孤儿数据**——附件可能被永久标记为 reported 但无对应报告、snapshots 可能存在但 report_id 指向不存在的报告、事件发布失败导致数据完整但通知丢失。

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

## 七、bonfire 客户端订阅集合的初始化流程

### 7.1 初始化时序

```
WebSocket 连接建立
   │
   ▼
① 认证：User::from_token(db, token)  [websocket.rs:L92-L101]
   │
   ▼
② State::from(user, session_id)      [state.rs:L77-L101]
   │  ├─ 创建空 HashSet subscribed
   │  ├─ subscribed.insert("{user_id}!")  ← 私有通道
   │  └─ subscribed.insert(user_id)       ← 用户通道
   │
   ▼
③ generate_ready_payload()           [impl.rs:L98-L332]
   │
   ├─ 3.1 加载用户数据（好友、服务器、频道、成员）
   │
   ├─ 3.2 self.reset_state().await    [state.rs:L159-L162]
   │    ├─ state = SubscriptionStateChange::Reset
   │    └─ subscribed.write().await.clear()  ← 清空！
   │
   ├─ 3.3 重新订阅（insert_subscription）：
   │    ├─ private_topic ("{user_id}!")      [impl.rs:L289]
   │    ├─ 所有关联用户的 user_id             [impl.rs:L291-L293]
   │    │   （包括自己、好友、已接收/发送的好友请求）
   │    ├─ 所有所在服务器的 server_id         [impl.rs:L295-L297]
   │    ├─ Bot 额外订阅 "{server_id}u"       [impl.rs:L298-L300]
   │    │   （服务器成员事件通道）
   │    └─ 所有所在频道的 channel_id          [impl.rs:L303-L305]
   │
   └─ 3.4 返回 Ready 事件给客户端
   │
   ▼
④ listener() 启动 [websocket.rs:L221-L403]
   │
   ├─ 首次循环调用 state.apply_state()
   │   ├─ 检测到 SubscriptionStateChange::Reset
   │   ├─ subscriber.unsubscribe_all().await
   │   └─ 遍历 subscribed 集合，逐个 subscriber.subscribe(id).await
   │      ← 此时 subscribed 中只有 3.3 中订阅的通道
   │
   └─ 进入 select! 循环等待消息

**注意：全程没有任何地方插入 "global" 通道！**
```

### 7.2 订阅通道的完整列表

| 通道命名 | 示例 | 订阅时机 | 用途 |
|---------|------|---------|------|
| `"{user_id}!"` | `"01J...!"` | 初始化 | 用户私有事件（私信、好友请求等） |
| `"{user_id}"` | `"01J..."` | 初始化 + 动态 | 用户相关事件（被 @、状态变更等） |
| `"{server_id}"` | `"ABC..."` | 初始化 + 动态 | 服务器全局事件（频道创建/删除等） |
| `"{server_id}u"` | `"ABC...u"` | Bot 初始化 + 动态 | 服务器成员事件（仅 Bot 订阅） |
| `"{channel_id}"` | `"XYZ..."` | 初始化 + 动态 | 频道事件（新消息、消息删除等） |
| `"global"` | `"global"` | **永不订阅** | **全局广播通道，无任何客户端订阅** |

### 7.3 动态订阅变更

除了初始化订阅外，运行时还会根据事件动态调整：

| 触发事件 | 订阅变更 | 代码位置 |
|---------|---------|---------|
| `ChannelCreate` | 订阅新频道 | [impl.rs:L464](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs#L464) |
| `ChannelDelete` | 取消订阅频道 | [impl.rs:L504](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs#L504) |
| `ChannelGroupJoin` | 订阅新群友的 user_id | [impl.rs:L508](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs#L508) |
| `ChannelGroupLeave` | 取消订阅（若不可访问） | [impl.rs:L513-L514](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs#L513-L514) |
| 服务器成员变更 | 重算服务器频道订阅 | [impl.rs:L335-L399](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs#L335-L399) |
| `ServerMemberLeave`（自己离开） | 重置并重新计算订阅 | [impl.rs:L630-L645](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs#L630-L645) |
| 服务器 15 分钟无活动 | `active_servers` LRU 过期，取消订阅 `{server_id}u` | [state.rs:L117-L134](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/state.rs#L117-L134) |

### 7.4 订阅应用到 Redis 的时机

[listener()](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/websocket.rs#L221-L403) 主循环的第一步就是 `state.apply_state().await`：

- **Reset**：`unsubscribe_all()` 然后批量 `subscribe()` 所有 `subscribed` 集合中的通道
- **Change { add, remove }**：逐个 `unsubscribe()` remove 列表，逐个 `subscribe()` add 列表
- **None**：无操作

`apply_state()` 在每次循环开始时执行，确保 Redis 订阅与本地 `subscribed` HashSet 保持一致。

---

## 八、ReportCreate 事件的实际消费者分析

### 8.1 发布端代码

[report_content.rs:L131](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L131)：

```rust
EventV1::ReportCreate(report.into()).global().await;
```

[client.rs:L402-L404](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs#L402-L404)：

```rust
pub async fn global(self) {
    self.p("global".to_string()).await;
}
```

### 8.2 "global" 通道的搜索结果

全局搜索 `"global"` 字符串（Rust 文件）：

| 文件 | 行 | 用途 |
|------|----|------|
| `events/client.rs` | 403 | `self.p("global".to_string()).await;` —— **仅发布** |
| **无其他文件** | — | **无任何订阅代码** |

### 8.3 为什么没有订阅者？

可能的设计意图推测：

1. **管理后台应独立订阅**：`"global"` 通道本意是给管理后台服务订阅，管理员上线后通过管理后台的 WebSocket 接收新举报通知。但管理后台尚未实现。
2. **误用了通道名**：`ReportCreate` 本应发布到某个管理员专用通道（如 `"admin_reports"`），但错误地使用了 `"global"`，而 `"global"` 通道从未被设计为客户端订阅的通道。
3. **遗留代码**：早期版本中 `"global"` 通道可能有订阅者，后续重构中订阅逻辑被移除，但发布代码未清理。

### 8.4 对安全报告流程的影响

**ReportCreate 事件是死信**。发布到 Redis 后，由于没有任何订阅者，消息立即被丢弃。这意味着：

- 管理员无法通过任何途径实时收到新举报通知
- 新举报只能通过管理后台的定时轮询（如果有的话）或手动刷新发现
- 但管理后台尚未实现，所以实际上新举报完全不可见

> **对比**：`ChannelStartTyping` 等事件发布到 `"{channel_id}"` 通道，而该通道在客户端初始化时就被订阅，所以能正常推送。`ReportCreate` 是唯一使用 `global()` 方法的事件。

### 8.5 其他使用 global() 的事件

[client.rs:L260-L273](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs#L260-L273) 中 `EventV1` 的变体列表：

```rust
Auth(AuthifierEvent),
Bulk { v: Vec<EventV1> },
// ... 各种频道、服务器、用户事件 ...
ReportCreate(Report),
```

检查 `EventV1` 的发布模式：只有 `ReportCreate` 使用 `global()`，其他事件要么使用 `private(id)`，要么使用 `server(id)`，要么使用 `p(channel)` 自定义通道。

---

## 九、事务边界与失败回滚机制

### 9.1 举报入库的四个独立写入操作

[report_content.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs) 中的写入顺序：

```
④ 附件标记（循环，可能多次）
   for file in files {
       db.mark_attachment_as_reported(&file).await?;
   }
   └─ MongoDB: db.attachments.update_one({_id: id}, {$set: {reported: true}})

⑥ 快照写入（循环，可能多次）
   for content in snapshots {
       let snapshot = Snapshot { ... report_id: id ... };
       db.insert_snapshot(&snapshot).await?;
   }
   └─ MongoDB: db.safety_snapshots.insert_one(snapshot)

⑦ 报告写入（一次）
   let report = Report { id, ... status: Created {}, ... };
   db.insert_report(&report).await?;
   └─ MongoDB: db.safety_reports.insert_one(report)

⑧ 事件发布（一次，无 ? 错误传播）
   EventV1::ReportCreate(report.into()).global().await;
   └─ Redis: PUBLISH "global" <event>
```

### 9.2 无事务证据

全局搜索事务相关代码：

| 搜索模式 | 结果 |
|----------|------|
| `start_session` | 仅在 `server_members/ops/mongodb.rs` 中出现，用于成员批量更新 |
| `with_transaction` | 零命中 |
| `transaction` | 仅在 `server_members` 中出现 |

**结论**：`report_content()` 没有使用任何 MongoDB 事务。每个 `.await?` 都是独立的数据库操作。

### 9.3 错误传播路径

```
db.mark_attachment_as_reported(&file).await?
   ↓ 失败 → return Err(...)
   ↓ 已成功的附件标记 **不会回滚**

db.insert_snapshot(&snapshot).await?
   ↓ 失败 → return Err(...)
   ↓ 已成功的附件标记、已成功的快照写入 **不会回滚**

db.insert_report(&report).await?
   ↓ 失败 → return Err(...)
   ↓ 已成功的附件标记、已成功的快照写入 **不会回滚**

EventV1::ReportCreate(...).global().await
   ↓ 失败 → 无 ? → 错误被静默忽略
   ↓ 数据库数据完整，但事件丢失
```

### 9.4 与其他模块的对比

唯一使用 MongoDB 事务的模块是 [server_members/ops/mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/server_members/ops/mongodb.rs)，用于：

- 批量删除服务器成员
- 同时更新多个集合（成员、频道、用户关系）

这表明代码库**具备事务能力**，但安全报告模块**没有使用**。

---

## 十、孤儿数据场景分析

### 10.1 场景 A：附件标记部分成功

```
files = ["A", "B", "C"]
   │
   ├─ mark_attachment_as_reported("A") → ✅ OK
   │  attachments.A.reported = true
   │
   ├─ mark_attachment_as_reported("B") → ❌ 网络错误
   │
   └─ return Err(...)

结果：
  - attachments.A.reported = true ✅
  - attachments.B.reported = false （未处理）
  - attachments.C.reported = false （未处理）
  - safety_snapshots 中无数据
  - safety_reports 中无数据
  - 事件未发布

孤儿数据：
  - attachments.A 被永久标记为 reported，但没有对应的 report
  - 没有 report_id 可以追溯，管理员不知道这个附件为什么被标记
```

### 10.2 场景 B：附件全部成功，快照部分成功

```
snapshots = [S1, S2]   （例如举报用户并附带消息上下文）
   │
   ├─ mark_attachment_as_reported("A") → ✅
   ├─ mark_attachment_as_reported("B") → ✅
   │
   ├─ insert_snapshot(S1) → ✅
   │  safety_snapshots.S1 = { report_id: "R1", ... }
   │
   ├─ insert_snapshot(S2) → ❌ 网络错误
   │
   └─ return Err(...)

结果：
  - 2 个附件标记为 reported
  - safety_snapshots.S1 已存在，report_id = "R1"
  - safety_snapshots.S2 不存在
  - safety_reports 中无数据 （report "R1" 尚未创建）

孤儿数据：
  - attachments 中的 2 个 reported 标记无对应 report
  - safety_snapshots.S1 的 report_id = "R1" 指向不存在的报告
  - S1 永远无法通过 report_id 查询到所属报告
```

### 10.3 场景 C：附件和快照全部成功，报告写入失败

```
   ├─ mark_attachment_as_reported 全部成功 ✅
   ├─ insert_snapshot(S1) 成功 ✅
   ├─ insert_snapshot(S2) 成功 ✅
   │
   ├─ insert_report(Report { id: "R1", ... }) → ❌ 网络错误
   │
   └─ return Err(...)

结果：
  - 2 个附件标记为 reported
  - safety_snapshots.S1 存在，report_id = "R1"
  - safety_snapshots.S2 存在，report_id = "R1"
  - safety_reports 中无 "R1"

孤儿数据：
  - attachments 中的 reported 标记无对应 report
  - safety_snapshots 中有 2 条孤儿快照，report_id 指向不存在的报告
  - 这 2 条快照占用存储空间，但无法被任何查询使用
```

### 10.4 场景 D：数据库全部成功，Redis 发布失败

```
   ├─ mark_attachment_as_reported 全部成功 ✅
   ├─ insert_snapshot 全部成功 ✅
   ├─ insert_report 成功 ✅
   │
   ├─ EventV1::ReportCreate(...).global().await → ❌ Redis 连接失败
   │  注意：这里没有 ?，错误被静默忽略
   │
   └─ return Ok(EmptyResponse)

结果：
  - 数据完整 ✅（附件、快照、报告都存在且一致）
  - 事件未发布 ❌

孤儿数据：
  - 无数据库层面的孤儿数据
  - 但管理后台（如果存在）不会收到实时通知
  - 报告状态永远停留在 Created，无人处理
```

### 10.5 场景 E：所有操作成功，但 ReportCreate 无人订阅

```
   ├─ 所有数据库操作成功 ✅
   ├─ Redis PUBLISH "global" event ✅
   │  Redis 成功接收，但无任何客户端订阅 "global"
   │  消息被 Redis 立即丢弃
   │
   └─ return Ok(EmptyResponse)

结果：
  - 数据完整 ✅
  - 事件发布成功，但无人接收 ❌

孤儿数据：
  - 无数据库层面的孤儿数据
  - 逻辑孤儿：报告存在但永远不会被处理
  - 因为管理后台尚未实现，管理员看不到新举报
```

### 10.6 孤儿数据的清洗现状

crond 有 `prune_dangling_files` 任务用于清理悬空附件，但：

| 清理任务 | 能否清理安全报告孤儿数据 |
|----------|-------------------------|
| `file_deletion` | 否（仅删标记为 deleted 的文件） |
| `prune_dangling_files` | 否（仅清理没有被任何 message/server/user 引用的附件） |
| `prune_members` | 否（仅清理服务器成员） |

**结论**：当前没有任何定时任务或机制清理安全报告产生的孤儿数据。这些数据会永久残留在数据库中，直到手动清理。

### 10.7 修复建议

| 问题 | 建议修复方案 |
|------|-------------|
| 无事务 | 使用 MongoDB 多文档事务包裹附件标记、快照写入、报告写入 |
| 无回滚 | 事务失败时自动回滚所有写入 |
| ReportCreate 死信 | 发布到管理员专用通道，并确保管理后台订阅该通道 |
| 孤儿数据 | 新增 crond 任务定期清理：<br>1. 快照的 report_id 不存在于 reports → 删除快照<br>2. 附件 reported = true 但没有关联 report → 重置 reported |

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
| bonfire client 入口 | [websocket.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/websocket.rs) | L42-L195 |
| bonfire listener | [websocket.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/websocket.rs) | L221-L403 |
| State 初始化 | [state.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/state.rs) | L77-L101 |
| State reset | [state.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/state.rs) | L159-L162 |
| State 应用订阅 | [state.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/state.rs) | L104-L151 |
| State insert_subscription | [state.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/state.rs) | L165-L185 |
| generate_ready_payload | [impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs) | L98-L332 |
| 事件处理分支 | [impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/impl.rs) | L434-L696 |
| mark_attachment_as_reported | [files/ops/mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/files/ops/mongodb.rs) | L121-L136 |
| MongoDB 事务使用示例 | [server_members/ops/mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/server_members/ops/mongodb.rs) | 事务相关 |
| AMQP 8 通道 | [amqp.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/amqp/amqp.rs) | L18-L29 |
| pushd 消费者 | [pushd/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/pushd/src/main.rs) | L28-L219 |
| crond 任务 | [crond/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/crond/src/main.rs) | L10-L23 |
| delta 启动 | [delta/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/main.rs) | L33-L97 |
