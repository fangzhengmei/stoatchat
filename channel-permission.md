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

---

## 7. 批量成员可见性计算与单用户权限计算的差异对比

除第 2 节描述的单用户路径外，代码库中还存在一条**批量权限计算路径**，用于一次性计算多名成员对某频道的可见性。两者在超时限制和语音发布/接收限制的实现上存在明显差异，本节结合代码逐一对比。

### 7.1 批量路径的用途与入口

批量计算的唯一调用方是推送守护进程 [mass_mention.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L112-L213)：当服务器频道发生 @everyone / @here 群体提及需要发推送时，用它快速筛出"能看到该频道"的成员，只给他们推通知。

入口结构体：[BulkDatabasePermissionQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L10-L25)

对外只暴露一个布尔结果方法 [members_can_see_channel()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L28-L57)，内部调用私有函数 [calculate_members_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L186-L313)，最终只取 `ViewChannel` 这一位。

### 7.2 批量路径执行流程

[calculate_members_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L186-L313) 内部按以下顺序处理每个用户：

1. **非成员短路**：`member.is_none()` → 0 权限
2. **特权短路**：`user.privileged` → GrantAllSafe
3. **服务器所有者短路**：`user.id == server.owner` → GrantAllSafe
4. **计算服务器级权限**：调用批量版 [calculate_server_permissions(&server, user, member)](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L316-L345)
5. **应用频道默认权限**：`permission.apply(channel_default_permissions)`
6. **按角色 rank 叠加频道角色权限**：同单用户路径的 rank 降序 apply
7. **（结束，无后续限制步骤）**

### 7.3 差异一：超时（Timeout）限制

这是两套路径最关键的差异。

**单用户路径**——超时限制执行**两次**：

