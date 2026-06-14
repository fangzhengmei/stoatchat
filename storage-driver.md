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
| `delete_channel` | 删除 invites → unreads → webhooks → messages → 从 server.channels 移除 → 附件批量标记 deleted → 删除 channel 本身 | 仅从 HashMap 中 `remove` channel，不清理任何关联数据（消息、附件、邀请等全部残留） |
| `delete_server` | 删除所有 messages → emojis 标记 Detached → 删除 channels → 删除 unreads/invites → 删除 members/bans → 附件批量标记 deleted | 仅从 HashMap 中 `remove` server |
| `delete_role` | 从 server_members 的 roles 数组 `$pull` → 从 channels 的 role_permissions `$unset` → 从 server.roles `$unset` | 仅从 server.roles 中 `remove` |

**后果**：Reference 后端删除频道后，消息、邀请、未读、Webhook 仍然残留在内存中；删除服务器后成员、封禁、频道等全部孤立。这些残留数据在 Reference 中还能被 fetch 到，产生"幽灵数据"。此外，Reference 的 `mark_attachments_as_deleted` 批量接口存在字段误用 Bug（详见 4.8 节），即便在有级联删除的路径上也无法正确标记附件删除状态。

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

### 4.8 附件批量标删：reported 与 deleted 字段误用导致双重击穿

**Bug 概述**：Reference 后端的 `mark_attachments_as_deleted`（批量版本）错误地将附件标记为 `reported=true`，而非 `deleted=true`，与 MongoDB 行为不一致，也与同名的单条版本（`mark_attachment_as_deleted`）不一致。

---

#### 4.8.1 代码证据

