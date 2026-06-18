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

#### 事件一：`ServerMemberJoin` —— 广播给所有服务器成员

```rust
EventV1::ServerMemberJoin {
    id: server.id.clone(),
    user: user.id.clone(),
    member: member.clone().into(),
}
.p(server.id.clone())  // 发布到 Redis Channel = 服务器 ID
.await;
```

作用：通知服务器上现有成员"有新用户加入"。但在服务器刚创建时，除了所有者还没有其他成员，这个事件实质上只有所有者能收到（但紧接着还有私有事件，所以这个主要是为加入流程统一设计）。

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
