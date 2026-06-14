# 用户关系状态协作脉络

本文档按代码执行顺序拆解好友请求、屏蔽与用户关系状态的对称维护方式，并说明并发接受或拒绝请求时的最终一致性保障。

---

## 1. 数据模型层

### 1.1 关系状态枚举

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L82-L91)

```rust
pub enum RelationshipStatus {
    None,         // 无关系
    User,         // 自己
    Friend,       // 好友
    Outgoing,     // 我方发出的好友请求（待对方接受）
    Incoming,     // 对方发来的好友请求（待我方接受）
    Blocked,      // 我方屏蔽了对方
    BlockedOther, // 对方屏蔽了我方
}
```

这 7 种状态构成了双方关系的完整状态机。注意 `Blocked` 和 `BlockedOther` 是成对出现的不对称状态。

### 1.2 关系存储结构

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L93-L98)

```rust
pub struct Relationship {
    #[serde(rename = "_id")]
    pub id: String,           // 对方用户 ID
    pub status: RelationshipStatus,
}
```

每个用户文档内嵌一个 `relations: Option<Vec<Relationship>>` 数组，保存该用户对其他所有用户的关系视图。

**设计要点**：关系是**双向冗余存储**的。A 对 B 的关系存在 A 的文档里，B 对 A 的关系存在 B 的文档里。这种设计使得读取时无需联表查询，但写入时必须同时更新双方文档，否则会出现数据不一致。

---

## 2. 核心协作流程（按调用顺序）

所有关系变更最终都汇聚到两个底层方法：`set_relationship`（单方写入）和 `apply_relationship`（双方对称写入 + 事件推送）。

### 2.1 第一步：单方写入 set_relationship

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L464-L492)

```rust
pub async fn set_relationship(
    &mut self,
    db: &Database,
    user_b: &User,
    status: RelationshipStatus,
) -> Result<()> {
    db.set_relationship(&self.id, &user_b.id, &status).await?;  // ① 写数据库

    // ② 同步更新内存中的 self 对象
    if let RelationshipStatus::None | RelationshipStatus::User = status {
        if let Some(relations) = &mut self.relations {
            relations.retain(|relation| relation.id != user_b.id); // None/User 状态直接移除
        }
    } else {
        let relation = Relationship { id: user_b.id.to_string(), status };
        if let Some(relations) = &mut self.relations {
            relations.retain(|relation| relation.id != user_b.id); // 先移除旧条目
            relations.push(relation);                              // 再追加新条目
        } else {
            self.relations = Some(vec![relation]);
        }
    }
    Ok(())
}
```

该方法职责单一：
1. 调用数据库抽象层 `db.set_relationship` 持久化
2. 同步更新当前 `&mut self` 的内存状态

关键点：`None` 和 `User` 状态不存储在 `relations` 数组中，直接移除条目即可。

### 2.2 第二步：双方对称写入 apply_relationship

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L494-L520)

```rust
pub async fn apply_relationship(
    &mut self,
    db: &Database,
    target: &mut User,
    local: RelationshipStatus,   // self 对 target 的新状态
    remote: RelationshipStatus,  // target 对 self 的新状态
) -> Result<()> {
    // ① 先更新 target 的关系（remote 视角）
    target.set_relationship(db, self, remote).await?;
    // ② 再更新 self 的关系（local 视角）
    self.set_relationship(db, target, local).await?;

    // ③ 向 target 推送事件：target 看到 self 的关系变化
    EventV1::UserRelationship {
        id: target.id.clone(),
        user: self.clone().into(db, Some(&*target)).await,
    }
    .private(target.id.clone())
    .await;

    // ④ 向 self 推送事件：self 看到 target 的关系变化
    EventV1::UserRelationship {
        id: self.id.clone(),
        user: target.clone().into(db, Some(&*self)).await,
    }
    .private(self.id.clone())
    .await;

    Ok(())
}
```

这是**所有关系操作的对称维护中枢**。任何关系变更都会：

1. 调用两次 `set_relationship`，分别写入双方文档
2. 通过 WebSocket 向双方各推送一条 `UserRelationship` 事件，保证客户端实时同步

**状态对称映射表**（`local` → `remote`）：

| 操作 | self 看到 target (local) | target 看到 self (remote) |
|------|--------------------------|---------------------------|
| A 发好友请求给 B | `Outgoing` | `Incoming` |
| B 接受请求 | `Friend` | `Friend` |
| A 拒绝/删除 B | `None` | `None` |
| A 屏蔽 B | `Blocked` | `BlockedOther` |
| A 取消屏蔽（双方无其他屏蔽） | `None` | `None` |
| A 取消屏蔽（B 仍屏蔽 A） | `BlockedOther` | `Blocked` |
| A、B 互相屏蔽 | `Blocked` | `Blocked` |

---

## 3. 四大业务操作详解

### 3.1 发送好友请求 / 接受请求：add_friend

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L522-L579)

API 入口：
- 发送请求：[send_friend_request.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/send_friend_request.rs) → `POST /friend`
- 接受请求：[add_friend.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/add_friend.rs) → `PUT /<target>/friend`

两个 API 都调用同一个 `add_friend` 方法，通过**当前关系状态**来区分是"发送新请求"还是"接受已有请求"。

```rust
pub async fn add_friend(
    &mut self,
    db: &Database,
    amqp: &AMQP,
    target: &mut User,
) -> Result<()> {
    match self.relationship_with(&target.id) {
        // ── 错误分支 ──────────────────────────────────────
        RelationshipStatus::User           => Err(NoEffect),
        RelationshipStatus::Friend         => Err(AlreadyFriends),
        RelationshipStatus::Outgoing       => Err(AlreadySentRequest),
        RelationshipStatus::Blocked        => Err(Blocked),
        RelationshipStatus::BlockedOther   => Err(BlockedByOther),

        // ── 接受对方发来的请求 ─────────────────────────────
        RelationshipStatus::Incoming => {
            _ = amqp.friend_request_accepted(self, target).await;  // 推送通知
            self.apply_relationship(db, target,
                RelationshipStatus::Friend,   // self 视角：变成好友
                RelationshipStatus::Friend    // target 视角：变成好友
            ).await
        }

        // ── 发送新的好友请求 ───────────────────────────────
        RelationshipStatus::None => {
            // 限频检查： outgoing 数量不能超限
            let count = self.relations.as_ref()
                .map(|r| r.iter().filter(|x| matches!(x.status, Outgoing)).count())
                .unwrap_or_default();
            if count >= self.limits().await.outgoing_friend_requests {
                return Err(TooManyPendingFriendRequests { max: ... });
            }

            _ = amqp.friend_request_received(target, self).await;  // 推送通知
            self.apply_relationship(db, target,
                RelationshipStatus::Outgoing,  // self 视角：已发出
                RelationshipStatus::Incoming   // target 视角：收到请求
            ).await
        }
    }
}
```

**执行流程拆解**：

1. API 层从数据库加载 `user`（当前登录用户）和 `target`（目标用户）
2. 调用 `user.add_friend(db, amqp, &mut target)`
3. `add_friend` 根据 `user.relationship_with(target)` 分支：
   - 若当前是 `Incoming` → 说明 target 之前给 user 发过请求 → 直接升级为 `Friend/Friend`
   - 若当前是 `None` → 发送新请求 → 设为 `Outgoing/Incoming`
4. `apply_relationship` 内部依次：
   - `target.set_relationship(db, user, remote)` → 写 target 文档
   - `self.set_relationship(db, target, local)` → 写 self 文档
   - 双方各推一条 WebSocket 事件

### 3.2 拒绝请求 / 删除好友：remove_friend

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L581-L597)

API 入口：[remove_friend.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/remove_friend.rs) → `DELETE /<target>/friend`

```rust
pub async fn remove_friend(&mut self, db: &Database, target: &mut User) -> Result<()> {
    match self.relationship_with(&target.id) {
        // Friend / Outgoing / Incoming 三种状态下都可以"移除"
        RelationshipStatus::Friend
        | RelationshipStatus::Outgoing
        | RelationshipStatus::Incoming => {
            self.apply_relationship(db, target,
                RelationshipStatus::None,    // 双方都清空
                RelationshipStatus::None
            ).await
        }
        _ => Err(create_error!(NoEffect)),
    }
}
```

**语义**：这个方法同时承担"拒绝好友请求"和"解除好友关系"两个动作，区分方式就是看当前状态——`Incoming/Outgoing` 对应请求场景，`Friend` 对应已建立的好友关系。但最终效果相同：双方关系都归零为 `None`。

### 3.3 屏蔽用户：block_user

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L599-L625)

API 入口：[block_user.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/block_user.rs) → `PUT /<target>/block`

```rust
pub async fn block_user(&mut self, db: &Database, target: &mut User) -> Result<()> {
    match self.relationship_with(&target.id) {
        RelationshipStatus::User | RelationshipStatus::Blocked => Err(NoEffect),

        // 特殊分支：对方已经屏蔽了我 → 现在我也屏蔽对方 → 进入双向屏蔽
        RelationshipStatus::BlockedOther => {
            self.apply_relationship(db, target,
                RelationshipStatus::Blocked,     // self 视角：我屏蔽了对方
                RelationshipStatus::Blocked      // target 视角：对方也屏蔽了我
            ).await
        }

        // 常规分支：无/好友/待处理请求 → 我屏蔽对方
        RelationshipStatus::None
        | RelationshipStatus::Friend
        | RelationshipStatus::Incoming
        | RelationshipStatus::Outgoing => {
            self.apply_relationship(db, target,
                RelationshipStatus::Blocked,     // self 视角：我屏蔽了对方
                RelationshipStatus::BlockedOther // target 视角：对方被我屏蔽
            ).await
        }
    }
}
```

**关键点**：
- 屏蔽是**覆盖式**操作，不管理之前是什么关系（None、Friend、Incoming、Outgoing 统统覆盖）
- 当对方已经屏蔽我（`BlockedOther`）时，我再屏蔽对方，双方状态统一变成 `Blocked/Blocked`，进入对称的双向屏蔽态

### 3.4 取消屏蔽：unblock_user

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L627-L653)

