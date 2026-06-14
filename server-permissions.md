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

## 九、两条权限计算路径的深度对比

系统中存在两条独立的权限计算路径：**单用户 trait 路径**和**批量同步路径**。两者计算结果在服务器级应该一致，但在频道级和后处理阶段存在结构性差异。

### 9.1 架构对比总览

| 维度 | 单用户 trait 路径 | 批量同步路径 |
|---|---|---|
| 入口结构 | `DatabasePermissionQuery` | `BulkDatabasePermissionQuery` |
| 计算函数 | `calculate_server_permissions::<P: PermissionQuery>` (async, 泛型) | `calculate_server_permissions(server, user, member)` (sync, 具体类型) |
| 服务器级计算 | 通过 trait 方法间接获取数据 | 直接引用 `&Server`, `&User`, `&Member` |
| 频道级计算 | `calculate_channel_permissions` 支持全部 5 种频道类型 | `calculate_members_permissions` 仅支持 `TextChannel`，其他类型直接 panic |
| 数据获取 | 按需异步拉取（lazy fetch） | 前置批量拉取（eager fetch） |
| 语音开关 | ✅ 服务器级检查 `can_publish` / `can_receive` | ❌ **缺失** |
| 频道级 Timeout | ✅ `ServerChannel` 分支中二次检查 | ❌ **缺失** |
| ViewChannel 守卫 | ✅ 无 `ViewChannel` 则 `revoke_all()` | ❌ **缺失** |
| 成员身份未知时 | 自动从 DB 补拉成员 | 直接判 0 |

### 9.2 两条路径的 `calculate_server_permissions` 逐行对比

