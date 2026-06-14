# 安全报告全链路时序：从用户举报到下游消费

## 零、关键结论先列

1. **RabbitMQ 与安全报告无直接关联**——AMQP 的 8 个通道全部用于推送通知（好友请求、消息通知、已读回执、DM 通话），安全报告不经过 RabbitMQ。
2. **Redis `"global"` 通道实际上没有任何订阅者**——ReportCreate 通过 `global()` 发布到 Redis `"global"` 通道，但 bonfire 客户端从不订阅该通道。该通道在代码中只有发布逻辑，没有订阅逻辑，ReportCreate 事件发布后即被丢弃，没有实际消费者。
3. **crond 守护任务与安全报告无任何关系**——4 个 crond 任务（file_deletion、prune_dangling_files、prune_members、acks）均不涉及安全报告的读写。
4. **Rejected / Resolved 状态在当前仓库中没有任何写入代码**——`AbstractReport` trait 只有 `insert_report` 一个方法，不存在 `update_report` / `fetch_report` 等方法；delta 路由中没有管理后台 API；也没有 admin panel 相关代码。状态流转的设计责任归属于独立的管理后台服务（尚未实现）。
5. **举报入库没有任何事务保障**——`report_content()` 中包含 4 类独立写入操作（附件标记、快照写入、报告写入、Redis 事件发布），彼此独立执行，无 MongoDB 事务包裹，无任何回滚机制。
6. **失败时会产生多种孤儿数据**——附件可能被永久标记为 reported 但无对应报告、snapshots 可能存在但 report_id 指向不存在的报告、事件发布失败导致数据完整但通知丢失。
7. **`safety_strikes` 集合是空壳**——只有迁移脚本创建集合和索引，没有对应的 Rust 模型、没有 Abstract trait、没有数据库操作方法、没有 API 路由。处罚体系完全未实现。
8. **`safety/report` 路由有限流保护**——限流桶名 `safety_report`，阈值 3 次/周期。另有通用 `safety` 桶 15 次/周期。
9. **`reported` 字段被写入但从未被读取使用**——`mark_attachment_as_reported` 将 `reported: true` 写入附件，但全代码库没有任何下游分支根据 `reported` 字段做逻辑判断。该字段是死字段。
10. **`generate_from_message` 抓取上下文时没有读权限校验**——用户可以举报任意频道的任意消息（只要知道 message_id），快照会抓取该频道前后各 15 条消息，无论举报人是否有权限访问该频道。这是一个越权读取漏洞。
11. **`safety/report` 路由没有绑定 IdempotencyKey**——客户端重复提交会产生重复举报记录。对比消息发送和 Webhook 执行都有幂等键保护。
12. **Bot 账号发起举报被禁止，但 Bot 可以被举报**——举报者是 bot 会返回 `IsBot` 错误；但 User 类型举报和 Message 类型举报都不检查被举报对象是否是 bot，bot 账号和 bot 发送的消息都可以正常被举报。
13. **`EventV1::ReportCreate` 载荷包含完整 Report 数据**——`author_id`、`notes`、`additional_context` 等字段全部原样塞进 Redis payload。虽然 `"global"` 通道当前无订阅者，但如果未来新增订阅者，敏感字段会被广播出去。
14. **限流桶计数维度是 session.id（优先）+ IP（兜底）**——Delta 服务的 Rocket 限流器优先使用会话 ID 作为标识符，同一用户的不同设备/浏览器有独立的限流桶；未登录用户按 IP 限流。不是按 user.id 限流。

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

## 十一、safety_strikes 集合状态分析

### 11.1 现状：只有集合定义，没有任何代码对接

`safety_strikes` 集合在迁移脚本中被创建，但**没有任何 Rust 代码与它交互**。

