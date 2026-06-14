# 频道消息发送 Pipeline 全链路梳理

本文档以 `POST /channels/<target>/messages` 为入口，逐层追踪消息从 HTTP 请求到达、写入 MongoDB、到事件广播至所有在线客户端的完整路径，并连带说明消息实体与未读状态的关联以及幂等保障机制。

---

## 一、整体数据流概览

```
Client (HTTP POST)
  │
  ▼
message_send (Rocket route handler)
  │  校验 → 权限 → 慢速模式 → 构造 Message
  ▼
Message::create_from_api (核心编排)
  │  nonce 幂等 → mentions 解析 → replies 校验 → flags 计算 → 附件处理
  ▼
Message::send
  ├─► Message::send_without_notifications
  │     ├─► db.insert_message                 ← MongoDB 写入
  │     ├─► EventV1::Message(...).p(channel)  ← Redis PubSub 广播
  │     ├─► tasks::last_message_id::queue      ← 防抖更新 channel.last_message_id
  │     └─► tasks::ack::queue_message           ← 异步处理未读/推送
  │
  └─► tasks::ack::queue_message (推送通知分支)
        └─► handle_ack_event::ProcessMessage
              ├─► db.add_mention_to_unread      ← channel_unreads 写入
              ├─► amqp.message_sent              ← AMQP 推送通知
              └─► amqp.mass_mention_message_sent ← @everyone / role 推送
```

---

## 二、接口处理层：Rocket Route Handler

