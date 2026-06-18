# 邀请码（Channel Invite）从签发到核销的代码运转梳理

> 仓库：Revolt 后端（Rust + Rocket + MongoDB）
> 范围：`channel_invites`（频道邀请码），即“邀请某人加入服务器/群组”的实体。注意：本仓库中“邀请码”指频道邀请码，与注册时 `invite_only` 的“注册邀请”是两套独立概念。
> 代码引用统一使用仓库相对路径，换环境可照常定位。

---

## 1. 总览：链路一图

```
签发(Issue)            查询(Fetch)              核销(Redeem/Join)          删除(Delete)
POST /channels/<id>/invites   GET /invites/<code>   POST /invites/<code>   DELETE /invites/<code>
        │                              │                     │                      │
   create_invite                 invite_fetch            invite_join            invite_delete
        │                              │                     │                      │
   InviteOthers 权限            as_invite 解析         can_acquire_server      creator? 或
        │                              │              + as_invite 解析         ManageServer 权限
   Invite::create_channel_invite        │              + Member::create /        │
   nanoid!(8) → insert_invite           │                add_user_to_group    delete_invite
```

路由挂载见 [crates/delta/src/routes/mod.rs](crates/delta/src/routes/mod.rs#L21-L44)：`/channels`、`/invites`、`/servers`、`/bots` 分别挂载各子路由。

| 操作 | 方法 + 路径 | 处理函数 | 文件 |
|------|------------|---------|------|
| 签发 | `POST /channels/<target>/invites` | `create_invite` | [crates/delta/src/routes/channels/invite_create.rs](crates/delta/src/routes/channels/invite_create.rs#L17-L37) |
| 查询 | `GET /invites/<target>` | `fetch` | [crates/delta/src/routes/invites/invite_fetch.rs](crates/delta/src/routes/invites/invite_fetch.rs#L10-L71) |
| 核销（加入） | `POST /invites/<target>` | `join` | [crates/delta/src/routes/invites/invite_join.rs](crates/delta/src/routes/invites/invite_join.rs#L10-L49) |
| 删除 | `DELETE /invites/<target>` | `delete` | [crates/delta/src/routes/invites/invite_delete.rs](crates/delta/src/routes/invites/invite_delete.rs#L14-L35) |
| 列出服务器全部邀请 | `GET /servers/<target>/invites` | `invites` | [crates/delta/src/routes/servers/invites_fetch.rs](crates/delta/src/routes/servers/invites_fetch.rs#L14-L30) |
| 邀请机器人加入 | `POST /bots/<target>/invite` | `invite_bot` | [crates/delta/src/routes/bots/invite.rs](crates/delta/src/routes/bots/invite.rs#L18-L64) |

---

## 2. 数据模型

邀请码实体定义在 [crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L11-L41)：

```rust
pub enum Invite {
    Server { code, server, creator, channel },   // 指向服务器内某频道
    Group { code, creator, channel },            // 指向群组频道
}
```

- `code`：8 位 nanoid，字母表为 54 字符的 Crockford 风格（去除易混淆的 `I/L/O/U` 等），见 [crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L5-L9) 与生成处 [crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L65) `nanoid::nanoid!(8, &ALPHABET)`。
- **没有 `uses`、`max_uses`、`expires_at`、`created_at` 等字段**——邀请码不记次数、不带时间戳。
- 对外 API 模型（含 `InviteResponse`/`InviteJoinResponse`）见 [crates/core/models/src/v0/channel_invites.rs](crates/core/models/src/v0/channel_invites.rs#L1-L103)。

数据库层（MongoDB 集合名 `channel_invites`）：
- 抽象 trait：[crates/core/database/src/models/channel_invites/ops.rs](crates/core/database/src/models/channel_invites/ops.rs#L10-L21) — `insert_invite` / `fetch_invite` / `fetch_invites_for_server` / `delete_invite`。
- MongoDB 实现：[crates/core/database/src/models/channel_invites/ops/mongodb.rs](crates/core/database/src/models/channel_invites/ops/mongodb.rs#L12-L46)。
- 内存参考实现（测试用）：[crates/core/database/src/models/channel_invites/ops/reference.rs](crates/core/database/src/models/channel_invites/ops/reference.rs#L9-L51)。

---

## 3. 签发（Issue）

入口 [crates/delta/src/routes/channels/invite_create.rs](crates/delta/src/routes/channels/invite_create.rs#L17-L37)：

1. **机器人拦截**：`if user.bot.is_some()` → `IsBot`。
2. **取频道**：`target.as_channel(db)`。
3. **权限校验**：构造 `DatabasePermissionQuery` 并调用 `calculate_channel_permissions`，再 `throw_if_lacking_channel_permission(ChannelPermission::InviteOthers)`。`InviteOthers = 1 << 25`，定义在 [crates/core/permissions/src/models/channel.rs](crates/core/permissions/src/models/channel.rs#L69)。
4. **生成并落库**：`Invite::create_channel_invite`（[crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L60-L85)）：
   - `Group` 频道 → `Invite::Group`；
   - `TextChannel` → `Invite::Server`；
   - 其他频道类型 → `InvalidOperation`。
   - 随后 `db.insert_invite(&invite)`。

### 签发限额（重要结论）

**代码中对“签发数量”没有任何限额**：

- 不限制单频道、单用户、单服务器可创建的邀请码数量；
- 不校验调用方已持有多少邀请码；
- `create_invite` 既不调用 `can_acquire_server`，也不读取任何 invite 计数。

> 唯一与“额度”相关的配置位于加入侧（见第 6 节），而非签发侧。也就是说，理论上一个有 `InviteOthers` 权限的用户可以无限签发邀请码。

---

## 4. 查询（Fetch）——新增边界

### 4.1 是否需要登录？**不需要，匿名可查**

路由签名：

```rust
pub async fn fetch(db: &State<Database>, target: Reference<'_>) -> Result<Json<v0::InviteResponse>>
```

**参数中没有 `user: User`**，对比需要登录的 `join` 路由（有 `user: User` 参数）：

```rust
pub async fn join(db: &State<Database>, amqp: &State<AMQP>, user: User, target: Reference<'_>) -> ...
```

证明代码：[crates/delta/src/routes/invites/invite_fetch.rs](crates/delta/src/routes/invites/invite_fetch.rs#L10-L12) vs [crates/delta/src/routes/invites/invite_join.rs](crates/delta/src/routes/invites/invite_join.rs#L11-L16)。

**结论**：任何人持有邀请码即可匿名查询其指向的服务器/群组信息，无需认证。

### 4.2 会暴露哪些信息？

`InviteResponse` 结构定义在 [crates/core/models/src/v0/channel_invites.rs](crates/core/models/src/v0/channel_invites.rs#L31-L84)：

| 类型 | 暴露字段 | 不暴露 |
|------|---------|-------|
| **Server 邀请** | `code`、`server_id`、`server_name`、`server_icon`、`server_banner`、`server_flags`、`channel_id`、`channel_name`、`channel_description`、`user_name`、`user_avatar`、`member_count`（成员数） | 创建者的 `user_id`（只给用户名+头像）、服务器成员列表、频道消息、权限设置 |
| **Group 邀请** | `code`、`channel_id`、`channel_name`、`channel_description`、`user_name`、`user_avatar` | 创建者的 `user_id`、群成员列表、群消息 |

注意：`member_count` 是实时统计的服务器成员数，见 [crates/delta/src/routes/invites/invite_fetch.rs](crates/delta/src/routes/invites/invite_fetch.rs#L31) `db.fetch_member_count(&server.id).await? as i64`。

---

## 5. 核销（Redeem / Join）

入口 [crates/delta/src/routes/invites/invite_join.rs](crates/delta/src/routes/invites/invite_join.rs#L10-L49)：

1. **机器人拦截**：`IsBot`。
2. **服务器获取额度校验**：`user.can_acquire_server(db)`（防滥用，见第 6 节）。
3. **解析邀请码**：`target.as_invite(db)`（见下文“解析”）。
4. **分支处理**：
   - `Invite::Server` → `db.fetch_server` → `Member::create(db, &server, &user, None)`。
   - `Invite::Group` → `db.fetch_channel` → `channel.add_user_to_group(db, amqp, &user, creator)`。

### 5.1 邀请码解析（`as_invite`）

实现在 [crates/core/database/src/util/reference.rs](crates/core/database/src/util/reference.rs#L49-L69)：

- 若 `target` 是合法 ULID（即服务器 ID）：
  - 取该 server，**必须 `server.discoverable == true`**，否则 `NotFound`；
  - 构造一个“虚拟邀请”指向服务器的第一个频道（`server.channels.into_iter().next()`），`creator = server.owner`。
- 否则按普通邀请码 `db.fetch_invite(code)` 查询。

> 同等逻辑也存在于 `Invite::find`（[crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L88-L105)）。这意味着可发现（discoverable）的服务器可被直接用其 ID 当作邀请码使用。

### 5.2 加入服务器：`Member::create`

[crates/core/database/src/models/server_members/model.rs](crates/core/database/src/models/server_members/model.rs#L101-L146)：

1. **封禁检查**：`db.fetch_ban(&server.id, &user.id)` 命中 → `Banned`。
2. **已在服务器检查**：`db.fetch_member(...)` 命中 → `AlreadyInServer`。
3. **创建成员**：`Member { ..Default::default() }`，其中默认值见 [crates/core/database/src/models/server_members/model.rs](crates/core/database/src/models/server_members/model.rs#L85-L98)：`roles: vec![]`、`joined_at: now`、`timeout: None`。随后 `insert_or_merge_member`。
4. **可见频道过滤**：对每个频道计算权限，仅保留调用方能 `ViewChannel` 的频道。
5. **事件广播**：`ServerMemberJoin`（public）与 `ServerCreate`（private 给该用户）；若配置了 `system_messages.user_joined` 则发系统消息。

### 5.3 加入群组：`add_user_to_group`

[crates/core/database/src/models/channels/model.rs](crates/core/database/src/models/channels/model.rs#L339-L401)：

1. **已在群组检查**：`recipients.contains(user.id)` → `AlreadyInGroup`。
2. **群规模上限**：`recipients.len() >= config.features.limits.global.group_size` → `GroupTooLarge`（默认 100，见 [crates/core/config/Revolt.toml](crates/core/config/Revolt.toml#L222)）。
3. 加入 recipients、写库、发 `ChannelGroupJoin` 与 `UserAdded` 系统消息。

### 5.4 并发边界：服务器数量检查与成员写入是否原子？**不是原子，存在竞态窗口**

核销路径 `join` 的实际执行顺序是（[crates/delta/src/routes/invites/invite_join.rs](crates/delta/src/routes/invites/invite_join.rs#L21-L27)）：

```
① user.can_acquire_server(db).await?    // count_documents 读取
   ...（中间还夹杂了 as_invite / fetch_server 等多次 DB 调用）...
② Member::create(db, &server, &user, None).await?  // insert_or_merge_member 写入
```

两步是**两次独立的数据库调用**，中间没有任何事务、锁或 session 级隔离。全代码库只有 `fetch_all_members_chunked` 和 `fetch_all_members_with_roles_chunked` 用了 `start_transaction()` + `ReadConcern::snapshot()`（[crates/core/database/src/models/server_members/ops/mongodb.rs](crates/core/database/src/models/server_members/ops/mongodb.rs#L98-L120)），而核销路径完全没用到。

竞态窗口内，如果同一用户并发 N 次核销，所有请求都可能在步骤①读到相同的“未超限”结果，然后各自走到步骤②并发写入。最终：

- `server_members` 集合 `_id` 是复合主键 `(server, user)`，并且在初始化脚本里建了复合索引 `compound_id`（[crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs](crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L187-L206)）。MongoDB 的 `_id` 本身隐含唯一性约束，**同一用户对同一服务器并发写入最终只有一条成功，其余会因主键冲突失败**。但失败不会被翻译成业务错误，而是以 5xx 数据库错误抛给上层（`insert_or_merge_member` 内部对正常分支用的是裸 `query!(self, insert_one, COL, &member)`，没有捕获并转成 `AlreadyInServer`）。
- 但 `insert_or_merge_member` 还有一个“软删除恢复”分支：若先查询到带 `pending_deletion_at` 的旧记录，则走 `find_one_and_update` 把它复活（[crates/core/database/src/models/server_members/ops/mongodb.rs](crates/core/database/src/models/server_members/ops/mongodb.rs#L17-L52)）。这条路径下的检查 (`existing.is_ok_and(|x| x.is_some())`) 也不是原子的——两个并发请求都可能同时判定“存在待恢复记录”，然后并发执行 update，结果是 MongoDB 端幂等，但应用层会返回两份“已恢复”的结果。

结论：**并发核销可以绕过 `can_acquire_server` 的计数检查**（虽然对“同一服务器重复加入”最终被复合主键兜住了，但对“同时加入多个不同服务器”的超限没有数据库层兜底，会让用户最终加入的服务器数超过配额）。

### 5.5 并发边界：群组重复加入是否可能重复写入？**可能——内存检查 + 无唯一数组约束会产生重复 recipient**

`add_user_to_group` 的时序：

```
① if let Channel::Group { recipients, .. } = self {
     if recipients.contains(&String::from(&user.id)) {   // 内存中检查
         return Err(create_error!(AlreadyInGroup));
     }
     recipients.push(String::from(&user.id));             // 内存中 push
  }
② db.add_user_to_group(id, &user.id).await?              // MongoDB $push 到数组
```

[crates/core/database/src/models/channels/model.rs](crates/core/database/src/models/channels/model.rs#L347-L364)

而 `add_user_to_group` 的 DB 层是裸 `$push`，**没有 `$addToSet`，也没有任何前置查询条件**：

```rust
self.col::<Document>(COL)
    .update_one(
        doc! { "_id": channel },
        doc! { "$push": { "recipients": user } },
    )
```

[crates/core/database/src/models/channels/ops/mongodb.rs](crates/core/database/src/models/channels/ops/mongodb.rs#L108-L123)

竞态：两个并发请求都在步骤①读到 `recipients` 里没有该用户 → 都通过检查 → 都执行 `$push` → **`recipients` 数组里最终会出现两个相同的 user_id**。同时这也绕过了 `recipients.len() >= group_size` 的群规模上限（如果两个并发请求都在边界上读，都认为还有 1 个空位，就可能把群推到 `group_size + 1` 甚至更多）。

修复方向：把条件下推到 MongoDB，用 `update_one(doc! { "_id": channel, "recipients": { "$ne": user } }, doc! { "$push": ... })` 或者直接改用 `$addToSet`。

### 5.6 并发边界：群组邀请非创建者删除的实际表现？**panic，当前请求线程异常终止**

删除分支 [crates/delta/src/routes/invites/invite_delete.rs](crates/delta/src/routes/invites/invite_delete.rs#L21-L32)：

```rust
match invite {
    Invite::Server { code, server, .. } => { /* 校验 ManageServer 权限后删除 */ }
    _ => unreachable!(),
}
```

- 当 `invite` 是 `Invite::Group` 且删除者不是创建者时，进入 `_ => unreachable!()`。
- Rust 的 `unreachable!()` 是 `panic!` 的别名，会让**当前处理请求的工作线程 panic**。
- Rocket 默认启用 `panic = "unwind"`，单个请求 panic 不会拉垮整个服务，但客户端会收到 500 而非业务错误码；同时日志里会留下 `thread '<unnamed>' panicked at 'internal error: entered unreachable code'` 之类的堆栈。
- 群组邀请没有类似“群组管理员”的权限 fallback，所以非创建者删除群组邀请的正常路径根本不存在——要么是创建者本人删，要么就是 500 panic。

### 5.7 并发边界：签发随机码冲突有没有重试？**没有——应用层无重试，依赖 MongoDB _id 隐含唯一性**

签发流程 [crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L60-L84)：

```rust
let code = nanoid::nanoid!(8, &ALPHABET);  // 单次生成
let invite = ... ;
db.insert_invite(&invite).await?;          // 直接插入
```

没有 `loop`、没有 `retry`、没有对“duplicate key”错误的捕获与换码重试。

数据库层侧：

- `channel_invites` 集合初始化时只 `create_collection`，**没有单独的 `createIndexes` 调用**（[crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs](crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L39-L41)），但 `Invite` 序列化后 `code` 字段被 `#[serde(rename = "_id")]` 映射为 MongoDB 的 `_id`（[crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L17-L31)），而 MongoDB 对 `_id` 自带全局唯一索引，因此重复的 code 插入一定会报 duplicate key error。
- 应用层没有把 `E11000 duplicate key error` 翻译成“生成新 code 重试”，而是通过 `query!` 宏直接转成 `InternalError`（500）返回给客户端。

概率层面：54 字符 × 8 位 = 54^8 ≈ 7.2e13 空间，碰撞概率极低，所以实际上这个缺口只在极端规模或故意攻击时才会触发，但从工程上讲缺失了冲突重试机制。

---

## 6. 有效期校验（重要结论）

**本实现没有“有效期”概念**：

- `Invite` 模型无 `expires_at` / `created_at` 字段（见 [crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs#L11-L41)）。
- 数据库层无 TTL 索引、无定时清理；`crond` 守护进程的任务只有 `acks / file_deletion / prune_dangling_files / prune_members`（见 [crates/daemons/crond/src/tasks/](crates/daemons/crond/src/tasks)），**不含邀请码清理**。
- 核销路径 `join` / `as_invite` **不读取任何时间字段**，因此邀请码一经签发即永久有效，直到：
  - 被显式删除（`DELETE /invites/<code>`）；
  - 所属频道被删除（频道删除会级联清理其邀请，见 [crates/core/database/src/models/channels/ops/mongodb.rs](crates/core/database/src/models/channels/ops/mongodb.rs#L268-L274)）；
  - 所属服务器被删除（级联清理，见 [crates/core/database/src/models/servers/ops/mongodb.rs](crates/core/database/src/models/servers/ops/mongodb.rs#L231)）。

> 若业务需要“有效期校验”，当前代码无现成支撑，需要新增字段与核销时的时间比较逻辑。

---

## 7. 角色配置与权限映射

### 7.1 签发权限
- 需要 `ChannelPermission::InviteOthers`（[crates/core/permissions/src/models/channel.rs](crates/core/permissions/src/models/channel.rs#L69)）。
- 该权限包含在 `DEFAULT_PERMISSION` 中（[crates/core/permissions/src/models/channel.rs](crates/core/permissions/src/models/channel.rs#L130-L141)），即普通成员默认即可签发邀请码（除非服务器/频道覆写权限将其移除）。

### 7.2 核销后的角色（加入服务器）
- `Member::create` 使用 `..Default::default()`，**`roles` 为空 `vec![]`**（[crates/core/database/src/models/server_members/model.rs](crates/core/database/src/models/server_members/model.rs#L85-L98)）。
- 即新成员**不会被显式赋予任何角色**，其权限来自服务器的默认权限（`@everyone` 对应的 `default_permissions`）经 `calculate_channel_permissions` 计算得出。
- 加入后可见频道由 `ViewChannel` 权限过滤（[crates/core/database/src/models/server_members/model.rs](crates/core/database/src/models/server_members/model.rs#L132-L145)）。

### 7.3 删除权限
[crates/delta/src/routes/invites/invite_delete.rs](crates/delta/src/routes/invites/invite_delete.rs#L14-L35)：
- 邀请码 **创建者** 可直接删除自己的邀请；
- 否则需要 `ChannelPermission::ManageServer`（仅 `Server` 类型，群组邀请非创建者无法删除——`unreachable!`）。

### 7.4 列出服务器邀请
[crates/delta/src/routes/servers/invites_fetch.rs](crates/delta/src/routes/servers/invites_fetch.rs#L14-L30)：需要 `ManageServer`，调用 `fetch_invites_for_server`。

### 7.5 机器人邀请（`invite_bot`，与普通邀请码不同）
[crates/delta/src/routes/bots/invite.rs](crates/delta/src/routes/bots/invite.rs#L18-L64)：
- `Server` 目标：需要 `ManageServer`，走 `Member::create` 把 bot 用户加入服务器；
- `Group` 目标：需要 `InviteOthers`，走 `add_user_to_group`；
- 还会校验 bot 是否 `public` 或调用者是否为 bot owner（`BotIsPrivate`）。

---

## 8. 防滥用机制汇总

| # | 机制 | 位置 | 说明 |
|---|------|------|------|
| 1 | 机器人拦截 | [crates/delta/src/routes/channels/invite_create.rs](crates/delta/src/routes/channels/invite_create.rs#L23)、[crates/delta/src/routes/invites/invite_join.rs](crates/delta/src/routes/invites/invite_join.rs#L17) | `user.bot.is_some()` → `IsBot`，bot 不能签发/核销 |
| 2 | 服务器获取上限 | [crates/core/database/src/models/users/model.rs](crates/core/database/src/models/users/model.rs#L280-L289) | `can_acquire_server`：用户已加入服务器数 ≤ 限额，否则 `TooManyServers`。新用户 50，默认 100 |
| 3 | 新用户降额 | [crates/core/database/src/models/users/model.rs](crates/core/database/src/models/users/model.rs#L229-L242) | 注册后 `new_user_hours`（默认 72h，[crates/core/config/Revolt.toml](crates/core/config/Revolt.toml#L231)）内使用 `new_user` 限额 |
| 4 | 服务器计数口径 | [crates/core/database/src/models/server_members/ops/mongodb.rs](crates/core/database/src/models/server_members/ops/mongodb.rs#L236-L245) | 统计 `Member` 文档数且 `pending_deletion_at` 不存在 |
| 5 | 封禁拦截 | [crates/core/database/src/models/server_members/model.rs](crates/core/database/src/models/server_members/model.rs#L109-L111) | `fetch_ban` 命中 → `Banned` |
| 6 | 重复加入拦截 | [crates/core/database/src/models/server_members/model.rs](crates/core/database/src/models/server_members/model.rs#L113-L115)、[crates/core/database/src/models/channels/model.rs](crates/core/database/src/models/channels/model.rs#L348-L350) | `AlreadyInServer` / `AlreadyInGroup` |
| 7 | 群规模上限 | [crates/core/database/src/models/channels/model.rs](crates/core/database/src/models/channels/model.rs#L352-L357) | `group_size`（默认 100）→ `GroupTooLarge` |
| 8 | 权限门控 | 见第 7 节 | 签发需 `InviteOthers`；删除/列出需 `ManageServer` |
| 9 | discoverable 才能用服务器 ID 当邀请码 | [crates/core/database/src/util/reference.rs](crates/core/database/src/util/reference.rs#L50-L68) | 非可发现服务器用其 ID 解析邀请 → `NotFound` |
| 10 | 速率限制 | [crates/delta/src/util/ratelimits.rs](crates/delta/src/util/ratelimits.rs#L64-L80) | 见下文 |

### 8.1 服务器数量等于限额时是否放行？**是，等于时仍放行，会超限 1 个**

代码证据：

```rust
pub async fn can_acquire_server(&self, db: &Database) -> Result<()> {
    if db.fetch_server_count(&self.id).await? <= self.limits().await.servers {
        Ok(())
    } else {
        Err(create_error!(TooManyServers {
            max: self.limits().await.servers
        }))
    }
}
```

[crates/core/database/src/models/users/model.rs](crates/core/database/src/models/users/model.rs#L281-L289)

比较符是 **`<=`**，而非 `<`。因此：
- 限额 100，当前已加入 100 → `100 <= 100` → `Ok(())`，**放行**，加入后变为 101；
- 当前已加入 101 → `101 <= 100` → `Err(TooManyServers)`，拒绝。

> 实际效果是允许“限额 + 1”，属于 off-by-one 边界。若预期“等于限额时拒绝”，应改为 `<`。

### 8.2 速率限制细节
速率分桶在 [crates/delta/src/util/ratelimits.rs](crates/delta/src/util/ratelimits.rs#L6-L80)：

- `POST /channels/<id>/invites`（签发）：路由段为 `channels`，落入 `"channels"` 桶，限 **15**/窗口，按频道 ID 维度限流（[crates/delta/src/util/ratelimits.rs](crates/delta/src/util/ratelimits.rs#L37-L45)）。
- `GET/POST/DELETE /invites/<code>`（查询/核销/删除）：路由段 `invites` **无专属桶**，落入默认 `_ => ("any", None)`，限 **20**/窗口（[crates/delta/src/util/ratelimits.rs](crates/delta/src/util/ratelimits.rs#L57)、[crates/delta/src/util/ratelimits.rs](crates/delta/src/util/ratelimits.rs#L78)）。

> 注意：`/invites/*` 没有按邀请码或按用户细分桶，仅有一个全局 `any` 桶（20/窗口）兜底。

### 8.3 潜在滥用面（当前未覆盖）
- **无使用次数上限**：邀请码可被无限次核销（无 `max_uses`）。
- **无有效期**：见第 6 节。
- **无签发数量上限**：见第 3 节。
- **等于限额时仍放行**：见 8.1。
- **服务器数量检查与成员写入非原子**：见 5.4，并发核销可绕过 `can_acquire_server` 限额。
- **群组并发加入可能产生重复 recipient 并绕过群规模上限**：见 5.5，内存检查 + `$push` 无原子性。
- **群组邀请非创建者删除会 500 panic**：见 5.6，`unreachable!()` 直接 panic 而非返回业务错误。
- **邀请码随机码冲突无重试**：见 5.7，应用层不捕获 duplicate key 并换码，冲突时返回 500。
- 因此对“恶意扩散邀请码”的防护主要依赖：速率限制、`ManageServer` 持有者手动删除、服务器封禁（`Banned`）以及 `can_acquire_server` 对加入方数量的限制，而非邀请码本身的额度/时效约束。

---

## 9. 关键文件索引

- 模型：[crates/core/database/src/models/channel_invites/model.rs](crates/core/database/src/models/channel_invites/model.rs)
- DB trait / MongoDB / 参考：[crates/core/database/src/models/channel_invites/ops.rs](crates/core/database/src/models/channel_invites/ops.rs) · [crates/core/database/src/models/channel_invites/ops/mongodb.rs](crates/core/database/src/models/channel_invites/ops/mongodb.rs) · [crates/core/database/src/models/channel_invites/ops/reference.rs](crates/core/database/src/models/channel_invites/ops/reference.rs)
- API 模型：[crates/core/models/src/v0/channel_invites.rs](crates/core/models/src/v0/channel_invites.rs)
- 路由：[crates/delta/src/routes/channels/invite_create.rs](crates/delta/src/routes/channels/invite_create.rs) · [crates/delta/src/routes/invites/invite_fetch.rs](crates/delta/src/routes/invites/invite_fetch.rs) · [crates/delta/src/routes/invites/invite_join.rs](crates/delta/src/routes/invites/invite_join.rs) · [crates/delta/src/routes/invites/invite_delete.rs](crates/delta/src/routes/invites/invite_delete.rs) · [crates/delta/src/routes/servers/invites_fetch.rs](crates/delta/src/routes/servers/invites_fetch.rs) · [crates/delta/src/routes/bots/invite.rs](crates/delta/src/routes/bots/invite.rs)
- 引用解析：[crates/core/database/src/util/reference.rs](crates/core/database/src/util/reference.rs)
- 权限：[crates/core/permissions/src/models/channel.rs](crates/core/permissions/src/models/channel.rs)
- 成员创建/默认值：[crates/core/database/src/models/server_members/model.rs](crates/core/database/src/models/server_members/model.rs)
- 群组加入：[crates/core/database/src/models/channels/model.rs](crates/core/database/src/models/channels/model.rs)（`add_user_to_group`）
- 用户限额：[crates/core/database/src/models/users/model.rs](crates/core/database/src/models/users/model.rs)（`can_acquire_server` / `limits`）
- 速率限制：[crates/delta/src/util/ratelimits.rs](crates/delta/src/util/ratelimits.rs)
- 配置默认值：[crates/core/config/Revolt.toml](crates/core/config/Revolt.toml) · 配置结构 [crates/core/config/src/lib.rs](crates/core/config/src/lib.rs)