API 入口：[unblock_user.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/unblock_user.rs) → `DELETE /<target>/block`

```rust
pub async fn unblock_user(&mut self, db: &Database, target: &mut User) -> Result<()> {
    match self.relationship_with(&target.id) {
        // 只有我当前处于 Blocked 状态（即我确实屏蔽了对方）才能取消
        RelationshipStatus::Blocked => match target.relationship_with(&self.id) {
            // 子分支 1：对方也还在屏蔽我 → 解除我的屏蔽后，我看到对方变成 BlockedOther
            RelationshipStatus::Blocked => {
                self.apply_relationship(db, target,
                    RelationshipStatus::BlockedOther, // self 视角：对方仍屏蔽我
                    RelationshipStatus::Blocked       // target 视角：target 仍屏蔽 self
                ).await
            }
            // 子分支 2：对方没屏蔽我 → 解除后双方归零
            RelationshipStatus::BlockedOther => {
                self.apply_relationship(db, target,
                    RelationshipStatus::None,
                    RelationshipStatus::None
                ).await
            }
            _ => Err(InternalError),
        },
        _ => Err(NoEffect),
    }
}
```

**为什么这里要读取 target 的关系状态？**

取消屏蔽不是简单地把自己改成 `None`，因为需要考虑对方是否也屏蔽了自己：
- 如果是双向屏蔽（双方都是 `Blocked`）→ 我取消后，我这边应该变成 `BlockedOther`（体现对方仍屏蔽我），对方不变
- 如果是单向屏蔽（我 `Blocked`，对方 `BlockedOther`）→ 我取消后，双方都变成 `None`

这体现了关系状态机的**完整性约束**：A 看到的 `BlockedOther` 必须对应 B 看到的 `Blocked`，反之亦然。

---

## 4. 并发场景下的最终一致性保障

### 4.1 问题场景

假设 A 和 B 同时向对方发送好友请求，或者同时接受对方的请求：
- 请求 1：A 进程加载了 A(None)、B(None) → 决定设置 Outgoing/Incoming
- 请求 2：B 进程加载了 A(None)、B(None) → 决定设置 Outgoing/Incoming

如果不加保护，双方最终可能出现 A.relations=[{B, Outgoing}] 且 B.relations=[{A, Outgoing}] 的不对称状态。

### 4.2 第一层保障：MongoDB 原子更新管道

位置：[mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/mongodb.rs#L248-L297)

```rust
async fn set_relationship(
    &self,
    user_id: &str,
    target_id: &str,
    relationship: &RelationshipStatus,
) -> Result<()> {
    if let RelationshipStatus::None = relationship {
        return self.pull_relationship(user_id, target_id).await;
    }

    self.col::<User>(COL)
        .update_one(
            doc! { "_id": user_id },
            vec![doc! {
                "$set": {
                    "relations": {
                        "$concatArrays": [
                            {
                                "$ifNull": [
                                    {
                                        "$filter": {               // ① 先过滤掉 target_id 的旧条目
                                            "input": "$relations",
                                            "cond": { "$ne": ["$$this._id", target_id] }
                                        }
                                    },
                                    []
                                ]
                            },
                            [                                          // ② 再追加新条目
                                { "_id": target_id, "status": format!("{relationship:?}") }
                            ]
                        ]
                    }
                }
            }],
        )
        .await
        .map(|_| ())
        .map_err(|_| create_database_error!("update_one", "user"))
}
```

**这是单次文档更新的原子性保障**。使用 MongoDB 的 Aggregation Pipeline Update（注意 `vec![doc!{...}]` 包裹表示走聚合管道）：

1. `$filter` 先把旧的同 `_id` 条目过滤掉
2. `$concatArrays` 把过滤后的数组和新条目拼接

整个 `$set` 在 MongoDB 中是**单文档原子操作**，不会出现两个并发写入各追加一条导致重复的情况。无论多少个并发请求同时更新同一用户的 `relations`，最终数组里每个 `_id` 都只会有一个条目。

### 4.3 第二层保障：基于状态的幂等性判断

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L529-L578) `add_friend` 的 match 分支

所有业务方法开头都做 `self.relationship_with(&target.id)` 判断，这是基于**内存中加载时的快照**进行的前置校验。虽然这个校验本身不是原子的（加载后、写入前可能被其他请求修改），但它和数据库层的原子更新配合形成了两层防御：

1. **前置校验挡住明显无效的操作**：比如自己加自己、已经是好友再加一次等
2. **数据库原子更新保证最终结果收敛**：即使前置检查基于旧快照，数据库端的 `$filter`+`$concatArrays` 也保证每个用户文档最终是自洽的

### 4.4 最终一致性的收敛机制

以"A 和 B 同时互相发好友请求"为例：

| 时间 | 进程 P_A（A 的视角） | 进程 P_B（B 的视角） |
|------|---------------------|---------------------|
| T1 | 读取 A.relations=[]，B.relations=[] | 读取 A.relations=[]，B.relations=[] |
| T2 | 判断 A→B 为 None → 决定写 Outgoing | 判断 B→A 为 None → 决定写 Outgoing |
| T3 | 写 B 文档：B.relations.push({A, Incoming}) ① | 写 A 文档：A.relations.push({B, Incoming}) ③ |
| T4 | 写 A 文档：A.relations.push({B, Outgoing}) ② | 写 B 文档：B.relations.push({A, Outgoing}) ④ |

最终数据库状态：
- A.relations = [{B, ???}] → 取决于②和③谁最后执行
- B.relations = [{A, ???}] → 取决于①和④谁最后执行

**可能的结果组合**：

| 结果 | A→B | B→A | 一致性 |
|------|-----|-----|--------|
| 1 | Outgoing | Incoming | ✅ 对称 |
| 2 | Incoming | Outgoing | ✅ 对称 |
| 3 | Outgoing | Outgoing | ❌ 不对称 |
| 4 | Incoming | Incoming | ❌ 不对称 |

**结果 3、4 为何可能出现？** 因为 `apply_relationship` 中两次 `set_relationship` 是分开的数据库调用，它们之间不是事务性的。

### 4.5 不对称状态的自愈：后续操作的自然收敛

即使出现结果 3 或 4，系统仍然是**最终一致**的，因为后续任何一方的操作都会重新读取数据库并纠正：

- 如果 A.relations=[{B, Outgoing}] 且 B.relations=[{A, Outgoing}]（结果 3）
  - A 调用 `add_friend` → 看到 `Outgoing` → 返回 `AlreadySentRequest`
  - B 调用 `add_friend` → 看到 `Outgoing` → 返回 `AlreadySentRequest`
  - 任一方调用 `remove_friend` → 双方重置为 `None/None`
  - 任一方再次操作时，会重新基于最新状态执行

- 如果 A.relations=[{B, Incoming}] 且 B.relations=[{A, Incoming}]（结果 4）
  - A 调用 `add_friend` → 看到 `Incoming` → 升级为 `Friend/Friend`，立即修复！
  - B 调用 `add_friend` → 同样修复

**特别是结果 4 的场景**：任何一方执行 `add_friend`（即"接受"）时，`apply_relationship` 会同时把双方都改成 `Friend/Friend`，不对称状态被一举修复。这体现了"接受"操作的收敛性——它会无条件覆盖为对称的 `Friend/Friend` 状态。

### 4.6 小结：一致性保障层次

```
┌─────────────────────────────────────────────┐
│  业务层：match self.relationship_with()     │  ← 前置状态校验，幂等性保护
├─────────────────────────────────────────────┤
│  协调层：apply_relationship                 │  ← 对称调用双方 set_relationship + 推事件
├─────────────────────────────────────────────┤
│  存储层：MongoDB $filter + $concatArrays    │  ← 单文档原子更新，防止重复条目
├─────────────────────────────────────────────┤
│  自愈层：后续操作的状态驱动收敛              │  ← 即使短暂不对称，下次操作自动修复
└─────────────────────────────────────────────┘
```

系统没有使用分布式事务或显式锁，而是通过：
1. 单文档原子性保证每个用户内部状态自洽
2. 对称写入协议尽量保证双方一致
3. 状态机驱动的幂等操作让不对称状态可以自然收敛

这是典型的 AP 系统设计思路——牺牲短暂强一致，换取高可用和高性能，同时保证最终一致。

---

## 5. 关键文件速查表