**Trait 定义**（[ops.rs#L43-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/files/ops.rs#L43-L44)）：

```rust
/// Mark multiple attachments as having been deleted.
async fn mark_attachments_as_deleted(&self, ids: &[String]) -> Result<()>;
```

注释明确：语义是"标记为已删除"。

**MongoDB 批量实现**（[mongodb.rs#L157-L174](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/files/ops/mongodb.rs#L157-L174)）：

```rust
async fn mark_attachments_as_deleted(&self, ids: &[String]) -> Result<()> {
    self.col::<Document>(COL)
        .update_many(
            doc! { "_id": { "$in": ids } },
            doc! { "$set": { "deleted": true } },
        )
        .await
}
```

设置 `deleted: true`，与注释一致。

**Reference 单条实现**（[reference.rs#L106-L114](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/files/ops/reference.rs#L106-L114)）：

```rust
async fn mark_attachment_as_deleted(&self, id: &str) -> Result<()> {
    let mut files = self.files.lock().await;
    if let Some(file) = files.get_mut(id) {
        file.deleted = Some(true);
        Ok(())
    } else { ... }
}
```

单条版本正确设置 `deleted = Some(true)`。

**Reference 批量实现**（[reference.rs#L117-L133](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/files/ops/reference.rs#L117-L133)）：

```rust
async fn mark_attachments_as_deleted(&self, ids: &[String]) -> Result<()> {
    let mut files = self.files.lock().await;
    // ... 校验存在性 ...
    for id in ids {
        if let Some(file) = files.get_mut(id) {
            file.reported = Some(true);  // ← Bug：应为 deleted
        }
    }
    Ok(())
}
```

批量版本设置 `reported = Some(true)`——明显是复制粘贴 `mark_attachment_as_reported` 时忘记改字段名。

---

#### 4.8.2 触发路径：两条调用链都会踩到

调用 `mark_attachments_as_deleted` 批量接口的路径有两条：

| 触发源 | 位置 | 场景 |
|--------|------|------|
| `Message::delete()` | [messages/model.rs#L1008](file:///d:/fz/0601-1\solo-dogfeeding/code/89-backend/crates/core/database/src/models/messages/model.rs#L1008) | 单条消息删除时，级联将附件标记为 deleted |
| `prune_dangling_files` 定时任务 | [prune_dangling_files.rs#L30](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/daemons/crond/src/tasks/prune_dangling_files.rs#L30) | 扫描悬挂文件后，将超时的悬挂文件批量标删 |

> 注意：`Message::bulk_delete`（消息批量删除）**不处理附件**，只删消息记录本身。这是另一处功能缺口，但不属于本 Bug 范畴。

---

#### 4.8.3 双重击穿：附件卡在生命周期夹缝

`deleted` 和 `reported` 两个布尔字段共同决定了附件在清理系统中的可见性。Bug 导致附件被错误地放入 `reported` 状态而非 `deleted` 状态，从而在两道清理闸口前同时"隐身"。

**第一道闸：已删除附件清理队列**

`fetch_deleted_attachments` 的过滤条件（[reference.rs#L37-L49](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/files/ops/reference.rs#L37-L49)）：

```rust
file.deleted.is_some_and(|v| v) && !file.reported.is_some_and(|v| v)
```

即：**已删除且未被举报** 的附件才会进入清理队列。被误标的附件：`deleted = None, reported = Some(true)` → 不满足 → **击穿**。物理删除任务永远看不到这些附件。

**第二道闸：悬挂文件检测**

`fetch_dangling_files` 的过滤条件（[reference.rs#L52-L59](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/files/ops/reference.rs#L52-L59)）：

```rust
file.used_for.is_none() && !file.deleted.is_some_and(|v| v)
```

即：**未被使用且未被删除** 的附件才算 dangling。被误标的附件：`used_for = Some(Message{...}), deleted = None` → `used_for.is_none()` 不成立 → **击穿**。悬挂检测任务也看不到这些附件。

**双重击穿的后果**：附件卡在 `reported=true + used_for=Some + deleted=false` 的中间状态——既不在删除队列里，也不在悬挂列表中，更不会被正常业务继续使用，成为永久"隐形"的孤儿记录。

---

#### 4.8.4 单条路径为何不受影响

值得注意的是，单条 `mark_attachment_as_deleted` 是正确的。以下调用走单条路径，不受 Bug 影响：

- 用户移除头像/背景：[edit_user.rs#L64](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/delta/src/routes/users/edit_user.rs#L64)
- 服务器移除 banner/icon：[server_edit.rs#L118](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/delta/src/routes/servers/server_edit.rs#L118)、[#L124](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/delta/src/routes/servers/server_edit.rs#L124)
- 角色移除 icon：[roles_edit.rs#L57](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/delta/src/routes/servers/roles_edit.rs#L57)
- 频道移除 icon：[channel_edit.rs#L105](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/delta/src/routes/channels/channel_edit.rs#L105)

这些场景都是单个附件，走单条接口，行为正确。只有**批量场景**（消息级联删除、悬挂文件定时清理）才会触发 Bug。

---

### 4.9 成员分块迭代器：offset 边界写反导致首次迭代即返回 None

**Bug 概述**：`ChunkedServerMembersGenerator::Reference` 分支的 `next()` 方法将终止条件 `offset >= data.len()` 错写为 `data.len() >= offset`，导致 offset 从 0 起步时首次调用就返回 `None`，迭代器永远无法产出任何成员。

---

#### 4.9.1 代码证据

[ops.rs#L55-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/server_members/ops.rs#L55-L63)：

```rust
ChunkedServerMembersGenerator::Reference { offset, data } => {
    if let Some(data) = data {
        if data.len() as i32 >= *offset {   // ← Bug：应为 *offset >= data.len() as i32
            None
        } else {
            let resp = &data[*offset as usize];
            *offset += 1;
            Some(resp.clone())
        }
    } else {
        warn!("...");
        None
    }
}
```

初始状态由 `new_reference` 设置（[ops.rs#L36-L41](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/server_members/ops.rs#L36-L41)）：

```rust
pub fn new_reference(data: Vec<Member>) -> Self {
    ChunkedServerMembersGenerator::Reference {
        offset: 0,
        data: Some(data),
    }
}
```

**逻辑推演**：offset=0, data.len()=N (N≥0)

| 条件 | 当前代码 | 正确逻辑 |
|------|---------|---------|
| 终止判断 | `N >= 0` → 恒为 true → 返回 None | `0 >= N` → 当 N>0 时为 false → 返回 data[0] |
| 首次迭代 | **永远返回 None** | 返回第一个元素 |

正确写法应为：

```rust
if *offset >= data.len() as i32 {
    None
} else { ... }
```

---

#### 4.9.2 下游影响链：pushd 全员@和角色@静默失败

迭代器由 pushd 守护进程的 `mass_mention` 消费者使用（[mass_mention.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs)）：

**路径一：@everyone / @here 触发**（[mass_mention.rs#L135-L188](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L135-L188)）

```rust
let mut db_query = self.db.fetch_all_members_chunked(&payload.server_id).await?;
loop {
    let mut chunk: Vec<Member> = vec![];
    for _ in 0..config.pushd.mass_mention_chunk_size {
        if let Some(member) = db_query.next().await {
            chunk.push(member);
        } else {
            exhausted = true;
            break;
        }
    }
    // ... 为 chunk 中的用户发送推送通知 ...
    if exhausted { break; }
}
```

**路径二：角色@触发**（[mass_mention.rs#L189-L209](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L189-L209)）

```rust
let mut role_members = self.db.fetch_all_members_with_roles_chunked(&payload.server_id, roles).await?;
while !exhausted {
    chunk.clear();
    for _ in 0..config.pushd.mass_mention_chunk_size {
        if let Some(member) = role_members.next().await {
            chunk.push(member);
        } else {
            exhausted = true;
            break;
        }
    }
    // ... 为 chunk 中的用户发送推送通知 ...
}
```

**影响链路**：

1. 用户在服务器中发送 `@everyone` 或 `@role` 消息
2. pushd 收到消息事件，调用 `fetch_all_members_chunked` 或 `fetch_all_members_with_roles_chunked`
3. 首次 `next()` 返回 None → chunk 为空 → `exhausted = true`
4. 循环立即退出，不进入推送逻辑
5. **结果**：所有服务器成员收不到 @everyone 通知；角色成员收不到 @role 通知——静默失败，无报错

MongoDB 后端使用 `SessionCursor` 驱动迭代，不受此 Bug 影响。只有 Reference 后端使用 `offset + Vec` 模拟，命中此 Bug。

---

### 4.10 图片 hash animated 字段：两套后端触发条件完全相反

**Bug 概述**：`set_attachment_hash_animated` 在 MongoDB 中只对 `animated` 字段尚未设置（`$exists: false`）的图片写入，而 Reference 中只对 `animated` 字段**已经设置**（`Some`）的图片覆盖——两者触发条件完全相反，导致同一调用在两套后端上行为互斥。

---

#### 4.10.1 代码证据

**MongoDB 实现**（[mongodb.rs#L55-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/file_hashes/ops/mongodb.rs#L55-L71)）：

```rust
async fn set_attachment_hash_animated(&self, hash: &str, animated: bool) -> Result<()> {
    self.col::<FileHash>(COL)
        .update_one(
            doc! {
                "_id": hash,
                "metadata.type": "Image",
                "metadata.animated": { "$exists": false },   // ← 只匹配 animated 不存在的记录
            },
            doc! {
                "$set": {
                    "metadata.animated": animated
                }
            },
        )
        .await
}
```

条件：`_id 匹配 AND 类型为 Image AND animated 字段不存在` → **只写未设 animated 的图片**

**Reference 实现**（[reference.rs#L45-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/file_hashes/ops/reference.rs#L45-L61)）：

```rust
async fn set_attachment_hash_animated(&self, hash: &str, animated: bool) -> Result<()> {
    let mut hashes = self.file_hashes.lock().await;
    if let Some(FileHash {
        metadata:
            Metadata::Image {
                animated: Some(animated_metadata),   // ← 只匹配 animated 已是 Some 的记录
                ..
            },
        ..
    }) = hashes.get_mut(hash)
    {
        *animated_metadata = animated;
        Ok(())
    } else {
        Err(create_error!(NotFound))
    }
}
```

条件：`hash 匹配 AND 类型为 Image AND animated 是 Some` → **只写已有 animated 的图片**

**对比**：

| 维度 | MongoDB | Reference |
|------|---------|-----------|
| 触发条件 | `animated` **不存在**（None） | `animated` **已存在**（Some） |
| 首次写入（animated=None） | ✅ 匹配 `$exists: false`，成功写入 | ❌ 不匹配 `Some`，返回 NotFound |
| 重复写入（animated=Some） | ❌ 不匹配 `$exists: false`，静默跳过 | ✅ 匹配 `Some`，覆盖写入 |
| 语义 | "只在未设时写一次" | "只在已设时覆盖" |

---

#### 4.10.2 调用上下文与实际影响

唯一调用方是 Autumn 文件服务（[api.rs#L389-L408](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/services/autumn/src/api.rs#L389-L408)）：

```rust
let is_animated = match &hash.metadata {
    Metadata::Image { animated: Some(value), .. } => *value,
    Metadata::Image { animated: None, .. } => {
        // ... 下载文件，检测是否动态图 ...
        let animated = is_animated(&named_file, &hash.content_type).unwrap_or(false);
        db.set_attachment_hash_animated(&hash.id, animated).await?;
        animated
    }
    _ => false,
};
```

**逻辑**：当 `animated` 为 `None` 时，才调用 `set_attachment_hash_animated` 做首次检测写入。

**MongoDB 行为**：`animated` 为 None → `$exists: false` 匹配 → 成功写入 → ✅ 符合预期

**Reference 行为**：`animated` 为 None → 不匹配 `Some(animated_metadata)` → 返回 `NotFound` 错误 → ❌ 与预期相反

**后果链**：

1. 用户上传图片，`FileHash` 初始创建时 `animated: None`
2. 首次访问图片时，Autumn 检测到 `animated: None`，调用 `set_attachment_hash_animated`
3. Reference 后端返回 `NotFound`，向上传播为 404 错误
4. **首次图片访问失败**，animated 元数据永远无法被写入
5. 后续每次访问都会重新触发检测（因为 `animated` 始终为 None），每次都会失败
6. 最终结果：Reference 后端下图片的 animated 标记永远为 None，动态图/静态图的区分完全失效

Reference 实现的正确写法应为：

```rust
if let Some(FileHash {
    metadata:
        Metadata::Image {
            animated: animated_metadata @ None,  // 匹配 None，而非 Some
            ..
        },
    ..
}) = hashes.get_mut(hash)
{
    *animated_metadata = Some(animated);
    Ok(())
} else {
    Err(create_error!(NotFound))
}
```

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
| 🔴 高 | `mark_attachments_as_deleted` 批量版本误用 `reported` 字段替代 `deleted`，导致附件双重击穿、永久卡在生命周期夹缝 | [reference.rs#L117-L133](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/files/ops/reference.rs#L117-L133) |
| 🔴 高 | `ChunkedServerMembersGenerator::Reference` 的 `next()` 边界条件写反，首次迭代即返回 None，pushd @everyone/@role 推送静默失败 | [ops.rs#L55-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/server_members/ops.rs#L55-L63) |
| 🔴 高 | `set_attachment_hash_animated` 触发条件相反：MongoDB 只写未设的，Reference 只写已设的，导致图片 animated 标记永远无法首次写入 | [reference.rs#L45-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/file_hashes/ops/reference.rs#L45-L61) vs [mongodb.rs#L55-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/89-backend/crates/core/database/src/models/file_hashes/ops/mongodb.rs#L55-L71) |
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
