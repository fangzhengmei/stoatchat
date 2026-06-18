# 聊天服务器（Server）创建初始化代码走向分析

## 0. 总体入口流程

创建服务器的 HTTP 入口位于：

[server_create.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/delta/src/routes/servers/server_create.rs)

核心调用链：

```
POST /servers/create
  → create_server()
    1. 校验 Bot 用户不能创建服务器
    2. 校验全局服务器创建限制
    3. 校验用户输入（validator）
    4. user.can_acquire_server(db) —— 检查用户服务器数量上限
    5. Server::create(db, data, &user, true) —— 创建服务器 + 默认频道
    6. Member::create(db, &server, &user, Some(channels)) —— 所有者加入 + 事件广播
    7. 返回 CreateServerLegacyResponse
```

---

## 1. 默认频道

### 1.1 创建位置

默认频道在 `Server::create` 函数内被创建，当 `create_default_channels = true` 时触发：

[servers/model.rs#L142-L188](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/servers/model.rs#L142-L188)

关键代码片段：

```rust
let channels: Vec<Channel> = if create_default_channels {
    vec![
        Channel::create_server_channel(
            db,
            &mut server,
            DataCreateServerChannel {
                channel_type: v0::LegacyServerChannelType::Text,
                name: "General".to_string(),
                ..Default::default()
            },
            false,  // update_server = false，稍后一次性保存
        )
        .await?,
    ]
} else {
    vec![]
};

// 将频道 ID 收集到服务器对象中
server.channels = channels.iter().map(|c| c.id().to_string()).collect();
db.insert_server(&server).await?;
```

### 1.2 频道具体构造

`Channel::create_server_channel` 的实现：

[channels/model.rs#L189-L253](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/channels/model.rs#L189-L253)

默认频道（Text 类型）被构造成 `Channel::TextChannel` 变体，字段如下：

| 字段 | 默认值 |
|------|--------|
| `id` | ULID 随机生成 |
| `server` | 所属服务器 ID |
| `name` | `"General"` |
| `description` | `None` |
| `icon` | `None` |
| `last_message_id` | `None` |
| `default_permissions` | `None`（继承服务器 default_permissions） |
| `role_permissions` | 空 HashMap |
| `nsfw` | `false` |
| `voice` | `None` |
| `slowmode` | `None` |

注意：由于 `update_server = false`，此时不会单独发送 `ChannelCreate` 事件，也不会单独更新 Server 的 channels 列表——这些操作被推迟到服务器整体插入后统一处理。

### 1.3 持久化

频道通过 `db.insert_channel(&channel)` 持久化到数据库，然后服务器整体通过 `db.insert_server(&server)` 持久化。

---

## 2. 默认角色

### 2.1 关键点：服务器初始化时**不创建**任何角色对象

`Server::create` 中的初始化为：

```rust
roles: HashMap::new(),      // 空的角色表
default_permissions: *DEFAULT_PERMISSION_SERVER as i64,
```

也就是说，新创建的服务器**没有内置角色**（如 "Owner"、"Member" 等 Role 对象），而是通过以下两种机制实现权限分层：

1. **所有者身份判定**（见第 3 节）—— `server.owner` 字段直接决定最高权限
2. **默认服务器权限**（`default_permissions`）——所有成员（非所有者）的基础权限

### 2.2 默认服务器权限常量

[channel.rs#L149-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/permissions/src/models/channel.rs#L149-L155)

```rust
pub static DEFAULT_PERMISSION_SERVER: Lazy<u64> = Lazy::new(|| {
    DEFAULT_PERMISSION.add(
        ChannelPermission::React
            + ChannelPermission::ChangeNickname
            + ChannelPermission::ChangeAvatar,
    )
});
```

其中 `DEFAULT_PERMISSION` 展开后的完整权限位为：

| 权限位 | 说明 |
|--------|------|
| `ViewChannel` (1<<20) | 查看频道 |
| `ReadMessageHistory` (1<<21) | 读取历史消息 |
| `SendMessage` (1<<22) | 发送消息 |
| `InviteOthers` (1<<25) | 创建邀请 |
| `SendEmbeds` (1<<26) | 发送嵌入内容 |
| `UploadFiles` (1<<27) | 上传文件 |
| `Connect` (1<<30) | 连接语音 |
| `Speak` (1<<31) | 语音发言 |
| `Listen` (1<<36) | 收听语音 |
| `Video` (1<<32) | 共享视频 |
| `React` (1<<29) | 消息反应 |
| `ChangeNickname` (1<<10) | 修改昵称 |
| `ChangeAvatar` (1<<12) | 修改头像 |

**不包含**：管理类权限（ManageChannel、ManageServer、ManagePermissions、ManageRole、ManageCustomisation、KickMembers、BanMembers 等），这些需要通过后续创建角色手动授予。

### 2.3 角色如何被创建（对比参考）

角色的创建是独立的 API，**不在服务器初始化流程中**：

- 路由：[roles_create.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/delta/src/routes/servers/roles_create.rs)
- 模型方法：[servers/model.rs#L316-L341](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/servers/model.rs#L316-L341) 的 `Role::create`

`Role::create` 中的默认值：
- `permissions: Default::default()` —— 即 `{ a: 0, d: 0 }`，新角色默认无任何权限，需要后续调用 `set_role_permission` 授予
- `rank: server.roles.len() as i64` —— 新角色排在最后

---

## 3. 所有者授权

### 3.1 所有者字段设置

在 `Server::create` 中：

```rust
let mut server = Server {
    id: ulid::Ulid::new().to_string(),
    owner: owner.id.to_string(),  // <-- 所有者 ID 直接写入
    ...
};
```

### 3.2 所有者权限判定逻辑

权限计算的核心函数位于：

[impl.rs#L48-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/permissions/src/impl.rs#L48-L78) 的 `calculate_server_permissions`

```rust
pub async fn calculate_server_permissions<P: PermissionQuery>(query: &mut P) -> PermissionValue {
    // ① 首先检查：特权用户 或 服务器所有者
    if query.are_we_privileged().await || query.are_we_server_owner().await {
        return ChannelPermission::GrantAllSafe.into();
    }
    // ② 非成员：无权限
    if !query.are_we_a_member().await {
        return 0_u64.into();
    }
    // ③ 普通成员：默认权限 + 角色覆盖
    let mut permissions: PermissionValue = query.get_default_server_permissions().await.into();
    for role_override in query.get_our_server_role_overrides().await {
        permissions.apply(role_override);
    }
    ...
}
```

**所有者判定实现**：

[permissions.rs#L115-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/util/permissions.rs#L115-L122)

```rust
async fn are_we_server_owner(&mut self) -> bool {
    if let Some(server) = &self.server {
        server.owner == self.perspective.id
    } else {
        false
    }
}
```

逻辑非常直接：比较 `server.owner == 当前用户.id`。

### 3.3 GrantAllSafe 的具体含义

[channel.rs#L108-L113](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/permissions/src/models/channel.rs#L108-L113)

```rust
GrantAllSafe = 0x000F_FFFF_FFFF_FFFF,  // 低 52 位全 1
GrantAll    = u64::MAX,                // 64 位全 1
```

所有者被授予的是 `GrantAllSafe`，即所有已定义权限位（防止 JavaScript 52 位安全整数问题）。

### 3.4 频道级权限中的所有者检查

[impl.rs#L119-L144](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/permissions/src/impl.rs#L119-L144)

```rust
ChannelType::ServerChannel => {
    query.set_server_from_channel().await;
    // 服务器所有者直接在频道内拥有所有权限
    if query.are_we_server_owner().await {
        ChannelPermission::GrantAllSafe.into()
    } else if query.are_we_a_member().await {
        ... // 普通成员权限计算
    }
}
```

结论：所有者身份是**独立于角色系统**的特殊授权路径，不依赖任何 Role 对象，直接在计算链的最前端短路返回全部权限。

### 3.5 所有者成员记录

`Member::create` 被调用时会为所有者创建一条 Member 记录：

[server_members/model.rs#L102-L201](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/model.rs#L102-L201)

```rust
let mut member = Member {
    id: MemberCompositeKey {
        server: server.id.to_string(),
        user: user.id.to_string(),
    },
    ..Default::default()   // roles: vec![], can_publish: true, can_receive: true
};
```

注意：所有者的 `roles` 字段也是空数组——它**不通过角色获得权限**，而是通过 owner 字段。

---

## 4. 事件广播

### 4.1 事件广播的三层抽象

```
应用层代码
  ↓ EventV1.p() / .private() / .server()
[client.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/events/client.rs#L366-L405)
  ↓ redis_kiss::publish(channel, event)
Redis Pub/Sub 消息队列
  ↓ Fred subscriber 订阅
[websocket.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs)  Bonfire 服务
  ↓ WebSocket send
前端客户端
```

### 4.2 事件发布方法

[client.rs#L366-L405](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/events/client.rs#L366-L405)

| 方法 | Redis Channel 格式 | 用途 |
|------|-------------------|------|
| `.p(channel)` | 原样使用 | 通用发布，频道名由调用方决定 |
| `.private(id)` | `{id}!` | 发送给单个用户的私有事件 |
| `.server(id)` | `{id}u` | 服务器成员事件（给服务器上所有成员） |
| `.global()` | `"global"` | 全局广播 |
| `.p_user(id, db)` | `{id}` + 遍历成员关系 | 用户事件，同时转发到用户所在所有服务器 |

底层实现（debug 模式下额外打印日志）：

```rust
pub async fn p(self, channel: String) {
    #[cfg(not(debug_assertions))]
    redis_kiss::p(channel, self).await;
    #[cfg(debug_assertions)]
    redis_kiss::publish(channel, self).await.unwrap();
}
```

### 4.3 服务器创建过程中的事件

在 `Member::create` 中触发**两个关键事件**：

[server_members/model.rs#L164-L184](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/model.rs#L164-L184)

#### 事件一：`ServerMemberJoin` —— 广播给服务器频道

```rust
EventV1::ServerMemberJoin {
    id: server.id.clone(),
    user: user.id.clone(),
    member: member.clone().into(),
}
.p(server.id.clone())  // 发布到 Redis Channel = 服务器 ID
.await;
```

作用：通知服务器上**已有在线成员**"有新用户加入"。

> **注意**：在服务器刚创建的场景下，这条消息发布时还没有任何人订阅该服务器频道（创建者本人也还没订阅，订阅发生在收到 ServerCreate 之后），因此实际上**无人能收到**。该事件的实际消费场景是"加入已有服务器"——其他已在线成员通过此事件更新成员列表。详见 6.6 节。

#### 事件二：`ServerCreate` —— 私有事件仅发给创建者

```rust
EventV1::ServerCreate {
    id: server.id.clone(),
    server: server.clone().into(),
    channels: channels.clone().into_iter().map(|channel| channel.into()).collect(),
    emojis: emojis.into_iter().map(|emoji| emoji.into()).collect(),
    voice_states,
}
.private(user.id.clone())  // 发布到 Redis Channel = "{user_id}!"
.await;
```

内容包含：
- 服务器完整信息
- 所有频道列表（即默认 General 频道）
- 所有表情（新服务器为空）
- 语音状态列表

客户端收到 `ServerCreate` 后即可将新服务器渲染到 UI 中。

### 4.4 系统消息（可选）

[server_members/model.rs#L186-L198](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/model.rs#L186-L198)

如果服务器配置了 `system_messages.user_joined` 频道，还会发送一条系统消息：

```rust
SystemMessage::UserJoined { id: user.id.clone() }
    .into_message(id.to_string())
    .send_without_notifications(db, None, None, false, false, false)
    .await
    .ok();
```

但对于刚创建的服务器，`system_messages` 初始为 `None`，所以这条不会触发。

### 4.5 Bonfire 订阅侧

Bonfire 是独立的 WebSocket 服务器进程：

[main.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/main.rs)

每个客户端连接后：

1. **认证阶段**：[websocket.rs#L71-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs#L71-L101) 读取 token，通过 `User::from_token` 验证
2. **Ready 阶段**：[websocket.rs#L118-L130](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs#L118-L130) 拉取用户所有服务器、频道、成员等数据，发送 `Authenticated` + `Ready` 事件
3. **订阅设置**：[websocket.rs#L221-L300](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs#L221-L300) 使用 Fred 客户端建立 Redis Pub/Sub 订阅

订阅管理逻辑（`state.apply_state()`）：
- **Reset**：清空并重新订阅所有频道
- **Change**：增量添加/移除订阅（add / remove）

用户至少订阅的频道包括：
- `{user_id}!` —— 私有事件频道（对应 `.private()` 方法），因此 `ServerCreate` 事件能送达
- `{server_id}` —— 用户加入的每个服务器（对应 `.p(server_id)`），因此 `ServerMemberJoin` 能送达
- `{server_id}u` —— 服务器成员频道（对应 `.server()` 方法）

### 4.6 完整时序图

```
用户 POST /servers/create
  │
  ▼
create_server() [delta]
  │
  ├─► Server::create()
  │     ├─ 构建 Server 对象 (owner=user.id, default_permissions=DEFAULT_PERMISSION_SERVER)
  │     ├─► Channel::create_server_channel()
  │     │     └─ 创建 TextChannel "General" → db.insert_channel()
  │     ├─ 收集频道 ID 到 server.channels
  │     └─ db.insert_server()
  │
  └─► Member::create()
        ├─ 检查 ban / 是否已在服务器
        ├─ 构建 Member 对象 → db.insert_or_merge_member()
        │
        ├─ ① EventV1::ServerMemberJoin
        │     └─ .p(server_id) → Redis Channel = server_id
        │
        ├─ ② EventV1::ServerCreate
        │     └─ .private(user_id) → Redis Channel = user_id!
        │
        └─ (可选) SystemMessage::UserJoined
              └─ 仅当 server.system_messages.user_joined 配置时

                      │
                      ▼
                Redis Pub/Sub
                      │
                      ▼
            Bonfire subscriber [bonfire]
                      │
                      ├─ 匹配到用户 WebSocket 连接
                      └─ WebSocket.send(编码后的事件)
                              │
                              ▼
                        前端客户端
                         ├─ ServerCreate: 在 UI 中渲染新服务器 + General 频道
                         └─ ServerMemberJoin: 更新服务器成员列表
```

---

## 5. 四段关系总结

| 模块 | 核心机制 | 关键代码位置 |
|------|---------|-------------|
| **默认频道** | Server::create 内硬编码创建一个名为 "General" 的 TextChannel，通过 update_server=false 与服务器批量持久化 | [servers/model.rs#L167-L186](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/servers/model.rs#L167-L186) |
| **默认角色** | 无预设 Role 对象；通过 `default_permissions` 字段给所有成员基础权限；角色系统由后续 API 独立创建 | [servers/model.rs#L155](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/servers/model.rs#L155)、[channel.rs#L149-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/permissions/src/models/channel.rs#L149-L155) |
| **所有者授权** | `server.owner == perspective.id` 的身份比较，在权限计算最前端短路返回 `GrantAllSafe`（低 52 位全 1），不依赖任何角色 | [impl.rs#L49-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/permissions/src/impl.rs#L49-L52)、[permissions.rs#L115-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/util/permissions.rs#L115-L122) |
| **事件广播** | 发布侧：EventV1.p() 写入 Redis Pub/Sub；订阅侧：Bonfire 用 Fred subscriber 订阅，通过 WebSocket 推送至匹配连接的客户端 | [client.rs#L366-L405](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/events/client.rs#L366-L405)、[server_members/model.rs#L164-L184](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/model.rs#L164-L184)、[websocket.rs#L221-L300](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs#L221-L300) |

---

## 6. 事件进入网关后的完整处理

本章聚焦于事件从 Redis Pub/Sub 进入 Bonfire WebSocket 网关之后的完整处理链。

### 6.1 网关接收流程总览

[websocket.rs#L323-L398](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs#L323-L398)

```
Redis Message (Fred subscriber.message_rx.recv())
  │
  ├─ 反序列化：根据 REDIS_PAYLOAD_TYPE (Json/Msgpack/Bincode) 解码为 EventV1
  │
  ├─ 特殊处理：EventV1::Auth → 可能转换为 Logout 事件
  │
  └─ 常规事件 → state.handle_incoming_event_v1(db, &mut event)
        │
        ├─ 返回 false → 丢弃，不发送给客户端
        └─ 返回 true  → write.lock().await.send(config.encode(&event))
                        → 通过 WebSocket 发送给前端
```

### 6.2 缓存写入

`State::handle_incoming_event_v1` 是事件处理的核心入口：

[impl.rs#L433-L696](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L433-L696)

#### 6.2.1 缓存数据结构

[state.rs#L35-L61](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/state.rs#L35-L61)

每个 WebSocket 连接维护独立的 `Cache`：

```rust
pub struct Cache {
    pub user_id: String,
    pub is_bot: bool,
    pub users: HashMap<String, User>,        // 用户信息缓存
    pub channels: HashMap<String, Channel>,  // 频道信息缓存
    pub members: HashMap<String, Member>,    // 成员信息缓存 (key = server_id)
    pub servers: HashMap<String, Server>,    // 服务器信息缓存
    pub seen_events: LruCache<String, ()>,   // 已见事件去重 (LRU, 容量 20)
}
```

注意：`members` 的 key 是 **server_id**，不是 member_id，即每个服务器缓存当前用户自己的成员身份。

#### 6.2.2 ServerCreate 事件触发的缓存写入

[impl.rs#L518-L548](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L518-L548)

```rust
EventV1::ServerCreate { id, server, channels, .. } => {
    // ① 订阅服务器事件频道
    self.insert_subscription(id.clone()).await;
    if self.cache.is_bot {
        self.insert_subscription(format!("{}u", id)).await;
    }

    // ② 服务器对象写入缓存
    self.cache.servers.insert(id.clone(), server.clone().into());

    // ③ 生成本地 Member 记录并写入缓存
    let member = Member {
        id: MemberCompositeKey {
            server: server.id.clone(),
            user: self.cache.user_id.clone(),
        },
        ..Default::default()   // roles=[], can_publish=true, can_receive=true
    };
    self.cache.members.insert(id.clone(), member);

    // ④ 所有频道写入缓存
    for channel in channels {
        self.cache
            .channels
            .insert(channel.id().to_string(), channel.clone().into());
    }

    // ⑤ 标记需要重算服务器权限
    queue_server = Some(id.clone());
}
```

关键细节：
- 这里写入的 Member 对象是**本地构造**的（`Default::default()`），不直接来自数据库
- 但由于服务器创建者必然是所有者，默认值（空角色列表 + 全语音权限）完全足够
- `queue_server = Some(id)` 触发后续的权限重算

#### 6.2.3 其他事件的缓存写入对比

| 事件 | 缓存操作 |
|------|---------|
| `ChannelCreate` | `cache.channels.insert(id, channel)` + 订阅 |
| `ChannelUpdate` | `channel.apply_options(data)` 增量更新 + 权限变化时可能转为 Create/Delete |
| `ChannelDelete` | `cache.channels.remove(id)` + 取消订阅 |
| `ServerUpdate` | `server.apply_options(data)` 增量更新；若 `default_permissions` 变化则标记重算 |
| `ServerDelete` | `cache.servers.remove(id)` + 移除所有关联频道缓存 + 移除成员缓存 |
| `ServerMemberUpdate` | 仅当 `id.user == self.cache.user_id` 时更新本地 member；若 roles 变化则标记重算 |
| `ServerRoleUpdate/Delete` | 更新 server.roles；若本用户拥有该角色则标记重算 |
| `UserRelationship` | `cache.users.insert(id, user)` + 按需订阅/退订用户事件 |

### 6.3 成员记录生成

成员记录生成有两条路径：**数据库持久化路径**和**网关本地缓存路径**。

#### 6.3.1 数据库路径：insert_or_merge_member

[server_members/model.rs#L117-L127](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/model.rs#L117-L127)

```rust
let mut member = Member {
    id: MemberCompositeKey {
        server: server.id.to_string(),
        user: user.id.to_string(),
    },
    ..Default::default()
};

if let Some(updated) = db.insert_or_merge_member(&member).await? {
    member = updated;
}
```

MongoDB 实现的关键逻辑：

[ops/mongodb.rs#L16-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/ops/mongodb.rs#L16-L52)

```rust
async fn insert_or_merge_member(&self, member: &Member) -> Result<Option<Member>> {
    // ① 查找是否有 pending_deletion_at 标记的"软删除"记录
    let existing = find_one({
        "_id.server": &member.id.server,
        "_id.user": &member.id.user,
        "pending_deletion_at": {"$exists": true}
    });

    if existing.is_ok_and(|x| x.is_some()) {
        // ② 有软删除记录 → 复活：更新 joined_at，移除 pending_deletion_at
        //    返回更新后的完整 Member（保留 timeout 等原有字段）
        find_one_and_update(
            {"$set": {"joined_at": ...}, "$unset": {"pending_deletion_at": ""}}
        ).return_document(After)
    } else {
        // ③ 无软删除记录 → 直接插入新文档，返回 None 表示使用传入的 member
        insert_one(&member)
    }
}
```

软删除（soft_delete_member）与复活机制的配合：

[ops/mongodb.rs#L269-L301](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/ops/mongodb.rs#L269-L301)

- 用户被踢出时，如果仍在 timeout 禁言期 → **不直接删除**，而是标记 `pending_deletion_at = timeout 到期时间`，同时清空 `joined_at/avatar/nickname/roles`
- 用户重新加入时 → 通过 `insert_or_merge_member` 找到软删除记录，重置 `joined_at` 并移除标记，保留 timeout（禁言期继续有效）
- crond 定时任务定期清理 `pending_deletion_at < now` 的记录

这就是 `muted_member_rejoin` 测试用例验证的场景。

#### 6.3.2 网关本地路径：ServerCreate 时构造

在网关侧，`ServerCreate` 事件处理时会在本地缓存中直接构造 Member：

[impl.rs#L532-L539](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L532-L539)

```rust
let member = Member {
    id: MemberCompositeKey {
        server: server.id.clone(),
        user: self.cache.user_id.clone(),
    },
    ..Default::default()
};
self.cache.members.insert(id.clone(), member);
```

这与 Ready 阶段从数据库批量拉取的行为一致：

[impl.rs#L219-L223](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L219-L223)

```rust
self.cache.members = members
    .iter()
    .cloned()
    .map(|x| (x.id.server.clone(), x))  // key = server_id
    .collect();
```

### 6.4 权限重算

#### 6.4.1 触发时机

`handle_incoming_event_v1` 通过三个变量决定是否需要重算：

```rust
let mut queue_server = None;  // 需要重算整个服务器
let mut queue_add = None;     // 需要新增单个订阅
let mut queue_remove = None;  // 需要移除单个订阅
```

以下事件会设置 `queue_server = Some(server_id)`：

| 事件 | 触发条件 |
|------|---------|
| `ServerCreate` | **无条件触发** |
| `ServerUpdate` | `data.default_permissions.is_some()`（服务器默认权限变更） |
| `ServerMemberUpdate` | `id.user == self.user_id` 且 (`data.roles.is_some()` 或移除了 Roles) |
| `ServerRoleUpdate` | 本用户拥有该角色（`member.roles.contains(role_id)`）且权限或 rank 变化 |
| `ServerRoleDelete` | 本用户拥有该角色 |

#### 6.4.2 重算逻辑：recalculate_server

[impl.rs#L334-L401](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L334-L401)

```rust
pub async fn recalculate_server(&mut self, db: &Database, id: &str, event: &mut EventV1) {
    if let Some(server) = self.cache.servers.get(id) {
        // ① 遍历所有已缓存的、属于该服务器的频道
        let mut added_channels = vec![];
        let mut removed_channels = vec![];

        for (channel_id, channel) in &self.cache.channels {
            if channel.server() == Some(id) {
                // ② 用当前用户的最新权限重新检查 ViewChannel
                if self.cache.can_view_channel(db, channel).await {
                    added_channels.push(channel_id.clone());
                } else {
                    removed_channels.push(channel_id.clone());
                }
            }
        }

        // ③ 处理权限被移除的频道
        for id in removed_channels {
            self.remove_subscription(&id).await;
            self.cache.channels.remove(&id);
            bulk_events.push(EventV1::ChannelDelete { id });
        }

        // ④ 处理权限新增的频道
        for id in added_channels {
            self.insert_subscription(id).await;
        }

        // ⑤ 检查服务器声明但缓存未知的频道（增量拉取）
        let known_ids = server.channels.iter().cloned().collect::<HashSet<_>>();
        let unknowns = known_ids.difference(&cached_channel_ids).collect::<Vec<_>>();

        if !unknowns.is_empty() {
            if let Ok(channels) = db.fetch_channels(&unknowns).await {
                let viewable = self.cache.filter_accessible_channels(db, channels).await;
                for channel in viewable {
                    self.cache.channels.insert(channel.id().to_string(), channel.clone());
                    self.insert_subscription(channel.id().to_string()).await;
                    bulk_events.push(EventV1::ChannelCreate(channel.into()));
                }
            }
        }

        // ⑥ 如果有增删事件，用 Bulk 包装替换原事件
        if !bulk_events.is_empty() {
            let mut new_event = EventV1::Bulk { v: bulk_events };
            std::mem::swap(&mut new_event, event);
            if let EventV1::Bulk { v } = event {
                v.push(new_event);  // 原 ServerCreate 也被塞进 Bulk 里
            }
        }
    }
}
```

#### 6.4.3 权限判定：can_view_channel

[impl.rs#L19-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L19-L46)

```rust
pub async fn can_view_channel(&self, db: &Database, channel: &Channel) -> bool {
    match &channel {
        Channel::TextChannel { server, .. } => {
            // 用缓存中的 member 和 server 构造权限查询
            let mut query = DatabasePermissionQuery::new(
                db, self.users.get(&self.user_id).unwrap()
            ).channel(channel);

            if let Some(member) = self.members.get(server) {
                query = query.member(member);
            }
            if let Some(server) = self.servers.get(server) {
                query = query.server(server);
            }

            calculate_channel_permissions(&mut query)
                .await
                .has_channel_permission(ChannelPermission::ViewChannel)
        }
        _ => true,  // DM/Group/SavedMessages 不需要额外判定
    }
}
```

对于服务器创建场景，由于用户是所有者，`calculate_channel_permissions` 会短路返回 `GrantAllSafe`，所有频道都可见。但该机制在角色变更、默认权限变更等场景至关重要。

### 6.5 频道订阅变化

#### 6.5.1 订阅状态管理

[state.rs#L10-L21](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/state.rs#L10-L21)

```rust
pub enum SubscriptionStateChange {
    None,
    Reset,                          // 清空所有订阅后重建
    Change { add: Vec<String>, remove: Vec<String> },  // 增量变更
}
```

`State` 维护两个同步状态：
- `state: SubscriptionStateChange` —— 待提交的变更（逻辑层）
- `subscribed: Arc<RwLock<HashSet<String>>>` —— 当前已生效的订阅集合（Redis 层）

#### 6.5.2 插入/移除订阅

[state.rs#L164-L208](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/state.rs#L164-L208)

```rust
pub async fn insert_subscription(&mut self, subscription: String) {
    let mut subscribed = self.subscribed.write().await;
    if subscribed.contains(&subscription) {
        return;  // 去重：已订阅则忽略
    }

    // 更新待提交的变更
    match &mut self.state {
        SubscriptionStateChange::None => {
            self.state = SubscriptionStateChange::Change {
                add: vec![subscription.clone()], remove: vec![]
            };
        }
        SubscriptionStateChange::Change { add, .. } => {
            add.push(subscription.clone());
        }
        SubscriptionStateChange::Reset => {}  // Reset 模式下无需记录增量
    }

    subscribed.insert(subscription);
}
```

核心设计：**先写 subscribed（立即生效的内存集合），再写 state（在下一轮 apply_state 时同步到 Redis）**。这防止了重复订阅检查与 Redis 实际订阅之间的竞态。

#### 6.5.3 订阅同步到 Redis

[state.rs#L103-L151](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/state.rs#L103-L151) +
[websocket.rs#L267-L306](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs#L267-L306)

`apply_state()` 在 listener 主循环的每次迭代开始时被调用：

```rust
loop {
    // ① 先同步订阅变更
    match state.apply_state().await {
        SubscriptionStateChange::Reset => {
            subscriber.unsubscribe_all().await;
            for id in subscribed.iter() {
                subscriber.subscribe(id).await;
            }
        }
        SubscriptionStateChange::Change { add, remove } => {
            for id in remove { subscriber.unsubscribe(id).await; }
            for id in add { subscriber.subscribe(id).await; }
        }
        SubscriptionStateChange::None => {}
    }

    // ② 再等待消息
    select! {
        message = message_rx.recv() => { ... },
        ...
    }
}
```

#### 6.5.4 服务器创建场景的订阅增量

服务器创建时触发的订阅变化（均写入内存 HashSet + 暂存到 state.Change，下一轮 `apply_state()` 才真正调用 Fred subscribe）：

| 步骤 | 代码位置 | 订阅动作 |
|------|---------|---------|
| 插入服务器本身 | [impl.rs#L525](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L525) | `insert_subscription(server_id)` |
| Bot 额外订阅成员频道 | [impl.rs#L527-L529](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L527-L529) | `insert_subscription("{server_id}u")` |
| recalculate_server 处理可见频道 | [impl.rs#L358-L360](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L358-L360) | `insert_subscription(channel_id)` 对每个可见频道 |

经过下一轮 `apply_state()` 同步后，用户 Redis 订阅集合中新增：
- `{server_id}` —— 服务器级别事件（ServerUpdate、ServerMemberJoin 等）
- `{server_id}u`（仅 Bot）—— 服务器成员事件
- `{channel_id}` —— 对每个可访问的频道（如 General 频道）

**注意**：由于订阅是延迟生效的，服务器创建时发布的 `ServerMemberJoin`（发布到 `{server_id}`）创建者本人收不到——它发布时订阅还没同步到 Redis。详见 6.6.2 节。

#### 6.5.5 active_servers 与成员频道

[state.rs#L103-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/state.rs#L103-L135)

`apply_state` 还额外维护 `active_servers`（LRU 时间缓存，900 秒 TTL，容量 5）：

```rust
if !self.cache.is_bot {
    // 根据 active_servers 的到期状态决定是否订阅/退订 {server_id}u
    // Valid → Subscribe("{server_id}u")
    // Expired → Unsubscribe("{server_id}u")
}
```

设计意图：普通用户只在"活跃服务器"上订阅成员级事件频道，避免订阅过多频道浪费资源；Bot 用户则订阅所有服务器的成员频道。

### 6.6 ServerMemberJoin 事件的空处理逻辑

服务器创建流程中会按以下顺序发布两个事件：

[server_members/model.rs#L164-L184](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/model.rs#L164-L184)

1. **先** `EventV1::ServerMemberJoin` → `.p(server_id)` 发布到 Redis Channel = `{server_id}`
2. **后** `EventV1::ServerCreate` → `.private(user_id)` 发布到 Redis Channel = `{user_id}!`

#### 6.6.1 服务器创建前用户的订阅状态

[state.rs#L77-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/state.rs#L77-L101)

WebSocket 连接建立时 `State::from(user)` 初始化的订阅集合：

```rust
let mut subscribed = HashSet::new();
let private_topic = format!("{}!", user.id);  // "{user_id}!"
subscribed.insert(private_topic.clone());      // 私有事件频道
subscribed.insert(user.id.clone());            // 用户自身事件频道
```

之后 Ready 阶段 `generate_ready_payload` 会追加订阅：

[impl.rs#L287-L305](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L287-L305)

- 所有好友/相关用户的 `{user_id}`
- 用户已加入的**已有**服务器的 `{server_id}`
- Bot 额外订阅 `{server_id}u`
- 用户可访问的**已有**频道的 `{channel_id}`

**关键点**：新建的服务器 ID 在 Ready 阶段不存在，所以**用户此时没有订阅 `{新server_id}`**。

#### 6.6.2 哪些连接能收到 ServerMemberJoin

Redis Pub/Sub 是**即时广播**，只有发布时已订阅该 channel 的客户端才能收到消息，不保留历史。

| 事件 | 发布到的 Channel | 发布时谁已订阅 | 实际结果 |
|------|-----------------|--------------|---------|
| `ServerMemberJoin` | `{新server_id}` | **没人**（服务器刚创建，没有任何成员在线，创建者本人也还没订阅） | 消息丢失，无人收到 |
| `ServerCreate` | `{user_id}!` | 创建者本人（Ready 阶段就已订阅私有频道） | 创建者正常收到 |

创建者对 `{新server_id}` 的订阅发生在**收到 ServerCreate 之后**：

[impl.rs#L523-L525](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L523-L525)

```rust
// 在 handle_incoming_event_v1 处理 ServerCreate 时执行
self.insert_subscription(id.clone()).await;
```

但 `insert_subscription` 只是修改内存的 `subscribed` HashSet 和暂存 `state.Change`，真正调用 Fred 的 `subscriber.subscribe()` 要等到下一轮循环的 `apply_state()`：

[websocket.rs#L268-L306](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/websocket.rs#L268-L306)

```rust
'out: loop {
    // ① 先 apply_state，把暂存的订阅变更同步到 Redis
    match state.apply_state().await {
        SubscriptionStateChange::Change { add, .. } => {
            for id in add { subscriber.subscribe(id).await; }
        }
        ...
    }
    // ② 再等待消息
    select! { message = message_rx.recv() => { ... } }
}
```

所以时序是：
1. ServerMemberJoin 发布到 `{server_id}` → 无人订阅 → 丢失
2. ServerCreate 发布到 `{user_id}!` → 创建者已订阅 → 收到
3. 创建者处理 ServerCreate → `insert_subscription(server_id)`（暂存）
4. 下一轮循环 → `apply_state()` → 真正 `subscriber.subscribe(server_id)`
5. 此后该服务器上的新事件创建者才能收到

因此，在服务器创建场景中，**创建者本人根本不会收到 ServerMemberJoin**。注释中说"We will always receive ServerCreate when joining a new server"的含义不是"两个事件都会收到所以 ServerMemberJoin 冗余"，而是**"当你作为新成员加入时，你收到的是 ServerCreate，而非 ServerMemberJoin"**。

#### 6.6.3 空处理 + 返回 true 的真实含义

[impl.rs#L564-L566](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L564-L566)

```rust
EventV1::ServerMemberJoin { .. } => {
    // We will always receive ServerCreate when joining a new server.
}
// ...
true  // 函数末尾默认返回 true
```

`handle_incoming_event_v1` 的返回值语义：
- `return true` → 事件通过 WebSocket **转发给前端客户端**
- `return false` → **丢弃**，不发送给客户端

ServerMemberJoin 的空匹配 `{}` + 默认返回 `true` 合起来的含义是：

| 行为 | 原因 |
|------|------|
| **不更新网关本地缓存** | `cache.members` 的 key 是 server_id，value 是**本用户自己**的 Member 记录（用于权限计算）。别的成员加入不影响本用户的缓存 |
| **不修改订阅状态** | 其他成员加入不改变本用户对任何频道/服务器的订阅 |
| **仍转发给前端** | 前端 UI 需要展示成员列表变化（比如侧边栏 +1、系统通知），这些是前端渲染逻辑，网关无需参与 |

#### 6.6.4 ServerMemberJoin 的实际消费场景

ServerMemberJoin 的目标接收者**不是新成员本人**，而是服务器上的**其他已有在线成员**：

**场景：用户 B 通过邀请加入已存在的服务器 S**
- 服务器 S 已存在，用户 A 是 S 的成员且在线
- 用户 A 在 Ready 阶段已订阅 `{S_id}`
- B 加入时，Member::create 发布 ServerMemberJoin 到 `{S_id}`
- A 收到 ServerMemberJoin 事件
- 网关侧：空处理（不更新 A 的缓存），但转发给 A 的前端
- A 的前端 UI：成员列表显示 B 加入

**场景：用户 A 创建新服务器 S（即服务器创建流程）**
- 创建时没有其他成员
- ServerMemberJoin 发布到 `{S_id}` 但无人订阅 → 无人收到
- 只有 ServerCreate 通过私有频道到达 A 本人

#### 6.6.5 成员记录不重复插入的保障（数据库层）

注意：这与"事件重复处理"是不同层次的问题，不要混淆。成员记录的唯一性由数据库层三层保障：

| 层级 | 机制 | 位置 |
|------|------|------|
| **第一层：应用前置校验** | `Member::create` 先 `fetch_member(server, user)`，已存在则返回 `Error::AlreadyInServer` | [server_members/model.rs#L109-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/model.rs#L109-L115) |
| **第二层：insert_or_merge 幂等** | 软删除记录存在则复活而非重复插入；返回 `Some(updated)` 表示走了复活分支 | [ops/mongodb.rs#L16-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/core/database/src/models/server_members/ops/mongodb.rs#L16-L52) |
| **第三层：数据库复合主键** | MongoDB 的 `_id` 是 `{server, user}` 复合对象，数据库层面保证唯一 | schema 定义 |

#### 6.6.6 补充：UserUpdate 事件的 seen_events 去重

对于 `EventV1::UserUpdate`，才存在真正的"同连接重复接收"问题——因为用户变更事件可能同时通过 `{user_id}`（被关注者的频道）和 `{server_id}`（用户所在服务器频道）等多条路径转发。网关层通过 LRU 去重：

[impl.rs#L643-L653](file:///d:/fz/0601-2/solo-dogfeeding/code/38-backend/crates/bonfire/src/events/impl.rs#L643-L653)

```rust
EventV1::UserUpdate { event_id, .. } => {
    if let Some(id) = event_id {
        if self.cache.seen_events.contains(id) {
            return false;  // 已处理过，丢弃，也不转发给前端
        }
        self.cache.seen_events.put(id.to_string(), ());
    }
    *event_id = None;  // 清除后再发给客户端，减少前端负担
}
```

`seen_events` 是容量 20 的 `LruCache<String, ()>`，只存 event_id 不存值。这是**真正的事件去重**，与 ServerMemberJoin 的"空处理"（不更新缓存但仍转发）语义完全不同。

---

## 7. 网关侧完整时序图（服务器创建场景）

```
Redis Pub/Sub
  │
  ├─ Channel: "{user_id}!" → EventV1::ServerCreate
  │     │
  │     ▼
  │   Fred subscriber.message_rx.recv()
  │     │
  │     ▼
  │   serde_json / rmp_serde / bincode 反序列化为 EventV1
  │     │
  │     ▼
  │   state.handle_incoming_event_v1(db, &mut event)
  │     │
  │     ├─ 匹配 EventV1::ServerCreate
  │     │   ├─ insert_subscription(server_id)
  │     │   │   └─ subscribed HashSet 插入 + state.Change.add 记录
  │     │   ├─ (Bot 额外) insert_subscription("{server_id}u")
  │     │   ├─ cache.servers.insert(server_id, server)
  │     │   ├─ cache.members.insert(server_id, Member::default())
  │     │   ├─ for channel in channels: cache.channels.insert(channel_id, channel)
  │     │   └─ queue_server = Some(server_id)
  │     │
  │     ├─ queue_server.is_some() → self.recalculate_server(db, server_id, event)
  │     │   ├─ 遍历缓存频道，can_view_channel 重算权限
  │     │   ├─ 对每个可见频道: insert_subscription(channel_id)
  │     │   ├─ 拉取服务器声明但缓存未知的频道 (db.fetch_channels)
  │     │   ├─ filter_accessible_channels 过滤
  │     │   ├─ 新增可见频道写入 cache.channels + 订阅 + ChannelCreate 事件
  │     │   └─ 若有新增/删除事件 → mem::swap 为 EventV1::Bulk { [ChannelCreate..., 原 ServerCreate] }
  │     │
  │     └─ 返回 true（需要发送给客户端）
  │
  ├─ Channel: "{server_id}" → EventV1::ServerMemberJoin
  │     │
  │     ▼
  │   state.handle_incoming_event_v1(db, &mut event)
  │     │
  │     └─ 匹配 EventV1::ServerMemberJoin { .. } => { /* 空处理 */ }
  │        └─ 返回 true（仍转发给客户端用于 UI 更新成员列表）
  │
  ▼
WebSocket send(config.encode(&event))
  │
  ▼
前端客户端
  ├─ ServerCreate (或 Bulk 包) → 渲染服务器、频道、表情、语音状态
  └─ ServerMemberJoin → (可选) 更新成员侧边栏
```