| 职责 | 文件 |
|------|------|
| 关系枚举 & 业务方法 | [model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs) |
| 数据库抽象 trait | [ops.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops.rs) |
| MongoDB 原子更新实现 | [ops/mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/mongodb.rs) |
| 内存数据库实现（测试用） | [ops/reference.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/reference.rs) |
| API：发送好友请求 | [send_friend_request.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/send_friend_request.rs) |
| API：接受好友请求 | [add_friend.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/add_friend.rs) |
| API：拒绝/删除好友 | [remove_friend.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/remove_friend.rs) |
| API：屏蔽用户 | [block_user.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/block_user.rs) |
| API：取消屏蔽 | [unblock_user.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/unblock_user.rs) |
| 关系→权限映射 | [permissions/impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/permissions/src/impl.rs#L8-L46) |
| 用户序列化（含关系视图转换） | [bridge/v0.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L994-L1205) |
| AMQP 推送通知 | [amqp.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/amqp/amqp.rs#L77-L141) |
| 权限查询桥接 | [permissions.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/permissions.rs#L52-L84) |
| Pushd 消费者：请求已接受 | [fr_accepted.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/daemons/pushd/src/consumers/inbound/fr_accepted.rs) |
| Pushd 消费者：收到请求 | [fr_received.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/daemons/pushd/src/consumers/inbound/fr_received.rs) |

---

## 6. None 状态的独立原子写入路径

### 6.1 两条路径的代码分叉

`set_relationship` 在入口处就做了分叉判断：

位置：[mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/mongodb.rs#L248-L257)

```rust
async fn set_relationship(
    &self,
    user_id: &str,
    target_id: &str,
    relationship: &RelationshipStatus,
) -> Result<()> {
    // ── 分叉点：None 状态走独立路径 ──
    if let RelationshipStatus::None = relationship {
        return self.pull_relationship(user_id, target_id).await;
    }

    // ── 其他状态走主路径：$filter + $concatArrays ──
    self.col::<User>(COL)
        .update_one(
            doc! { "_id": user_id },
            vec![doc! {
                "$set": {
                    "relations": {
                        "$concatArrays": [
                            { "$ifNull": [
                                { "$filter": {
                                    "input": "$relations",
                                    "cond": { "$ne": ["$$this._id", target_id] }
                                }},
                                []
                            ]},
                            [{ "_id": target_id, "status": format!("{relationship:?}") }]
                        ]
                    }
                }
            }],
        )
        .await
        ...
}
```

独立路径 `pull_relationship` 使用 `$pull` 操作符：

位置：[mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/mongodb.rs#L299-L317)

```rust
async fn pull_relationship(&self, user_id: &str, target_id: &str) -> Result<()> {
    self.col::<User>(COL)
        .update_one(
            doc! { "_id": user_id },
            doc! {
                "$pull": {
                    "relations": { "_id": target_id }
                }
            },
        )
        .await
        ...
}
```

### 6.2 为什么需要两条独立路径？

| 路径 | 操作符 | 语义 |
|------|--------|------|
| 主路径 | `$filter` + `$concatArrays` | **替换式写入**：先过滤掉旧条目，再追加新条目。保证数组中每个 `_id` 唯一 |
| None 路径 | `$pull` | **删除式写入**：直接移除匹配的条目。数组中不再保留该 `_id` |

**不合并成一条路径的原因**：
1. `None` 状态在业务语义上是"没有关系"，不应该在 `relations` 数组中占据位置
2. `$pull` 是专门针对数组删除的优化原语，比先过滤再拼接更高效
3. `$pull` 可以一次性删除所有匹配 `_id` 的条目（理论上如果数组中有重复，会全部删掉），而 `$filter` 也能做到但语义上是"先过滤掉，再拼接空"

### 6.3 两条路径并存时如何不互相破坏？

两者本质上是**幂等且可交换**的操作，所以并发执行也不会破坏数据：

**场景 1：并发执行 `set_relationship(None)` 和 `set_relationship(Friend)`**

- 如果 `$pull` 先执行 → 数组中移除 target_id → 然后 `$filter`+`$concatArrays` 过滤（空）+ 追加 Friend → 最终数组：[{target_id, Friend}]
- 如果 `$filter`+`$concatArrays` 先执行 → 数组中变为 [{target_id, Friend}] → 然后 `$pull` 移除 → 最终数组：[]

最终结果取决于谁最后执行，但**两种结果都是自洽的**——要么有关系、要么没有关系，不会出现同 `_id` 多条目，也不会出现状态错乱。

**场景 2：并发执行两个 `set_relationship(None)`**

两个 `$pull` 并发执行，结果都是数组中不包含 target_id，完全幂等。

**场景 3：并发执行 `set_relationship(Outgoing)` 和 `set_relationship(Incoming)`**

都是主路径，谁后执行谁覆盖，最终只有一个条目，状态为后写入的那个。

**关键结论**：两条路径的原子性保证是统一的——都是 MongoDB 单文档原子操作。虽然语义不同（一个是删除、一个是替换），但它们的操作都是针对数组中同一个 `_id` 的，而且都是幂等的，所以并发执行不会产生损坏状态，最多是最终结果取决于时序，这符合 AP 系统的设计目标。

---

## 7. AMQP 消息推送与 WebSocket 实时事件的分工

系统中有**两套并行的通知通道**，分别处理不同的场景：

### 7.1 两套通道的完整流程

```
add_friend / remove_friend / block_user / unblock_user
         │
         ├─→ AMQP 推送（离线通知）
         │     │
         │     ├─ 调用 amqp.friend_request_received() 或 amqp.friend_request_accepted()
         │     ├─ 发布到 RabbitMQ exchange
         │     └─ pushd 守护进程消费 → 查询用户 session → 按平台(APN/FCM/VAPID)分发推送
         │
         └─→ WebSocket 事件（在线实时同步）
               │
               ├─ 构造 EventV1::UserRelationship { id, user }
               ├─ 调用 .private(user_id) 发布到 Redis channel "user_id!"
               └─ bonfire 网关进程广播给该用户的所有在线 WebSocket 连接
```

### 7.2 AMQP 推送通道详解

**发送位置**：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L537) 和 [model.rs#L567](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L567)

```rust
// 接受好友请求时
_ = amqp.friend_request_accepted(self, target).await;

// 发送好友请求时
_ = amqp.friend_request_received(target, self).await;
```

注意：只有 `add_friend` 会发 AMQP 推送，`remove_friend`、`block_user`、`unblock_user` **都不发推送**。

**AMQP 消息结构**：[amqp.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/amqp/amqp.rs#L77-L141)

- `FRAcceptedPayload`：告诉 pushd"某用户接受了你的好友请求"
- `FRReceivedPayload`：告诉 pushd"你收到了一条好友请求"

**Pushd 消费者**：
- [fr_accepted.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/daemons/pushd/src/consumers/inbound/fr_accepted.rs)：处理请求被接受的通知
- [fr_received.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/daemons/pushd/src/consumers/inbound/fr_received.rs)：处理收到新请求的通知

消费者逻辑：查询目标用户的所有 session，对每个有推送订阅（`subscription` 字段）的 session，按其 `endpoint` 类型（apn/fcm/vapid）分发到对应的平台推送队列。

### 7.3 WebSocket 实时事件通道详解

**发送位置**：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L505-L517)

```rust
// 向 target 推送事件：target 看到 self 的关系变化
EventV1::UserRelationship {
    id: target.id.clone(),
    user: self.clone().into(db, Some(&*target)).await,  // 注意：这里用 target 的视角序列化 self
}
.private(target.id.clone())  // 发布到 channel "target_id!"
.await;

// 向 self 推送事件：self 看到 target 的关系变化
EventV1::UserRelationship {
    id: self.id.clone(),
    user: target.clone().into(db, Some(&*self)).await,  // 注意：这里用 self 的视角序列化 target
}
.private(self.id.clone())  // 发布到 channel "self_id!"
.await;
```

**关键点**：
- **每个关系变更推送两条事件**，分别发给双方
- 事件中的 `user` 字段是**从接收者视角序列化**的对方用户对象，包含正确的 `relationship` 字段
- `.private(id)` 是发布到 Redis channel `"{id}!"`，bonfire 网关会广播给该用户的所有在线连接

**事件消费**：[bonfire/impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/bonfire/src/events/impl.rs#L654-L655)

```rust
EventV1::UserRelationship { id, user, .. } => {
    self.cache.users.insert(id.clone(), user.clone().into());
```

客户端收到事件后更新本地缓存，界面实时刷新关系状态。

### 7.4 分工对比表

| 维度 | AMQP 推送通道 | WebSocket 事件通道 |
|------|--------------|-------------------|
| **触发时机** | 仅好友请求发送/接受时 | 所有关系变更（加/删/请求/屏蔽/取消屏蔽） |
| **受众** | 离线用户（通过手机推送） | 在线用户（通过 WebSocket 长连接） |
| **时效性** | 异步，延迟取决于推送平台 | 实时，毫秒级 |
| **可靠性** | RabbitMQ 持久化队列，至少一次投递 | Redis Pub/Sub，最多一次投递（在线才能收到） |
| **内容** | 极简通知（标题+图标） | 完整用户对象（含新关系状态） |
| **是否双向** | 单向（只通知被动方） | 双向（通知双方） |
| **持久化** | AMQP 消息持久化 + 手机系统推送中心 | 仅客户端本地缓存 |

**设计意图**：
- **AMQP 负责"触达"**：确保用户即使不在线也能通过手机推送知道有新的好友请求
- **WebSocket 负责"同步"**：确保在线用户的界面状态与服务器实时一致

两套通道是互补而非互斥的：如果用户同时在线且有手机推送，可能会同时收到 WebSocket 事件和手机推送。客户端需要做去重处理。

---

## 8. 屏蔽状态到权限禁用模块的完整链路

屏蔽不是单纯的"标签"，它会直接影响用户之间的所有交互权限。整条链路是**声明式**的，没有硬编码判断。

### 8.1 完整调用链

```
API 调用（如发消息、看资料）
    ↓
权限检查：calculate_user_permissions(query)
    ↓
query.user_relationship()  ← 从 perspective.relations 数组中读取关系
    ↓
match relationship {
    Blocked | BlockedOther => 只允许 Access 权限
    ...
}
    ↓
权限不满足 → 返回 PermissionDenied 错误
```

### 8.2 关系状态如何注入权限计算

**第一步：DatabasePermissionQuery 实现 PermissionQuery trait**

位置：[permissions.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/permissions.rs#L52-L84)

```rust
async fn user_relationship(&mut self) -> RelationshipStatus {
    if let Some(other_user) = &self.user {
        if self.perspective.id == other_user.id {
            return RelationshipStatus::User;
        }
        // ... 省略 bot 特殊处理 ...

        // 从 perspective（当前登录用户）的 relations 数组中查找
        if let Some(relations) = &self.perspective.relations {
            for entry in relations {
                if entry.id == other_user.id {
                    return match entry.status {
                        // 数据库层枚举 → 权限层枚举 的一一映射
                        crate::RelationshipStatus::None => RelationshipStatus::None,
                        crate::RelationshipStatus::User => RelationshipStatus::User,
                        crate::RelationshipStatus::Friend => RelationshipStatus::Friend,
                        crate::RelationshipStatus::Outgoing => RelationshipStatus::Outgoing,
                        crate::RelationshipStatus::Incoming => RelationshipStatus::Incoming,
                        crate::RelationshipStatus::Blocked => RelationshipStatus::Blocked,
                        crate::RelationshipStatus::BlockedOther => RelationshipStatus::BlockedOther,
                    };
                }
            }
        }
    }
    RelationshipStatus::None
}
```

这里做了一次**枚举转译**：将数据库层的 `crate::RelationshipStatus` 映射为权限层的 `revolt_permissions::RelationshipStatus`。虽然两个枚举定义完全相同，但通过 trait 边界解耦。

**第二步：权限计算函数根据关系状态分支**

位置：[permissions/impl.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/permissions/src/impl.rs#L8-L46)

```rust
pub async fn calculate_user_permissions<P: PermissionQuery>(query: &mut P) -> PermissionValue {
    if query.are_we_privileged().await {
        return u64::MAX.into();  // 特权用户全开
    }
    if query.are_the_users_same().await {
        return u64::MAX.into();  // 自己对自己全开
    }

    let mut permissions = 0_u64;
    match query.user_relationship().await {
        // ── 好友：全开 ─────────────────────────────
        RelationshipStatus::Friend => return u64::MAX.into(),

        // ── 屏蔽/被屏蔽：只保留 Access ─────────────
        RelationshipStatus::Blocked | RelationshipStatus::BlockedOther => {
            return (UserPermission::Access as u64).into()
        }

        // ── 待处理请求：只保留 Access ─────────────
        RelationshipStatus::Incoming | RelationshipStatus::Outgoing => {
            permissions = UserPermission::Access as u64;
        }

        // ── 无关系：0 基础权限，再看是否有共同连接 ──
        _ => {}
    }

    // 如果有共同服务器/群组，追加 Access + ViewProfile 权限
    if query.have_mutual_connection().await {
        permissions = UserPermission::Access as u64 + UserPermission::ViewProfile as u64;
        // ... 省略 bot 特殊处理 ...
    }

    permissions.into()
}
```

**屏蔽状态的权限效果**：

```
Blocked / BlockedOther → 仅 Access 权限
```

`UserPermission::Access` 是什么？看权限定义：

位置：[permissions/models/user.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/permissions/src/models/user.rs#L19-L24)

```rust
pub enum UserPermission {
    Access = 1 << 0,       // 基本"可见性"权限
    ViewProfile = 1 << 1,  // 查看对方资料
    SendMessage = 1 << 2,  // 发送消息
    Invite = 1 << 3,       // 邀请到服务器
}
```

所以屏蔽状态下：
- ❌ 不能查看对方资料（`ViewProfile` = 0）
- ❌ 不能发送消息（`SendMessage` = 0）
- ❌ 不能邀请到服务器（`Invite` = 0）
- ✅ 仅保留最基本的 `Access`（意味着用户"存在"，不会 404）

### 8.3 屏蔽如何影响具体操作

**场景 1：A 屏蔽了 B，B 试图给 A 发消息**

1. API 层创建 DM 通道时调用权限检查
2. `calculate_user_permissions` 中 B 看到 A 的关系是 `BlockedOther`
3. 返回仅 `Access` 权限，不包含 `SendMessage`
4. 进一步在 [permissions/impl.rs#L95-L106](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/permissions/src/impl.rs#L95-L106) 中：
   ```rust
   if permissions.has_user_permission(UserPermission::SendMessage) {
       (*DEFAULT_PERMISSION_DIRECT_MESSAGE).into()
   } else {
       (*DEFAULT_PERMISSION_VIEW_ONLY).into()  // 只能看，不能发
   }
   ```
5. 最终返回 `VIEW_ONLY` 权限，发送消息失败

**场景 2：A 屏蔽了 B，B 试图查看 A 的资料**

1. 调用 `fetch_profile` API
2. 权限检查返回仅 `Access`，不含 `ViewProfile`
3. 资料内容被过滤，仅返回基本信息（头像、用户名），不返回状态、个人简介等

**设计亮点**：屏蔽的权限控制是**单点声明**的——只在 `calculate_user_permissions` 中写一次判断，所有上层操作（发消息、看资料、加好友、邀请入群等）都会自动应用这个权限结果，不需要在每个 API 中单独判断"是否被屏蔽"。

---

## 9. 协调层两次写入之间的部分失败后果

### 9.1 代码中的风险点

`apply_relationship` 使用 `?` 运算符串联两次写入，这意味着**第一次写入成功后如果第二次写入失败，会直接返回错误，没有回滚机制**：

位置：[model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L494-L520)

```rust
pub async fn apply_relationship(
    &mut self,
    db: &Database,
    target: &mut User,
    local: RelationshipStatus,
    remote: RelationshipStatus,
) -> Result<()> {
    // ① 第一次写入：更新 target 的关系
    target.set_relationship(db, self, remote).await?;  // ← 如果这里成功...

    // ② 第二次写入：更新 self 的关系
    self.set_relationship(db, target, local).await?;    // ← ...但这里失败了！

    // ③ 事件推送（如果前两步都成功才会执行）
    EventV1::UserRelationship { ... }.private(target.id.clone()).await;
    EventV1::UserRelationship { ... }.private(self.id.clone()).await;

    Ok(())
}
```

### 9.2 可能的失败场景与后果

**失败场景 1：第一次写入（target）成功，第二次写入（self）失败**

后果：**不对称状态**——target 文档已更新，但 self 文档还是旧状态。

示例：A 接受 B 的好友请求
- 写 B 文档成功：B.relations = [{A, Friend}]
- 写 A 文档失败：A.relations = [{B, Incoming}] （还是旧状态）
- 事件推送未执行

最终状态：
- B 看到 A 是 `Friend`
- A 看到 B 还是 `Incoming`
- 客户端没有收到 WebSocket 事件，界面不会更新

**失败场景 2：两次写入都成功，但事件推送失败**

后果：**数据一致，但客户端不同步**——数据库中双方都是 `Friend`，但用户界面上可能还是旧状态，直到下次刷新。

**失败场景 3：第一次写入（target）失败**

后果：**无影响**——`?` 直接返回错误，两次都没写，状态保持原样。

### 9.3 可能导致第二次写入失败的原因

1. **网络分区**：第一次写入时 MongoDB 主节点可达，第二次写入时主节点不可达
2. **MongoDB 主节点切换**：两次写入之间发生了主备切换
3. **文档大小超限**：极端情况下，第一次写入后 target 文档刚好到了 16MB 限制，第二次写入时 self 文档也超限（但这个几乎不可能）
4. **并发修改导致的 write conflict**：虽然 MongoDB 的单文档更新是原子的，但在某些极端的事务/多文档写入场景下可能出现冲突（但这里没有事务）

### 9.4 不对称状态的类型与严重程度

按业务操作分类：

| 操作 | 部分失败后的不对称状态 | 可自愈吗？ | 影响程度 |
|------|-----------------------|-----------|----------|
| **发送好友请求** | A→B: None, B→A: Incoming | ✅ A 重试 add_friend 会重新发送 | 低 |
| **接受好友请求** | A→B: Incoming, B→A: Friend | ✅ A 再次调用 add_friend 看到 Incoming → 升级为 Friend，立即修复 | 中 |
| **拒绝/删除好友** | A→B: Friend, B→A: None | ✅ A 调用 remove_friend 会重置为 None/None | 中 |
| **A 屏蔽 B** | A→B: Friend, B→A: BlockedOther | ❌ A 再次调用 block_user 会看到旧状态（如 Friend）→ 重新执行屏蔽 → 修复 | 中 |
| **A 取消屏蔽 B**<br>(单向屏蔽) | A→B: Blocked, B→A: None | ❌ A 再次调用 unblock_user 会看到 Blocked → 重新执行取消 → 修复 | 低 |
| **A 取消屏蔽 B**<br>(双向屏蔽) | A→B: Blocked, B→A: BlockedOther | ❌ A 再次调用 unblock_user 会看到 Blocked → 重新执行取消 → 修复 | 低 |
| **A、B 互相屏蔽** | A→B: BlockedOther, B→A: Blocked | ✅ A 再次调用 block_user 看到 BlockedOther → 设为 Blocked/Blocked → 修复 | 低 |

**自愈分析**：
- **所有场景都可以通过用户再次操作自愈**——因为业务方法都是基于数据库中读取的最新状态来判断下一步动作的
- 最危险的是"接受好友请求"场景的部分失败：B 看到 A 已经是好友了，可以发消息，但 A 看到 B 还是待接受状态，可能会困惑为什么自己没点接受对方就"已经是好友"了
- 但即使这种场景，A 只要点一下"接受"按钮，`add_friend` 会看到 `Incoming` 状态并正确升级为 `Friend/Friend`

### 9.5 系统是否应该加分布式事务？

当前设计**有意不使用**分布式事务（MongoDB 4.0+ 支持多文档事务），原因：

1. **性能开销**：事务需要两阶段提交，延迟和吞吐量都会受影响
2. **冲突概率低**：两次写入都是针对不同文档的，且都是简单的数组更新，并发冲突概率极低
3. **自愈能力强**：如上面分析，所有不对称状态都可以通过用户操作自然修复
4. **AP 优先**：社交关系场景下，短暂的不一致比不可用更可接受

**但可以改进的地方**：
- 在 `apply_relationship` 中捕获第二次写入的错误，记录日志并触发一个补偿任务（如延迟重试）
- 增加一个定期的一致性校验脚本，扫描不对称的关系对并自动修复

### 9.6 内存对象与数据库的不一致风险

还有一个细节：`set_relationship` 会同步更新内存中的 `&mut self` 对象。如果第一次 `target.set_relationship(...)` 成功（target 内存对象已更新），但第二次 `self.set_relationship(...)` 失败，那么：

- `target` 内存对象的关系已经是新状态
- `self` 内存对象的关系还是旧状态
- 数据库中 target 是新状态，self 是旧状态

这意味着**函数返回错误后，调用者手中的内存对象已经部分修改了**。但由于错误会通过 `?` 向上冒泡，API 层会直接返回错误给客户端，不会继续使用这些内存对象，所以这个问题在实际中不会造成影响。

---

## 10. None / User 状态在三处写入行为的确切分歧

`set_relationship` 被三个实体执行，当状态为 `None` 或 `User` 时，三处的写入行为**并不一致**。这看似是 bug，实际上是刻意的设计分层。

### 10.1 三处代码逐行对照

**① 内存对象层**——[model.rs#L473-L476](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L473-L476)

```rust
if let RelationshipStatus::None | RelationshipStatus::User = status {
    if let Some(relations) = &mut self.relations {
        relations.retain(|relation| relation.id != user_b.id);  // 移除条目
    }
    // 注意：如果 self.relations 是 None，什么都不做
}
```

**② 参考实现（内存数据库）**——[reference.rs#L120-L121](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/reference.rs#L120-L121)

```rust
if let RelationshipStatus::User | RelationshipStatus::None = &relationship {
    self.pull_relationship(user_id, target_id).await  // 转发到 pull
}
```

`pull_relationship` 内部：[reference.rs#L145-L156](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/reference.rs#L145-L156)

```rust
async fn pull_relationship(&self, user_id: &str, target_id: &str) -> Result<()> {
    let mut users = self.users.lock().await;
    let user = users.get_mut(user_id).ok_or_else(|| create_error!(NotFound))?;
    if let Some(relations) = &mut user.relations {
        relations.retain(|relation| relation.id != target_id);  // 移除条目
    }
    Ok(())
}
```

**③ 生产实现（MongoDB）**——[mongodb.rs#L254-L256](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/mongodb.rs#L254-L256)

```rust
if let RelationshipStatus::None = relationship {
    return self.pull_relationship(user_id, target_id).await;  // 只判断 None，不判断 User
}
```

`pull_relationship` 内部：[mongodb.rs#L300-L316](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/ops/mongodb.rs#L300-L316)

```rust
async fn pull_relationship(&self, user_id: &str, target_id: &str) -> Result<()> {
    self.col::<User>(COL)
        .update_one(
            doc! { "_id": user_id },
            doc! { "$pull": { "relations": { "_id": target_id } } },
        )
        .await ...
}
```

### 10.2 分歧一览

| 维度 | 内存对象层 | 参考实现 | MongoDB 生产实现 |
|------|-----------|---------|-----------------|
| **None 状态** | `retain` 移除条目 | 转发 `pull` → `retain` 移除条目 | 转发 `pull` → `$pull` 移除条目 |
| **User 状态** | `retain` 移除条目 | 转发 `pull` → `retain` 移除条目 | ⚠️ **不拦截**，走主路径的 `$filter`+`$concatArrays` |
| **效果差异** | None 和 User 行为一致 | None 和 User 行为一致 | None 走 `$pull`，User 走替换式写入 |

### 10.3 这个分歧会不会出问题？

**结论：不会**。原因是 `User` 状态（"对方就是自己"）在实际业务流中**永远不会作为参数传入** `set_relationship`。

追查所有调用路径：

```
add_friend      → 传 None/Outgoing/Incoming/Friend
remove_friend   → 传 None/None
block_user      → 传 Blocked/BlockedOther/None
unblock_user    → 传 BlockedOther/None/Blocked
```

没有任何业务方法会传入 `RelationshipStatus::User`。`User` 状态只在 `relationship_with()` 和 `user_relationship()` 中作为**读取时的特殊返回值**，表示"对方就是自己"，不会参与写入。

因此 MongoDB 实现中 `if let RelationshipStatus::None = relationship` 只判断 `None` 而不判断 `User`，实际上是完全正确的——`User` 状态根本不会到达这里。参考实现多加了一个 `User` 判断属于防御性编程，无害但多余。

### 10.4 三处行为的另一个细微差别

当 `self.relations` 为 `None` 时：

| 层 | 行为 |
|----|------|
| 内存对象层 | `if let Some(relations)` 不匹配 → **跳过，不操作** |
| 参考实现 | `if let Some(relations)` 不匹配 → **跳过，不操作** |
| MongoDB 生产实现 | `$filter` 在 `$ifNull` 保护下返回 `[]`，`$pull` 在空数组上也无害 → **始终执行 update_one，但无副作用** |

这意味着：如果用户的 `relations` 字段本身是 `None`（从未有过任何关系），MongoDB 的 `$pull` 操作仍然会发出一次写操作（虽然不会实际修改任何数据），而内存实现则完全跳过。这是一个微小的性能差异，不影响正确性。

---

## 11. 待处理请求命中共同关系时的权限计算：是覆盖赋值而非叠加

### 11.1 需要修正的说法

前文第 8 节中对 `Incoming`/`Outgoing` 状态命中共同关系时的权限描述可能被理解为"叠加权限"，但代码实际是**覆盖赋值**。

### 11.2 代码执行流精确分析

位置：[permissions/impl.rs#L17-L39](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/permissions/src/impl.rs#L17-L39)

```rust
let mut permissions = 0_u64;                          // ← 初始化为 0
match query.user_relationship().await {
    RelationshipStatus::Friend => return u64::MAX.into(),
    RelationshipStatus::Blocked | RelationshipStatus::BlockedOther => {
        return (UserPermission::Access as u64).into()
    }
    RelationshipStatus::Incoming | RelationshipStatus::Outgoing => {
        permissions = UserPermission::Access as u64;   // ← 覆盖赋值，不是 +=
    }
    _ => {}                                            // ← None 分支：permissions 保持 0
}

// 后续判断共同关系
if query.have_mutual_connection().await {
    permissions = UserPermission::Access as u64         // ← 再次覆盖赋值！
                 + UserPermission::ViewProfile as u64;
    ...
}
```

### 11.3 关键：两次都是 `=` 不是 `+=`

这意味着：

**Incoming/Outgoing + 有共同关系**：

```
permissions = 0                                    // 初始化
permissions = Access                               // match 分支覆盖赋值
permissions = Access + ViewProfile                  // 共同关系分支覆盖赋值
```

最终：`Access + ViewProfile`

**如果误以为是叠加（`+=`）**，会认为结果是 `Access + Access + ViewProfile = Access + ViewProfile`（Access 重复但位运算无影响，所以结果碰巧相同）。但意图是完全不同的：共同关系分支并**不是在 Incoming/Outgoing 的 Access 基础上追加 ViewProfile**，而是**用 Access+ViewProfile 完全替换掉之前的 Access**。

**实际效果差异的证明——None 状态 + 有共同关系**：

```
permissions = 0                                    // 初始化
// None 命中 _ => {} 分支，permissions 不变
permissions = Access + ViewProfile                  // 共同关系分支覆盖赋值
```

最终：`Access + ViewProfile`

对比 Incoming/Outgoing + 有共同关系，结果**完全相同**。也就是说，`Incoming`/`Outgoing` 分支中 `permissions = Access` 这个赋值在有共同关系的情况下**没有任何实际效果**，因为后面会被覆盖。它的意义仅体现在**没有共同关系**时：

| 关系状态 | 有共同关系 | 最终权限 |
|---------|-----------|---------|
| None | ✅ | Access + ViewProfile |
| None | ❌ | 0 |
| Incoming/Outgoing | ✅ | Access + ViewProfile |
| Incoming/Outgoing | ❌ | **Access** |
| Friend | 任意 | 全开 |
| Blocked/BlockedOther | 任意 | 仅 Access |

所以 `Incoming`/`Outgoing` 分支的真正意义是：**即使没有共同服务器/群组，有待处理请求的双方也至少拥有 Access 权限**（可以看到对方的存在）。这与完全无关的陌生人（None + 无共同关系 → 0 权限）形成区别。

---

## 12. 机器人所有者分支的权限落点追踪

### 12.1 代码路径

位置：[permissions.rs#L56-L62](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/permissions.rs#L56-L62)

```rust
} else if let Some(bot) = &other_user.bot {
    // For the purposes of permissions checks,
    // assume owner is the same as bot
    if self.perspective.id == bot.owner {
        return RelationshipStatus::User;  // ← 机器人所有者看到的关系状态是 User
    }
}
```

**场景**：当前登录用户（perspective）是某个机器人的所有者，正在查询自己对那个机器人的权限。

这段代码的含义是：**机器人所有者看待自己的机器人，等同于看待自己**，返回 `RelationshipStatus::User`。

### 12.2 User 状态落入权限计算的哪条分支？

回到 [permissions/impl.rs#L8-L46](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/permissions/src/impl.rs#L8-L46)：

```rust
pub async fn calculate_user_permissions<P: PermissionQuery>(query: &mut P) -> PermissionValue {
    if query.are_we_privileged().await {
        return u64::MAX.into();        // ① 特权检查
    }
    if query.are_the_users_same().await {
        return u64::MAX.into();        // ② 同一人检查
    }

    let mut permissions = 0_u64;
    match query.user_relationship().await {
        RelationshipStatus::Friend => return u64::MAX.into(),
        RelationshipStatus::Blocked | RelationshipStatus::BlockedOther => { ... }
        RelationshipStatus::Incoming | RelationshipStatus::Outgoing => { ... }
        _ => {}                         // ③ User 状态落在这里！
    }
    ...
}
```

**`User` 状态落入 `_ => {}` 通配分支**，即 permissions 保持为 `0`。

然后继续到共同关系判断：

```rust
if query.have_mutual_connection().await {
    permissions = Access + ViewProfile;

    if query.user_is_bot().await || query.are_we_a_bot().await {
        permissions += SendMessage;
    }
}
```

### 12.3 但等等——② 号检查会先命中吗？

关键问题：如果 perspective.id == bot.owner，`are_the_users_same()` 会不会在 ② 处就返回 `u64::MAX`？

看 [permissions.rs#L43-L49](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/permissions.rs#L43-L49)：

```rust
async fn are_the_users_same(&mut self) -> bool {
    if let Some(other_user) = &self.user {
        self.perspective.id == other_user.id
    } else {
        false
    }
}
```

**不会命中**。`are_the_users_same()` 比较的是 `perspective.id == other_user.id`，而机器人所有者场景中 perspective.id 是 bot.owner，other_user.id 是 bot 的用户 ID。这两个 ID **不相同**（机器人和所有者是不同的用户账号），所以 ② 号检查不会提前返回。

### 12.4 完整落点路径

```
机器人所有者查询对自己机器人的权限
    ↓
are_we_privileged() → false（一般不是特权用户）
    ↓
are_the_users_same() → false（owner_id ≠ bot_user_id）
    ↓
user_relationship() → User（因为 perspective.id == bot.owner）
    ↓
match User → _ => {} → permissions = 0
    ↓
have_mutual_connection() → 查询共同服务器/群组
    ├─ 有共同关系 → permissions = Access + ViewProfile
    │                + user_is_bot() == true → permissions += SendMessage
    │                → 最终：Access + ViewProfile + SendMessage
    └─ 无共同关系 → permissions = 0
```

### 12.5 这合理吗？

机器人所有者查询自己机器人的权限时：

- **有共同服务器/群组**：获得 `Access + ViewProfile + SendMessage`（因为 `user_is_bot()` 为 true，自动追加 `SendMessage`）
- **无共同服务器/群组**：获得 `0` 权限

这暴露了一个问题：机器人所有者**在没有任何共同服务器/群组的情况下，对自己的机器人没有任何权限**。看起来 `user_relationship()` 返回 `User` 的意图是让所有者对机器人拥有完全控制权，但实际上 `User` 状态在 `calculate_user_permissions` 中没有对应的全开分支，而是落入 `_ => {}` 被忽略。

**对比真正的"自己看自己"**：当 perspective.id == other_user.id 时，② 号检查会直接返回 `u64::MAX`。而机器人所有者虽然被赋予 `User` 关系状态，却享受不到同等待遇。

这可能是代码中的一个**逻辑缺陷**或**待完善的设计**：要么应该在 `calculate_user_permissions` 中给 `User` 分支加上 `return u64::MAX.into()`，要么应该让 `are_the_users_same()` 也检查机器人所有者关系。当前的行为意味着机器人所有者对机器人的权限取决于是否有共同连接，而不是像代码注释所说的"assume owner is the same as bot"。

---

## 13. 关系字段进入客户端的完整序列化链路

关系状态不是直接从数据库原样输出给客户端的，而是经过了**四条不同的序列化路径**。每条路径在不同的业务场景下使用，对关系字段的处理方式也各不相同。

### 13.1 四条序列化路径总览

所有路径都在 [bridge/v0.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L994-L1197) 中，是 `crate::User` 的方法，返回 `v0::User`（API 层模型）。

| 方法 | 签名 | 关系处理方式 | 典型使用场景 |
|------|------|-------------|-------------|
| `into_self()` | `self, force_online` | 强制 `relationship=User`，携带全部 relations 列表 | 查看自己的资料 |
| `into()` | `self, db, perspective` | 完整权限引擎 + 关系数组读取 | 查看单个陌生人/好友资料 |
| `into_known()` | `self, perspective, is_online` | 直接读关系数组 + 仅判断是否被拉黑 | 批量获取用户（群成员列表） |
| `into_known_static()` | `self, is_online` | 强制 `relationship=None`，不带 relations | 消息作者等无视角场景 |

### 13.2 路径一：into_self（自己看自己）

位置：[bridge/v0.rs#L1164-L1197](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1164-L1197)

```rust
pub async fn into_self(self, force_online: bool) -> User {
    User {
        relations: self.relations.unwrap_or_default()  // ① 完整返回所有关系列表
            .into_iter()
            .map(|relation| relation.into())
            .collect(),
        ...
        relationship: RelationshipStatus::User,        // ② 强制 User 状态
        ...
    }
}
```

**特点**：
- 不接受 perspective 参数，天然就是"自己看自己"
- `relationship` 硬编码为 `User`，不需要任何计算
- `relations` 完整返回，客户端可以看到自己对所有人的关系状态
- 在线状态也完全公开（除非设为隐身）

**调用场景**：
- `GET /users/@me` —— [fetch_self.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/fetch_self.rs)
- `GET /users/<target>` 且 target 是自己 —— [fetch_user.rs#L17-L18](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/users/fetch_user.rs#L17-L18)
- 机器人管理 API（创建/编辑机器人，查看自己的机器人列表）

### 13.3 路径二：into（常规权限路径）

位置：[bridge/v0.rs#L994-L1070](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L994-L1070)

这是最完整、计算量最大的一条路径。关系和可见性是**分开计算**的：

```rust
pub async fn into<'a, P>(self, db: &Database, perspective: P) -> User
where P: Into<Option<&'a crate::User>>
{
    let perspective = perspective.into();
    let (relationship, can_see_profile) = if self.bot.is_some() {
        // ① 机器人短路：直接返回 None + true
        (RelationshipStatus::None, true)
    } else if let Some(perspective) = perspective {
        let mut query = DatabasePermissionQuery::new(db, perspective).user(&self);

        if perspective.id == self.id {
            // ② 字面自己看自己：直接返回 User + true
            (RelationshipStatus::User, true)
        } else {
            (
                // ③ 关系状态：直接从 perspective.relations 数组中查找
                perspective.relations.as_ref().map(|relations| {
                    relations.iter()
                        .find(|r| r.id == self.id)
                        .map(|r| r.status.clone().into())
                        .unwrap_or_default()
                }).unwrap_or_default(),
                // ④ 资料可见性：调用完整权限引擎计算
                calculate_user_permissions(&mut query)
                    .await
                    .has_user_permission(UserPermission::ViewProfile),
            )
        }
    } else {
        (RelationshipStatus::None, false)
    };
    // ... 组装 User 对象 ...
}
```

**关键点**：
- `relationship` 直接从 `perspective.relations` 数组读取，**不经过权限引擎**
- `can_see_profile` 由 `calculate_user_permissions()` 完整计算，考虑好友、屏蔽、共同连接等所有因素
- 机器人有最高优先级的短路：直接设为 `None` + `true`，跳过后续所有计算

**调用场景**：
- `GET /users/<target>` 且 target 不是自己 —— 单个用户详情查询

### 13.4 路径三：into_known（已知用户路径）

位置：[bridge/v0.rs#L1075-L1134](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1075-L1134)

```rust
pub async fn into_known<'a, P>(self, perspective: P, is_online: bool) -> User
where P: Into<Option<&'a crate::User>>
{
    let (relationship, can_see_profile) = if self.bot.is_some() {
        (RelationshipStatus::None, true)                // ① 同样的机器人短路
    } else if let Some(perspective) = perspective {
        if perspective.id == self.id {
            (RelationshipStatus::User, true)             // ② 同样的自己短路
        } else {
            let relationship = perspective.relations     // ③ 同样的关系读取
                .as_ref()
                .map(...find...map...unwrap_or_default())
                .unwrap_or_default();

            let can_see_profile = relationship != RelationshipStatus::BlockedOther;
                                                         // ④ 只判断是否被对方拉黑！
            (relationship, can_see_profile)
        }
    } else {
        (RelationshipStatus::None, false)
    };
    // ...
    relations: vec![],                                    // ⑤ relations 永远是空数组
    // ...
}
```

**注释原文**：
> Convert user object into user model assuming mutual connection
> Relations will never be included, i.e. when we process ourselves

"assuming mutual connection"——**假设已经有共同连接**（如在同一个服务器/频道里）。

**与 into() 的核心区别**：
- `can_see_profile` 的计算方式不同：`into()` 调用完整权限引擎，`into_known()` 只判断 `relationship != BlockedOther`
- `relations` 字段：`into()` 只有自己看自己时才返回完整列表，`into_known()` 永远返回空数组

**调用场景**：
- 批量获取用户列表（群成员、服务器成员等）
- `User::fetch_many_ids_as_mutuals()` —— [model.rs#L360-L374](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L360-L374)
- bonfire 网关推送 Ready 事件时的用户列表 —— [bonfire/impl.rs#L278-L282](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/bonfire/src/events/impl.rs#L278-L282)

### 13.5 路径四：into_known_static（静态无视角路径）

位置：[bridge/v0.rs#L1137-L1162](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1137-L1162)

```rust
pub async fn into_known_static(self, is_online: bool) -> User {
    User {
        relations: vec![],                          // 空
        ...
        relationship: RelationshipStatus::None,     // 强制 None
        ...
    }
}
```

注释说明：`events client will populate this from cache`——事件客户端会从缓存填充关系字段。

**特点**：
- 完全不接受 perspective 参数，没有任何视角概念
- `relationship` 硬编码为 `None`
- `relations` 是空数组
- 在线状态作为参数传入（由调用方批量查询后传入）

**调用场景**：
- 消息作者（发送消息时序列化发送者）—— [message_send.rs#L166-L168](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/delta/src/routes/channels/message_send.rs#L166-L168)
- 其他不需要视角、由客户端自行补充关系信息的场景

---

## 14. 机器人短路：所有者的内置 User 状态在序列化层被拍回 None

### 14.1 代码中的"双重人格"

机器人所有者关系状态在**两层**有不同的处理，且结果互相矛盾：

**第一层：权限引擎层 —— 返回 User**

位置：[permissions.rs#L56-L62](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/permissions.rs#L56-L62)

```rust
} else if let Some(bot) = &other_user.bot {
    // For the purposes of permissions checks,
    // assume owner is the same as bot
    if self.perspective.id == bot.owner {
        return RelationshipStatus::User;  // 权限计算时：所有者 = 用户自己
    }
}
```

**第二层：序列化层 —— 直接拍回 None**

位置：[bridge/v0.rs#L1000-L1001](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1000-L1001) 和 [bridge/v0.rs#L1080-L1081](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1080-L1081)

```rust
// into() 和 into_known() 中完全相同的短路
if self.bot.is_some() {
    (RelationshipStatus::None, true)  // 序列化输出时：机器人 = 无关系 + 可见
}
```

### 14.2 为什么会有这个分歧？

这不是 bug，而是**两套独立逻辑服务于不同目的**：

| 层面 | 返回值 | 目的 |
|------|--------|------|
| 权限引擎层 | `User` | 让所有者对自己的机器人拥有特殊权限（虽然实际上落入了 `_ => {}` 分支，见第 12 节） |
| 序列化层 | `None` | 客户端 UI 显示需求——机器人不应该显示"好友"、"待接受"等关系状态 |

机器人是一个特殊的实体：
- **对普通用户**：机器人是一个"用户"，但不应该有好友、请求等关系概念
- **对所有者**：机器人是自己的资产，但客户端界面上仍然显示为"无关系"，因为关系系统是为人与人设计的

序列化层的短路优先级更高（它在 if 链的最前面），所以**客户端看到的机器人关系永远是 None**，不管权限引擎怎么想。

### 14.3 两条路径的完整执行顺序对比

**场景：机器人所有者查看自己的机器人**

```
into() 路径：
  1. self.bot.is_some() → true
     → 直接返回 (None, true)
     → 权限引擎根本不会被调用！
     → user_relationship() 中的 User 分支完全不会执行
```

```
calculate_user_permissions() 单独调用时（如频道权限计算）：
  1. are_we_privileged() → false
  2. are_the_users_same() → false（owner_id ≠ bot_id）
  3. user_relationship() → User（命中 bot.owner 分支）
  4. match User → _ => {} → permissions = 0
  5. have_mutual_connection() → ...
```

所以：**序列化时永远看不到 User 状态，只有在纯权限计算场景下才会返回 User，但 User 又被权限计算函数忽略了**。这就形成了一个"两层都有特殊处理，但两层都没真正生效"的奇特状态。

### 14.4 为什么机器人短路在最前面？

看代码结构：

```rust
if self.bot.is_some() {
    (RelationshipStatus::None, true)       // ① 机器人短路（最优先）
} else if let Some(perspective) = perspective {
    if perspective.id == self.id {
        (RelationshipStatus::User, true)   // ② 自己短路
    } else {
        ...                                 // ③ 常规计算
    }
} else {
    (RelationshipStatus::None, false)      // ④ 无视角
}
```

机器人短路在自己短路**之前**。这意味着即使机器人的 ID 和 perspective 的 ID 相同（理论上不可能，因为机器人是另一个账号），也会先命中机器人短路。

但更实际的含义是：**机器人永远不会走"自己看自己"的分支**，哪怕某个 API 不小心把机器人自己传成了 perspective，也会被机器人短路拦截。这是一种防御性设计——机器人不应该有"自己看自己"的概念。

---

## 15. 两套"自己看自己"判定并存的真实行为

### 15.1 两套判定分别在哪里

代码中有**两处独立的**"自己看自己"判定，服务于不同的层次：

**第一套：字面 ID 比对 —— 序列化层**

位置：[bridge/v0.rs#L1005-L1006](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1005-L1006) 和 [bridge/v0.rs#L1083-L1084](file:///d:/fz/0601-1/solo-dogfeeding/code/85-frontend/crates/core/database/src/util/bridge/v0.rs#L1083-L1084)

```rust
if perspective.id == self.id {
    (RelationshipStatus::User, true)
}
```

直接比较两个用户 ID 字符串。

**第二套：机器人所有者扩展 —— 权限查询层**

位置：[permissions.rs#L56-L62](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/permissions.rs#L56-L62)

```rust
} else if let Some(bot) = &other_user.bot {
    if self.perspective.id == bot.owner {
        return RelationshipStatus::User;
    }
}
```

不仅比较 ID，还扩展了"如果 perspective 是对方机器人的所有者，也算自己"。

### 15.2 两套判定的覆盖关系

```
                    ┌─────────────────────────────────┐
                    │     序列化层：字面 ID 比对      │
                    │  perspective.id == self.id     │
                    └─────────────────────────────────┘
                                    │
                  命中 → (User, true) ← 直接返回，不调用权限引擎
                                    │
                              未命中 ↓
                    ┌─────────────────────────────────┐
                    │      权限查询层：bot.owner      │
                    │  perspective.id == bot.owner   │
                    └─────────────────────────────────┘
                                    │
                  命中 → 返回 User ← 但 calculate_user_permissions 中落入 _ => {}
                                    │
                              未命中 ↓
                          继续常规关系匹配
```

**关键结论**：两套判定是**串行**的，不是并行的。序列化层的判定在前，如果命中直接返回，根本不会走到权限查询层。只有当序列化层判定不命中时，权限查询层的扩展判定才有可能生效——但它生效的结果又被 `calculate_user_permissions` 的 match 忽略了。

### 15.3 各场景下的实际行为

| 场景 | 序列化层判定 | 权限层判定 | 最终 relationship | 最终 can_see_profile |
|------|-------------|-----------|-------------------|---------------------|
| 真·自己看自己 | ✅ 命中 | ❌ 不会执行 | `User` | `true` |
| 机器人所有者看机器人 | ❌ 不命中（但被机器人短路拦截） | ❌ 不会执行 | `None`（机器人短路） | `true`（机器人短路） |
| 陌生人看陌生人 | ❌ 不命中 | ❌ 不命中 | 从 relations 读 | 权限引擎计算 |
| 纯权限计算（无序列化） | —— | ✅ 命中 | `User`（仅内部返回） | 按 `_ => {}` 分支计算 |

### 15.4 为什么需要两套？

从设计意图推断：

1. **序列化层的字面比对**是为了**性能和正确性**：自己看自己时不需要查权限引擎，直接返回 `User` + 完整可见性，又快又准。

2. **权限层的机器人所有者扩展**是为了**语义一致性**：注释写着 "assume owner is the same as bot"，本意是让所有者对机器人拥有"对自己一样"的权限。

但两套判定的**衔接出了问题**：
- 序列化层不知道机器人所有者的概念，只看 ID
- 权限层虽然知道机器人所有者，但它返回的 `User` 状态在 `calculate_user_permissions` 中没有对应的全开分支

结果就是：**机器人所有者扩展只存在于 `user_relationship()` 的返回值里，上下都不接**——上面序列化层有机器人短路把它盖掉，下面权限计算函数有 `_ => {}` 把它吞掉。

### 15.5 一个反直觉的推论

因为机器人短路在序列化层的最前面，而机器人短路返回 `(None, true)`，所以：

> **所有人看机器人，关系都是 None，资料都可见**

包括：
- 机器人所有者看自己的机器人 → None
- 陌生人看机器人 → None
- 机器人看自己（理论上不会发生）→ None

这符合社交平台上机器人的设计：机器人是公开可见的服务实体，没有好友关系的概念。`user_relationship()` 中 `bot.owner` 返回 `User` 的分支，在序列化场景下永远不会暴露给客户端。

---

## 16. 关系变更事件推送到客户端的完整链路

### 16.1 事件构造时选择的序列化路径

`apply_relationship` 中构造关系变更事件时，**调用的是 `into()`（常规权限路径）**，不是 `into_known()`：

位置：[model.rs#L505-L517](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/models/users/model.rs#L505-L517)

```rust
// 向 target 推送：target 看到 self 的关系变化
EventV1::UserRelationship {
    id: target.id.clone(),
    user: self.clone().into(db, Some(&*target)).await,   // ← 调用 into()，传 target 作为 perspective
}
.private(target.id.clone())
.await;

// 向 self 推送：self 看到 target 的关系变化
EventV1::UserRelationship {
    id: self.id.clone(),
    user: target.clone().into(db, Some(&*self)).await,   // ← 调用 into()，传 self 作为 perspective
}
.private(self.id.clone())
.await;
```

**注意 perspective 的交换**：
- 给 target 发的事件里，user 对象是"self 从 target 视角"序列化 → `self.clone().into(db, Some(target))`
- 给 self 发的事件里，user 对象是"target 从 self 视角"序列化 → `target.clone().into(db, Some(self))`

这确保每个用户收到的事件中，对方的 `relationship` 字段和 `can_see_profile`（进而决定 online/status 字段）都是**从接收者自己的视角**算出来的。

### 16.2 为什么事件用 `into()` 不用 `into_known()`？

`into_known()` 的注释是 "assuming mutual connection"——假设已经有共同连接。但关系变更事件的触发场景是**任意**的：
- 可能两个陌生人刚发了好友请求（还没共同连接）
- 可能刚解除好友（不再有共同连接）

所以不能用假设性的 `into_known()`，必须用完整的 `into()` 精确计算每一次的关系状态和资料可见性。即使有共同连接，`into()` 的结果也不会比 `into_known()` 差（最多只是计算成本更高）。

### 16.3 事件到达客户端后的缓存更新

bonfire 网关收到 `UserRelationship` 事件后：

位置：[bonfire/impl.rs#L654-L661](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/bonfire/src/events/impl.rs#L654-L661)

```rust
EventV1::UserRelationship { id, user, .. } => {
    self.cache.users.insert(id.clone(), user.clone().into());
    // ↑ 直接覆盖缓存中的该用户对象

    if self.cache.can_subscribe_to_user(id) {
        self.insert_subscription(id.clone()).await;
    } else {
        self.remove_subscription(id).await;
    }
}
```

然后再通过 WebSocket 把整个事件广播给客户端。客户端收到后同样更新本地缓存，界面自动刷新。

---

## 17. 相同屏蔽关系在两条路径下得到反向可见性的代码分析

### 17.1 先建立场景

**A 屏蔽了 B**，所以：
- A.relations = `[{ B, Blocked }]`        ← A 知道自己屏蔽了 B
- B.relations = `[{ A, BlockedOther }]`   ← B 知道自己被 A 屏蔽了

现在看 **B 查 A**（被屏蔽者查屏蔽者）的两种场景。

### 17.2 场景一：B 通过用户详情页查 A → 走 `into()` 常规路径

B 调用 `GET /users/<A_id>`，API 层调 `a_user.into(db, Some(&b_user))`。

`into()` 内部的计算：

位置：[bridge/v0.rs#L1000-L1024](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1000-L1024)

```rust
// perspective = B，self = A
if self.bot.is_some() { ... }           // A 不是机器人 → 跳过
else if let Some(perspective) = perspective {
    let mut query = DatabasePermissionQuery::new(db, perspective).user(&self);

    if perspective.id == self.id { ... } // B.id ≠ A.id → 跳过
    else {
        // ① relationship：从 B.relations 中找 A
        let relationship = perspective.relations.as_ref().map(|relations| {
            relations.iter()
                .find(|r| r.id == self.id)   // 找到 BlockedOther
                .map(|r| r.status.clone().into())
                .unwrap_or_default()
        }).unwrap_or_default();
        // relationship = BlockedOther ✓

        // ② can_see_profile：调用完整权限引擎
        let can_see = calculate_user_permissions(&mut query)
            .await
            .has_user_permission(UserPermission::ViewProfile);
    }
}
```

**进入 `calculate_user_permissions`**：[permissions/impl.rs#L8-L22](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/permissions/src/impl.rs#L8-L22)

```rust
// perspective = B, other_user = A
are_we_privileged() → false
are_the_users_same() → false（B.id ≠ A.id）

match query.user_relationship().await {
    // B 看到 A 的关系是 BlockedOther → 命中
    RelationshipStatus::Blocked | RelationshipStatus::BlockedOther => {
        return (UserPermission::Access as u64).into()  // 只给 Access
    }
    ...
}
```

所以：
```
can_see_profile = has_permission(ViewProfile)
                = (Access & ViewProfile) != 0
                = false
```

**场景一结果**：
- `relationship = BlockedOther` ✓
- `can_see_profile = false` → A 的在线状态隐藏，状态文本隐藏

### 17.3 场景二：A 和 B 在同一个群，B 看群成员列表看到 A → 走 `into_known()` 已知用户路径

调用 `a_user.into_known(Some(&b_user), is_online)`。

`into_known()` 内部的计算：

位置：[bridge/v0.rs#L1080-L1100](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/core/database/src/util/bridge/v0.rs#L1080-L1100)

```rust
// perspective = B，self = A
if self.bot.is_some() { ... }           // A 不是机器人 → 跳过
else if let Some(perspective) = perspective {
    if perspective.id == self.id { ... } // B.id ≠ A.id → 跳过
    else {
        // ① relationship：和 into() 完全相同的读取逻辑
        let relationship = perspective.relations     // 从 B.relations 找 A
            .as_ref()
            .map(|relations| {
                relations.iter()
                    .find(|r| r.id == self.id)   // 找到 BlockedOther
                    .map(|r| r.status.clone().into())
                    .unwrap_or_default()
            })
            .unwrap_or_default();
        // relationship = BlockedOther ✓

        // ② can_see_profile：⚠️ 和 into() 完全不同的逻辑！
        let can_see_profile = relationship != RelationshipStatus::BlockedOther;
        //                        BlockedOther != BlockedOther
        //                      = false
    }
}
```

**场景二结果**：
- `relationship = BlockedOther` ✓
- `can_see_profile = false` → A 的在线状态隐藏，状态文本隐藏

等等，**结果相同**？那用户说的"反向的资料可见性结果"是什么意思呢？

### 17.4 反转视角：A 查 B（屏蔽者查被屏蔽者）

这才是两条路径产生**反向结果**的场景。

**A.relations = [{ B, Blocked }]，B.relations = [{ A, BlockedOther }]**

现在看 **A 查 B**：

---

**场景一：A 通过用户详情页查 B → `into()` 常规路径**

`b_user.into(db, Some(&a_user))`

```
① relationship：从 A.relations 找 B → Blocked
② calculate_user_permissions()：
   match Blocked → 命中 Blocked 分支 → 仅 Access
   can_see = (Access & ViewProfile) != 0 → **false**
```

结果：A 看不到 B 的在线状态和资料。✓ 合理，因为 A 屏蔽了 B，不想看 B 的信息。

---

**场景二：A 和 B 在同一个群，A 看群成员列表看到 B → `into_known()` 已知用户路径**

`b_user.into_known(Some(&a_user), is_online)`

```
① relationship：从 A.relations 找 B → Blocked
② can_see_profile = relationship != BlockedOther
                     Blocked != BlockedOther
                   = **true**  ← ⚠️ 这里！
```

结果：**A 能看到 B 的在线状态和资料**！

---

### 17.5 反向结果的总结表

| 场景 | 谁查谁 | 查看到的关系 | 常规路径 `into()` can_see | 已知路径 `into_known()` can_see | 结果 |
|------|--------|-------------|-------------------------|------------------------------|------|
| A 屏蔽 B | B 查 A | BlockedOther | **false** | **false** | 一致 |
| A 屏蔽 B | A 查 B | Blocked | **false** | **true** | ⚠️ 反向！ |

**反向的原因**：`into_known()` 的判断条件是 `relationship != BlockedOther`（我有没有被对方拉黑），而不是 `relationship != Blocked`（我有没有拉黑对方）。

- 当关系是 `Blocked`（我拉黑了对方）：
  - 常规路径：权限引擎 `match Blocked → 仅 Access` → `can_see = false`
  - 已知路径：`Blocked != BlockedOther` → `can_see = true`
  - 结果完全相反！

**设计意图的推断**：`into_known()` 注释是"assuming mutual connection"（假设已有共同连接）。在同一个群里，你即使屏蔽了某个人，**群成员列表中仍然需要显示对方的在线状态**，因为这是公共空间的必要信息。而通过用户详情页查对方时（`into()` 路径），因为你屏蔽了对方，所以隐藏对方的状态和资料。

这是一个刻意的设计差异，不是 bug：**屏蔽影响的是"点对点的可见性"，不影响"公共空间的存在性"**。

---

## 18. 初始连接事件的混合序列化策略与客户端反查机制

### 18.1 Ready 事件的用户序列化：混合策略

位置：[bonfire/impl.rs#L277-L285](file:///d:/fz/0601-1/solo-dogfeeding/code/85-backend/crates/bonfire/src/events/impl.rs#L277-L285)

```rust
// ① 其他所有用户：走 into_known（已知用户路径）
let mut users: Vec<v0::User> = join_all(users.into_iter().map(|other_user| async {
    let is_online = online_ids.contains(&other_user.id);
    other_user.into_known(&user, is_online).await  // ← 已知用户路径
})).await;

// ② 自己：单独追加 into_self（自身路径）
users.push(user.into_self(true).await);              // ← 自身路径（force_online=true）
```

**为什么用混合策略？**

| 用户类型 | 序列化方法 | 原因 |
|---------|-----------|------|
| 其他用户 | `into_known(perspective, is_online)` | Ready 时能列出的用户都和你有共同连接（好友、同群成员），符合 "assuming mutual connection" 前提；批量场景下性能也更好 |
| 自己 | `into_self(true)` | 自己必须看到完整的 `relations` 列表和 `relationship=User`，这是 `into_known()` 给不了的 |

### 18.2 混合策略下的 data layout

Ready 事件发送到客户端后，用户数组的结构是：

```
users: [
    // 其他用户们：into_known() 序列化
    { id: "B", relationship: "Friend",   relations: [], online: true,  status: {...} },
    { id: "C", relationship: "Blocked",  relations: [], online: false, status: null  },
    { id: "D", relationship: "Incoming", relations: [], online: true,  status: {...} },
    ...
    // 最后一个：into_self() 序列化
    { id: "A", relationship: "User",     relations: [            // ← 完整关系数组！
        { _id: "B", status: "Friend"   },
        { _id: "C", status: "Blocked"  },
        { _id: "D", status: "Incoming" },
        { _id: "E", status: "Outgoing" },
        ...
    ], online: true, status: {...} },
]
```

**关键特征**：
- 其他用户的 `relations` 都是空数组（`into_known()` 永远返回 `vec![]`）
- 只有自己（最后一个元素）的 `relations` 有完整内容
- 其他用户的 `relationship` 字段是"我对对方的关系状态"（从 `into_known()` 的 perspective 视角读取）

### 18.3 客户端的反查机制

客户端拿到 Ready 事件后，有**两套关系信息源**：

**源 1：每个其他用户对象的 `relationship` 字段**
- 已经是"我对这个人的关系"，直接显示即可
- 用于 UI 渲染：好友列表标"好友"、请求列表标"待接受"、屏蔽按钮状态等

**源 2：自己用户对象的 `relations` 数组**
- 完整的关系索引表，按 `_id` 反查
- 用于补充源 1 没有的信息，或在源 1 过期时做缓存

### 18.4 为什么两套信息源可以并存？

它们实际上是**完全冗余**的——因为 `into_known()` 中读取 `relationship` 的代码：

```rust
let relationship = perspective.relations
    .as_ref()
    .map(|relations| {
        relations.iter()
            .find(|r| r.id == self.id)          // 从 perspective.relations 中找
            .map(|r| r.status.clone().into())
            .unwrap_or_default()
    })
    .unwrap_or_default();
```

这里的 `perspective` 就是当前登录用户自己，`perspective.relations` 正是 `into_self()` 会返回的那个完整列表。所以：

```
other_user.relationship （源1）
  = perspective.relations.find(|r| r.id == other_user.id).map(|r| r.status)
  = my_relations.find(|r| r._id == other_user._id).map(|r| r.status)
  = 通过源 2（自己的 relations）按 other_user._id 反查到的状态
```

两套信息源来自**同一个底层数据**，只是投影方式不同：
- 源 1：把关系状态**散列**到每个对方用户对象上（方便 UI 直接读取）
- 源 2：把关系状态**集中**在自己对象的数组里（方便批量查询和缓存）

### 18.5 关系变更事件到来时的更新顺序

当 `UserRelationship` 事件到达时（参见第 16 节，事件用 `into()` 序列化）：

```
客户端收到 { id: "B", user: { id: "B", relationship: "Friend", ... } }
```

客户端更新流程：

1. **覆盖缓存中的 B 用户对象** → 源 1 自动更新（新的 relationship 已经在 user 对象里）
2. **同时按 id 反查自己的 relations 数组** → 如果有 B 的条目，也更新那条的 status
3. UI 重新渲染 → 从新的源 1 读取 relationship 显示

因为两套源最终都指向同一个事实（数据库中的 relations 数组），所以无论先更新哪一个，最终都会一致。而且关系变更事件本身已经附带了正确的 relationship 值，所以即使不更新源 2，客户端也不会显示错误——只是源 2 暂时"脏"了，下次 Ready 或手动刷新时会自然同步。

### 18.6 为什么 Ready 不直接给每个用户塞完整的 relations？

1. **数据量控制**：如果每个用户都携带完整的 relations 数组，假设每个用户有 100 个关系，Ready 中有 100 个用户，总数据量就是 100 × 100 = 10,000 条关系记录。实际上只需要 100 条（自己的那份）就足够了。

2. **信息安全**：其他用户的 relations 对客户端来说是**无意义的隐私数据**——你只需要知道你对对方是什么关系，不需要知道对方和其他人是什么关系。

3. **视角一致性**：关系是"有向边"（A→B 和 B→A 可能不同），每个客户端只能看到**自己的视角**，这是由 perspective 机制保证的。如果给你看 B 的 relations 数组，你会看到 B 对其他人的关系，这在语义上和权限上都是不允许的。
