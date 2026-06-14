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