**文件**: [message_send.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/delta/src/routes/channels/message_send.rs#L24-L193)

### 2.1 入口签名

```rust
#[post("/<target>/messages", data = "<data>")]
pub async fn message_send(
    db: &State<Database>,
    amqp: &State<AMQP>,
    user: User,                          // 认证守卫：已登录用户
    target: Reference<'_>,              // 频道 ID 引用
    data: Json<v0::DataMessageSend>,     // 请求体
    idempotency: IdempotencyKey,         // 从 Header 提取的幂等键
) -> Result<Json<v0::Message>>
```

### 2.2 处理步骤

1. **请求体校验** — `data.validate()` 调用 validator 对 `DataMessageSend` 做字段级校验。

2. **频道权限检查** — 通过 `DatabasePermissionQuery` 构建权限查询上下文，`calculate_channel_permissions` 计算 结果后逐项验证：
   - `SendMessage` — 发消息基础权限
   - `Masquerade` — 使用昵称/头像伪装
   - `ManageRole` — 伪装中使用角色色值
   - `SendEmbeds` — 发送嵌入内容
   - `UploadFiles` — 上传附件

3. **慢速模式 (Slowmode)** — 对 `TextChannel` 类型的频道，若配置了 `slowmode > 0` 且用户没有 `BypassSlowmode` 权限，使用 Redis `SET NX + EX` 原子操作做防刷检查：
   - Key: `slowmode:{user_id}:{channel_id}`
   - 若 Key 已存在 → 返回 `InSlowmode` 错误，附带 `retry_after` TTL
   - 若 Key 不存在 → 设置 Key 并发布 `UserSlowmodes` 私有事件

4. **Interactions 校验** — 如果请求中包含 `interactions` 字段，验证 reactions 列表中的 emoji 可用。

5. **新用户提及限制** — 在可发现的 (discoverable) 公开服务器中，注册不足 12 小时的用户禁止 @提及。

6. **委托给 `Message::create_from_api`** — 将 `db`、`amqp`、`channel`、`data`、`idempotency` 等参数传入核心编排方法，完成消息构造、持久化与广播。

---

## 三、核心编排层：Message::create_from_api

**文件**: [model.rs (Message)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L269-L609)

### 3.1 消息实体构造

```
Message {
    id:          Ulid::new()           // 服务端生成的 ULID，兼具时间序与唯一性
    nonce:       idempotency.into_key() // 幂等键写入 nonce 字段，回传客户端
    channel:     channel.id()
    author:      author.id()
    content:     data.content
    mentions:    解析后的被 @ 用户 ID 列表
    role_mentions: 解析后的被 @ 角色 ID 列表
    replies:     验证后的回复消息 ID 列表
    flags:       位域 (SuppressNotifications | MentionsEveryone | MentionsOnline)
    attachments: 已确认的附件 File 列表
    embeds:      处理后的嵌入内容
    masquerade:  伪装信息
    interactions:交互配置
    webhook:     Webhook 发送时的信息
    ...
}
```

### 3.2 关键子流程

#### 3.2.1 Nonce 幂等消费

```rust
idempotency.consume_nonce(data.nonce).await
```

详见 [第五节：幂等保障](#五幂等保障)。

#### 3.2.2 空消息拦截

content、attachments、embeds 三者均为空时返回 `EmptyMessage` 错误。

#### 3.2.3 Mention 解析与过滤

1. 使用 `revolt_parser::parse_message` 从 content 中提取 `@user`、`@everyone`、`@online`、`<@role_id>` 等原始提及。
2. 合并请求 flags 中的 `MentionsEveryone` / `MentionsOnline` 标记。
3. **权限二次校验**：如果触发了 mass mention，需要 `MentionEveryone` 或 `MentionRoles` 权限。
4. **可见性过滤**：
   - DM/Group：仅保留 recipients 中的 user_mentions
   - TextChannel：先查 server members 确认存在性，再用 `BulkDatabasePermissionQuery` 过滤出能查看该频道的成员
   - SavedMessages：清空所有 mentions

#### 3.2.4 Reply 校验

- 逐条 `db.fetch_message(&id)` 验证被回复消息存在
- `fail_if_not_exists = true` (默认) → 消息不存在则报错
- `fail_if_not_exists = false` → 静默丢弃该条 reply
- `mention = true` 时将被回复消息的 author 加入 mentions

#### 3.2.5 附件与嵌入

- 附件：`File::use_attachment` 将临时文件标记为已使用
- 嵌入：`attach_sendable_embed` 处理 `SendableEmbed` 中的 media 附件

---

## 四、持久化与事件广播层

### 4.1 Message::send → send_without_notifications

**文件**: [model.rs (send)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L671-L734)

`send` 方法分为两步：

#### Step 1: send_without_notifications

**文件**: [model.rs (send_without_notifications)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L612-L667)

1. **MongoDB 写入** — `db.insert_message(self)` 将完整 Message 文档写入 `messages` 集合。

   **文件**: [mongodb.rs (insert_message)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/ops/mongodb.rs#L23-L25)
   ```rust
   async fn insert_message(&self, message: &Message) -> Result<()> {
       query!(self, insert_one, COL, &message).map(|_| ())
   }
   ```

2. **Redis PubSub 广播** — `EventV1::Message(...).p(channel)` 将消息事件发布到以 channel ID 为名的 Redis topic。

3. **异步更新 last_message_id** — `tasks::last_message_id::queue` 入队一个防抖任务，延迟写入 `channel.last_message_id`。

   **文件**: [last_message_id.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/tasks/last_message_id.rs)
   - 使用 `HashMap<String, DelayedTask<Task>>` 按频道聚合
   - 同一频道连续多条消息时，只保留最新 ID，延迟提交
   - DM 频道同时设置 `active = true`

4. **入队未读/推送任务** — `tasks::ack::queue_message` 入队 `AckEvent::ProcessMessage`。

#### Step 2: 推送通知分支

`send` 方法在 `send_without_notifications` 之后，根据条件决定是否推送通知：

- 条件：`!suppress_notifications && (is_dm_or_group || mentions.is_some() || contains_mass_push_mention)`
- 满足时再次调用 `tasks::ack::queue_message`，携带 `PushNotification` 信息

### 4.2 事件广播：Redis PubSub → Bonfire WebSocket

#### 4.2.1 发布端

**文件**: [client.rs (EventV1::p)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/events/client.rs#L367-L377)

```rust
pub async fn p(self, channel: String) {
    redis_kiss::p(channel, self).await;   // 发布到 Redis
}
```

事件路由规则：
| 方法 | Topic 格式 | 订阅者 |
|---|---|---|
| `.p(channel_id)` | `channel_id` | 订阅了该频道的所有 WebSocket 连接 |
| `.private(user_id)` | `{user_id}!` | 仅该用户本人的 WebSocket 连接 |
| `.server(server_id)` | `{server_id}u` | 该服务器所有成员（bot 专用） |
| `.global()` | `global` | 全局订阅者 |

#### 4.2.2 订阅端 (Bonfire)

**文件**: [websocket.rs (listener)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/bonfire/src/websocket.rs#L221-L403)

1. 每个 WebSocket 连接启动时，`State::from(user)` 初始化订阅集合：
   - 必定订阅 `{user_id}!`（私有 topic）和 `{user_id}`（用户 topic）
   - 订阅所有可查看的 channel topic
   - 订阅所有所在 server topic

2. `listener` 函数使用 fred (Redis client) 订阅这些 topic，循环接收消息：
   - 反序列化 `EventV1`（支持 JSON / Msgpack / Bincode）
   - `state.handle_incoming_event_v1` 处理事件（更新本地缓存、调整订阅）
   - 通过 WebSocket 写入帧推送到客户端

3. `Message` 事件特殊处理：在 [impl.rs#L664-L676](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/bonfire/src/events/impl.rs#L664-L676) 中，会基于本地缓存重建 `relationship` 字段（因为消息广播给所有频道订阅者，但 relationship 是按接收者视角不同的）。

---

## 五、消息实体与未读状态的关联

### 5.1 ChannelUnread 数据模型

**文件**: [model.rs (ChannelUnread)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/channel_unreads/model.rs#L1-L24)

```
ChannelUnread {
    id: ChannelCompositeKey {    // 复合主键
        channel: String,          // 频道 ID
        user: String,             // 用户 ID
    },
    last_id: Option<String>,      // 该用户在此频道最后已读消息 ID
    mentions: Option<Vec<String>>, // @了该用户的消息 ID 列表
}
```

MongoDB 中存储于 `channel_unreads` 集合，`_id` 为 `{ channel, user }` 复合键。

### 5.2 未读写入：消息发送时

**文件**: [ack.rs (handle_ack_event::ProcessMessage)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/tasks/ack.rs#L120-L153)

消息发送后，`ProcessMessage` 事件在 ack worker 中被处理：

1. 收集所有被提及的用户 ID
2. 对每个用户，筛选出其中提及了该用户的消息 ID
3. 调用 `db.add_mention_to_unread(channel, user, message_ids)` 写入

**文件**: [mongodb.rs (add_mention_to_unread)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/channel_unreads/ops/mongodb.rs#L86-L110)

```rust
async fn add_mention_to_unread(&self, channel_id, user_id, message_ids) {
    // $push + $each 追加 mentions 数组，upsert=true 自动创建文档
    .update_one(
        { "_id.channel": channel_id, "_id.user": user_id },
        { "$push": { "mentions": { "$each": message_ids } } },
        .with_options(UpdateOptions::builder().upsert(true).build())
    )
}
```

注意：`last_id` **不在消息发送时设置**，只在用户主动 ack 时更新。

### 5.3 未读清除：用户 Ack 时

**文件**: [channel_ack.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/delta/src/routes/channels/channel_ack.rs#L15-L36)

```
PUT /channels/<target>/ack/<message>
```

1. 权限校验 `ViewChannel`
2. 调用 `channel.ack(user_id, message_id, amqp)`

**文件**: [model.rs (Channel::ack)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/channels/model.rs#L647-L657)

```rust
pub async fn ack(&self, user, message, amqp) {
    EventV1::ChannelAck { id, user, message_id }
        .private(user)           // 通知该用户的其他设备
        .await;
    acker::ack_channel(user, self.id(), message, amqp).await
}
```

**文件**: [acker.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/util/acker.rs#L7-L24)

- Redis `GETSET` 实现 **幂等 ack**：`acker:{user}+{channel}` → 只有值变化时才触发 AMQP `process_ack`
- AMQP `process_ack` 最终走到 `ack.rs` worker 中的 `AckMessage` 分支

**文件**: [mongodb.rs (acknowledge_message)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/channel_unreads/ops/mongodb.rs#L17-L49)

```rust
async fn acknowledge_message(&self, channel_id, user_id, message_id) {
    .find_one_and_update(
        { "_id.channel": channel_id, "_id.user": user_id },
        {
            "$pull": { "mentions": { "$lte": message_id } },  // 清除 <= message_id 的提及
            "$set": { "last_id": message_id }                  // 更新已读位置
        },
        .upsert(true) .return_document(After)
    )
}
```

ULID 天然递增，因此 `$lte: message_id` 可以安全地移除所有比已读位置更早的 mention 记录。

### 5.4 未读查询

**文件**: [get_unreads.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/delta/src/routes/sync/get_unreads.rs)

```
GET /sync/unreads
```

直接 `db.fetch_unreads(&user.id)` 返回该用户所有频道的 `ChannelUnread` 列表。

---

## 六、幂等保障：重复发送防重

幂等保障分为两层，在请求链路的不同位置分别拦截。

### 6.1 第一层：Idempotency-Key Header

**文件**: [idempotency.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/util/idempotency.rs#L86-L127)

```rust
// Rocket FromRequest 实现
impl<'r> FromRequest<'r> for IdempotencyKey {
    async fn from_request(request) -> Outcome<Self> {
        if let Some(key) = request.headers().get("Idempotency-Key").next() {
            // Key 长度不能超过 64
            // 在全局 LRU Cache (容量 1000) 中查重
            // 若已存在 → 返回 409 Conflict (DuplicateNonce)
            // 若不存在 → 写入 Cache 并返回
        }
        // 未提供 Header → 自动生成 ULID 作为 key
    }
}
```

- 全局 LRU 缓存 `TOKEN_CACHE`，容量 1000
- 在请求进入 handler 之前就拦截重复的 `Idempotency-Key`
- 若客户端未提供 Header，自动生成 ULID（不会冲突）

### 6.2 第二层：Nonce 消费

**文件**: [idempotency.rs (consume_nonce)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/util/idempotency.rs#L27-L39)

```rust
pub async fn consume_nonce(&mut self, v: Option<String>) -> Result<()> {
    if let Some(v) = v {
        if cache.get(&v).is_some() {
            return Err(create_error!(DuplicateNonce));
        }
        cache.put(v.clone(), ());
        self.key = v;   // nonce 值覆盖 idempotency key
    }
    Ok(())
}
```

在 `Message::create_from_api` 内部调用，将客户端请求体中的 `nonce` 字段与同一个 LRU Cache 做二次查重。

### 6.3 双层防护流程

```
请求到达
  │
  ├─ 有 Idempotency-Key Header?
  │    ├─ Yes → LRU Cache 查重 → 重复则 409 Conflict
  │    └─ No  → 自动生成 ULID key
  │
  ▼
message_send handler
  │
  ├─ data.nonce 存在?
  │    ├─ Yes → consume_nonce → LRU Cache 二次查重 → 重复则 DuplicateNonce
  │    │        → nonce 值覆盖 idempotency.key
  │    └─ No  → 跳过，使用 Header 层的 key
  │
  ▼
消息写入 MongoDB
  → message.nonce = idempotency.into_key()
```

**注意事项**：
- LRU Cache 是进程内的（非分布式），重启后丢失
- 容量 1000 意味着只保留最近 1000 个 key
- 这是一个**尽力而为** (best-effort) 的幂等保障，适用于防抖和短时间内的重复提交
- MongoDB 层面没有对 nonce 建唯一索引，所以不保证跨进程/跨重启的严格幂等

---

## 七、完整时序图

```
Client          Delta (Rocket)        Database (MongoDB)    Redis           Bonfire (WS)     ack Worker
  │                 │                      │                  │                 │                │
  │─ POST /messages ─►│                     │                  │                 │                │
  │  (Idempotency-Key)│                     │                  │                 │                │
  │                 │─ FromRequest ──────────────────────────►│                 │                │
  │                 │  (LRU cache check)   │                  │                 │                │
  │                 │◄─ Ok ───────────────────────────────────│                 │                │
  │                 │                      │                  │                 │                │
  │                 │─ validate ───────►   │                  │                 │                │
  │                 │─ permission check ──►│                  │                 │                │
  │                 │─ slowmode check ───────────────────────►│                 │                │
  │                 │                      │                  │                 │                │
  │                 │─ Message::create_from_api              │                 │                │
  │                 │  ├─ consume_nonce (LRU) ──────────────►│                 │                │
  │                 │  ├─ parse mentions                     │                 │                │
  │                 │  ├─ validate replies ──►│              │                 │                │
  │                 │  └─ construct Message  │              │                 │                │
  │                 │                      │                  │                 │                │
  │                 │─ Message::send        │                  │                 │                │
  │                 │  └─ send_without_notifications         │                 │                │
  │                 │     ├─ insert_message ─►│              │                 │                │
  │                 │     │                 │                  │                 │                │
  │                 │     ├─ EventV1::Message.p(channel) ───────────────────►│                │
  │                 │     │                 │                  │  ── WS push ──►│ (clients)      │
  │                 │     │                 │                  │                 │                │
  │                 │     ├─ last_message_id::queue           │                 │                │
  │                 │     │  (debounced)    │                  │                 │                │
  │                 │     │                 │                  │                 │                │
  │                 │     └─ ack::queue_message (mentions)    │                 │                │
  │                 │                       │                  │                 │                │
  │                 │─ ack::queue_message (push notifications)│                 │                │
  │                 │                      │                  │                 │                │
  │◄─ 200 Ok (Message) │                  │                  │                 │                │
  │                 │                      │                  │                 │                │
  │                 │                      │    (async)       │                 │                │
  │                 │                      │                  │                 │                │
  │                 │                      │◄──────────────────────────────────────────────────│
  │                 │                      │ add_mention_to_unread             │                │
  │                 │                      │                  │                 │                │
  │                 │                      │◄──────────────────────────────────────────────────│
  │                 │                      │ amqp.message_sent / mass_mention  │                │
```

---

## 八、关键设计总结

| 环节 | 设计决策 | 原因 |
|---|---|---|
| 消息 ID | ULID (服务端生成) | 时间有序 + 全局唯一，可天然按 ID 排序、做范围查询 |
| 幂等 | 两层 LRU Cache (进程内) | 轻量防抖，适合短时重复提交；非严格分布式幂等 |
| 事件广播 | Redis PubSub | 解耦 API 进程与 WS 进程；支持多 Bonfire 实例横向扩展 |
| 未读写入 | 异步 ack worker + 防抖 | 避免每条消息同步写 MongoDB；高频频道自动合并 |
| last_message_id | 异步防抖更新 | 同一频道短时间内多条消息只写一次 DB |
| 慢速模式 | Redis SET NX + EX | 原子操作，无需额外清理；TTL 自动过期 |
| Mention 过滤 | 先解析后校验可见性 | 防止通过 @提及探测不可见用户是否存在 |
| Ack 幂等 | Redis GETSET | 同一用户同一频道重复 ack 相同 message_id 时跳过 |

---

## 九、校验与异步分支薄弱点逐条剖析

下文沿着 `message_send` → `Message::create_from_api` → `Message::send` 的执行顺序，将代码中容易遗漏或误读的分支逻辑逐一对照源码讲清。

### 9.1 `send` 与 `send_without_notifications` 的互斥推送投递

**文件**: [model.rs#L671-L734 (send)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L671-L734) 与 [model.rs#L612-L667 (send_without_notifications)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L612-L667)

这两个方法共同承担"推送通知"的职责，但分工极为精确——**同一条消息只会被其中一个方法入队带推送通知的 `ProcessMessage`，不会重复**。

`send` 调用 `send_without_notifications` 时传入 `mentions_elsewhere = true`：

```rust
// send 方法, L681-L688
self.send_without_notifications(
    db, user.clone(), member.clone(),
    matches!(channel, Channel::DirectMessage { .. }),
    generate_embeds,
    true,   // ← mentions_elsewhere = true
).await?;
```

`send_without_notifications` 内部据此跳过自身的 mention 入队：

```rust
// send_without_notifications, L636-L651
if !mentions_elsewhere {
    if let Some(mentions) = &self.mentions {
        tasks::ack::queue_message(
            self.channel.to_string(),
            AckEvent::ProcessMessage {
                messages: vec![(
                    None,                          // ← push = None
                    self.clone(),
                    mentions.clone(),
                    self.has_suppressed_notifications(),
                )],
            },
        ).await;
    }
}
```

注意此分支中 `push = None`——它**只负责写未读 (add_mention_to_unread)**，不触发推送通知。

随后 `send` 方法根据条件决定是否投递带推送通知的任务：

```rust
// send 方法, L696-L731
if !self.has_suppressed_notifications()
    && (is_dm_or_group || self.mentions.is_some() || self.contains_mass_push_mention())
{
    tasks::ack::queue_message(
        self.channel.to_string(),
        AckEvent::ProcessMessage {
            messages: vec![(
                Some(PushNotification::from(...).await),  // ← push = Some
                self.clone(),
                match channel {
                    Channel::DirectMessage { recipients, .. }
                    | Channel::Group { recipients, .. } =>
                        recipients.iter().filter(|uid| *uid != author.id()).cloned().collect(),
                    Channel::TextChannel { .. } => self.mentions.clone().unwrap_or_default(),
                    _ => vec![],
                },
                false,  // ← silenced = false (本分支已排除 suppress)
            )],
        },
    ).await;
}
```

**两条路径的互斥关系**：

| 场景 | `send_without_notifications` 入队？ | `send` 入队？ | 效果 |
|---|---|---|---|
| `send` 调用 (正常发消息) | ❌ (`mentions_elsewhere=true`) | ✅ (条件满足时) | 推送+未读 由 `send` 一次搞定 |
| `send_without_notifications` 直接调用 (系统消息等) | ✅ (有 mentions 时) | 不调用 | 只写未读，不推通知 |

**薄弱点**：当 `send` 的推送条件不满足 (如 suppress_notifications=true 且无 mass mention)，但消息确实有 mentions 时——`send` 不入队，而 `send_without_notifications` 又被 `mentions_elsewhere=true` 跳过——**mention 未读记录不会被写入**。不过实际场景中 suppress 的是通知推送而非 mention 记录，这在 `send_without_notifications` 里已经用 `silenced` 字段区分：即使 `silenced=true`，mention 仍会被入队。所以真正的缺口是：当 `send` 接管了 mention 处理权 (`mentions_elsewhere=true`) 却因为推送条件不满足而不入队时，mention 未读就丢失了。实际上 `send` 的推送条件 `mentions.is_some()` 恰好覆盖了"有 mention"的情况，所以**只要 mentions 存在就一定会入队**，这个互斥是安全的。

### 9.2 Embed 抓取的异步入队

**文件**: [process_embeds.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/tasks/process_embeds.rs)

嵌入链接抓取在 `send_without_notifications` 中被入队：

```rust
// model.rs, L654-L664
if generate_embeds {
    if let Some(content) = &self.content {
        tasks::process_embeds::queue(
            self.channel.to_string(),
            self.id.to_string(),
            content.clone(),
        ).await;
    }
}
```

Worker 的工作流程：

1. 从容量 10,000 的 `Queue<EmbedTask>` 中取出任务
2. 调用 [generate](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/tasks/process_embeds.rs#L85-L170)：
   - 用正则剔除 code block (\`\`\`...\`\`\` / \`...\`) 和尖括号链接 (`<http...>`)
   - 剔除引用行 (`>`)
   - 用 `linkify` 提取所有 URL，去重后取前 `max_embeds` 条
   - 对每个 URL 并发请求 `january` 服务的 `/embed?url=...` 接口
   - 用 `Semaphore` 限制最大并发连接数
3. 抓取成功后调用 `Message::append` 将 embed 追加到消息

**薄弱点**：
- **消息先返回客户端，embed 后到**：客户端收到的 `Message` 初始不含 link embed，需通过 `MessageAppend` 事件异步获得。客户端必须处理此事件才能展示完整内容。
- **失败静默**：`generate` 返回 `Err` 时 worker 不做任何事（不重试），embed 就永远不会出现。
- **January 单点依赖**：如果 January 服务不可用，所有链接嵌入都不会生成，且不会影响消息发送本身（降级为纯文本）。
- **linkify 正则剥离不完整**：只处理了 code block 和 `<http>` 语法，inline code 中的链接如果被 linkify 在正则替换前扫描到，可能误抓取。实际执行顺序是先正则替换再 linkify，所以 code block 内容已被清空，是安全的。

### 9.3 入口处的总长校验与 flag 三条快速失败

**文件**: [model.rs#L284-L331](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L284-L331)

#### 总长校验 (`validate_sum`)

```rust
Message::validate_sum(
    &data.content,
    data.embeds.as_deref().unwrap_or_default(),
    limits.message_length,
)?;
```

**文件**: [model.rs#L977-L998 (validate_sum)](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L977-L998)

将 `content.len()` + 每个 embed 的 `description.len()` 累加，超过 `limits.message_length` 则返回 `PayloadTooLarge`。

**薄弱点**：只统计了 content 和 embed description 的字符数，**不计 embed 的 title / url / icon_url / media / colour**，也不计 attachment 的元数据大小。这是一个"内容长度"而非"消息总字节数"的校验。

#### Flag 三条快速失败

```rust
if let Some(raw_flags) = &data.flags {
    // 快速失败 1: 值 > 7（三位 bit 全组合也只有 7）
    if raw_flags > &7 {
        return Err(create_error!(InvalidProperty));
    }

    let flags = MessageFlagsValue(*raw_flags);
    suppress_notifications = flags.has(MessageFlags::SuppressNotifications);
    mentions_everyone = allow_mentions && flags.has(MessageFlags::MentionsEveryone);
    mentions_online = allow_mentions && flags.has(MessageFlags::MentionsOnline);

    // 快速失败 2: 非机器人用户设置 @everyone/@online 标记
    if user.as_ref().is_some_and(|u| u.bot.as_ref().is_none())
        && (mentions_everyone || mentions_online)
    {
        return Err(create_error!(IsNotBot));
    }

    // 快速失败 3: @everyone 与 @online 互斥
    if mentions_everyone && mentions_online {
        return Err(create_error!(InvalidFlagValue));
    }
}
```

三条快速失败路径：

| # | 条件 | 错误类型 | 含义 |
|---|---|---|---|
| 1 | `raw_flags > 7` | `InvalidProperty` | 当前只用了 bit 0/1/2，任何 >7 的值都非法 |
| 2 | 非机器人 && (`mentions_everyone` \|\| `mentions_online`) | `IsNotBot` | 只有 bot 才能通过 flags 设置 mass mention |
| 3 | `mentions_everyone` && `mentions_online` | `InvalidFlagValue` | 两种 mass mention 互斥，不可同时指定 |

**薄弱点**：
- 快速失败 2 只拦截**通过 flags 设置**的 mass mention，不拦截**通过消息正文 `@everyone`** 触发的。正文中写 `@everyone` 的普通用户由后面的 `MentionEveryone` 权限校验拦截。
- `allow_mentions` 为 false 时（新用户在可发现服务器），`mentions_everyone` / `mentions_online` 被强制为 false，快速失败 2 和 3 不会被触发，但也不会报错——**静默降级**而非拒绝。

### 9.4 Redis 端 slowmode 用户索引集

**文件**: [message_send.rs#L89-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/delta/src/routes/channels/message_send.rs#L89-L97)

```rust
if set_result.is_some() {
    let idx_key = format!("slowmode_idx:{}", user.id);
    conn.sadd::<_, _, ()>(&idx_key, channel_id.as_str()).await.ok();
    conn.expire::<_, _, ()>(&idx_key, *channel_slowmode as usize).await.ok();
}
```

当 slowmode 键成功设置（`NX` 生效）时，同时维护一个用户维度的索引集：

- **Key**: `slowmode_idx:{user_id}` — Redis Set
- **Member**: 该用户当前被限速的 channel_id
- **TTL**: 与最新设置的 slowmode 键相同

**用途**：Bonfire WebSocket 连接建立时，通过 [fetch_user_slowmodes](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/bonfire/src/websocket.rs#L541-L578) 批量读取该索引集，一次性获取用户所有活跃的 slowmode 状态。

```rust
let channel_ids: Vec<String> = conn.smembers(&idx_key).await.unwrap_or_default();
// 批量 TTL 查询
let mut pipe = redis_kiss::redis::pipe();
for channel_id in &channel_ids {
    pipe.ttl(format!("slowmode:{}:{}", user_id, channel_id));
}
let ttls: Vec<i64> = pipe.query_async(&mut conn).await.unwrap_or_default();
```

**薄弱点**：
- **TTL 不精确**：索引集的 `EXPIRE` 被设置为 `*channel_slowmode`（频道慢速模式时长），但后续如果用户在同一频道的另一个 slowmode 周期中发消息，TTL 会被刷新。如果用户跨多个不同 slowmode 时长的频道发消息，TTL 会被**最后设置的频道时长**覆盖——可能过早或过晚过期。
- **过期成员清理不即时**：索引集的成员只在 Bonfire 读取时惰性清理（TTL<=0 的 member 被移除），其余时间可能包含已过期的 channel_id。
- **`sadd` / `expire` 非原子**：两个操作分开执行，如果中间进程崩溃，索引集可能缺少该 member 或 TTL 未设置。
- **`.ok()` 吞错误**：`sadd` 和 `expire` 的错误被静默忽略，Redis 连接问题不会阻塞消息发送。

### 9.5 新用户提及限制只在可发现服务器生效

**文件**: [message_send.rs#L143-L157](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/delta/src/routes/channels/message_send.rs#L143-L157)

```rust
let allow_mentions = if let Some(server) = query.server_ref() {
    if server.discoverable {
        (ulid::Ulid::from_string(&user.id)
            .unwrap()
            .datetime()
            .elapsed()
            .expect("Time went backwards"))
            >= Duration::from_hours(12)
    } else {
        true
    }
} else {
    true
};
```

逻辑分解：

| 条件 | `allow_mentions` |
|---|---|
| 非服务器频道 (DM/Group/SavedMessages) | `true` |
| 服务器频道 + `discoverable = false` | `true` |
| 服务器频道 + `discoverable = true` + 用户注册 >= 12h | `true` |
| 服务器频道 + `discoverable = true` + 用户注册 < 12h | `false` |

`allow_mentions` 被传入 `create_from_api`，影响：
1. **flags 解析**：`mentions_everyone = allow_mentions && flags.has(MentionsEveryone)` — false 时静默降级
2. **reply mention**：`if mention && allow_mentions { user_mentions.insert(...) }` — false 时不自动 @被回复者
3. **`allow_mass_mentions`**：`allow_mentions && config.features.mass_mentions_enabled` — 双重门控

**薄弱点**：
- **只限可发现服务器**：非可发现服务器的新用户不受限，可以立刻 @everyone。这是有意为之——可发现服务器对滥用更敏感。
- **时间判断用 ULID**：`Ulid::from_string(&user.id).datetime()` 依赖 ULID 的时间戳编码。如果用户 ID 不是 ULID（理论上不会，但没有校验），`unwrap()` 会 panic。
- **`expect("Time went backwards")`**：用户注册时间不可能晚于当前时间，但如果系统时钟回拨，会 panic。
- **与权限系统独立**：这个检查在权限检查之后、消息构造之前，是额外的安全层。即使有 `MentionEveryone` 权限，新用户仍然被限制。

### 9.6 Mass mention 全局开关

**文件**: [model.rs#L303](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L303) 与 [model.rs#L404-L443](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L404-L443)

```rust
let allow_mass_mentions = allow_mentions && config.features.mass_mentions_enabled;
```

全局开关 `config.features.mass_mentions_enabled` 在两处发挥作用：

**第一处：role 提及的存在性过滤前置条件**

```rust
if allow_mass_mentions && server_id.is_some() && !role_mentions.is_empty() {
    let server_data = db.fetch_server(server_id.unwrap().as_str()).await.expect(...);
    role_mentions.retain(|role_id| server_data.roles.contains_key(role_id));
}
```

只有 `allow_mass_mentions = true` 时才做 role 存在性过滤。如果 `mass_mentions_enabled = false`，这整段被跳过——role_mentions 保留原始值。

**第二处：mass mention 的静默清除 vs 权限校验分流**

```rust
if !config.features.mass_mentions_enabled
    && (mentions_everyone || mentions_online || !role_mentions.is_empty())
{
    // 全局开关关闭 → 静默清除所有 mass mention
    mentions_everyone = false;
    mentions_online = false;
    role_mentions.clear();
} else if mentions_everyone || mentions_online || !role_mentions.is_empty() {
    // 全局开关开启 → 进入权限校验
    ...
}
```

**薄弱点**：
- **开关关闭时是静默降级**：不报错，只是把 mentions_everyone / mentions_online / role_mentions 清空。消息照常发送，但 flags 中不会设置 mass mention 位。
- **role 存在性过滤依赖全局开关**：当 `mass_mentions_enabled = false` 时，role_mentions 不经过 `server_data.roles.contains_key` 过滤就直接被 clear 了，所以没有安全问题。但如果未来有人修改逻辑让 clear 分支不清空 role_mentions，就会出现未验证的 role ID 泄漏到消息中。
- **`allow_mass_mentions` 不等于 `mass_mentions_enabled`**：前者还叠加了 `allow_mentions`（新用户限制），这意味着新用户在可发现服务器里即使全局开关开启，也无法触发 role 存在性过滤——但此时 role_mentions 本身也不应该存在（parser 解析出的 role mentions 会被后续清除逻辑处理）。

### 9.7 Role 提及的存在性过滤

**文件**: [model.rs#L394-L401](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L394-L401)

```rust
if allow_mass_mentions && server_id.is_some() && !role_mentions.is_empty() {
    let server_data = db
        .fetch_server(server_id.unwrap().as_str())
        .await
        .expect("Failed to fetch server");

    role_mentions.retain(|role_id| server_data.roles.contains_key(role_id));
}
```

三重前置条件：
1. `allow_mass_mentions` = `allow_mentions && config.features.mass_mentions_enabled`
2. `server_id.is_some()` — 只在服务器频道 (TextChannel) 中执行
3. `!role_mentions.is_empty()` — 无 role mention 则跳过

过滤方式：从数据库获取完整 server 数据，`retain` 只保留 server 中真实存在的 role ID。

**薄弱点**：
- **`.expect("Failed to fetch server")` 会 panic**：如果数据库查询失败，整个请求线程崩溃而非返回错误。这与其他地方用 `?` 传播错误的做法不一致。
- **`server_id.unwrap()`**：虽然外层已检查 `is_some()`，但 `unwrap()` 不如 `if let Some(server) = server_id` 安全。
- **不检查用户是否有该 role**：只验证 role 存在于 server，不验证 mentioning 用户是否拥有该 role 或有权 @该 role。权限校验在后面的 `MentionRoles` 检查中完成（只看有无 `MentionRoles` 权限，不看能否 @特定 role）。
- **DM/Group 频道被排除**：这些频道的 `server_id = None`，role_mentions 在后续的用户可见性过滤中被 `role_mentions.clear()` 清除。

### 9.8 回复消息的错误类型分流

**文件**: [model.rs#L458-L489](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L458-L489)

```rust
for ReplyIntent { id, mention, fail_if_not_exists } in entries {
    match db.fetch_message(&id).await {
        Ok(message) => {
            if mention && allow_mentions {
                user_mentions.insert(message.author.to_owned());
            }
            if !replies.contains(&message.id) {
                replies.push(message.id);
            }
        }
        Err(e) => {
            if !matches!(e.error_type, ErrorType::NotFound)
                || fail_if_not_exists.unwrap_or(true)
            {
                return Err(e);
            }
        }
    }
}
```

错误分流逻辑：

```
db.fetch_message(&id) 返回 Err(e)
  │
  ├─ e.error_type != NotFound?
  │    └─ Yes → return Err(e)     // 非 NotFound 错误直接上抛（如数据库故障 → InternalError）
  │
  └─ e.error_type == NotFound?
       ├─ fail_if_not_exists = Some(true)  → return Err(e)    // 要求必须存在，不存在则报错
       ├─ fail_if_not_exists = Some(false) → continue          // 容忍不存在，静默丢弃该 reply
       └─ fail_if_not_exists = None         → return Err(e)    // 默认行为 = true
```

**薄弱点**：
- **非 NotFound 错误被原样上抛**：如果 `fetch_message` 因数据库故障返回 `InternalError`，即使 `fail_if_not_exists = false`，消息发送也会失败。这是正确行为——数据库故障不应静默忽略。
- **`fail_if_not_exists` 默认值**：`None` 等同于 `true`，即默认要求被回复消息必须存在。客户端如果不传此字段，旧消息被删除后回复会失败。
- **O(n²) 去重**：代码注释说明 `replies.contains(&message.id)` 是 O(n²)，但在 `message_replies` 限制较小（默认 5）时比 HashSet 快。
- **mention 不受 `fail_if_not_exists` 影响**：当 reply 消息存在且 `mention = true` 时，被回复者一定被加入 mentions，无论其是否有权查看当前频道。后续的 mention 可见性过滤会处理这个情况。

### 9.9 系统作者常量 ULID

**文件**: [model.rs#L353-L357](file:///d:/fz/0601-1/solo-dogfeeding/code/81-backend/crates/core/database/src/models/messages/model.rs#L353-L357)

```rust
let (author_id, webhook) = match &author {
    MessageAuthor::User(user) => (user.id.clone(), None),
    MessageAuthor::Webhook(webhook) => (webhook.id.clone(), Some((*webhook).clone())),
    MessageAuthor::System { .. } => ("00000000000000000000000000".to_string(), None),
};
```

系统消息的 author 被硬编码为 26 个零（ULID 格式，时间戳为 Unix 纪元）。

**用途**：所有系统事件消息（用户加入/离开/被踢/被禁、频道重命名/描述修改/所有权转移、消息置顶/取消置顶、通话开始等）的 author 都是这个常量。

**薄弱点**：
- **不是一个真实用户 ID**：数据库中不存在 ID 为 `00000000000000000000000000` 的用户文档。客户端在展示时需要特殊处理——不能用这个 ID 去查用户资料。
- **与 `MessageAuthor::System` 的区分**：`System` 变体还携带 `username` 和 `avatar` 信息，用于展示。但 `author_id` 被固定为全零 ULID 后，这些展示信息不会出现在 `Message.author` 字段中，而是通过 `masquerade` 或客户端特殊逻辑处理。
- **全零 ULID 的时间戳**：ULID 的前 10 位是毫秒时间戳，全零对应 1970-01-01。如果用 ULID 排序，系统消息的 author ID 会排在所有真实用户之前。
- **消息删除权限**：`delete_message` 的权限检查通常需要 author 匹配或 `ManageMessages` 权限。系统消息的 author 是全零 ULID，没有用户能匹配，所以**只有 `ManageMessages` 权限才能删除系统消息**——这其实是正确的行为。
