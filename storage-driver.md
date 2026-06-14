# Storage Driver 架构分析

## 1. 整体架构概览

本项目采用 **Trait 抽象 + 枚举分派** 的经典策略，在同一套模型接口下并存两套后端实现：

| 层级 | 文件位置 | 职责 |
|------|----------|------|
| 抽象接口 | `crates/core/database/src/models/<model>/ops.rs` | 用 `#[async_trait]` trait 定义每个模型的数据库操作签名 |
| 模型本体 | `crates/core/database/src/models/<model>/model.rs` | 数据结构定义、业务逻辑（创建/更新/删除的上下文编排） |
| MongoDB 实现 | `crates/core/database/src/models/<model>/ops/mongodb.rs` | 把 trait 操作翻译为 MongoDB 查询 |
| Reference 实现 | `crates/core/database/src/models/<model>/ops/reference.rs` | 用进程内 `HashMap + Mutex` 做内存模拟 |
| 驱动注册 | `crates/core/database/src/drivers/mod.rs` | `Database` 枚举 + `DatabaseInfo` 连接工厂 |
| 统一分派 | `crates/core/database/src/models/mod.rs` | `AbstractDatabase` 超级 trait + `Deref` 动态分派 |

核心分派机制在 [mod.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/mod.rs#L46-L86)：

```rust
pub trait AbstractDatabase:
    Sync + Send
    + AbstractMigrations + AbstractBots + AbstractChannels
    + AbstractChannelInvites + AbstractChannelUnreads + AbstractWebhooks
    + AbstractEmojis + AbstractAttachmentHashes + AbstractAttachments
    + AbstractMessages + AbstractPolicyChange + AbstractRatelimitEvents
    + AbstractReport + AbstractSnapshot + AbstractServerBans
    + AbstractServerMembers + AbstractServers + AbstractUserSettings
    + AbstractUsers
{}

impl AbstractDatabase for ReferenceDb {}
impl AbstractDatabase for MongoDb {}

impl Deref for Database {
    type Target = dyn AbstractDatabase;
    fn deref(&self) -> &Self::Target {
        match &self {
            Database::Reference(dummy) => dummy,
            Database::MongoDb(mongo) => mongo,
        }
    }
}
```

上层业务代码持有 `Database`，通过 `Deref` 获得 `&dyn AbstractDatabase`，调用 `db.insert_channel(...)` 时由 Rust 动态分派至对应后端。

---

## 2. 抽象操作接口的职责分界

### 2.1 Trait 层：只定义 "做什么"，不规定 "怎么做"

以 [AbstractChannels](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/channels/ops.rs#L8-L52) 为例：

```rust
#[async_trait]
pub trait AbstractChannels: Sync + Send {
    async fn insert_channel(&self, channel: &Channel) -> Result<()>;
    async fn fetch_channel(&self, channel_id: &str) -> Result<Channel>;
    async fn fetch_channels<'a>(&self, ids: &'a [String]) -> Result<Vec<Channel>>;
    async fn find_direct_messages(&self, user_id: &str) -> Result<Vec<Channel>>;
    async fn find_saved_messages_channel(&self, user_id: &str) -> Result<Channel>;
    async fn find_direct_message_channel(&self, user_a: &str, user_b: &str) -> Result<Channel>;
    async fn add_user_to_group(&self, channel_id: &str, user_id: &str) -> Result<()>;
    async fn set_channel_role_permission(&self, ...) -> Result<()>;
    async fn update_channel(&self, id: &str, channel: &PartialChannel, remove: Vec<FieldsChannel>) -> Result<()>;
    async fn remove_user_from_group(&self, channel_id: &str, user_id: &str) -> Result<()>;
    async fn delete_channel(&self, channel_id: &Channel) -> Result<()>;
}
```

**关键设计要点：**

- **参数已脱驱动化**：trait 方法接收的是 `Channel`、`PartialChannel`、`FieldsChannel` 等领域模型，而非 `Document`/`Bson` 等 MongoDB 类型。驱动特定类型被限制在具体实现内部。
- **Partial + Fields 模式**：更新操作统一用 `PartialXxx`（可选字段结构体）+ `Vec<FieldsXxx>`（要清除的字段列表）表达，两套后端各自解读。
- **返回值统一为 `Result<T>`**：错误类型是项目自有的 `revolt_result::Result`，而非 MongoDB 的原生错误。

### 2.2 Model 层：编排业务逻辑

[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/channels/model.rs) 中的 `impl Channel` 承担了三大职责：

1. **业务校验**（如频道数量上限、群组人数上限）
2. **调用 trait 抽象方法**操作数据库
3. **发送事件通知**（`EventV1::ChannelCreate` 等）

```rust
pub async fn create_server_channel(db: &Database, server: &mut Server, data: ...) -> Result<Channel> {
    // 1. 校验上限
    if server.channels.len() > config.features.limits.global.server_channels { ... }
    // 2. 构造模型
    let channel = Channel::TextChannel { ... };
    // 3. 持久化（走 trait）
    db.insert_channel(&channel).await?;
    // 4. 更新关联（走 trait）
    server.update(db, PartialServer { channels: Some(...) }, vec![]).await?;
    // 5. 发事件
    EventV1::ChannelCreate(channel.clone().into()).p(server.id.clone()).await;
    Ok(channel)
}
```

Model 层不感知底层数据库，所有存储操作都通过 `db.xxx()` 走 trait 分派。

### 2.3 Driver 层：只做存储翻译

以 MongoDB 驱动为例，其唯一职责是把 trait 语义翻译为 MongoDB 操作：

```rust
impl AbstractChannels for MongoDb {
    async fn insert_channel(&self, channel: &Channel) -> Result<()> {
        query!(self, insert_one, COL, &channel).map(|_| ())
    }
    async fn add_user_to_group(&self, channel: &str, user: &str) -> Result<()> {
        self.col::<Document>(COL).update_one(
            doc! { "_id": channel },
            doc! { "$push": { "recipients": user } },
        ).await.map(|_| ()).map_err(...)
    }
}
```

驱动层不做业务校验、不发事件——这些全部留在 Model 层。

---

## 3. 两套后端的具体实现方式

### 3.1 MongoDB 驱动

[MongoDb](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/drivers/mongodb.rs) 是对 `mongodb::Client` 的薄封装，提供 `insert_one`、`find_one`、`update_one`、`delete_one` 等通用方法。

**关键特性：**

- 利用 MongoDB 的 **原子操作符**：`$push`、`$pull`、`$addToSet`、`$unset`、`$set` 实现细粒度局部更新
- 使用 `$in`、`$all`、`$elemMatch` 做集合查询
- 聚合管道（`aggregate`）用于复杂查询，如 [fetch_mutual_server_ids](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/users/ops/mongodb.rs#L157-L206)
- `$text` + `$meta:textScore` 支持全文搜索排序
- **事务与快照读**：`fetch_all_members_chunked` 使用 `start_transaction().read_concern(ReadConcern::snapshot())` 保证遍历一致性
- `query!` 宏在 debug 模式下硬解包错误，在 release 模式下转换为 `create_database_error!`

### 3.2 Reference 实现

[ReferenceDb](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/drivers/reference.rs) 用 `Arc<Mutex<HashMap<K, V>>>` 模拟每个集合：

```rust
pub struct ReferenceDb {
    pub bots: Arc<Mutex<HashMap<String, Bot>>>,
    pub channels: Arc<Mutex<HashMap<String, Channel>>>,
    pub messages: Arc<Mutex<HashMap<String, Message>>>,
    pub server_members: Arc<Mutex<HashMap<MemberCompositeKey, Member>>>,
    // ...
}
```

**关键特性：**

- 每次 `lock().await` 获取整张表的互斥锁
- 查询用 `.values().filter(...).cloned().collect()` 遍历全表
- 更新用 `get_mut()` + `apply_options()` + `remove_field()` 原地修改
- 没有事务、没有快照读、没有聚合管道

---

## 4. 两种后端的一致性差异——逐项对比

### 4.1 级联删除：MongoDB 完整 vs Reference 阉割

这是**最严重的一致性缺陷**。

| 操作 | MongoDB 行为 | Reference 行为 |
|------|-------------|---------------|
| `delete_channel` | 删除 invites → unreads → webhooks → messages(含附件标记) → 从 server.channels 移除 → 附件标记 deleted → 删除 channel 本身 | 仅从 HashMap 中 `remove` channel，不清理任何关联数据 |
| `delete_server` | 删除所有 messages(含附件标记) → emojis 标记 Detached → 删除 channels → 删除 unreads/invites → 删除 members/bans → 附件标记 deleted | 仅从 HashMap 中 `remove` server |
| `delete_role` | 从 server_members 的 roles 数组 `$pull` → 从 channels 的 role_permissions `$unset` → 从 server.roles `$unset` | 仅从 server.roles 中 `remove` |

**后果**：Reference 后端删除频道后，消息、邀请、未读、Webhook 仍然残留在内存中；删除服务器后成员、封禁、频道等全部孤立。这些残留数据在 Reference 中还能被 fetch 到，产生"幽灵数据"。

### 4.2 查询语义：等价 vs 近似

| 操作 | MongoDB | Reference | 差异 |
|------|---------|-----------|------|
| `find_direct_messages` | `$or: [{channel_type: DirectMessage/Group, recipients: user_id}, {channel_type: SavedMessages, user: user_id}]` | `.filter(\|ch\| ch.contains_user(user_id))` | Reference 只匹配 Group 类型（`contains_user` 只在 `Channel::Group` 返回 true），不匹配 DirectMessage 和 SavedMessages |
| `find_saved_messages_channel` | `channel_type: SavedMessages, user: user_id` | `channels.get(user_id)`（用 user_id 当 key 查） | **语义完全不同**：MongoDB 按 `channel_type + user` 字段查，Reference 用 user_id 当 HashMap key，几乎不可能命中 |
| `fetch_channels(ids)` | `$in` 查询，缺失的静默跳过 | 逐个 `get`，任一缺失返回 `NotFound` | MongoDB 宽容返回已找到的，Reference 严格全有或全无 |
| `add_mention_to_unread` | `upsert: true`，不存在则创建 | 先 get_mut，不存在则 insert | 语义一致但实现路径不同 |

### 4.3 未实现功能：`todo!()` 占位

Reference 后端中以下方法直接 `todo!()`，调用会 panic：

| 模型 | 方法 |
|------|------|
| Users | `fetch_session_by_token`、`fetch_mutual_user_ids`、`fetch_mutual_channel_ids`、`fetch_mutual_server_ids`、`remove_push_subscription_by_session_id`、`update_session_last_seen` |
| ServerMembers | `remove_dangling_members` |

这些功能涉及 authifier Session（外部依赖）或聚合查询，Reference 用 `todo!()` 表示"尚无动机实现"。

### 4.4 行为分歧：同一接口返回不同结果

| 操作 | MongoDB | Reference | 影响 |
|------|---------|-----------|------|
| `set_channel_role_permission` | 直接 `$set role_permissions.{role_id}`，无论 role 是否已存在 | 先检查 `role_permissions.get(role_id).is_some()`，不存在则返回 `NotFound` | MongoDB 幂等地设置，Reference 要求角色权限已存在才可更新 |
| `set_relationship(None)` | 仅当 `RelationshipStatus::None` 时走 `pull_relationship` | 当 `RelationshipStatus::None` 或 `User` 时走 `pull_relationship` | Reference 多处理了 `User` 变体，MongoDB 没有这层逻辑 |
| `acknowledge_message` 返回值 | 使用 `find_one_and_update` + `upsert` + `ReturnDocument::After`，返回更新后的完整文档 | 手动构造/修改 HashMap 中的值，返回 `get(&key).cloned()` | MongoDB 原子返回 after-state，Reference 先写再读（非原子但单锁下等效） |
| `acknowledge_channels` | 删除旧 unreads + 批量 insert 新记录（`last_id` 当前时间，无 mentions） | 逐个调用 `acknowledge_message` | MongoDB 丢弃所有 mentions 信息，Reference 保留 |
| `fetch_messages` pinned 过滤 | `filter.insert("pinned", pinned)` — 匹配 pinned 字段等于给定值 | `message.pinned.unwrap_or_default() == pinned` 返回 false 时**过滤掉** | **逻辑反转**：MongoDB 保留 pinned==true 的，Reference 在 pinned==true 时 `== pinned` 为 false 反而过滤掉了。这是一个 Bug |

### 4.5 并发与一致性模型

| 维度 | MongoDB | Reference |
|------|---------|-----------|
| 隔离级别 | 支持事务（`start_transaction` + `ReadConcern::snapshot`） | 全局 Mutex，串行化所有操作 |
| 并发粒度 | 文档级锁（MongoDB WiredTiger） | 表级锁（整个 `channels` HashMap 一把锁） |
| 跨集合原子性 | 可用事务保证 | 单锁串行化天然一致但性能极差 |
| 持久性 | 写入 journal 后确认 | 进程崩溃即丢失 |

### 4.6 `insert_or_merge_member` 的分歧

| MongoDB | Reference |
|---------|-----------|
| 如果找到 `pending_deletion_at` 的记录，用 `find_one_and_update` 合并（重置 joined_at，清除 pending_deletion_at），返回更新后的 Member | 如果 key 已存在直接报错 `create_database_error!("insert", "member")`，不处理合并 |

### 4.7 消息存取接口深度差异

消息模型（`AbstractMessages`）是两套后端差异最密集的领域。以下逐接口展开分析。

---

#### 4.7.1 `delete_messages`——Reference 的 retain 条件与 MongoDB 语义不一致

**Trait 签名**（[ops.rs#L44](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops.rs#L44)）：

```rust
async fn delete_messages(&self, channel: &str, ids: &[String]) -> Result<()>;
```

语义意图：删除 **同时满足** "属于指定 channel" **且** "ID 在 ids 列表中" 的消息。

**MongoDB 实现**（[mongodb.rs#L300-L312](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/mongodb.rs#L300-L312)）：

```rust
self.col::<Document>(COL)
    .delete_many(doc! {
        "channel": channel,
        "_id": { "$in": ids }
    })
    .await
```

删除条件：`channel == channel AND _id IN ids`，是 **AND 合取**。

**Reference 实现**（[reference.rs#L282-L289](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/reference.rs#L282-L289)）：

```rust
self.messages
    .lock()
    .await
    .retain(|id, message| message.channel != channel && !ids.contains(id));
```

保留条件：`channel != channel AND id NOT IN ids`。

**逻辑推导**：

| 条件 | MongoDB 保留（delete 的否定） | Reference 保留 |
|------|------------------------------|----------------|
| De Morgan 展开 | `channel != channel ∨ id NOT IN ids` | `channel != channel ∧ id NOT IN ids` |

正确 retain 应为 **OR**（De Morgan 定律），但 Reference 写成了 **AND**。

**用例推演**（假设 `channel = "ch1"`, `ids = ["msgB"]`）：

| 消息 | channel | id | MongoDB 行为 | Reference 行为 | 差异 |
|------|---------|----|-------------|---------------|------|
| A | "ch1" | "msgA" | 保留（id 不在 ids 中） | **删除**（channel 匹配 ∧ id 不在 ids → AND 不满足 → 不保留） | 🔴 误删同频道非目标消息 |
| B | "ch2" | "msgB" | 保留（channel 不匹配） | **删除**（channel 不匹配 ∧ id 在 ids 中 → AND 不满足 → 不保留） | 🔴 误删跨频道消息 |
| C | "ch1" | "msgB" | 删除（两个条件都匹配） | 删除（channel 匹配 ∧ id 在 ids → 都不满足 → 不保留） | ✅ 一致 |
| D | "ch2" | "msgA" | 保留（两个条件都不匹配） | 保留（channel 不匹配 ∧ id 不在 ids → 都满足 → 保留） | ✅ 一致 |

**结论**：Reference 实现把 AND 误写为 AND（而非正确的 OR），导致影响范围分为三段：

1. **目标频道里的消息全被删**——无论是否出现在 ids 列表中。Case A（ch1, msgA）和 Case C（ch1, msgB）都无法被保留，因为 `channel != channel` 始终为 false，使得整个 AND 表达式为 false，消息全部被 retain 丢弃。
2. **其他频道里命中 ids 列表的消息被误删**——Case B（ch2, msgB）中 `channel != channel` 为 true，但 `!ids.contains(id)` 为 false（因为 msgB 在列表中），AND 不满足，消息被错误删除。
3. **其他频道里不命中 ids 列表的消息正常保留**——Case D（ch2, msgA）中两个子条件均为 true，AND 满足，消息被保留。

正确写法应为：

```rust
.retain(|id, message| message.channel != channel || !ids.contains(id));
//                                             ^^          ^
```

这是整个消息模块最严重的逻辑 Bug——一次批量删除会连同目标频道的全部消息和其他频道中恰好 ID 匹配的消息一起清理掉。

---

#### 4.7.2 `fetch_messages`——全文索引与子串匹配的本质差异

**Trait 签名**（[ops.rs#L20](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops.rs#L20)）：

```rust
async fn fetch_messages(&self, query: MessageQuery) -> Result<Vec<Message>>;
```

`MessageFilter` 中的搜索字段定义（[model.rs#L184-L193](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/model.rs#L184-L193)）：

```rust
pub struct MessageFilter {
    pub channel: Option<String>,
    pub author: Option<String>,
    pub query: Option<String>,      // "Search query"
    pub pinned: Option<bool>,       // "Search for pinned"
}
```

**MongoDB 实现**（[mongodb.rs#L45-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/mongodb.rs#L45-L56)）：

```rust
let is_search_query = if let Some(query) = query.filter.query {
    filter.insert(
        "$text",
        doc! { "$search": query },
    );
    true
} else {
    false
};
```

**Reference 实现**（[reference.rs#L51-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/reference.rs#L51-L58)）：

```rust
if let Some(query) = &query.filter.query {
    if let Some(content) = &message.content {
        if !content.to_lowercase().contains(query) {
            return false;
        }
    } else {
        return false;
    }
}
```

两者的搜索机制存在 **六个维度** 的本质差异：

| 维度 | MongoDB `$text` | Reference `to_lowercase().contains()` |
|------|-----------------|---------------------------------------|
| **分词方式** | 按语言规则分词（空格/标点分割），匹配整词。例如搜索 `"hello"` 不匹配 `"helloworld"` | 不分词，将整个 content 视为连续字符串做子串匹配。搜索 `"hello"` 可匹配 `"helloworld"` |
| **大小写** | 文本索引默认不区分大小写（取决于索引的 `default_language`） | 手动 `.to_lowercase()` 转换 content，但 **query 原样保留**——若 query 含大写字母则不会命中（`"Hello".to_lowercase().contains("Hello")` 为 false） |
| **停止词** | MongoDB 内置停止词表（英语默认约 127 个），搜索 `"the"` 可能零结果 | 无停止词概念，`"the"` 可匹配任何包含该子串的内容 |
| **词干提取** | MongoDB 对英语等语言做 stemming（`"running"` → `"run"`），搜索 `"run"` 可匹配 `"running"` | 无词干提取，`"run"` 不匹配 `"running"` |
| **排序** | 支持 `MessageSort::Relevance`，通过 `$meta: "textScore"` 按相关度排序；`Latest`/`Oldest` 按 `_id` 排序 | **未实现**，代码注释 `// FIXME: sorting, etc (will be required for tests)`，结果顺序不确定 |
| **分页/限流** | 通过 `FindOptions::builder().limit(limit)` 在数据库层截断 | **未实现** limit，返回全部匹配结果 |

**额外问题：query 的大小写陷阱**

Reference 中 `content.to_lowercase().contains(query)` 只对 content 做了小写化，但 query 本身未做小写化。当用户传入 `query = "Hello"` 时：
- `"hello world".to_lowercase().contains("Hello")` → `"hello world".contains("Hello")` → **false**

而 MongoDB 的 `$text` 搜索不区分大小写，会正确匹配。这意味着 Reference 中对含大写字母的搜索词会产生假阴性。

---

#### 4.7.3 `fetch_messages` 的 pinned 过滤 Bug——逻辑完全反转

**`MessageFilter` 字段注释**（[model.rs#L192](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/model.rs#L192)）：

```rust
/// Search for pinned
pub pinned: Option<bool>,
```

注释含义：`pinned = Some(true)` 时应 **保留** 已置顶消息，`pinned = Some(false)` 时应保留未置顶消息。

**MongoDB 实现**（[mongodb.rs#L58-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/mongodb.rs#L58-L60)）：

```rust
if let Some(pinned) = query.filter.pinned {
    filter.insert("pinned", pinned);
};
```

MongoDB 的行为：将 `pinned: true` 或 `pinned: false` 作为等值查询条件插入 filter，**保留** 匹配的文档。符合注释语义。

**Reference 实现**（[reference.rs#L61-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/reference.rs#L61-L65)）：

```rust
if let Some(pinned) = query.filter.pinned {
    if message.pinned.unwrap_or_default() == pinned {
        return false
    }
}
```

逻辑是：当 `message.pinned` 的值 **等于** `pinned` 参数时，**排除** 该消息（`return false`）。

**逐值推演**：

| `pinned` 参数 | `message.pinned` | `unwrap_or_default()` | `== pinned` | 结果 | 预期 |
|---------------|------------------|-----------------------|-------------|------|------|
| `Some(true)` | `Some(true)` | `true` | `true == true` = **true** | **排除** | 保留 ✅ |
| `Some(true)` | `None` | `false` | `false == true` = false | 保留 | 排除 ✅ |
| `Some(true)` | `Some(false)` | `false` | `false == true` = false | 保留 | 排除 ✅ |
| `Some(false)` | `Some(false)` | `false` | `false == false` = **true** | **排除** | 保留 ✅ |
| `Some(false)` | `Some(true)` | `true` | `true == false` = false | 保留 | 排除 ✅ |
| `Some(false)` | `None` | `false` | `false == false` = **true** | **排除** | 保留 ✅ |

**结论**：Reference 的 pinned 过滤逻辑 **完全反转**——它排除了应该保留的消息，保留了应该排除的消息。正确写法应为：

```rust
if message.pinned.unwrap_or_default() != pinned {
    return false
}
```

即：当消息的 pinned 状态与查询参数 **不等** 时才排除，**相等** 时保留。

**影响范围**：此 Bug 使得以下场景全部行为异常：
- `pinned=true` 查询会返回所有 **非置顶** 消息
- `pinned=false` 查询会返回所有 **已置顶** 消息
- 与 MongoDB 的行为完全相反

---

## 5. 架构模式总结

### 5.1 职责分界三棱图

```
┌──────────────────────────────────────────────────┐
│  Route 层 (delta/)                                │
│  HTTP 解析 → 权限检查 → 调用 Model 方法           │
├──────────────────────────────────────────────────┤
│  Model 层 (models/*/model.rs)                     │
│  业务校验 → 构造模型 → 调用 db.xxx() → 发事件     │
│  ⚠ 不感知底层驱动                                 │
├──────────────────────────────────────────────────┤
│  Trait 层 (models/*/ops.rs)                       │
│  定义操作签名，约束输入输出类型                     │
│  ⚠ 不包含实现                                     │
├──────────────────────────────────────────────────┤
│  Driver 层 (models/*/ops/mongodb.rs, reference.rs)│
│  MongoDB: 翻译为 BSON 查询 / 聚合管道             │
│  Reference: 翻译为 HashMap 操作                   │
│  ⚠ 不做业务校验、不发事件                         │
└──────────────────────────────────────────────────┘
```

### 5.2 关键适配机制

1. **`IntoDocumentPath` trait**：MongoDB 专用，让 `FieldsXxx` 枚举映射为 BSON 字段路径，用于 `$unset` 操作。Reference 不需要此 trait。
2. **`PartialXxx` + `FieldsXxx`**：统一表达"部分更新"。MongoDB 用 `$set` + `$unset` 分开处理；Reference 用 `apply_options()` + `remove_field()` 原地修改。
3. **`ChunkedServerMembersGenerator` 枚举**：MongoDB 用游标 + 事务会话，Reference 用 Vec + offset 偏移。两套流式遍历完全不同的底层机制。
4. **`query!` 宏**：统一错误处理——debug 模式硬解包快速失败，release 模式转为 `create_database_error!`。
5. **`database_derived!` 宏**：自动为驱动结构体添加 `Clone` derive，消除两套实现的样板差异。

### 5.3 Reference 实现的定位

Reference 后端的核心定位是**测试替身**，而非生产替代品。证据如下：

- `drop_database()` 为空操作（测试用完即弃）
- `migrate_database()` 直接返回 `Ok(())`
- 多个方法 `todo!()`（涉及外部系统或复杂聚合的功能被搁置）
- 级联删除只做单表 `remove`，不清理关联数据
- `fetch_messages` 的排序/分页逻辑注释了 `// FIXME: sorting, etc (will be required for tests)`

---

## 6. 一致性风险清单

| 风险等级 | 问题 | 位置 |
|----------|------|------|
| 🔴 高 | `delete_messages` retain 条件误用 AND 替代 OR，导致目标频道消息全删、其他频道命中列表的消息被误删 | [reference.rs#L282-L289](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/reference.rs#L282-L289) |
| 🔴 高 | `fetch_messages` pinned 过滤逻辑完全反转，`pinned=true` 返回非置顶消息 | [reference.rs#L61-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/reference.rs#L61-L65) |
| 🔴 高 | `delete_channel`/`delete_server` 级联删除差异导致 Reference 残留幽灵数据 | channels/ops/reference.rs、servers/ops/reference.rs |
| 🔴 高 | `find_saved_messages_channel` 在 Reference 中用 user_id 做 HashMap key，与 MongoDB 语义不符 | channels/ops/reference.rs#L55-L61 |
| 🔴 高 | `find_direct_messages` 在 Reference 中只匹配 Group，遗漏 DirectMessage 和 SavedMessages | channels/ops/reference.rs#L45-L52 |
| 🟡 中 | `fetch_messages` 全文索引 vs 子串匹配：分词/词干/停止词/排序/分页全部不等价，且 query 未小写化导致假阴性 | [reference.rs#L51-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/reference.rs#L51-L58) vs [mongodb.rs#L45-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/ops/mongodb.rs#L45-L56) |
| 🟡 中 | `fetch_channels` 在 Reference 中任一 ID 缺失即报错，MongoDB 宽容返回已有项 | channels/ops/reference.rs#L32-L42 |
| 🟡 中 | `set_channel_role_permission` 在 Reference 中要求已有才能更新，MongoDB 幂等设置 | channels/ops/reference.rs#L85-L112 |
| 🟡 中 | `insert_or_merge_member` 在 Reference 中不支持合并语义 | server_members/ops/reference.rs#L10-L19 |
| 🟢 低 | `acknowledge_channels` 在 MongoDB 中丢弃 mentions，Reference 中保留 | channel_unreads/ops 对比 |
| 🟢 低 | `set_relationship(None)` 分歧：Reference 额外处理了 `User` 变体 | users/ops/reference.rs#L120 |
