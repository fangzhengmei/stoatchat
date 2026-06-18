# 邀请码（Channel Invite）从签发到核销的代码运转梳理

> 仓库：Revolt 后端（Rust + Rocket + MongoDB）
> 范围：`channel_invites`（频道邀请码），即“邀请某人加入服务器/群组”的实体。注意：本仓库中“邀请码”指频道邀请码，与注册时 `invite_only` 的“注册邀请”是两套独立概念。

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

路由挂载见 [routes/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/mod.rs#L21-L44)：`/channels`、`/invites`、`/servers`、`/bots` 分别挂载各子路由。

| 操作 | 方法 + 路径 | 处理函数 | 文件 |
|------|------------|---------|------|
| 签发 | `POST /channels/<target>/invites` | `create_invite` | [invite_create.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/channels/invite_create.rs#L17-L37) |
| 查询 | `GET /invites/<target>` | `fetch` | [invite_fetch.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_fetch.rs#L10-L71) |
| 核销（加入） | `POST /invites/<target>` | `join` | [invite_join.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_join.rs#L10-L49) |
| 删除 | `DELETE /invites/<target>` | `delete` | [invite_delete.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_delete.rs#L14-L35) |
| 列出服务器全部邀请 | `GET /servers/<target>/invites` | `invites` | [invites_fetch.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/servers/invites_fetch.rs#L14-L30) |
| 邀请机器人加入 | `POST /bots/<target>/invite` | `invite_bot` | [bots/invite.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/bots/invite.rs#L18-L64) |

---

## 2. 数据模型

邀请码实体定义在 [channel_invites/model.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/model.rs#L11-L41)：

```rust
pub enum Invite {
    Server { code, server, creator, channel },   // 指向服务器内某频道
    Group { code, creator, channel },            // 指向群组频道
}
```

- `code`：8 位 nanoid，字母表为 54 字符的 Crockford 风格（去除易混淆的 `I/L/O/U` 等），见 [model.rs#L5-L9](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/model.rs#L5-L9) 与生成处 [model.rs#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/model.rs#L65) `nanoid::nanoid!(8, &ALPHABET)`。
- **没有 `uses`、`max_uses`、`expires_at`、`created_at` 等字段**——邀请码不记次数、不带时间戳。
- 对外 API 模型（含 `InviteResponse`/`InviteJoinResponse`）见 [models/v0/channel_invites.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/models/src/v0/channel_invites.rs#L1-L103)。

数据库层（MongoDB 集合名 `channel_invites`）：
- 抽象 trait：[ops.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/ops.rs#L10-L21) — `insert_invite` / `fetch_invite` / `fetch_invites_for_server` / `delete_invite`。
- MongoDB 实现：[ops/mongodb.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/ops/mongodb.rs#L12-L46)。
- 内存参考实现（测试用）：[ops/reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/ops/reference.rs#L9-L51)。

---

## 3. 签发（Issue）

入口 [create_invite](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/channels/invite_create.rs#L17-L37)：

1. **机器人拦截**：`if user.bot.is_some()` → `IsBot`。
2. **取频道**：`target.as_channel(db)`。
3. **权限校验**：构造 `DatabasePermissionQuery` 并调用 `calculate_channel_permissions`，再 `throw_if_lacking_channel_permission(ChannelPermission::InviteOthers)`。`InviteOthers = 1 << 25`，定义在 [channel.rs#L69](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/permissions/src/models/channel.rs#L69)。
4. **生成并落库**：`Invite::create_channel_invite`（[model.rs#L60-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/model.rs#L60-L85)）：
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

## 4. 核销（Redeem / Join）

入口 [join](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_join.rs#L10-L49)：

1. **机器人拦截**：`IsBot`。
2. **服务器获取额度校验**：`user.can_acquire_server(db)`（防滥用，见第 6 节）。
3. **解析邀请码**：`target.as_invite(db)`（见下文“解析”）。
4. **分支处理**：
   - `Invite::Server` → `db.fetch_server` → `Member::create(db, &server, &user, None)`。
   - `Invite::Group` → `db.fetch_channel` → `channel.add_user_to_group(db, amqp, &user, creator)`。

### 4.1 邀请码解析（`as_invite`）

实现在 [reference.rs#L49-L69](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/util/reference.rs#L49-L69)：

- 若 `target` 是合法 ULID（即服务器 ID）：
  - 取该 server，**必须 `server.discoverable == true`**，否则 `NotFound`；
  - 构造一个“虚拟邀请”指向服务器的第一个频道（`server.channels.into_iter().next()`），`creator = server.owner`。
- 否则按普通邀请码 `db.fetch_invite(code)` 查询。

> 同等逻辑也存在于 `Invite::find`（[model.rs#L88-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/model.rs#L88-L105)）。这意味着可发现（discoverable）的服务器可被直接用其 ID 当作邀请码使用。

### 4.2 加入服务器：`Member::create`

[server_members/model.rs#L101-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/model.rs#L101-L146)：

1. **封禁检查**：`db.fetch_ban(&server.id, &user.id)` 命中 → `Banned`。
2. **已在服务器检查**：`db.fetch_member(...)` 命中 → `AlreadyInServer`。
3. **创建成员**：`Member { ..Default::default() }`，其中默认值见 [server_members/model.rs#L85-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/model.rs#L85-L98)：`roles: vec![]`、`joined_at: now`、`timeout: None`。随后 `insert_or_merge_member`。
4. **可见频道过滤**：对每个频道计算权限，仅保留调用方能 `ViewChannel` 的频道。
5. **事件广播**：`ServerMemberJoin`（public）与 `ServerCreate`（private 给该用户）；若配置了 `system_messages.user_joined` 则发系统消息。

### 4.3 加入群组：`add_user_to_group`

[channels/model.rs#L339-L401](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channels/model.rs#L339-L401)：

1. **已在群组检查**：`recipients.contains(user.id)` → `AlreadyInGroup`。
2. **群规模上限**：`recipients.len() >= config.features.limits.global.group_size` → `GroupTooLarge`（默认 100，见 [Revolt.toml#L222](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/config/Revolt.toml#L222)）。
3. 加入 recipients、写库、发 `ChannelGroupJoin` 与 `UserAdded` 系统消息。

---

## 5. 有效期校验（重要结论）

**本实现没有“有效期”概念**：

- `Invite` 模型无 `expires_at` / `created_at` 字段（见 [model.rs#L11-L41](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/model.rs#L11-L41)）。
- 数据库层无 TTL 索引、无定时清理；`crond` 守护进程的任务只有 `acks / file_deletion / prune_dangling_files / prune_members`（见 [daemons/crond/src/tasks/](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/daemons/crond/src/tasks)），**不含邀请码清理**。
- 核销路径 `join` / `as_invite` **不读取任何时间字段**，因此邀请码一经签发即永久有效，直到：
  - 被显式删除（`DELETE /invites/<code>`）；
  - 所属频道被删除（频道删除会级联清理其邀请，见 [channels/ops/mongodb.rs#L268-L274](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channels/ops/mongodb.rs#L268-L274)）；
  - 所属服务器被删除（级联清理，见 [servers/ops/mongodb.rs#L231](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/servers/ops/mongodb.rs#L231)）。

> 若业务需要“有效期校验”，当前代码无现成支撑，需要新增字段与核销时的时间比较逻辑。

---

## 6. 角色配置与权限映射

### 6.1 签发权限
- 需要 `ChannelPermission::InviteOthers`（[channel.rs#L69](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/permissions/src/models/channel.rs#L69)）。
- 该权限包含在 `DEFAULT_PERMISSION` 中（[channel.rs#L130-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/permissions/src/models/channel.rs#L130-L141)），即普通成员默认即可签发邀请码（除非服务器/频道覆写权限将其移除）。

### 6.2 核销后的角色（加入服务器）
- `Member::create` 使用 `..Default::default()`，**`roles` 为空 `vec![]`**（[server_members/model.rs#L85-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/model.rs#L85-L98)）。
- 即新成员**不会被显式赋予任何角色**，其权限来自服务器的默认权限（`@everyone` 对应的 `default_permissions`）经 `calculate_channel_permissions` 计算得出。
- 加入后可见频道由 `ViewChannel` 权限过滤（[server_members/model.rs#L132-L145](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/model.rs#L132-L145)）。

### 6.3 删除权限
[invite_delete.rs#L14-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_delete.rs#L14-L35)：
- 邀请码 **创建者** 可直接删除自己的邀请；
- 否则需要 `ChannelPermission::ManageServer`（仅 `Server` 类型，群组邀请非创建者无法删除——`unreachable!`）。

### 6.4 列出服务器邀请
[invites_fetch.rs#L14-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/servers/invites_fetch.rs#L14-L30)：需要 `ManageServer`，调用 `fetch_invites_for_server`。

### 6.5 机器人邀请（`invite_bot`，与普通邀请码不同）
[bots/invite.rs#L18-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/bots/invite.rs#L18-L64)：
- `Server` 目标：需要 `ManageServer`，走 `Member::create` 把 bot 用户加入服务器；
- `Group` 目标：需要 `InviteOthers`，走 `add_user_to_group`；
- 还会校验 bot 是否 `public` 或调用者是否为 bot owner（`BotIsPrivate`）。

---

## 7. 防滥用机制汇总

| # | 机制 | 位置 | 说明 |
|---|------|------|------|
| 1 | 机器人拦截 | [invite_create.rs#L23](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/channels/invite_create.rs#L23)、[invite_join.rs#L17](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_join.rs#L17) | `user.bot.is_some()` → `IsBot`，bot 不能签发/核销 |
| 2 | 服务器获取上限 | [users/model.rs#L280-L289](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/users/model.rs#L280-L289) | `can_acquire_server`：用户已加入服务器数 ≤ 限额，否则 `TooManyServers`。新用户 50，默认 100 |
| 3 | 新用户降额 | [users/model.rs#L229-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/users/model.rs#L229-L242) | 注册后 `new_user_hours`（默认 72h，[Revolt.toml#L231](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/config/Revolt.toml#L231)）内使用 `new_user` 限额 |
| 4 | 服务器计数口径 | [server_members/ops/mongodb.rs#L236-L245](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/ops/mongodb.rs#L236-L245) | 统计 `Member` 文档数且 `pending_deletion_at` 不存在 |
| 5 | 封禁拦截 | [server_members/model.rs#L109-L111](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/model.rs#L109-L111) | `fetch_ban` 命中 → `Banned` |
| 6 | 重复加入拦截 | [server_members/model.rs#L113-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/model.rs#L113-L115)、[channels/model.rs#L348-L350](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channels/model.rs#L348-L350) | `AlreadyInServer` / `AlreadyInGroup` |
| 7 | 群规模上限 | [channels/model.rs#L352-L357](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channels/model.rs#L352-L357) | `group_size`（默认 100）→ `GroupTooLarge` |
| 8 | 权限门控 | 见第 6 节 | 签发需 `InviteOthers`；删除/列出需 `ManageServer` |
| 9 | discoverable 才能用服务器 ID 当邀请码 | [reference.rs#L50-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/util/reference.rs#L50-L68) | 非可发现服务器用其 ID 解析邀请 → `NotFound` |
| 10 | 速率限制 | [util/ratelimits.rs#L64-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/util/ratelimits.rs#L64-L80) | 见下文 |

### 7.1 速率限制细节
速率分桶在 [DeltaRatelimits](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/util/ratelimits.rs#L6-L80)：

- `POST /channels/<id>/invites`（签发）：路由段为 `channels`，落入 `"channels"` 桶，限 **15**/窗口，按频道 ID 维度限流（[ratelimits.rs#L37-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/util/ratelimits.rs#L37-L45)）。
- `GET/POST/DELETE /invites/<code>`（查询/核销/删除）：路由段 `invites` **无专属桶**，落入默认 `_ => ("any", None)`，限 **20**/窗口（[ratelimits.rs#L57](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/util/ratelimits.rs#L57)、[ratelimits.rs#L78](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/util/ratelimits.rs#L78)）。

> 注意：`/invites/*` 没有按邀请码或按用户细分桶，仅有一个全局 `any` 桶（20/窗口）兜底。

### 7.2 潜在滥用面（当前未覆盖）
- **无使用次数上限**：邀请码可被无限次核销（无 `max_uses`）。
- **无有效期**：见第 5 节。
- **无签发数量上限**：见第 3 节。
- 因此对“恶意扩散邀请码”的防护主要依赖：速率限制、`ManageServer` 持有者手动删除、服务器封禁（`Banned`）以及 `can_acquire_server` 对加入方数量的限制，而非邀请码本身的额度/时效约束。

---

## 8. 关键文件索引

- 模型：[channel_invites/model.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/model.rs)
- DB trait / MongoDB / 参考：[ops.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/ops.rs) · [ops/mongodb.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/ops/mongodb.rs) · [ops/reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channel_invites/ops/reference.rs)
- API 模型：[models/v0/channel_invites.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/models/src/v0/channel_invites.rs)
- 路由：[channels/invite_create.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/channels/invite_create.rs) · [invites/invite_fetch.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_fetch.rs) · [invites/invite_join.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_join.rs) · [invites/invite_delete.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/invites/invite_delete.rs) · [servers/invites_fetch.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/servers/invites_fetch.rs) · [bots/invite.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/routes/bots/invite.rs)
- 引用解析：[util/reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/util/reference.rs)
- 权限：[permissions/models/channel.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/permissions/src/models/channel.rs)
- 成员创建/默认值：[server_members/model.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/server_members/model.rs)
- 群组加入：[channels/model.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/channels/model.rs)（`add_user_to_group`）
- 用户限额：[users/model.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/database/src/models/users/model.rs)（`can_acquire_server` / `limits`）
- 速率限制：[delta/util/ratelimits.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/delta/src/util/ratelimits.rs)
- 配置默认值：[Revolt.toml](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/config/Revolt.toml) · 配置结构 [config/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/39-backend/crates/core/config/src/lib.rs)
