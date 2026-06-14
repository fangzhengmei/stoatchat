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