| 位置 | 代码 | 说明 |
|------|------|------|
| 服务器级（[impl.rs#L73-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/impl.rs#L73-L75)） | `permissions.restrict(*ALLOW_IN_TIMEOUT)` | 第一次：在服务器角色 apply 之后 |
| 频道级（[impl.rs#L132-L134](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/impl.rs#L132-L134)） | `permissions.restrict(*ALLOW_IN_TIMEOUT)` | 第二次：在频道默认+频道角色 apply 之后再执行一次 |

第二次 restrict 的作用是**兜底**：即使频道默认权限或频道角色覆盖尝试给超时成员授予额外权限（如 SendMessage），也会被 restrict 清除，只保留 `ViewChannel | ReadMessageHistory`。

**批量路径**——超时限制只执行**一次**：

| 位置 | 代码 | 说明 |
|------|------|------|
| 服务器级（[bulk_permissions.rs#L340-L342](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L340-L342)） | `permissions.restrict(*ALLOW_IN_TIMEOUT)` | 仅此一次，在服务器角色 apply 之后 |
| 频道级 | **缺失** | 频道默认权限和频道角色覆盖 apply 之后，没有再次 restrict |

**影响**：在批量路径中，如果一个超时成员的频道角色覆盖授予了 `ViewChannel` 之外的权限（如 `SendMessage`），这些权限**不会被清除**，会保留在最终 `PermissionValue` 中。

不过对可见性结果而言，`ALLOW_IN_TIMEOUT = ViewChannel | ReadMessageHistory` 已经保留了 `ViewChannel`，所以超时成员仍然"能看到"频道（只要服务器级或频道级没有显式 deny 掉 ViewChannel），可见性布尔值在多数场景下一致。差异主要体现在**存储的 PermissionValue 语义**上，而非可见性判定本身。

### 7.4 差异二：语音发布/接收限制（can_publish / can_receive）

**单用户路径**——完整实现了语音限制（[impl.rs#L64-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/impl.rs#L64-L71)）：

```rust
if !query.do_we_have_publish_overwrites().await {
    permissions.revoke(ChannelPermission::Speak as u64);
    permissions.revoke(ChannelPermission::Video as u64);
}

if !query.do_we_have_receive_overwrites().await {
    permissions.revoke(ChannelPermission::Listen as u64);
}
```

对应 `Member` 上的 `can_publish`、`can_receive` 布尔字段。`can_publish = false` 时撤销 Speak、Video；`can_receive = false` 时撤销 Listen。

**批量路径**——**完全缺失**这两段逻辑。

批量版 [calculate_server_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L316-L345) 的函数体中没有任何对 `can_publish` / `can_receive` 的引用，[calculate_members_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L186-L313) 中也没有补做。

**影响**：批量路径计算出的 `PermissionValue` 中，Speak、Video、Listen 三项**不受** `can_publish` / `can_receive` 约束。但由于批量路径只取 `ViewChannel` 位做可见性判断，而 Speak/Video/Listen 与 ViewChannel 是不同位，因此这一差异**不影响可见性结果**，只影响存储的权限值语义。

### 7.5 差异三：无 ViewChannel 时的 revoke_all

**单用户路径**（[impl.rs#L136-L138](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/impl.rs#L136-L138)）：

```rust
if !permissions.has_channel_permission(ChannelPermission::ViewChannel) {
    permissions.revoke_all();
}
```

无 ViewChannel → 全部位清零，保证"看不到就什么都不给"。

**批量路径**——不做 revoke_all。

[members_can_see_channel()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L28-L57) 仅调用 `has_channel_permission(ViewChannel)` 取布尔值，即使无 ViewChannel 也不清零其它位。

**影响**：对可见性判定无影响（只查 ViewChannel 位），但 `cached_member_perms` 中缓存的 PermissionValue 会保留多余的位，若未来复用该缓存做更细粒度的权限判断则可能产生偏差。

### 7.6 差异四：遗留的 dead_code 方法

[BulkDatabasePermissionQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L10-L25) 上挂着三个 `#[allow(dead_code)]` 方法：

| 方法 | 行 | 说明 |
|------|----|------|
| [get_default_channel_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L136-L153) | L136 | 未被调用，实际逻辑内联在 calculate_members_permissions 中 |
| [get_channel_type()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L155-L167) | L155 | 标记 `dead_code, deprecated`，未被调用 |
| [get_channel_role_overrides()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L170-L182) | L170 | 未被调用，实际逻辑内联 |

这些方法看起来是早期设计时试图复用 `PermissionQuery` trait 的遗留物，后来批量路径改为直接内联逻辑，但方法未清理。它们不影响运行时行为，但会让阅读者误以为批量路径也走 trait 抽象。

### 7.7 差异总结表

| 维度 | 单用户路径（impl.rs） | 批量路径（bulk_permissions.rs） | 对可见性结果的影响 |
|------|----------------------|-------------------------------|-------------------|
| 超时 restrict 次数 | 2 次（服务器级 + 频道级） | 1 次（仅服务器级） | 通常无影响（ViewChannel 已被保留） |
| can_publish 撤销 Speak/Video | 有 | **无** | 无影响（非 ViewChannel 位） |
| can_receive 撤销 Listen | 有 | **无** | 无影响（非 ViewChannel 位） |
| 无 ViewChannel → revoke_all | 有 | **无** | 无影响（仅查 ViewChannel 位） |
| 特权/所有者短路 | 有 | 有（等价） | 无 |
| 非成员 → 0 | 有 | 有（等价） | 无 |
| 角色排序 | rank 降序 apply | rank 降序 apply（等价） | 无 |

### 7.8 结论

批量路径是单用户路径的**简化版**：它省略了所有与 `ViewChannel` 无关的限制步骤（语音限制、频道级超时兜底、revoke_all），因为其唯一用途是判定可见性。在当前仅调用 `members_can_see_channel()` 的场景下，这些省略不会导致可见性误判。

但需要注意两点风险：
1. **语义不一致**：同一个超时成员，经单用户路径和批量路径算出的 `PermissionValue` 可能不同（批量路径多出 Speak/Video/Listen 等位）。
2. **复用风险**：若未来直接复用 `cached_member_perms` 做更细粒度判断（如能否发言），会因缺少语音限制和频道级超时兜底而产生越权。
