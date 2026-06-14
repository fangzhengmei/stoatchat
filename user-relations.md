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