| 层面 | 是否存在 | 代码位置 |
|------|---------|---------|
| 集合创建 | ✅ 有 | [init.rs:L79-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L79-L81) |
| 迁移脚本 | ✅ 有 | revision 21 创建集合，revision 22 添加 `moderator_id` 字段 |
| Rust 模型 struct | ❌ 无 | 全代码库无 `Strike` / `SafetyStrike` 结构体 |
| Abstract trait | ❌ 无 | 无 `AbstractStrike` trait |
| 数据库 ops 实现 | ❌ 无 | 无 MongoDB / Reference 操作实现 |
| API 路由 | ❌ 无 | 无任何创建/查询/更新 strike 的路由 |
| 事件 | ❌ 无 | 无 `StrikeCreate` / `StrikeUpdate` 等事件变体 |

### 11.2 迁移脚本中的线索

[scripts.rs:L688-L708](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs#L688-L708) 提供了一些设计意图线索：

- **Revision 21** (2023-05-31)：创建 `safety_strikes` 集合
- **Revision 22** (2023-05-31)：为所有文档补充 `moderator_id` 字段

从 `moderator_id` 字段名可以推断，strike 的设计意图是：
- 每个 strike 记录一次用户违规处罚
- `moderator_id` 记录执行处罚的管理员
- 与安全报告（safety_reports）联动，结案后对用户执行 strike

### 11.3 设计意图推断

```
  安全报告 (safety_reports)           处罚记录 (safety_strikes)
  ┌─────────────────────────┐        ┌─────────────────────────┐
  │ id: 报告ID               │        │ id: 处罚ID              │
  │ author_id: 举报人        │ 结案后  │ user_id: 被处罚用户     │
  │ content: 被举报内容      │ ──────▶ │ reason: 处罚原因        │
  │ status: Created          │  触发   │ moderator_id: 执行者    │
  │ notes: 管理员备注        │        │ created_at: 处罚时间     │
  └─────────────────────────┘        └─────────────────────────┘
           ▲                                       │
           │                                       │
      用户举报                              用户状态变更
      （已实现）                             （未实现）
```

**当前状态**：只有左边的举报输入，没有右边的处罚输出。报告创建后永远停留在 `Created` 状态，没有任何机制将其转化为 strike 处罚。

### 11.4 责任归属

与 `ReportStatus` 的状态流转一样，`safety_strikes` 的操作代码也应归属于**独立的管理后台服务**。本仓库（revolt-backend）只负责数据存储层和用户端的举报入口。

---

## 十二、safety/report 路由的限流桶与阈值

### 12.1 限流配置位置

限流定义在 [ratelimits.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/util/ratelimits.rs) 的 `DeltaRatelimits` 实现中。

### 12.2 限流解析逻辑

`resolve_bucket()` 方法根据 HTTP 请求的路径段和方法决定限流桶：

```rust
match (segment, resource, method) {
    // ...
    ("safety", Some("report"), _) => ("safety_report", Some("report")),
    ("safety", _, _) => ("safety", None),
    // ...
}
```

| 路由路径 | 桶名 | 资源 ID |
|---------|------|--------|
| `POST /safety/report` | `safety_report` | `"report"`（固定） |
| 其他 `/safety/*` 路由 | `safety` | `None` |

### 12.3 限流阈值

`resolve_bucket_limit()` 方法定义各桶的阈值：

| 桶名 | 阈值 | 说明 |
|------|------|------|
| `safety_report` | **3** | 创建举报的限流，非常严格 |
| `safety` | 15 | 其他安全相关 API 的通用桶 |
| `messaging` | 10 | 发消息的限流（对比参考） |
| `auth` | 15 | 认证相关限流 |
| `any` | 20 | 默认桶 |

**注意**：`safety_report` 桶的阈值（3）远低于发消息的阈值（10），说明设计上对举报行为有更严格的频率控制，防止恶意刷屏式举报。

### 12.4 限流实现机制

限流基于 `revolt_ratelimits` crate，使用 Redis 存储计数器。从 `ratelimit_events` 模型可推断：

- 每个桶 + 用户/IP 对应一个计数器
- 有时间窗口（滑动或固定窗口）
- 超过阈值返回 429 错误

### 12.5 潜在问题

1. **资源 ID 固定为 "report"**：`Some("report")` 是硬编码的字符串，不是动态的。这意味着限流是按**全局**计算的，不是按每个被举报对象计算。用户 A 举报用户 X 和举报用户 Y 都消耗同一个 `safety_report` 桶的额度。

2. **阈值较低**：3 次/周期的阈值对于正常使用可能偏紧，但对于防止恶意举报是合理的。

---

## 十三、mark_attachment_as_reported 字段的下游读取分析

### 13.1 写入位置

`mark_attachment_as_reported()` 方法将 `reported: true` 写入 `attachments` 集合：

- MongoDB 实现：[files/ops/mongodb.rs:L121-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/files/ops/mongodb.rs#L121-L136)
- Reference 实现：[files/ops/reference.rs:L95-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/files/ops/reference.rs#L95-L101)

### 13.2 字段定义

File 模型中的 `reported` 字段：
- 数据库模型：[files/model.rs:L32-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/files/model.rs#L32-L34)
- API 模型：[v0/files.rs:L21-L23](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/models/src/v0/files.rs#L21-L23)

字段特性：
- 类型：`Option<bool>`
- 序列化：`skip_serializing_if = "Option::is_none"`（为 None 时不输出）
- 含义：文件是否被举报过

### 13.3 下游读取情况排查

全局搜索 `reported` 字段的读取/判断逻辑：

| 代码位置 | 用途 | 是否为业务逻辑判断 |
|---------|------|-------------------|
| [bridge/v0.rs:L378](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/util/bridge/v0.rs#L378) | 模型转换（db → v0） | ❌ 只是透传 |
| [bridge/v0.rs:L397](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/util/bridge/v0.rs#L397) | 模型转换（v0 → db） | ❌ 只是透传 |
| [files/ops/reference.rs:L45](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/files/ops/reference.rs#L45) | Reference 实现中的条件过滤 | ❌ 仅测试/内存实现 |
| [files/ops/mongodb.rs:L43](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/files/ops/mongodb.rs#L43) | MongoDB 投影字段 | ❌ 只是查询时包含该字段 |
| **业务逻辑判断** | — | **零命中** |

### 13.4 结论：reported 是死字段

**`reported` 字段被写入但从未被任何业务逻辑读取使用。** 具体表现：

1. **没有查询过滤**：没有任何 API 根据 `reported=true` 过滤文件列表
2. **没有条件分支**：没有任何 `if file.reported` 的判断逻辑
3. **没有管理后台读取**：管理后台尚未实现，自然也没有读取该字段
4. **序列化透传**：只有模型层的双向转换透传了该字段，但如果文件通过 API 返回给客户端，客户端会看到 `reported` 字段（如果为 true 的话）

### 13.5 设计意图推测

该字段的设计意图应该是：

1. **防止重复举报**：标记已举报的附件，避免重复进入审核流程
2. **快速筛选**：管理员可以按 `reported=true` 筛选待审核的附件
3. **状态标记**：作为文件生命周期的一部分（normal → reported → deleted）

但由于管理后台未实现，这些功能都不存在。当前只有"写入"，没有"读取"和"使用"。

### 13.6 安全隐患

虽然 `reported` 字段本身是死字段，但结合之前的孤儿数据分析：

- 举报失败时附件可能被永久标记为 `reported=true`
- 由于没有读取逻辑，这些"假阳性"标记不会被发现
- 如果未来管理后台实现了按 `reported` 筛选，会出现大量无对应报告的孤儿标记

---

## 十四、generate_from_message 的权限校验漏洞

### 14.1 正常消息获取的权限校验

作为对比，标准的消息获取 API [message_fetch.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/channels/message_fetch.rs#L14-L33) 有严格的权限校验：

```rust
// 1. 获取频道
let channel = target.as_channel(db).await?;

// 2. 计算频道权限
let mut query = DatabasePermissionQuery::new(db, &user).channel(&channel);
calculate_channel_permissions(&mut query)
    .await
    .throw_if_lacking_channel_permission(ChannelPermission::ViewChannel)?;

// 3. 获取消息并校验归属
let message = msg.as_message(db).await?;
if message.channel != channel.id() {
    return Err(create_error!(NotFound));
}
```

三步校验：频道存在 → 有权限 → 消息属于该频道。

### 14.2 举报路由的权限校验缺失

而在 [report_content.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L47-L56) 的消息举报逻辑中：

```rust
ReportedContent::Message { id, .. } => {
    let message = db.fetch_message(id).await?;  // 直接按 ID 取消息！

    // Users cannot report themselves
    if message.author == user.id {
        return Err(create_error!(CannotReportYourself));
    }

    let (snapshot, files) = SnapshotContent::generate_from_message(db, message).await?;
    (vec![snapshot], files)
}
```

**只有一步校验**：不能举报自己的消息。

**缺少的校验**：
1. ❌ 不校验消息所在的频道是否存在
2. ❌ 不校验用户是否有 `ViewChannel` 权限
3. ❌ 不校验用户是否能访问该服务器/频道
4. ❌ 不校验消息是否属于用户可见的频道

### 14.3 漏洞影响：越权读取消息上下文

更严重的是，`generate_from_message()` 会抓取消息前后各 **15 条** 上下文消息：

```rust
// 抓取之前的 15 条
let prior_context = db.fetch_messages(MessageQuery {
    filter: MessageFilter {
        channel: Some(message.channel.to_string()),
        ..Default::default()
    },
    limit: Some(15),
    time_period: MessageTimePeriod::Absolute {
        before: Some(message.id.to_string()),
        ...
    },
}).await?;

// 抓取之后的 15 条
let leading_context = db.fetch_messages(MessageQuery {
    filter: MessageFilter {
        channel: Some(message.channel.to_string()),
        ..Default::default()
    },
    limit: Some(15),
    time_period: MessageTimePeriod::Absolute {
        after: Some(message.id.to_string()),
        ...
    },
}).await?;
```

**攻击路径**：

```
攻击者（无权限访问私密频道 X）
   │
   │  知道/猜到频道 X 中的某条消息 ID
   │
   ▼
POST /safety/report
{ content: { type: "Message", id: "消息ID" } }
   │
   ▼
delta 直接 fetch_message(id) → 成功获取消息内容
   │
   ▼
generate_from_message() → 抓取频道 X 中前后各 15 条消息
   │
   ▼
31 条私密消息被存入 safety_snapshots 快照中
   │
   ▼
虽然攻击者看不到返回结果（只有 204 No Content），
但如果快照能通过其他途径读取（如管理后台漏洞），
就能获取大量私密信息
```

### 14.4 漏洞严重程度评估

| 维度 | 评估 |
|------|------|
| 可利用性 | 高 —— 只需知道 message_id 即可发起 |
| 信息泄露量 | 中 —— 31 条消息，约一个屏幕的聊天内容 |
| 直接危害 | 低 —— 攻击者无法直接读取返回（API 返回 204） |
| 间接危害 | 中 —— 快照永久存储，若管理后台有泄露则放大 |
| 修复成本 | 低 —— 添加几行权限校验即可 |

### 14.5 用户举报场景的校验缺失

除了消息举报，**用户举报** 场景也有类似问题：

```rust
ReportedContent::User { id, message_id, .. } => {
    let reported_user = db.fetch_user(id).await?;  // 直接按 ID 取用户
    // 只校验了不能举报自己
    if reported_user.id == user.id {
        return Err(create_error!(CannotReportYourself));
    }

    // 如果提供了 message_id，也会抓取该消息的快照
    let message = if let Some(id) = message_id {
        db.fetch_message(id).await.ok()  // 注意这里用了 .ok()，失败静默
    } else {
        None
    };
    // ...
}
```

用户信息本身通常是公开的（头像、用户名等），所以越权读取用户资料的危害不大。但附带的 `message_id` 同样存在消息越权读取问题。

### 14.6 服务器举报场景

服务器举报场景：
```rust
ReportedContent::Server { id, .. } => {
    let server = db.fetch_server(id).await?;
    if server.owner == user.id {  // 只校验了不能举报自己的服务器
        return Err(create_error!(CannotReportYourself));
    }
    let (snapshot, files) = SnapshotContent::generate_from_server(server)?;
    // ...
}
```

服务器信息（名称、图标、描述等）通常是公开的，即使是私密服务器，基本信息也可能通过搜索等途径暴露。所以服务器举报的越权问题相对较轻。

### 14.7 修复建议

**核心修复点**：在 `report_content()` 中，对被举报内容进行权限校验，确保举报人有权访问。

具体措施：

1. **消息举报**：
   - 通过 `channel_id` 获取频道
   - 计算用户在该频道的权限
   - 确保有 `ViewChannel` 权限才允许举报和生成快照

2. **服务器举报**：
   - 校验用户是否是服务器成员（或服务器是公开的）

3. **快照生成**：
   - 上下文消息抓取也应在权限校验通过后进行
   - 或在 `generate_from_message` 内部增加权限校验参数

4. **深度防御**：
   - 快照数据不应包含完整的消息内容元数据
   - 管理后台查看快照时应再次校验管理员权限

---

## 十五、safety/report 路由的 IdempotencyKey 缺失分析

### 15.1 现状：举报路由没有幂等键保护

[safety/report 路由](file:///d:/fz/0601-1\solo-dogfeeding\code\90-backend\crates\delta\src\routes\safety\report_content.rs#L27-L31) 的函数签名：

```rust
pub async fn report_content(
    db: &State<Database>,
    user: User,
    data: Json<DataReportContent>,
) -> Result<EmptyResponse> {
```

**没有 `IdempotencyKey` 参数。**

### 15.2 对比：有幂等键保护的路由

作为对比，消息发送和 Webhook 执行都有幂等键：

| 路由 | 是否有 IdempotencyKey | 代码位置 |
|------|----------------------|---------|
| `POST /channels/{id}/messages` | ✅ 有 | [message_send.rs:L30](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/channels/message_send.rs#L30) |
| `POST /webhooks/{id}/{token}` | ✅ 有 | [webhook_execute.rs:L24](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/webhooks/webhook_execute.rs#L24) |
| `POST /safety/report` | ❌ 无 | [report_content.rs:L27-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L27-L31) |

### 15.3 IdempotencyKey 的工作原理

`IdempotencyKey` 从请求头 `X-Idempotency` 中提取：

- 如果请求带有幂等键，服务器会检查该键是否已被使用过
- 如果已使用过，直接返回之前的结果，不重复执行业务逻辑
- 如果未使用过，执行业务逻辑并缓存结果

这样可以防止客户端因网络重试、用户双击等原因导致的重复提交。

### 15.4 缺失幂等键的影响

由于举报路由没有幂等键保护，**客户端重复提交会产生重复的举报记录**：

| 场景 | 结果 |
|------|------|
| 用户误双击提交按钮 | 产生 2 条内容完全相同的举报 |
| 网络重试（客户端超时重发） | 产生多条重复举报 |
| 脚本恶意刷举报 | 受限流限制（3 次/周期），但每个周期内都可以提交多条 |

### 15.5 为什么举报路由没有幂等键？

可能的设计原因：

1. **优先级低**：举报不是核心功能，重复提交的危害不大（最多增加审核工作量）
2. **限流兜底**：3 次/周期的限流已经限制了重复提交的频率
3. **遗漏**：开发时忘记添加，属于功能缺失
4. **返回 EmptyResponse**：举报返回 204 无内容，没有需要缓存的响应体，实现幂等键的价值较低

### 15.6 与限流的关系

幂等键和限流是两个不同层面的保护：

| 维度 | 限流 | 幂等键 |
|------|------|--------|
| 作用 | 限制频率 | 防止重复 |
| 粒度 | 时间窗口 | 请求级 |
| 相同内容多次提交 | 消耗额度但允许 | 直接返回缓存结果 |
| 不同内容提交 | 都消耗额度 | 都正常执行 |

当前举报路由只有限流，没有幂等键。

---

## 十六、Bot 账号的举报过滤逻辑

### 16.1 举报者侧：Bot 不能发起举报

[report_content.rs:L39-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L39-L42)：

```rust
// Bots cannot create reports
if user.bot.is_some() {
    return Err(create_error!(IsBot));
}
```

**举报者如果是 Bot 账号，直接返回 `IsBot` 错误。**

这与其他社交功能的设计一致：

| 功能 | Bot 是否被禁止 | 代码位置 |
|------|---------------|---------|
| 发起举报 | ✅ 禁止 | [report_content.rs:L40](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L40) |
| 加好友 | ✅ 禁止 | [add_friend.rs:L21](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/users/add_friend.rs#L21) |
| 发好友请求 | ✅ 禁止 | [send_friend_request.rs:L22](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/users/send_friend_request.rs#L22) |
| 创建服务器 | ✅ 禁止 | [server_create.rs:L21](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/servers/server_create.rs#L21) |
| 服务器确认 | ✅ 禁止 | [server_ack.rs:L21](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/servers/server_ack.rs#L21) |

设计意图：Bot 是程序控制的账号，不应该主动发起社交互动（包括举报）。

### 16.2 被举报者侧：Bot 可以被举报

**User 类型举报**（举报用户账号）：

[report_content.rs:L69-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L69-L75)：

```rust
ReportedContent::User { id, message_id, .. } => {
    let reported_user = db.fetch_user(id).await?;

    // Users cannot report themselves
    if reported_user.id == user.id {
        return Err(create_error!(CannotReportYourself));
    }
    // ... 没有检查 reported_user.bot
}
```

**只检查了不能举报自己，没有检查被举报用户是不是 Bot。**

**Message 类型举报**（举报消息）：

消息的 `author` 字段可能指向 Bot 账号，但代码中只检查了 `message.author == user.id`（不能举报自己），没有检查消息作者是不是 Bot。

### 16.3 设计意图分析

为什么举报者要过滤 Bot，但被举报者不过滤？

```
设计意图：

  举报者（主动方）        被举报者（被动方）
  ┌───────────────┐      ┌───────────────┐
  │ Bot 被禁止    │      │ Bot 可以被举报│
  │ (主动社交行为) │      │ (接受审核)     │
  └───────────────┘      └───────────────┘
```

合理的设计逻辑：

1. **Bot 不应该主动举报**：举报是人的行为，Bot 作为程序不应主动发起举报
2. **Bot 可以被举报**：Bot 发送垃圾消息、恶意内容等，用户应该能举报 Bot
3. **一致性**：Bot 发送的消息可以被举报，Bot 账号本身也可以被举报

### 16.4 潜在问题

虽然设计上 Bot 可以被举报是合理的，但有一个潜在问题：

**Bot 账号通常由开发者管理，对 Bot 的举报应该如何处理？**

- 是处罚 Bot 账号本身（封禁、限制）？
- 还是处罚 Bot 的所有者？
- 处罚是否会传播到所有者的其他 Bot？

这些问题在当前代码中没有答案，因为管理后台和 strike 处罚体系都尚未实现。

---

## 十七、EventV1::ReportCreate 载荷的敏感字段分析

### 17.1 ReportCreate 事件包含完整 Report 对象

[EventV1 枚举定义](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs#L268)：

```rust
/// New report
ReportCreate(Report),
```

`ReportCreate` 变体直接包含 `Report` 结构体，没有做任何字段裁剪或脱敏。

### 17.2 Report 结构体的完整字段

[Report 模型](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/models/src/v0/safety_reports.rs#L5-L21)：

```rust
pub struct Report {
    pub id: String,                  // 报告 ID
    pub author_id: String,           // 举报人 ID ⚠️
    pub content: ReportedContent,    // 被举报内容
    pub additional_context: String,  // 举报描述 ⚠️
    #[serde(flatten)]
    pub status: ReportStatus,        // 状态
    #[serde(default)]
    pub notes: String,               // 管理员备注 ⚠️
}
```

### 17.3 发布前的转换

[report_content.rs:L131](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L131)：

```rust
EventV1::ReportCreate(report.into()).global().await;
```

`report.into()` 调用 `From<crate::Report> for Report` 实现，将数据库模型转换为 API 模型。

[bridge/v0.rs:L606-L617](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/util/bridge/v0.rs#L606-L617)：

```rust
impl From<crate::Report> for Report {
    fn from(value: crate::Report) -> Self {
        Report {
            id: value.id,
            author_id: value.author_id,       // 原样拷贝
            content: value.content,           // 原样拷贝
            additional_context: value.additional_context, // 原样拷贝
            status: value.status,             // 原样拷贝
            notes: value.notes,               // 原样拷贝
        }
    }
}
```

**所有字段全部原样拷贝，没有任何脱敏或裁剪。**

### 17.4 敏感字段清单

| 字段 | 敏感性 | 说明 |
|------|--------|------|
| `author_id` | 中 | 暴露举报人身份。如果事件被广播给普通用户，大家都知道是谁举报的 |
| `additional_context` | 中高 | 举报描述文本，可能包含举报人提供的敏感信息 |
| `notes` | 高 | 管理员备注，可能包含内部审核意见。但当前创建时 `notes` 是空字符串 |
| `content`（含消息快照引用） | 中 | 被举报内容的 ID 和类型，本身不敏感，但可能间接泄露信息 |

### 17.5 当前风险评估

由于 `"global"` 通道**目前没有任何订阅者**，所以这些敏感字段实际上不会泄露给任何人。

```
风险现状：

  Redis "global" 通道
    │
    ├─ 发布者: delta (ReportCreate 事件)
    │
    └─ 订阅者: 空（无任何客户端订阅）
        ↓
        当前风险：低（事件发布后立即被丢弃）
```

但这是一个**潜在风险**：

1. 如果未来有人为 `"global"` 通道添加订阅者，敏感字段会立即暴露
2. 如果事件被错误地发布到其他通道（如 `server(id)` 或 `private(id)`），也会造成泄露
3. 如果 Redis 本身不安全（如未设置密码），攻击者可以直接监听所有 PUBLISH 消息

### 17.6 设计意图与改进方向

**当前设计**：ReportCreate 事件包含完整 Report 数据，推送到 `"global"` 通道。

**推测的设计意图**：管理后台服务应该订阅 `"global"` 通道（或专用的管理员通道），管理员上线后实时接收新举报通知，包含完整信息以便快速审核。

**改进建议**：

| 改进方向 | 具体措施 |
|---------|---------|
| 通道隔离 | 不要推送到 `"global"`，推送到专用的管理员通道（如 `"admin_reports"`） |
| 字段裁剪 | 面向普通用户的事件不应包含 `author_id`、`notes` 等敏感字段 |
| 分级推送 | 给管理员推完整信息，给普通用户推最小信息（或不推） |
| 权限校验 | bonfire 收到事件后，应校验接收者是否有查看报告的权限 |

---

## 十八、限流桶的计数维度分析

### 18.1 Delta 服务使用 Rocket 限流实现

`report_content.rs` 是 Delta 服务的 Rocket 路由，所以使用的是 `revolt_ratelimits::rocket` 模块的限流实现，而不是 axum 版本。

### 18.2 限流标识符的选择逻辑

[rocket.rs:L60-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/ratelimits/src/rocket.rs#L60-L65)：

```rust
let identifier = if let Outcome::Success(session) = request.guard::<Session>().await
{
    session.id  // 有会话 → 用 session.id
} else {
    to_real_ip(request).await  // 无会话 → 用 IP
};
```

**计数维度：优先使用 session.id，兜底使用 IP。**

### 18.3 session.id vs user.id

重要区别：

| 维度 | session.id | user.id |
|------|-----------|---------|
| 含义 | 会话 ID（每次登录一个） | 用户 ID（一个用户只有一个） |
| 数量 | 一个用户可能有多个 session | 一个用户只有一个 user_id |
| 限流粒度 | 每个设备/浏览器独立限流 | 全账号统一限流 |
| 多设备影响 | 各设备互不影响 | 一台设备耗光，所有设备都受限 |

**Delta 服务选择的是 session.id，不是 user.id。**

### 18.4 实际限流效果

对于 `safety_report` 桶（阈值 3 次/周期）：

| 场景 | 实际限流效果 |
|------|-------------|
| 用户在浏览器 A 举报 | 消耗 session_A 的额度，3 次后 A 受限 |
| 用户切换到浏览器 B 举报 | 消耗 session_B 的额度，独立计算，B 还可以举报 3 次 |
| 用户同时用手机 App 举报 | 消耗 session_C 的额度，又可以举报 3 次 |
| 未登录用户举报 | 消耗 IP 的额度 |
| 同一 IP 下多个未登录用户 | 共享同一 IP 的限流桶 |

### 18.5 与 axum 版本的对比

作为对比，axum 版本的限流使用 user.id：

[axum.rs:L63-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/ratelimits/src/axum.rs#L63-L67)：

```rust
let identifier = if let Ok(user) = parts.extract_with_state::<User, _>(state).await {
    user.id  // axum 版本用 user.id
} else {
    to_real_ip(parts).await
};
```

| 版本 | 登录用户标识符 | 未登录用户标识符 |
|------|--------------|----------------|
| Rocket (Delta) | `session.id` | IP |
| Axum | `user.id` | IP |

**为什么会有这个差异？** 可能是不同团队开发，或者 Rocket 版本更老，后续 axum 版本做了优化。

### 18.6 对举报限流的影响

使用 session.id 作为限流维度，对于举报功能的影响：

| 方面 | 影响 |
|------|------|
| 正常用户 | 体验更好，多设备不会互相影响 |
| 恶意用户 | 可以通过创建多个 session（多开浏览器、多设备）绕过限流 |
| 实际防护效果 | 比 user.id 弱，但比纯 IP 强 |
| 配合 Bot 禁止 | Bot 不能发起举报，所以恶意脚本无法用 Bot 账号绕限流 |

### 18.7 限流周期

从 [ratelimiter.rs:L54](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/ratelimits/src/ratelimiter.rs#L54) 可以看出：

```rust
reset: now().add(Duration::from_secs(10)).as_millis(),
```

**限流窗口是 10 秒**。

所以 `safety_report` 桶的实际限流是：**每个 session 每 10 秒最多 3 次举报**。

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
| 文件 reported 字段 | [files/model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/files/model.rs) | L32-L34 |
| 文件 reported (v0) | [v0/files.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/models/src/v0/files.rs) | L21-L23 |
| reported 桥接转换 | [bridge/v0.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/util/bridge/v0.rs) | L378, L397 |
| MongoDB 事务使用示例 | [server_members/ops/mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/server_members/ops/mongodb.rs) | 事务相关 |
| AMQP 8 通道 | [amqp.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/amqp/amqp.rs) | L18-L29 |
| pushd 消费者 | [pushd/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/pushd/src/main.rs) | L28-L219 |
| crond 任务 | [crond/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/crond/src/main.rs) | L10-L23 |
| delta 启动 | [delta/main.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/main.rs) | L33-L97 |
| 限流配置 | [ratelimits.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/util/ratelimits.rs) | L1-L81 |
| 正常消息获取权限校验 | [message_fetch.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/channels/message_fetch.rs) | L14-L33 |
| generate_from_message | [safety_snapshots/model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/model.rs) | L42-L93 |
| safety_strikes 集合创建 | [init.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs) | L79-L81 |
| safety_strikes 迁移脚本 | [scripts.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs) | L688-L708 |
| IdempotencyKey 导入 | [message_send.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/channels/message_send.rs) | L7, L30 |
| Webhook 幂等键 | [webhook_execute.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/webhooks/webhook_execute.rs) | L3, L24 |
| Bot 禁止举报判断 | [report_content.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs) | L39-L42 |
| Bot 禁止加好友 | [add_friend.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/users/add_friend.rs) | L21 |
| Report 桥接转换 | [bridge/v0.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/util/bridge/v0.rs) | L606-L617 |
| Rocket 限流实现 | [rocket.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/ratelimits/src/rocket.rs) | L60-L65 |
| Axum 限流实现 | [axum.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/ratelimits/src/axum.rs) | L63-L67 |
| 限流窗口周期 | [ratelimiter.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/ratelimits/src/ratelimiter.rs) | L54 |
| Report v0 模型 | [v0/safety_reports.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/models/src/v0/safety_reports.rs) | L5-L21 |