**单用户 trait 版本** — [impl.rs#L49-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/impl.rs#L49-L78)：

```rust
pub async fn calculate_server_permissions<P: PermissionQuery>(query: &mut P) -> PermissionValue {
    // ① 特权 OR 服务器拥有者 → GrantAllSafe
    if query.are_we_privileged().await || query.are_we_server_owner().await { ... }
    // ② 非成员 → 0
    if !query.are_we_a_member().await { ... }
    // ③ 取默认权限
    let mut permissions = query.get_default_server_permissions().await.into();
    // ④ 依次 apply 角色覆盖
    for role_override in query.get_our_server_role_overrides().await { permissions.apply(role_override); }
    // ⑤ 语音开关
    if !query.do_we_have_publish_overwrites().await { revoke(Speak+Video); }
    if !query.do_we_have_receive_overwrites().await { revoke(Listen); }
    // ⑥ Timeout
    if query.are_we_timed_out().await { permissions.restrict(*ALLOW_IN_TIMEOUT); }
    permissions
}
```

**批量同步版本** — [bulk_permissions.rs#L316-L345](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L316-L345)：

```rust
fn calculate_server_permissions(server: &Server, user: &User, member: &Member) -> PermissionValue {
    // ① 特权 OR 服务器拥有者 → GrantAllSafe
    if user.privileged || server.owner == user.id { ... }
    // ② 无（调用方已保证成员存在）
    // ③ 取默认权限
    let mut permissions: PermissionValue = server.default_permissions.into();
    // ④ 依次 apply 角色覆盖
    let mut roles = server.roles.iter().filter(...).map(...).collect();
    roles.sort_by(|a, b| b.0.cmp(&a.0));
    for role in role_overrides { permissions.apply(role); }
    // ⑤ 语音开关 — ❌ 缺失
    // ⑥ Timeout
    if member.in_timeout() { permissions.restrict(*ALLOW_IN_TIMEOUT); }
    permissions
}
```

**差异汇总**：

| 步骤 | trait 版本 | 批量版本 | 差异 |
|---|---|---|---|
| 特权判断 | `are_we_privileged()` | `user.privileged` | 等价 |
| 拥有者判断 | `are_we_server_owner()` | `server.owner == user.id` | 等价 |
| 成员检查 | `are_we_a_member()` 可能补拉 | 调用方已过滤 | 批量版不做（外层保证） |
| 角色排序 | trait 内 `get_our_server_role_overrides` | 内联排序逻辑 | 等价 |
| 语音开关 | ✅ 检查 `can_publish`/`can_receive` | ❌ **完全缺失** | **行为差异** |
| Timeout | ✅ | ✅ | 等价 |

### 9.3 频道级计算的差异

单用户路径的 `calculate_channel_permissions` 在 `ServerChannel` 分支中（[impl.rs#L119-L144](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/impl.rs#L119-L144)）执行了三步后处理：

1. **Timeout 二次检查**：即使服务器级已 restrict，频道级再次 `restrict(*ALLOW_IN_TIMEOUT)`
2. **ViewChannel 守卫**：若无 `ViewChannel` 权限，`revoke_all()` 清零

批量路径的 `calculate_members_permissions`（[bulk_permissions.rs#L256-L310](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L256-L310)）在频道级**都没有**这两个后处理步骤。

这意味着：
- 批量路径可能返回一个没有 `ViewChannel` 权限但仍有其他权限位的非零值
- `members_can_see_channel` 通过 `has_channel_permission(ChannelPermission::ViewChannel)` 单独检查弥补了 ViewChannel 守卫的缺失，但其他使用批量权限值的场景可能出问题
- 批量路径的 Timeout 限制仅在服务器级生效（`calculate_server_permissions` 同步版中），频道级不再二次 restrict——但由于 `restrict` 是 AND 操作，重复执行结果相同，所以实际差异仅在于语音开关的缺失

---

## 十、批量补拉机制：missing_users 与 missing_members

`BulkDatabasePermissionQuery` 的 Builder 模式允许调用方只提供 `users` 或 `members` 其中之一，缺失的另一个维度会在计算时自动补拉。

### 10.1 补拉逻辑

位于 [calculate_members_permissions](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L206-L245)：

```
调用方提供了 members 但没提供 users:
  → 从 members 中提取 user_id 列表
  → database.fetch_users(&ids[..]) 批量拉取
  → 存入 cached_users 和 users

调用方提供了 users 但没提供 members:
  → 从 users 中提取 user_id 列表
  → database.fetch_members(&server.id, &ids[..]) 批量拉取
  → 存入 cached_members 和 members
```

### 10.2 双缓存设计

```rust
pub struct BulkDatabasePermissionQuery<'a> {
    users: Option<Vec<User>>,           // 调用方传入或补拉后的用户列表
    members: Option<Vec<Member>>,       // 调用方传入或补拉后的成员列表
    pub(crate) cached_users: Option<Vec<User>>,    // 补拉产生的用户数据
    pub(crate) cached_members: Option<Vec<Member>>, // 补拉产生的成员数据
    cached_member_perms: Option<HashMap<String, PermissionValue>>, // 权限结果缓存
}
```

`cached_users` / `cached_members` 与 `users` / `members` 分开存储，原因：
- `cached_*` 标记"这些数据是补拉产生的，属于调用方不持有的额外数据"，供外部读取
- `users` / `members` 是实际计算用的列表

### 10.3 Builder 的互斥约束

[`.members()`](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L112-L121) 和 [`.users()`](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L123-L132) 两个 Builder 方法会**互斥清空**对方：

```rust
pub fn members(self, members: &'z [Member]) -> BulkDatabasePermissionQuery<'z> {
    BulkDatabasePermissionQuery {
        members: Some(members.to_owned()),
        cached_member_perms: None,
        users: None,           // ← 清空 users
        cached_members: None,
        cached_users: None,
        ..self
    }
}

pub fn users(self, users: &'z [User]) -> BulkDatabasePermissionQuery<'z> {
    BulkDatabasePermissionQuery {
        users: Some(users.to_owned()),
        cached_member_perms: None,
        members: None,         // ← 清空 members
        cached_members: None,
        cached_users: None,
        ..self
    }
}
```

这确保了不会出现"旧 users + 新 members"的不一致状态。任一 setter 调用后，另一维度必然为 `None`，需要在 `calculate_members_permissions` 中补拉。

### 10.4 成员查找效率

补拉完成后，成员列表被转为 HashMap 以实现 O(1) 查找（[bulk_permissions.rs#L247-L254](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs#L247-L254)）：

```rust
let members: HashMap<&String, &Member, RandomState> = HashMap::from_iter(
    query.members.as_ref().unwrap()
        .iter()
        .map(|m| (&m.id.user, m)),
);
```

以 `user.id` 为 key，遍历用户列表时快速定位对应成员。如果用户不是服务器成员（`members.get(&user.id)` 返回 `None`），直接赋 0 权限。

---

## 十一、Timeout 和语音开关在两条链路中的位置差异

### 11.1 单用户链路中的完整后处理链

```
calculate_channel_permissions (ServerChannel 分支)
  │
  ├─ calculate_server_permissions (trait 版)
  │     ├─ 角色叠加完成后
  │     ├─ ✅ can_publish 检查 → revoke(Speak + Video)
  │     ├─ ✅ can_receive 检查 → revoke(Listen)
  │     └─ ✅ Timeout → restrict(ALLOW_IN_TIMEOUT)
  │
  ├─ apply 频道默认权限
  ├─ apply 频道角色覆盖
  │
  ├─ ✅ Timeout 二次 restrict(ALLOW_IN_TIMEOUT)    ← 频道级后处理
  └─ ✅ ViewChannel 守卫 → revoke_all()             ← 频道级后处理
```

**关键细节**：Timeout 在单用户路径中被检查了两次：
1. 服务器级（`calculate_server_permissions` 内）：角色叠加后 restrict
2. 频道级（`calculate_channel_permissions` 的 `ServerChannel` 分支）：频道覆盖后再次 restrict

第二次 restrict 是防御性的——理论上，如果频道覆盖的 allow 包含了 `ViewChannel` 或 `ReadMessageHistory` 之外的权限，第二次 restrict 会将其收回。但由于 `restrict` 是 AND 操作，而第一次 restrict 已经将权限限制到 `ALLOW_IN_TIMEOUT`（只有 ViewChannel + ReadMessageHistory），频道覆盖不可能恢复超出此范围的权限（因为角色的 allow 位中不可能有超出 ALLOW_IN_TIMEOUT 的位被设置——除非角色显式 allow 了其他权限）。**因此第二次 restrict 的实际效果是：即使频道覆盖 allow 了额外权限，Timeout 成员仍无法获得。**

### 11.2 批量链路中的后处理链

```
calculate_members_permissions
  │
  ├─ calculate_server_permissions (同步版)
  │     ├─ 角色叠加完成后
  │     ├─ ❌ 无 can_publish / can_receive 检查
  │     └─ ✅ Timeout → restrict(ALLOW_IN_TIMEOUT)
  │
  ├─ apply 频道默认权限
  ├─ apply 频道角色覆盖
  │
  ├─ ❌ 无 Timeout 二次 restrict
  └─ ❌ 无 ViewChannel 守卫
```

**缺失分析**：

| 缺失项 | 影响 | 是否有外部弥补 |
|---|---|---|
| `can_publish` / `can_receive` | 被 mute/deafen 的成员在批量计算中仍获得 Speak/Video/Listen 权限 | `members_can_see_channel` 只检查 ViewChannel，不检查语音权限 |
| 频道级 Timeout 二次 restrict | Timeout 成员如果频道覆盖 allow 了额外权限，不会被收回 | 无弥补 |
| ViewChannel 守卫 | 没有 ViewChannel 权限的成员可能返回非零权限值 | `members_can_see_channel` 通过单独检查 ViewChannel 弥补了"可见性判断"场景 |

**结论**：批量路径的 `calculate_members_permissions` 目前仅在 `members_can_see_channel` 中被调用，该方法只需要判断 `ViewChannel` 权限，因此缺失的后处理对当前使用场景影响有限。但如果未来在其他场景复用该函数，需要补齐这些后处理步骤。

---

## 十二、throw_permission_override 位运算拦截机制

### 12.1 拦截逻辑详解

[throw_permission_override](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L92-L113) 是权限委托安全的核心守卫，防止用户将自身不具备的权限授予他人。

```rust
pub async fn throw_permission_override<C>(
    &self,                          // self = 操作者当前权限值
    current_value: C,               // 目标当前的 Override（Option<Override>）
    next_value: &Override,          // 目标即将设置的 Override
) -> Result<()>
where C: Into<Option<Override>>
```

**两种场景**：

**场景 A：从无到有（current_value = None）**

```rust
if !self.has(next_value.allows()) {
    return Err(create_error!(CannotGiveMissingPermissions));
}
```

所有即将 allow 的权限位，操作者必须全部拥有。这是最直观的检查：你不能授予你没有的权限。

**场景 B：从旧值变更（current_value = Some(old)）**

```rust
if !self.has(!current_value.allows() & next_value.allows())
    || !self.has(current_value.denies() & !next_value.denies())
{
    return Err(create_error!(CannotGiveMissingPermissions));
}
```

两个条件必须同时满足，否则拦截：

**条件 1**：`!old_allow & new_allow` — 新增的 allow 位

```
旧值 allow = 0b1010
新值 allow = 0b1110
新增 allow = ~0b1010 & 0b1110 = 0b0101 & 0b1110 = 0b0100
```

新增的 allow 位（从无到有）操作者必须拥有。你不能开放你本身不具备的权限。

**条件 2**：`old_deny & ~new_deny` — 移除的 deny 位

```
旧值 deny = 0b1100
新值 deny = 0b1000
移除 deny = 0b1100 & ~0b1000 = 0b1100 & 0b0111 = 0b0100
```

移除的 deny 位（从拒绝变为中性/允许）操作者必须拥有。你不能"解禁"你本身不具备的权限——因为解禁等同于授予。

### 12.2 位运算真值表

| 旧 allow | 新 allow | 旧 deny | 新 deny | 含义 | 需要检查 |
|---|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 保持中性 | 无需 |
| 0 | 1 | 0 | 0 | 新增 allow | ✅ 操作者需拥有此位 |
| 1 | 1 | 0 | 0 | 保持 allow | 无需 |
| 1 | 0 | 0 | 0 | 撤销 allow | 无需（收权总是允许的） |
| 0 | 0 | 0 | 1 | 新增 deny | 无需（收权总是允许的） |
| 0 | 0 | 1 | 1 | 保持 deny | 无需 |
| 0 | 0 | 1 | 0 | 移除 deny | ✅ 操作者需拥有此位 |
| 1 | 1 | 1 | 1 | 同时 allow+deny（deny 胜出） | 无需 |

**核心原则**：只有"扩大权限范围"的操作才需要拦截——包括新增 allow 和移除 deny。"缩小权限范围"的操作（新增 deny、撤销 allow）总是允许的。

### 12.3 调用位置

| 路由 | 文件 | 场景 |
|---|---|---|
| 服务器角色权限设置 | [permissions_set.rs (server)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/permissions_set.rs#L46-L48) | 修改服务器角色的 allow/deny |
| 频道角色权限设置 | [permissions_set.rs (channel)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/channels/permissions_set.rs#L36-L39) | 修改频道角色的 allow/deny |
| 频道默认权限设置 | [permissions_set_default.rs (channel)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/channels/permissions_set_default.rs#L53-L55) | 修改频道默认的 allow/deny |
| 服务器默认权限设置 | [permissions_set_default.rs (server)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/permissions_set_default.rs#L33-L40) | 设置默认权限（从 None 开始） |

---

## 十三、角色 Rank 互锁机制

Rank 互锁是权限系统的层级安全守卫，确保低优先级角色的持有者无法操作高优先级角色。互锁贯穿于角色编辑、角色删除、角色排序、成员编辑四个操作。

### 13.1 互锁基础：Member::get_ranking

[get_ranking](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/server_members/model.rs#L243-L254) 取成员所有角色中的最小 rank 值：

```rust
pub fn get_ranking(&self, server: &Server) -> i64 {
    let mut value = i64::MAX;         // 默认最低优先级
    for role in &self.roles {
        if let Some(role) = server.roles.get(role) {
            if role.rank < value {
                value = role.rank;    // 取最小 rank = 最高优先级
            }
        }
    }
    value
}
```

**无角色成员的 ranking = `i64::MAX`**（最低优先级），拥有 rank=0 角色的成员 ranking=0（最高优先级）。服务器拥有者不经过此计算，直接拥有绝对权限。

### 13.2 互锁在角色删除中的体现

[roles_delete.rs#L29-L38](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_delete.rs#L29-L38)：

```rust
let member_rank = query.get_member_rank().unwrap_or(i64::MIN);
let role = server.roles.remove(&role_id).ok_or_else(|| create_error!(NotFound))?;
if role.rank <= member_rank {
    return Err(create_error!(NotElevated));
}
```

**互锁规则**：`role.rank <= member_rank` 时拒绝。即只能删除 rank **严格大于**自己 ranking 的角色（rank 更大 = 优先级更低）。

`unwrap_or(i64::MIN)` 处理了成员不在服务器中的极端情况——此时 `i64::MIN` 确保几乎所有角色的 rank 都大于它，从而阻止操作。

### 13.3 互锁在角色编辑中的体现

[roles_edit.rs#L38-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_edit.rs#L38-L44)：

```rust
let member_rank = query.get_member_rank().unwrap_or(i64::MIN);
if let Some(mut role) = server.roles.remove(&role_id) {
    if role.rank <= member_rank {
        return Err(create_error!(NotElevated));
    }
    // ... proceed with edit
}
```

与删除逻辑完全一致：不能编辑 rank 不高于自己的角色。注意编辑只涉及角色元数据（名称、颜色、图标），不涉及权限修改——权限修改走 `permissions_set` 路由。

### 13.4 互锁在角色排序中的体现

[roles_edit_positions.rs#L45-L69](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_edit_positions.rs#L45-L69)：

```rust
if server.owner != user.id {
    let member_top_rank = query.get_member_rank();
    if server.roles.iter()
        .filter(|(_, role)| {
            if let Some(top_rank) = member_top_rank {
                role.rank <= top_rank      // 找出所有高于或等于自己的角色
            } else { true }
        })
        .any(|(id, _)| {
            existing_order.iter().position(|x| x == id)
                != new_order.iter().position(|x| x == id)  // 检查这些角色的位置是否被改变
        })
    {
        return Err(create_error!(NotElevated));
    }
}
```

**互锁规则**：非拥有者不能改变 rank ≤ 自己 ranking 的角色在排序中的位置。这意味着：
- 你不能把高于你的角色往下挪（降低其优先级）
- 你不能把低于你的角色往上挪到高于你的位置（提升其优先级）
- 你只能在自己排名以下的角色之间重新排序

### 13.5 互锁在成员编辑中的双重体现

[member_edit.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/member_edit.rs) 包含两层互锁：

**第一层：目标成员的 ranking 互锁**（[L144-L152](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/member_edit.rs#L144-L152)）

```rust
let our_ranking = query.get_member_rank().unwrap_or(i64::MIN);
if member.id.user != user.id
    && member.get_ranking(query.server_ref().as_ref().unwrap()) <= our_ranking
{
    return Err(create_error!(NotElevated));
}
```

不能对 ranking 不高于自己的成员执行管理操作。注意这里用的是 `<=`（不是 `<`），即 ranking 相等的成员也不能互操作。

**第二层：角色分配的 rank 互锁**（[L155-L169](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/member_edit.rs#L155-L169)）

```rust
if let Some(roles) = &data.roles {
    let added_roles: Vec<&&String> = new_roles.difference(&current_roles).collect();
    for role_id in added_roles {
        if let Some(role) = server.roles.remove(*role_id) {
            if role.rank <= our_ranking {
                return Err(create_error!(NotElevated));
            }
        } else {
            return Err(create_error!(InvalidRole));
        }
    }
}
```

只能分配 rank **严格高于**自己 ranking 的角色给他人。这防止了"我给自己或他人添加一个比我当前最高优先级还高的角色"的提权攻击。

**Timeout 的反向互锁**（[L81-L93](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/member_edit.rs#L81-L93)）

```rust
if data.timeout.is_some() {
    if member.id.user == user.id {
        return Err(create_error!(CannotTimeoutYourself));
    }
    if target_permissions.has_channel_permission(ChannelPermission::TimeoutMembers) {
        return Err(create_error!(IsElevated));
    }
}
```

Timeout 操作除了需要 `TimeoutMembers` 权限外，还有反向检查：如果目标成员拥有 `TimeoutMembers` 权限，则不能对其执行 Timeout。这是一种"同级保护"——拥有管理权限的成员不能被互相 Timeout。

### 13.6 Rank 互锁全景图

```
操作               互锁检查                              代码位置
─────────────────────────────────────────────────────────────────────
删除角色           role.rank > member_rank              roles_delete.rs#L37
编辑角色           role.rank > member_rank              roles_edit.rs#L42
角色排序           不能移动 rank ≤ 自己的角色的位置      roles_edit_positions.rs#L48-L68
成员编辑(总)       target_ranking > our_ranking          member_edit.rs#L148-L152
成员编辑(角色)     added_role.rank > our_ranking         member_edit.rs#L161-L165
成员编辑(Timeout)  目标不能有 TimeoutMembers 权限        member_edit.rs#L87-L89
权限设置(服务器)   role.rank > member_rank              permissions_set(server)#L40
权限设置(频道)     role.rank > member_rank              permissions_set(channel)#L32
```

**统一的互锁模式**：所有检查都遵循"只能操作 rank 严格低于自己 ranking 的对象"这一原则，用 `role.rank <= member_rank` 或 `target_ranking <= our_ranking` 来拦截越级操作。

---

## 十四、关键代码索引

| 职责 | 文件 |
|---|---|
| 权限位定义 | [channel.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/channel.rs) |
| Override 结构 | [mod.rs (models)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs) |
| PermissionValue 操作 | [mod.rs (models)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L14-L114) |
| throw_permission_override | [mod.rs (models)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/models/mod.rs#L92-L113) |
| PermissionQuery trait | [trait.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/trait.rs) |
| 权限计算核心逻辑（trait 版） | [impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/impl.rs) |
| 数据库适配器（单用户） | [permissions.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/permissions.rs) |
| 批量权限计算（含同步版 calculate_server_permissions） | [bulk_permissions.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/util/bulk_permissions.rs) |
| 角色删除路由 | [roles_delete.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_delete.rs) |
| 角色编辑路由（rank 互锁） | [roles_edit.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_edit.rs) |
| 角色排序路由（rank 互锁） | [roles_edit_positions.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/roles_edit_positions.rs) |
| 成员编辑路由（双重 rank 互锁） | [member_edit.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/member_edit.rs) |
| 服务器角色权限设置 | [permissions_set.rs (server)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/servers/permissions_set.rs) |
| 频道角色权限设置 | [permissions_set.rs (channel)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/delta/src/routes/channels/permissions_set.rs) |
| 角色删除 DB 操作 | [mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/servers/ops/mongodb.rs#L113-L156) |
| 成员 ranking | [model.rs (members)](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/models/server_members/model.rs#L243-L254) |
| 语音权限同步 | [voice/mod.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/database/src/voice/mod.rs#L441-L466) |
| 权限单元测试 | [test.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/permissions/src/test.rs) |
| 服务器模型 | [servers.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/models/src/v0/servers.rs) |
| 频道模型 | [channels.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/models/src/v0/channels.rs) |
| 成员模型 | [server_members.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/82-backend/crates/core/models/src/v0/server_members.rs) |
