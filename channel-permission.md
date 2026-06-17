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

批量路径有**两个实际调用场景**，其中一个场景内部又分为两条分支，仅其中一条分支使用了可见性过滤：

| 调用方 | 场景 | 是否使用 members_can_see_channel() 做可见性过滤 |
|--------|------|----------------------------------------------|
| [Message::create_from_api()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/models/messages/model.rs#L514-L520) | 发送消息时过滤 @提及目标（第二级过滤） | **是**，对所有被提及用户统一过滤 |
| [pushd/mass_mention.rs @everyone 分支](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L135-L188) | @everyone 全员推送通知 | **否**，直接对全服务器成员推送，不调用可见性过滤 |
| [pushd/mass_mention.rs @role 分支](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L189-L240) | @role 角色提及推送通知 | **是**，只给能看到频道的角色成员推送 |

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

---

## 8. 成员可见性计算的所有业务场景

成员可见性判定（"某成员能否看到某频道"）贯穿于后端多个子系统，每条路径使用的计算方式、过滤时机和目的各不相同。本节按业务场景逐一拆解。

### 8.1 场景总览

| 场景 | 细分 | 使用路径 | 入口模块 | 是否做频道可见性过滤 | 核心目的 |
|------|------|---------|----------|---------------------|---------|
| 发送消息时过滤 @提及目标 | 单用户提及（@成员） | 批量路径 | delta → Message::create_from_api | **是**（第二级过滤） | 防止在隐藏频道中 @ 了看不到该频道的成员 |
| 群体推送通知 | @everyone / @here 全员提及 | 批量路径（不调用可见性过滤） | pushd/mass_mention.rs @everyone 分支 | **否**，直接给全服务器成员加未读+推送 | 离线成员收到全员提及推送 |
| 群体推送通知 | @role 角色提及 | 批量路径（调用可见性过滤） | pushd/mass_mention.rs @role 分支 | **是**，只给能看到频道的角色成员推送 | 离线角色成员收到推送，信息不外泄 |
| 拉取服务器详情时过滤频道列表 | — | 单用户路径 | delta/routes/servers/server_fetch | **是**（对每个频道单独计算） | 返回给客户端的频道列表只包含可见频道 |
| WebSocket Ready 时过滤频道订阅 | — | 单用户路径 | bonfire/events/impl (filter_accessible_channels) | **是**（对每个频道单独计算） | 只订阅用户可见的频道事件主题 |
| 权限变更时重新计算订阅 | — | 单用户路径 | bonfire/events/impl (recalculate_server) | **是**（对每个频道单独计算） | 权限变化后自动订阅/退订对应频道 |

---

### 8.2 消息通知的两层架构总览

在展开各场景之前，需要先理解后端消息通知的**两层架构**。所有消息的未读写入和推送发送都涉及两个异步处理层，三条路径（普通用户提及、全员提及、角色提及）在这两层中的行为各不相同。

```
消息发送 (Message::send)
       │
       ▼
  ack 任务队列 (tasks::ack::queue_message)
       │
       ├── 第一层：ack worker (handle_ack_event)
       │    ├── 给 recipients 中的用户加未读 (add_mention_to_unread)
       │    ├── 给 recipients 发普通推送 (amqp.message_sent)
       │    └── 如果是 mass mention → 发 mass_mention 事件 (amqp.mass_mention_message_sent)
       │
       └── 第二层：pushd mass_mention 消费者
            ├── @everyone 分支：全服成员加未读 + 全服离线成员发推送（不做可见性过滤）
            └── @role 分支：可见性过滤后的角色成员发推送（不加未读）
```

**第一层：ack 任务层** — 入口在 [tasks::ack::handle_ack_event()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/tasks/ack.rs#L84-L210)

所有消息都会经过这一层。它处理：
- 给 `recipients`（收件人列表）中的每个用户**逐个加未读**：`db.add_mention_to_unread(channel, user, message_ids)`
- 给 `recipients` 中的用户发**普通推送**：`amqp.message_sent(recipients, push)`
- 如果消息包含 `@everyone` 或 `@role`（即 `contains_mass_push_mention()` 返回 true），则额外通过 `amqp.mass_mention_message_sent()` 发一个 **mass mention 事件**到独立队列，由 pushd 进一步处理

**第二层：pushd mass_mention 层** — 入口在 [MassMessageConsumer::consume()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L95-L245)

只有 `@everyone` 和 `@role` 消息才会到达这一层。这一层的目的是处理"收件人数量巨大、无法在 ack 任务中同步处理"的群体提及。

关键注意点：
- **ack 层的 recipients**（TextChannel 场景下）等于 `message.mentions`，即**单独 @ 的用户**，不包含全员成员或角色成员
- 因此 ack 层只给少量被单独 @ 的用户加未读和发推送，而大量的全员/角色成员需要在 pushd 层处理
- 但 pushd 层中，@everyone 和 @role 的处理方式存在**多处不一致**，详见 8.3 节

---

### 8.3 场景一：普通用户提及（@成员）

入口位置：消息写入阶段在 [Message::create_from_api()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/models/messages/model.rs#L491-L534)，未读与推送阶段在 [ack::handle_ack_event()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/tasks/ack.rs#L120-L185)

#### 8.3.1 消息写入时的两级过滤

当消息内容包含用户提及（`@某成员`）时，在 `TextChannel` 场景下会经历**两级过滤**：

##### 第一级：服务器成员身份过滤

```rust
let valid_members = db.fetch_members(server.as_str(), &mentions_vec[..]).await;
if let Ok(valid_members) = valid_members {
    let valid_mentions = HashSet::<&String, RandomState>::from_iter(
        valid_members.iter().map(|m| &m.id.user),
    );
    user_mentions.retain(|m| valid_mentions.contains(m));
```

先把所有不是该服务器成员的被提及用户剔除。比如 A 服务器成员 @ 了 B 服务器的成员，这一级就把 B 过滤掉。

##### 第二级：频道可见性过滤

```rust
if !user_mentions.is_empty() {
    let member_channel_view_perms =
        BulkDatabasePermissionQuery::from_server_id(db, server)
            .await
            .channel(&channel)
            .members(&valid_members)
            .members_can_see_channel()
            .await;

    user_mentions
        .retain(|m| *member_channel_view_perms.get(m).unwrap_or(&false));
}
```

调用批量路径 [members_can_see_channel()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/util/bulk_permissions.rs#L28-L57)，对所有剩余的被提及成员一次性计算"能否看到当前频道"。不能看到的，从 `user_mentions` 集合中移除，最终消息的 `mentions` 字段只保留能看到该频道的用户 ID。

这两级过滤的业务目的：
- **防止信息泄露**：如果在私有频道中 @ 了一个看不到该频道的成员，该成员不应出现在消息的 `mentions` 列表中
- **未读计数准确**：只有能看到频道的成员才会在后续 ack 任务中被加未读

#### 8.3.2 ack 任务层的未读写入与推送发送

消息写入数据库后，通过 `Message::send()` 调用 `tasks::ack::queue_message()` 将消息入队，由 ack worker 异步处理。

**未读写入**（[ack.rs#L120-L152](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/tasks/ack.rs#L120-L152)）：

```rust
let mut users: HashSet<&String> = HashSet::new();
messages.iter().for_each(|(_, _, recipents, _)| {
    users.extend(recipents.iter());
});

for user in users {
    let message_ids: Vec<String> = messages
        .iter()
        .filter_map(|(_, message, recipients, _)| {
            if recipients.contains(user) {
                Some(message.id.clone())
            } else { None }
        })
        .collect();

    if !message_ids.is_empty() {
        db.add_mention_to_unread(channel, user, &message_ids).await?;
    }
}
```

遍历所有 recipients（即 `message.mentions` 中的用户 ID），逐个调用 `add_mention_to_unread()` 给每个用户加未读。注意是**逐个用户写入**，不是批量写入。

**推送发送**（[ack.rs#L156-L180](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/tasks/ack.rs#L156-L180)）：

```rust
for (push, message, recipients, silenced) in messages {
    if *silenced || push.is_none()
        || (recipients.is_empty() && !message.contains_mass_push_mention())
    {
        continue;
    }
    amqp.message_sent(recipients.clone(), push.clone().unwrap()).await;
}
```

通过 `amqp.message_sent()` 给所有 recipients 发推送通知。

#### 8.3.3 普通提及路径总结

| 阶段 | 处理内容 | 是否做可见性过滤 | 未读写入方式 |
|------|---------|----------------|-------------|
| 消息写入 | 两级过滤后将结果写入 message.mentions 字段 | **是**（成员身份 + 频道可见性） | 不在此阶段写 |
| ack 任务层 | 给 recipients（= message.mentions）加未读 + 发普通推送 | 否（依赖写入阶段已过滤） | 逐个用户调用 add_mention_to_unread |
| pushd mass_mention 层 | 不经过（普通提及不触发 mass mention 事件） | — | — |

#### 8.3.4 单元测试验证

该逻辑在 [message_send.rs 的 message_mention_constraints 测试](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/delta/src/routes/channels/message_send.rs#L208-L403) 中被完整覆盖了三种情况：
1. 被提及者不是服务器成员 → 被过滤
2. 被提及者是服务器成员但频道不可见 → 被过滤
3. 被提及者拥有能看到频道的角色 → 保留

测试断言三次发送的 `message.mentions` 字段分别为空、空、非空。

---

### 8.4 场景二：@everyone / @role 群体推送通知

入口位置：ack 层在 [ack.rs#L182-L205](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/database/src/tasks/ack.rs#L182-L205)，pushd 层在 [mass_mention.rs::consume()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L95-L245)

当消息标记了 `MentionsEveryone` 或包含角色提及时，ack 层会通过 `amqp.mass_mention_message_sent()` 触发第二层处理。此场景下 @everyone 和 @role 的处理逻辑存在**多处关键差异**，包括是否做可见性过滤、是否写未读等。

#### 8.4.1 ack 层的共同处理

@everyone 和 @role 在 ack 层的处理是**完全相同**的：

1. **给单独 @ 的用户加未读和发推送**：recipients = `message.mentions`（只包含被单独 @ 的用户，**不包含**全员成员或角色成员），这部分处理与普通提及完全一致
2. **检测到 mass mention 后触发第二层**：如果 `contains_mass_push_mention()` 返回 true（即有 MentionsEveryone 标志或 role_mentions 非空），则收集到 `mass_mentions` 向量中，最后通过 `amqp.mass_mention_message_sent(server, mass_mentions)` 发送到独立队列

关键区别都发生在 **pushd mass_mention 层**。

#### 8.4.2 分支 A：@everyone —— 不做可见性过滤、直接加未读

代码位置：[mass_mention.rs#L135-L188](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L135-L188)

```rust
if flags.has(MessageFlags::MentionsEveryone) {
    let mut db_query = self.db.fetch_all_members_chunked(&payload.server_id).await?;
    loop {
        // 分块拉取全服成员
        let (userids, ack_chnl, exhausted, online_users, existing_mentions, _) =
            next_chunk(&mut db_query, &push).await?;

        // 直接给全服成员批量加未读，不检查可见性
        self.db
            .add_mention_to_many_unreads(push.channel.id(), &userids, &ack_chnl)
            .await;

        // 过滤掉在线用户和已有提及，剩余的发推送
        let target_users: Vec<String> = userids
            .iter()
            .filter(|uid| !online_users.contains(*uid) && !existing_mentions.contains(*uid))
            .cloned()
            .collect();

        self.fire_notification_for_users(&push, &target_users).await?;
    }
}
```

**未读处理**：调用 `add_mention_to_many_unreads()` **批量**给全服所有成员加未读。这是一个批量数据库操作（MongoDB 中是 `update_many`），比逐个用户写入高效得多。

**推送处理**：过滤掉**在线用户**（在线用户通过 WebSocket 实时收到消息，不需要推送）和**已被单独 @ 的用户**（这些用户在 ack 层已经收到过普通推送，避免重复推送），剩余的离线用户收到推送。

**可见性过滤**：**完全不做**。`BulkDatabasePermissionQuery` 对象虽然在函数开头构造了，但在 @everyone 分支中从未被使用。

#### 8.4.3 分支 B：@role —— 做可见性过滤、不加未读

代码位置：[mass_mention.rs#L189-L240](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L189-L240)

```rust
} else if let Some(roles) = &push.message.role_mentions {
    let mut role_members =
        self.db.fetch_all_members_with_roles_chunked(&payload.server_id, roles).await?;
    while !exhausted {
        // 分块拉取拥有该角色的成员
        let (chunk, _, exhausted, online_users, existing_mentions, query) =
            next_chunk(&mut role_members, &push).await?;

        // 先做可见性过滤：只保留能看到频道的成员
        let mut q = query.clone().members(&chunk);
        let viewing_members: Vec<String> = q
            .members_can_see_channel()
            .await
            .iter()
            .filter_map(|(uid, viewable)| {
                if *viewable && !existing_mentions.contains(uid) {
                    Some(uid.clone())
                } else { None }
            })
            .collect();

        // 再过滤掉在线用户，剩余的发推送
        let targets: Vec<String> = viewing_members
            .into_iter()
            .filter(|uid| !online_users.contains(uid))
            .collect();

        self.fire_notification_for_users(&push, &targets).await?;
    }
}
```

**未读处理**：**完全不加未读**。@role 分支中没有任何 `add_mention_to_*` 的调用。这是与 @everyone 分支最关键的差异之一——角色成员虽然能收到推送通知，但他们的频道未读计数中的 mentions 数量**不会增加**。

**推送处理**：先做可见性过滤，再过滤掉在线用户和已单独 @ 的用户，剩余的收到推送。

**可见性过滤**：**做**。调用 `members_can_see_channel()` 批量计算每个角色成员对该频道的可见性，只有 `viewable == true` 的成员才保留。

#### 8.4.4 @everyone 与 @role 的完整差异对比

| 维度 | @everyone 全员提及 | @role 角色提及 |
|------|-------------------|--------------|
| pushd 层代码位置 | [mass_mention.rs#L135-L188](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L135-L188) | [mass_mention.rs#L189-L240](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/daemons/pushd/src/consumers/inbound/mass_mention.rs#L189-L240) |
| 成员来源 | `fetch_all_members_chunked(server_id)` 全服成员 | `fetch_all_members_with_roles_chunked(server_id, roles)` 指定角色成员 |
| 是否调用 `members_can_see_channel()` | **否**，完全不调用 | **是**，对每个分块成员调用 |
| pushd 层未读写入 | `add_mention_to_many_unreads()` 批量加未读 | **完全不加未读**（无任何 add_mention 调用） |
| 未读写入方式 | 批量（update_many） | — |
| 可见性过滤 | 无（全服成员一视同仁） | 可见性过滤后，只保留 `viewable == true` 的成员 |
| 推送过滤条件 | `不在线 && 不在 existing_mentions` | `能看到频道 && 不在线 && 不在 existing_mentions` |
| 查询对象（query）使用 | 只用于初始化，不在 @everyone 分支中调用 | 克隆后调用 `query.members(chunk).members_can_see_channel()` |
| ack 层是否处理 | 是（给单独 @ 的用户加未读+推送） | 是（给单独 @ 的用户加未读+推送） |

#### 8.4.5 处理逻辑为什么会有这样的差异

从代码结构推断，这一差异源于**预期语义的不同**：

- **@role 的语义是"通知拥有该角色的人"**，但角色本身不保证频道可见性。比如 @Moderator 可能在某个内部频道中提到了管理员，但并非所有管理员都能看到该内部频道（可能有些管理员属于不同部门）。因此必须在推送前做可见性过滤，确保消息内容不泄露给看不到频道的角色成员。

- **@everyone 的语义是"通知全服务器所有人"**，典型使用场景是服务器公告。如果 @everyone 也按频道可见性过滤，则公告频道的可见性与 @everyone 的"全员通知"意图矛盾——如果管理者希望某个公告让所有人看到，就会把公告频道设为全员可见；如果希望隐藏公告频道，本就不应在隐藏频道里使用 @everyone。代码选择了"信任使用者的设置、不做额外过滤、保证通知到达率"的策略，代价是隐藏频道中使用 @everyone 可能导致信息泄露给看不到该频道的成员。

至于 **@role 不加未读**，这更可能是一个**实现遗漏**而非有意设计：
- @everyone 分支的 `add_mention_to_many_unreads` 调用紧邻在成员拉取之后
- @role 分支在可见性过滤后有了 `viewing_members` 列表，理论上同样可以调用 `add_mention_to_many_unreads` 给这些可见成员加未读，但代码中没有这一步
- 从功能一致性角度，被 @role 提及的用户应该和被 @everyone 提及的用户一样，在未读计数中体现为提及未读

这一差异导致的实际效果是：收到 @role 推送的用户点开频道后，看到的是普通未读（last_id 未读），而非红色的提及未读（mentions 未读）。

#### 8.4.6 信息泄露风险说明

如果在**私有频道**（默认 deny ViewChannel，仅特定角色 allow）中发送包含敏感内容的 @everyone 消息，由于全员提及路径不做可见性过滤：
1. 所有服务器成员都会在该频道下增加一个未读计数（即使他们在客户端左侧列表中看不到该频道条目）
2. 不在线的所有服务器成员都会收到包含消息正文的推送通知（敏感内容直接泄露到移动端/桌面端）

在使用 @everyone 功能时应意识到该行为。

---

### 8.5 场景三：拉取服务器详情时过滤频道列表

入口位置：[server_fetch.rs::fetch()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/delta/src/routes/servers/server_fetch.rs#L27-L39)

当客户端请求服务器详情并带上 `include_channels=true` 参数时，服务端需要返回该用户可见的所有频道。

```rust
let all_channels = db.fetch_channels(&server.channels).await?;
let mut visible_channels: Vec<v0::Channel> = vec![];

for channel in all_channels {
    let mut channel_query = query.clone().channel(&channel);
    if calculate_channel_permissions(&mut channel_query)
        .await
        .has_channel_permission(ChannelPermission::ViewChannel)
    {
        visible_channels.push(channel.into());
    }
}
```

**使用单用户路径** [calculate_channel_permissions()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/core/permissions/src/impl.rs#L81-L147)，对每个频道单独计算一次完整权限，然后检查 `ViewChannel` 位。

由于这里是单用户视角且频道数量通常远小于成员数量（一个服务器最多几十上百个频道），使用完整的单用户路径是合理的——不需要批量优化，且完整路径确保了超时、语音限制、revoke_all 等全部约束都被正确应用。

### 8.6 场景四：WebSocket Ready 时过滤频道订阅

入口位置：[bonfire/events/impl.rs::generate_ready_payload()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/bonfire/src/events/impl.rs#L140-L145)

用户连接 WebSocket、发送认证后，服务端返回 Ready 事件，其中包含所有可见频道的列表以及订阅对应事件主题。

```rust
let mut channels = db.find_direct_messages(&user.id).await?;
channels.append(&mut db.fetch_channels(&channel_ids).await?);

// Filter server channels by permission.
let channels = self.cache.filter_accessible_channels(db, channels).await;
```

内部调用 [Cache::filter_accessible_channels()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/bonfire/src/events/impl.rs#L49-L62)，再逐个调用 [Cache::can_view_channel()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/bonfire/src/events/impl.rs#L21-L46)。

`can_view_channel()` 本质上也是单用户路径的封装：

```rust
let mut query = DatabasePermissionQuery::new(db, self.users.get(&self.user_id).unwrap())
    .channel(channel);
// ...
calculate_channel_permissions(&mut query)
    .await
    .has_channel_permission(ChannelPermission::ViewChannel)
```

对非 TextChannel 类型（DM、Group、SavedMessages）直接返回 `true`，因为这些频道的成员集合本身就是白名单，不需要额外权限判断。

### 8.7 场景五：权限变更时重新计算订阅

入口位置：[bonfire/events/impl.rs::recalculate_server()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/bonfire/src/events/impl.rs#L335-L399)

当服务器或频道的权限配置发生变更时（服务器默认权限修改、角色权限修改、频道默认权限修改、频道角色覆盖修改、成员角色变更等），bonfire 会重新计算该服务器下所有频道对当前用户的可见性，并动态订阅/退订对应频道的事件主题。

触发来源分布在 [handle_incoming_event_v1()](file:///d:/fz/0601-2/solo-dogfeeding/code/19-backend/crates/bonfire/src/events/impl.rs#L434-L684) 中，会设置 `queue_server` 标记的事件包括：

| 事件 | 触发条件 |
|------|---------|
| `EventV1::ServerUpdate` | `data.default_permissions.is_some()` |
| `EventV1::ServerRoleUpdate` | `data.rank.is_some() \|\| data.permissions.is_some()` |
| `EventV1::ServerMemberUpdate` | `data.roles.is_some() \|\| data.timeout.is_some()` |
| `EventV1::ChannelUpdate` | 对 TextChannel 始终触发 |

`recalculate_server()` 的核心逻辑：

```rust
for (channel_id, channel) in &self.cache.channels {
    if channel.server() == Some(id) {
        if self.cache.can_view_channel(db, channel).await {
            added_channels.push(channel_id.clone());
        } else {
            removed_channels.push(channel_id.clone());
        }
    }
}
```

对缓存中属于该服务器的每个频道，调用单用户路径的 `can_view_channel()`，然后对比变化：
- 从不可见变为可见 → 订阅该频道的事件主题，并向客户端发送 `ChannelCreate` 事件
- 从可见变为不可见 → 退订该频道的事件主题，并向客户端发送 `ChannelDelete` 事件

这保证了客户端实时感知权限变更对频道可见性的影响，不需要刷新页面。

### 8.8 两条路径的选择策略总结

代码库在不同场景下选择批量路径还是单用户路径，遵循以下原则：

| 原则 | 批量路径（含可见性过滤的调用） | 单用户路径 |
|------|------------------------------|-----------|
| **适用场景** | N 个成员 × 1 个频道 | 1 个用户 × N 个频道 |
| **性能优势** | 一次数据库查询批量拉取成员、角色、权限，避免 N 次往返 | 完整权限语义准确，适合需要精确判断多种权限位的场景 |
| **语义完整性** | 省略了语音限制、频道级超时兜底、revoke_all（见第 7 节） | 所有限制步骤完整执行 |
| **调用方** | Message::create_from_api（过滤 @提及，L514-L520）、pushd/mass_mention 中 **仅 @role 分支**（L189-L240）。注意：@everyone 分支（L135-L188）虽走批量流水线，但完全不调用可见性过滤。 | server_fetch（频道列表）、bonfire Ready、bonfire 权限重算 |

实际调用 `members_can_see_channel()` 做可见性过滤的两处场景（Message @提及过滤、@role 推送）都只需要判断 `ViewChannel` 这一位，因此批量路径省略的限制步骤不影响结果正确性。若未来有场景需要批量判断 `SendMessage`、`Speak` 等更细粒度权限，则不能直接复用当前批量路径，需要补齐缺失的限制步骤。

