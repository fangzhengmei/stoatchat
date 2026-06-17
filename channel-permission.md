# 频道权限计算路径分析

本文档按代码执行顺序拆解 Revolt 后端频道权限的判定逻辑，重点说明：判定顺序、优先级、allow/deny 冲突如何处理。

---

## 1. 核心数据结构

### 1.1 PermissionValue —— 位掩码权限值

定义位置：[models/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/models/mod.rs#L11-L138)

```rust
pub struct PermissionValue(u64);
```

内部是 64 位整数，每一位对应一种 `ChannelPermission`（如 `ViewChannel = 1 << 20`）。

关键操作方法：

| 方法 | 位运算 | 作用 |
|------|--------|------|
| `allow(v)` | `self.0 \|= v` | 按位或，授予权限 |
| `revoke(v)` | `self.0 &= !v` | 按位与取反，撤销权限 |
| `restrict(v)` | `self.0 &= v` | 按位与，保留指定权限，其余清除 |
| `revoke_all()` | `self.0 = 0` | 清零所有权限 |
| `has(v)` | `(self.0 & v) == v` | 检查是否拥有全部指定权限 |

### 1.2 Override —— 权限覆盖对

定义位置：[models/server.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/models/server.rs#L5-L13)

```rust
pub struct Override {
    pub allow: u64,  // 允许的权限位
    pub deny: u64,   // 拒绝的权限位
}
```

### 1.3 Override::apply() —— 叠加的核心语义

定义位置：[models/mod.rs#L24-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/models/mod.rs#L24-L27)

```rust
pub fn apply(&mut self, v: Override) {
    self.allow(v.allow);
    self.revoke(v.deny);
}
```

**先 allow 后 deny**。这意味着在**同一个 Override 内部**，deny 优先级高于 allow：如果某位同时出现在 allow 和 deny 中，最终结果是 deny（因为 allow 先置 1，deny 随后清 0）。

---

## 2. ServerChannel（服务器频道）权限完整判定流程

入口函数：[calculate_channel_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/impl.rs#L81-L147)

以下按代码执行顺序逐步拆解：

### 第 0 步：特权短路

```rust
if query.are_we_privileged().await {
    return ChannelPermission::GrantAllSafe.into();
}
```

如果当前用户是特权用户（系统/管理员），直接授予全部安全权限，跳过所有后续判断。

### 第 1 步：服务器所有者短路

```rust
if query.are_we_server_owner().await {
    return ChannelPermission::GrantAllSafe.into();
}
```

服务器所有者拥有该服务器下所有频道的全部权限。

### 第 2 步：非服务器成员直接拒权

```rust
if query.are_we_a_member().await {
    // 继续计算
} else {
    return 0_u64.into();
}
```

不是服务器成员 → 0 权限。

### 第 3 步：计算服务器级权限（calculate_server_permissions）

调用：[calculate_server_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/impl.rs#L49-L78)

此函数内部按以下顺序执行：

#### 3.1 特权/所有者短路（同上，略）

#### 3.2 取服务器默认权限作为基底

```rust
let mut permissions: PermissionValue =
    query.get_default_server_permissions().await.into();
```

对应数据库字段：`Server.default_permissions`（纯 u64，仅 allow 语义）。

#### 3.3 按角色 rank 叠加服务器角色权限

```rust
for role_override in query.get_our_server_role_overrides().await {
    permissions.apply(role_override);
}
```

**角色排序规则**（定义在 [permissions.rs#L154-L177](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/permissions.rs#L154-L177)）：

```rust
roles.sort_by(|a, b| b.0.cmp(&a.0));  // 按 rank 降序排列
```

即 `rank` 值**大**的角色排在前面，先 apply；rank 值**小**的排在后面，后 apply。

由于 `apply()` 是后执行覆盖先执行的结果（deny 可撤销之前 allow 的权限），因此实际效果是：**rank 值越小的角色优先级越高**（后执行，可以覆盖前面的结果）。

#### 3.4 语音发布/接收覆盖

```rust
if !query.do_we_have_publish_overwrites().await {
    permissions.revoke(ChannelPermission::Speak as u64);
    permissions.revoke(ChannelPermission::Video as u64);
}

if !query.do_we_have_receive_overwrites().await {
    permissions.revoke(ChannelPermission::Listen as u64);
}
```

- `can_publish = false` → 强制撤销 Speak、Video
- `can_receive = false` → 强制撤销 Listen

对应 `Member` 上的独立布尔字段。

#### 3.5 超时（Timeout）限制

```rust
if query.are_we_timed_out().await {
    permissions.restrict(*ALLOW_IN_TIMEOUT);
}
```

`ALLOW_IN_TIMEOUT = ViewChannel | ReadMessageHistory`，即超时状态下只保留浏览和读历史。

### 第 4 步：应用频道默认权限

```rust
permissions.apply(query.get_default_channel_permissions().await);
```

对应数据库字段：`TextChannel.default_permissions`（类型 `Option<OverrideField>`，即 allow+deny）。

如果频道未设置 `default_permissions`，则使用空 Override（allow=0, deny=0），不做任何变更。

### 第 5 步：按角色 rank 叠加频道角色权限

```rust
for role_override in query.get_our_channel_role_overrides().await {
    permissions.apply(role_override);
}
```

对应数据库字段：`TextChannel.role_permissions: HashMap<String, OverrideField>`。

排序规则与服务器角色完全一致：按 rank 降序排列后依次 apply。

数据来源：[permissions.rs#L252-L290](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/permissions.rs#L252-L290)

### 第 6 步：频道级别超时限制

```rust
if query.are_we_timed_out().await {
    permissions.restrict(*ALLOW_IN_TIMEOUT);
}
```

与服务器级超时逻辑相同，再执行一次（确保即使频道覆盖尝试授予额外权限，也被超时限制住）。

### 第 7 步：无 ViewChannel 则全部清零

```rust
if !permissions.has_channel_permission(ChannelPermission::ViewChannel) {
    permissions.revoke_all();
}
```

这是一条"全有或全无"规则：如果最终看不到频道，则连其它任何权限都不保留（防止出现"看不到频道但能发消息"之类的不一致状态）。

---

## 3. 其它频道类型的权限判定

### 3.1 SavedMessages（已保存消息）

```rust
if query.do_we_own_the_channel().await {
    DEFAULT_PERMISSION_SAVED_MESSAGES.into()  // GrantAllSafe
} else {
    0_u64.into()
}
```

只有频道所有者（即用户本人）拥有全部权限，其它人 0。

### 3.2 DirectMessage（私信）

```rust
if query.are_we_part_of_the_channel().await {
    // 先把对方设为 user，计算用户间关系权限
    query.set_recipient_as_user().await;
    let permissions = calculate_user_permissions(query).await;
    if permissions.has_user_permission(UserPermission::SendMessage) {
        (*DEFAULT_PERMISSION_DIRECT_MESSAGE).into()
    } else {
        (*DEFAULT_PERMISSION_VIEW_ONLY).into()
    }
} else {
    0_u64.into()
}
```

取决于与对方的用户关系：好友/自己 → 完全权限；陌生人但有互关 → 可发送消息等完整权限；被拉黑/无关系 → 仅 View。

### 3.3 Group（群组）

```rust
if query.do_we_own_the_channel().await {
    ChannelPermission::GrantAllSafe.into()
} else if query.are_we_part_of_the_channel().await {
    (*DEFAULT_PERMISSION_VIEW_ONLY
        | query.get_default_channel_permissions().await.allow)
        .into()
} else {
    0_u64.into()
}
```

- 群主 → 全部权限
- 群成员 → `ViewChannel | ReadMessageHistory` 作为基底，再 OR 上群组设置的 allow 权限（Group 的 `permissions` 字段只有 allow 语义，没有 deny）
- 非成员 → 0

---

## 4. 优先级总览（从低到高，后者覆盖前者）

针对 ServerChannel，按"被覆盖者 → 覆盖者"的方向排列：

| 层级 | 来源 | 数据字段 | 类型 |
|------|------|----------|------|
| 1（最低） | 服务器默认 | `Server.default_permissions` | u64（仅 allow） |
| 2 | 服务器角色（按 rank 降序 apply，rank 小的优先级更高） | `Server.roles[id].permissions` | Override（allow+deny） |
| 3 | 语音成员状态 | `Member.can_publish` / `can_receive` | 布尔（强制 deny） |
| 4 | 服务器超时 | `Member.timeout` | restrict（强制白名单） |
| 5 | 频道默认 | `TextChannel.default_permissions` | Override（allow+deny） |
| 6 | 频道角色（按 rank 降序 apply，rank 小的优先级更高） | `TextChannel.role_permissions[id]` | Override（allow+deny） |
| 7 | 频道超时 | 同 `Member.timeout` | restrict（强制白名单） |
| 8（最高） | 无 ViewChannel → 清零 | 派生 | 强制全部 0 |

### 关于 rank 排序的细节

代码中按 rank **降序**排列（rank 大的先 apply），但由于后 apply 的会覆盖先 apply 的，因此实际优先级是 **rank 数值越小 → 优先级越高**。

这与 `set_role_permissions` 路由中 rank 的语义一致（操作者的 rank 必须严格大于被操作角色的 rank，才能修改其权限），说明 rank 数值设计上就是"越小越高级"。

---

## 5. 允许/拒绝冲突处理规则

冲突可能发生在两个层面：

### 5.1 单个 Override 内部 allow/deny 同时置位

由于 `apply()` 实现为：

```rust
self.allow(v.allow);   // |=
self.revoke(v.deny);   // &= !
```

同一位如果同时出现在 allow 和 deny 中，最终结果是 **deny 胜出**。

### 5.2 不同层级/不同角色之间的冲突

按上一节优先级表，**高层级完全覆盖低层级**：

- 高层级的 allow 可以授予低层级 deny 的权限
- 高层级的 deny 可以撤销低层级 allow 的权限

示例（来自 [test.rs#L205-L315](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/test.rs#L205-L315)）：

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 服务器默认: View + Send + ReadHistory | View, Send, ReadHistory |
| 2 | 服务器角色: allow(Upload+React), deny(ReadHistory) | View, Send, Upload, React |
| 3 | 频道默认: deny(Send) | View, Upload, React |
| 4 | 频道角色: deny(React) | View, Upload |

最终结果：`ViewChannel | UploadFiles`，与测试断言一致。

---

## 6. 关于"成员覆盖"的说明

当前 `TextChannel` 数据模型中**不存在成员级别的频道权限覆盖**（即没有类似 `user_permissions: HashMap<UserId, OverrideField>` 的字段）。

权限覆盖粒度仅限两级：
- **频道默认**（相当于 @everyone 的频道覆盖）
- **频道角色**（按角色覆盖）

如果需要针对特定成员调整频道权限，只能通过创建专属角色并设置频道角色覆盖来实现。
