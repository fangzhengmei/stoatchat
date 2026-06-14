# 服务器权限模型深度分析

本文基于代码对 Revolt 后端的多层权限模型进行完整解读，涵盖权限位定义、全局权限计算、频道级覆盖优先级判定、批量权限计算入口，以及角色删除后权限回收的连锁影响。

---

## 一、权限位体系总览

权限系统由两个独立的枚举驱动，均采用位标志（bitflags）方式编码为 `u64`。

### 1.1 用户权限 `UserPermission`

定义于 [user.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/user.rs#L19-L24)，占低 4 位：

| 位 | 权限 | 含义 |
|---|---|---|
| `1 << 0` | `Access` | 可访问该用户 |
| `1 << 1` | `ViewProfile` | 可查看该用户资料 |
| `1 << 2` | `SendMessage` | 可向该用户发消息 |
| `1 << 3` | `Invite` | 可邀请该用户 |

`UserPermission` 仅用于非服务器场景（DM、用户间交互），与服务器权限无关。

### 1.2 频道权限 `ChannelPermission`

定义于 [channel.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/channel.rs#L22-L113)，是服务器 + 频道权限的核心位标志，划分为四个区段：

**通用管理权限（bit 0–4）**

| 位 | 权限 | 含义 |
|---|---|---|
| `1 << 0` | `ManageChannel` | 管理频道 |
| `1 << 1` | `ManageServer` | 管理服务器 |
| `1 << 2` | `ManagePermissions` | 管理权限 |
| `1 << 3` | `ManageRole` | 管理角色 |
| `1 << 4` | `ManageCustomisation` | 管理自定义（含表情） |

**成员管理权限（bit 6–13）**

| 位 | 权限 | 含义 |
|---|---|---|
| `1 << 6` | `KickMembers` | 踢出低 rank 成员 |
| `1 << 7` | `BanMembers` | 封禁低 rank 成员 |
| `1 << 8` | `TimeoutMembers` | 对低 rank 成员禁言 |
| `1 << 9` | `AssignRoles` | 分配角色给低 rank 成员 |
| `1 << 10` | `ChangeNickname` | 修改自己的昵称 |
| `1 << 11` | `ManageNicknames` | 修改低 rank 成员昵称 |
| `1 << 12` | `ChangeAvatar` | 修改自己的头像 |
| `1 << 13` | `RemoveAvatars` | 移除低 rank 成员头像 |

**频道交互权限（bit 20–29, 37–39）**

| 位 | 权限 | 含义 |
|---|---|---|
| `1 << 20` | `ViewChannel` | 查看频道 |
| `1 << 21` | `ReadMessageHistory` | 读取历史消息 |
| `1 << 22` | `SendMessage` | 发送消息 |
| `1 << 23` | `ManageMessages` | 管理消息 |
| `1 << 24` | `ManageWebhooks` | 管理 Webhook |
| `1 << 25` | `InviteOthers` | 邀请他人 |
| `1 << 26` | `SendEmbeds` | 发送嵌入内容 |
| `1 << 27` | `UploadFiles` | 上传文件 |
| `1 << 28` | `Masquerade` | 伪装消息（自定义昵称/头像） |
| `1 << 29` | `React` | 使用表情回应 |
| `1 << 37` | `MentionEveryone` | @所有人 |
| `1 << 38` | `MentionRoles` | @角色 |
| `1 << 39` | `BypassSlowmode` | 绕过慢速模式 |

**语音权限（bit 30–36）**

| 位 | 权限 | 含义 |
|---|---|---|
| `1 << 30` | `Connect` | 加入语音频道 |
| `1 << 31` | `Speak` | 语音说话 |
| `1 << 32` | `Video` | 共享视频 |
| `1 << 33` | `MuteMembers` | 静音低 rank 成员 |
| `1 << 34` | `DeafenMembers` | 停音低 rank 成员 |
| `1 << 35` | `MoveMembers` | 移动成员 |
| `1 << 36` | `Listen` | 收听他人 |

**特殊值**

| 值 | 含义 |
|---|---|
| `GrantAllSafe = 0x000F_FFFF_FFFF_FFFF` | 安全全权（排除高 4 位保留区） |
| `GrantAll = u64::MAX` | 绝对全权 |

---

## 二、Override 结构——权限覆盖的基本单位

[Override](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L8-L13) 是整个权限系统的原子操作单元：

```rust
pub struct Override {
    pub allow: u64,   // 允许的权限位
    pub deny: u64,    // 拒绝的权限位
}
```

`Override` 的语义是"对当前权限值施加一次修改"：先 `allow`（按位 OR），再 `deny`（按位 AND NOT）。对应到 [PermissionValue::apply](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L24-L27)：

```rust
pub fn apply(&mut self, v: Override) {
    self.allow(v.allow);   // self.0 |= v.allow
    self.revoke(v.deny);   // self.0 &= !v.deny
}
```

**关键语义**：`deny` 优先于 `allow`。当同一个 `Override` 中 `allow` 和 `deny` 有重叠位时，`deny` 会胜出（因为先 OR 再 AND NOT，deny 在后执行）。但在多角色叠加场景中，高 rank 角色的 `allow` 可以覆盖低 rank 角色的 `deny`（详见第四节）。

数据库存储层使用紧凑的 [OverrideField](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L52-L57)（字段名 `a` / `d`），与 `Override` 通过 `From` 互转。

---

## 三、全局权限计算——服务器层

入口函数为 [calculate_server_permissions](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/impl.rs#L49-L78)。

### 3.1 计算流程

```
① 特权用户 / 服务器拥有者 → 直接返回 GrantAllSafe
② 非成员 → 返回 0
③ 取服务器默认权限 (server.default_permissions) 作为基底
④ 按角色 rank 从高到低排序，依次 apply 角色的 Override
⑤ 检查成员级语音开关 (can_publish / can_receive)
⑥ 检查 Timeout 状态
```

### 3.2 角色覆盖的排序与叠加

[get_our_server_role_overrides](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/permissions.rs#L154-L177) 的实现逻辑：

1. 从成员的 `roles` 列表中筛选出服务器 `roles` HashMap 中存在的角色
2. 将每个角色的 `permissions: OverrideField` 转换为 `Override`
3. 按 `role.rank` **降序**排序（rank 数值越小 → 优先级越高 → 排在后面）
4. 返回排序后的 `Vec<Override>` 供调用方依次 `apply`

**rank 的含义**：`rank` 越小，角色优先级越高。排序后从高 rank（低优先级）到低 rank（高优先级）依次 apply，这意味着**低 rank（高优先级）角色的 allow/deny 会覆盖高 rank 角色的结果**。

### 3.3 默认服务器权限

[DEFAULT_PERMISSION_SERVER](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/channel.rs#L149-L155) 在创建服务器时作为 `default_permissions` 写入：

```
DEFAULT_PERMISSION_VIEW_ONLY + SendMessage + InviteOthers + SendEmbeds + UploadFiles
+ Connect + Speak + Listen + Video + React + ChangeNickname + ChangeAvatar
```

### 3.4 语音开关

成员级 `can_publish` / `can_receive` 字段（默认 `true`）在服务器权限计算后额外检查：
- `can_publish = false` → 收回 `Speak` + `Video`
- `can_receive = false` → 收回 `Listen`

### 3.5 Timeout 限制

[ALLOW_IN_TIMEOUT](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/channel.rs#L124-L125) 仅保留 `ViewChannel + ReadMessageHistory`：

```rust
pub static ALLOW_IN_TIMEOUT: Lazy<u64> =
    Lazy::new(|| ChannelPermission::ViewChannel + ChannelPermission::ReadMessageHistory);
```

被 Timeout 的成员执行 `permissions.restrict(*ALLOW_IN_TIMEOUT)`，即用 AND 操作将权限限制到只有查看能力。

---

## 四、频道级权限覆盖——优先级判定

入口函数为 [calculate_channel_permissions](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/impl.rs#L81-L147)。

### 4.1 按频道类型的分支逻辑

| 频道类型 | 权限来源 |
|---|---|
| `SavedMessages` | 仅拥有者获得 `GrantAllSafe`，其余为 0 |
| `DirectMessage` | 基于用户间关系计算 `UserPermission`，再叠加 `DEFAULT_PERMISSION_DIRECT_MESSAGE` 或 `DEFAULT_PERMISSION_VIEW_ONLY` |
| `Group` | 拥有者获得 `GrantAllSafe`；参与者获得 `DEFAULT_PERMISSION_VIEW_ONLY | channel.permissions` |
| `ServerChannel` | **三层叠加**（详见下文） |

### 4.2 ServerChannel 的三层权限叠加

这是最核心的权限计算路径，代码在 [impl.rs#L119-L144](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/impl.rs#L119-L144)：

```
第一层：服务器基础权限
  └─ calculate_server_permissions() 的完整结果

第二层：频道默认权限覆盖
  └─ channel.default_permissions (OverrideField → Override)
  └─ apply 到第一层结果

第三层：频道角色权限覆盖
  └─ 按 role.rank 降序排列的角色 Override 列表
  └─ 依次 apply 到第二层结果

后处理：
  └─ Timeout 限制（同服务器层）
  └─ ViewChannel 检查：若无 ViewChannel 权限，直接 revoke_all()
```

### 4.3 频道角色覆盖的选取逻辑

[get_our_channel_role_overrides](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/permissions.rs#L252-L290)：

1. 从 `channel.role_permissions` 中筛选成员拥有的角色 ID
2. **关键步骤**：还要在 `server.roles` 中查到该角色才能使用——如果角色已被删除，`server.roles.get(id)` 返回 `None`，该条目会被 `filter_map` 过滤掉
3. 按 `role.rank` 降序排列后依次 apply

### 4.4 完整优先级链（从低到高）

```
server.default_permissions         ← 最底层：所有人都有
  ↓ apply
角色 Override (高 rank → 低 rank)  ← 服务器级角色叠加
  ↓ apply
channel.default_permissions        ← 频道默认覆盖
  ↓ apply
频道角色 Override (高 rank → 低 rank) ← 频道级角色叠加
  ↓ 后处理
Timeout / 语音开关 / ViewChannel 检查
```

**核心原则**：后执行的 `apply` 覆盖先执行的结果。同一层内，低 rank（高优先级）角色的 allow 可以恢复被高 rank 角色 deny 掉的权限，反之亦然。

### 4.5 测试用例验证

[test.rs#L205-L315](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/test.rs#L205-L315) 中的 `validate_server_permissions` 测试精确展示了三层叠加：

- 服务器默认：`ViewChannel + SendMessage + ReadMessageHistory`
- 角色覆盖：allow `UploadFiles + React`，deny `ReadMessageHistory`
  → 结果：`ViewChannel + SendMessage + UploadFiles + React`
- 频道默认覆盖：deny `SendMessage`
  → 结果：`ViewChannel + UploadFiles + React`
- 频道角色覆盖：deny `React`
  → 结果：`ViewChannel + UploadFiles` ✅

---

## 五、批量权限计算入口

### 5.1 单用户入口：DatabasePermissionQuery

[DatabasePermissionQuery](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/permissions.rs#L12-L26) 是面向单用户的 Builder 模式权限计算器：

```rust
let mut query = DatabasePermissionQuery::new(db, &user)
    .server(&server)
    .member(&member)
    .channel(&channel);
let perms = calculate_channel_permissions(&mut query).await;
```

它实现了 [PermissionQuery](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/trait.rs#L4-L73) trait，将数据库查询与纯逻辑解耦：

- 数据层（`DatabasePermissionQuery`）负责从 MongoDB 获取 User / Server / Member / Channel 数据
- 逻辑层（`calculate_*` 函数）只依赖 trait 方法，完全无数据库依赖

### 5.2 批量入口：BulkDatabasePermissionQuery

[BulkDatabasePermissionQuery](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L11-L25) 用于一次计算多个成员在某个频道上的权限，核心场景是判断"哪些成员能看到这个频道"。

核心方法 [calculate_members_permissions](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L186-L313)：

1. 解构频道数据：`id`, `role_permissions`, `default_permissions`
2. 确保用户列表和成员列表都存在（缺失则从数据库批量拉取）
3. 对每个用户：
   - 非成员 → 0
   - 特权用户 / 服务器拥有者 → `GrantAllSafe`
   - 调用 [calculate_server_permissions](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L316-L345)（同步版本，直接操作数据而非 trait）
   - apply 频道默认权限
   - 按 rank 排序后 apply 频道角色覆盖
   - 检查 Timeout

典型调用：

```rust
BulkDatabasePermissionQuery::new(db, server)
    .channel(&channel)
    .members(&members)
    .members_can_see_channel()
    .await
// → HashMap<String, bool>：每个成员是否能看到该频道
```

### 5.3 权限验证辅助

[PermissionValue](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L14-L114) 提供了两个关键验证方法：

- **throw_if_lacking_channel_permission**：检查是否拥有指定权限，缺失则抛出 `MissingPermission` 错误
- **throw_permission_override**：验证用户能否将权限授予他人。逻辑是：
  - 新增的 allow（`~旧.allow & 新.allow`）必须自己拥有
  - 减少的 deny（`旧.deny & ~新.deny`）必须自己拥有
  - 确保用户不能把自己没有的权限授予他人

---

## 六、角色删除后的权限回收——连锁影响

### 6.1 删除入口

[roles_delete.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_delete.rs#L16-L49) 是角色删除的 HTTP 入口：

```rust
pub async fn delete(db, user, target, role_id, voice_client) -> Result<EmptyResponse> {
    // 1. 权限校验：需要 ManageRole 权限
    // 2. rank 校验：只能删除 rank 低于自己的角色
    // 3. 从 server.roles 中移除
    // 4. 调用 role.delete(db, &server.id)
    // 5. 遍历所有频道，同步语音权限
}
```

### 6.2 数据库层连锁操作

[delete_role](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/servers/ops/mongodb.rs#L113-L156) 执行三步原子操作：

**第一步：从所有成员中移除该角色**

```javascript
// MongoDB: server_members 集合
db.server_members.updateMany(
    { "_id.server": server_id },
    { "$pull": { "roles": role_id } }
)
```

每个成员的 `roles: Vec<String>` 中不再包含此角色 ID。这意味着：
- 该角色在服务器级权限计算中不再参与 `get_our_server_role_overrides`（因为 `member_roles.contains(id)` 为 false）
- 该角色在频道级权限计算中不再参与 `get_our_channel_role_overrides`（同理）

**第二步：从所有频道中移除该角色的权限覆盖**

```javascript
// MongoDB: channels 集合
db.channels.updateOne(
    { "server": server_id },  // 注意：只更新了第一个匹配的频道！
    { "$unset": { "role_permissions.<role_id>": 1 } }
)
```

> ⚠️ **潜在缺陷**：此处使用的是 `update_one` 而非 `update_many`，意味着只有第一个频道会被清理该角色的 `role_permissions` 条目。其他频道中该角色的覆盖条目将残留在数据库中，成为"幽灵覆盖"。

**第三步：从服务器中移除角色定义**

```javascript
// MongoDB: servers 集合
db.servers.updateOne(
    { "_id": server_id },
    { "$unset": { "roles.<role_id>": 1 } }
)
```

### 6.3 "幽灵覆盖"问题分析

由于第二步只清理了第一个频道，其他频道中 `channel.role_permissions` 里可能仍残留已删除角色的覆盖条目。然而，这不会导致运行时错误，原因是：

**权限计算时的防御性过滤**——[get_our_channel_role_overrides](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/permissions.rs#L269-L277)：

```rust
.filter_map(|(id, permission)| {
    server.roles.get(id).map(|role| {
        let v: Override = (*permission).into();
        (role.rank, v)
    })
})
```

`server.roles.get(id)` 在角色被删除后返回 `None`，`filter_map` 会丢弃该条目。因此即使频道数据库中残留覆盖数据，运行时计算也不会用到它。

**但存在隐患**：
1. 数据库中存在无效数据，增加存储和理解成本
2. 如果未来以相同 ID 重新创建角色，残留的覆盖条目会意外生效
3. 成员 `roles` 列表中也会残留（第一步用了 `updateMany` 是正确的，但需确认所有成员都被清理）

### 6.4 语音权限同步

角色删除后，路由层遍历服务器所有频道，调用 [sync_voice_permissions](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/voice/mod.rs#L441-L466)：

```rust
for channel_id in &server.channels {
    let channel = Reference::from_unchecked(channel_id).as_channel(db).await?;
    sync_voice_permissions(db, voice_client, &channel, Some(&server), Some(&role_id)).await?;
}
```

这确保了正在语音频道中的成员在角色被删除后，其语音权限（Connect / Speak / Video 等）立即被重新计算并同步到 LiveKit 语音服务器。

### 6.5 成员 ranking 的变化

[Member::get_ranking](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/server_members/model.rs#L243-L254) 取成员所有角色中的最小 rank 值（最高优先级）：

```rust
pub fn get_ranking(&self, server: &Server) -> i64 {
    let mut value = i64::MAX;
    for role in &self.roles {
        if let Some(role) = server.roles.get(role) {
            if role.rank < value {
                value = role.rank;
            }
        }
    }
    value
}
```

角色删除后，如果该角色恰好是某成员的最高优先级角色，该成员的 ranking 会升高（数值变大 → 优先级降低），可能导致：
- 无法管理原来能管理的低 rank 成员
- 无法操作原来能操作的频道权限
- 踢人/封禁等操作的"低于自己 ranking"的范围缩小

### 6.6 事件通知

[Role::delete](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/servers/model.rs#L381-L391) 会发布 `ServerRoleDelete` 事件：

```rust
EventV1::ServerRoleDelete {
    id: server_id.to_string(),
    role_id: self.id.clone(),
}
.p(server_id.to_string())
.await;
```

客户端收到此事件后应本地移除该角色的 UI 显示和缓存，但**不会自动重新计算权限**——客户端通常需要在下次操作时才会触发权限重新计算。

### 6.7 连锁影响总结图

```
角色删除
  │
  ├─→ server.roles 移除角色定义
  │     └─→ get_our_server_role_overrides 不再包含此角色
  │     └─→ get_our_channel_role_overrides 不再包含此角色
  │     └─→ Member::get_ranking 忽略此角色
  │
  ├─→ server_members.$pull roles
  │     └─→ 成员不再持有此角色
  │     └─→ 权限计算时 member_roles.contains(id) = false
  │     └─→ 成员 ranking 可能升高（优先级降低）
  │
  ├─→ channels.$unset role_permissions (仅第一个频道！⚠️)
  │     └─→ 其他频道残留覆盖数据（运行时被 filter_map 过滤）
  │
  ├─→ sync_voice_permissions 遍历所有频道
  │     └─→ 重算语音权限并同步到 LiveKit
  │
  └─→ ServerRoleDelete 事件广播
        └─→ 客户端移除角色 UI
```

---

## 七、权限设置时的安全校验

### 7.1 服务器级角色权限设置

[permissions_set.rs (server)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/permissions_set.rs#L16-L61)：

1. 需要 `ManagePermissions` 权限
2. 不能修改 rank ≥ 自己 ranking 的角色
3. 通过 `throw_permission_override` 确保不能授予自己没有的权限

### 7.2 频道级角色权限设置

[permissions_set.rs (channel)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/channels/permissions_set.rs#L15-L56)：

1. 需要 `ManagePermissions` 权限
2. 不能修改 rank ≥ 自己 ranking 的角色
3. 同样通过 `throw_permission_override` 校验

### 7.3 默认权限设置

- 服务器默认权限：[permissions_set_default.rs (server)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/permissions_set_default.rs) — 需要 `ManagePermissions`，且只能设置自己拥有的权限
- 频道默认权限：[permissions_set_default.rs (channel)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/channels/permissions_set_default.rs) — 需要 `ManagePermissions`，`Group` 使用纯 allow 值，`TextChannel` 使用 Override（allow/deny）

---

## 八、关键代码索引

| 职责 | 文件 |
|---|---|
| 权限位定义 | [channel.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/channel.rs) |
| Override 结构 | [mod.rs (models)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs) |
| PermissionValue 操作 | [mod.rs (models)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L14-L114) |
| PermissionQuery trait | [trait.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/trait.rs) |
| 权限计算核心逻辑 | [impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/impl.rs) |
| 数据库适配器 | [permissions.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/permissions.rs) |
| 批量权限计算 | [bulk_permissions.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs) |
| 角色删除路由 | [roles_delete.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_delete.rs) |
| 角色删除 DB 操作 | [mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/servers/ops/mongodb.rs#L113-L156) |
| 成员 ranking | [model.rs (members)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/server_members/model.rs#L243-L254) |
| 语音权限同步 | [voice/mod.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/voice/mod.rs#L441-L466) |
| 权限单元测试 | [test.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/test.rs) |
| 服务器模型 | [servers.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/models/src/v0/servers.rs) |
| 频道模型 | [channels.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/models/src/v0/channels.rs) |
| 成员模型 | [server_members.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/models/src/v0/server_members.rs) |
