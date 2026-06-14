# 安全报告与后台迁移任务协同关系梳理

## 一、整体架构概览

系统围绕安全报告模块形成了四条主线的协同：**报告创建 → 状态流转 → 迁移脚本 → 异步任务**。各模块之间通过数据库集合、Redis Pub/Sub 事件总线、AMQP 消息队列三层进行解耦通信。

```
用户请求
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│  delta (HTTP API 服务)                                        │
│  ┌──────────────────┐    ┌──────────────────────────────┐   │
│  │ report_content.rs│───▶│  1. 内容快照生成              │   │
│  │  POST /safety/   │    │  2. 附件标记为已举报          │   │
│  │  report          │    │  3. 写入 safety_snapshots     │   │
│  └────────┬─────────┘    │  4. 写入 safety_reports      │   │
│           │              │  5. 发送 ReportCreate 事件    │   │
│           │              └──────────────┬───────────────┘   │
│           │                             │                   │
│           ▼                             ▼                   │
│  ┌─────────────────┐          ┌──────────────────┐          │
│  │ migrate_database│          │ start_workers()  │          │
│  │ (启动时执行)    │          │ (后台任务线程)    │          │
│  └────────┬────────┘          └────────┬─────────┘          │
└───────────┼────────────────────────────┼────────────────────┘
            │                            │
            │ MongoDB                    │ Redis Pub/Sub
            │                            │ AMQP (RabbitMQ)
            ▼                            ▼
┌──────────────────────────┐  ┌──────────────────────────────┐
│ migrations 集合          │  │  bonfire (WebSocket 服务)     │
│ safety_reports 集合      │  │  - 订阅 Redis 事件通道        │
│ safety_snapshots 集合    │  │  - 向在线客户端推送事件       │
│ safety_strikes 集合      │  │  - 管理客户端订阅状态         │
└──────────────────────────┘  └──────────────────────────────┘
```

---

## 二、环节一：安全报告创建

### 2.1 入口与调用链

HTTP API 入口定义在 [report_content.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs)：

- 路由：`POST /safety/report`
- 所在模块：[safety](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/mod.rs)
- 函数签名：`pub async fn report_content(db, user, data) -> Result<EmptyResponse>`

### 2.2 创建流程详解

| 步骤 | 操作 | 关键代码位置 |
|------|------|-------------|
| 1 | 数据校验：验证 `additional_context` 长度 ≤ 1000 | [report_content.rs:L33-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L33-L37) |
| 2 | 权限校验：Bot 账号不能创建举报 | [report_content.rs:L40-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L40-L42) |
| 3 | 自举报校验：不能举报自己的消息/服务器/用户 | [report_content.rs:L51-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L51-L53), [L62-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L62-L64), [L73-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L73-L75) |
| 4 | 内容快照生成 | 见下方 2.3 节 |
| 5 | 附件标记为已举报 | [report_content.rs:L100-L102](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L100-L102) |
| 6 | 生成报告 ULID | [report_content.rs:L105](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L105) |
| 7 | 批量插入快照文档 | [report_content.rs:L108-L117](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L108-L117) |
| 8 | 插入报告文档 | [report_content.rs:L120-L129](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L120-L129) |
| 9 | 发送全局事件广播 | [report_content.rs:L131](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/routes/safety/report_content.rs#L131) |

### 2.3 内容快照生成机制

快照逻辑定义在 [safety_snapshots/model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/model.rs) 的 `SnapshotContent` 枚举中：

- **消息举报**：生成 `SnapshotContent::Message`，包含被举报消息 + 前后各 15 条上下文消息
  - 方法：[SnapshotContent::generate_from_message](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/model.rs#L42-L93)
- **服务器举报**：生成 `SnapshotContent::Server`，保存服务器完整信息
  - 方法：[SnapshotContent::generate_from_server](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/model.rs#L96-L104)
- **用户举报**：生成 `SnapshotContent::User`，若提供了 `message_id` 则额外生成消息快照
  - 方法：[SnapshotContent::generate_from_user](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/model.rs#L107-L120)

### 2.4 数据模型

**Report 模型**（[safety_reports/model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/model.rs)）：
```rust
pub struct Report {
    pub id: String,              // ULID 主键
    pub author_id: String,       // 举报人 ID
    pub content: ReportedContent,// 被举报内容（Message/Server/User）
    pub additional_context: String, // 附加说明
    pub status: ReportStatus,    // 状态（见第三节）
    pub notes: String,           // 管理员备注
}
```

**Snapshot 模型**（[safety_snapshots/model.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/model.rs#L8-L16)）：
```rust
pub struct Snapshot {
    pub id: String,              // ULID 主键
    pub report_id: String,       // 关联的报告 ID（已建索引）
    pub content: SnapshotContent,// 快照内容
}
```

### 2.5 数据库持久化

- Report 写入：[AbstractReport::insert_report](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/ops.rs#L10-L12) → [MongoDB 实现](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_reports/ops/mongodb.rs#L11-L15)
- Snapshot 写入：[AbstractSnapshot::insert_snapshot](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/safety_snapshots/ops.rs#L9-L12)

---

## 三、环节二：状态流转

### 3.1 状态枚举定义

报告状态定义在 [v0/safety_reports.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/models/src/v0/safety_reports.rs#L122-L135)，使用 `#[serde(tag = "status")]` 做内联标记：

```rust
pub enum ReportStatus {
    Created {},                                  // 待处理（初始状态）
    Rejected { rejection_reason, closed_at },    // 已驳回
    Resolved { closed_at },                      // 已处理
}
```

另有简化版枚举 `ReportStatusString` 用于仅表示状态类型的场景。

### 3.2 状态流转路径

```
                    ┌──────────────┐
                    │   Created    │  初始状态，创建报告时自动写入
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
     ┌────────────────┐       ┌────────────────┐
     │    Rejected    │       │    Resolved    │
     │ rejection_reason│       │  closed_at     │
     │ closed_at      │       └────────────────┘
     └────────────────┘
```

> **注意**：当前代码库中，状态流转的管理后台 API（驳回/结案）尚未在 delta 路由中暴露。模型层和数据库层已完整支持状态字段，管理操作预计由独立的后台管理服务完成。

### 3.3 事件广播机制

报告创建完成后通过 `EventV1::ReportCreate(report.into()).global().await` 广播全局事件：

- 事件定义：[client.rs:L268](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs#L268)
- `global()` 方法实现：[client.rs:L402-L404](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/events/client.rs#L402-L404)，发布到 Redis 通道 `"global"`
- 底层发布：使用 `redis_kiss::p()` 或 `redis_kiss::publish()` 发送到 Redis Pub/Sub
- 消费端：bonfire 服务的 WebSocket 客户端订阅该通道，向在线用户推送

---

## 四、环节三：迁移脚本

### 4.1 迁移启动入口

服务启动时自动执行迁移，在 [delta/main.rs:L33-L34](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/main.rs#L33-L34)：

```rust
let db = revolt_database::DatabaseInfo::Auto.connect().await.unwrap();
db.migrate_database().await.unwrap();
```

### 4.2 迁移总控逻辑

迁移调度位于 [admin_migrations/ops/mongodb.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb.rs#L17-L32)：

```
migrate_database()
   │
   ├─ 数据库已存在？
   │   ├─ 是 → scripts::migrate_database()  （增量迁移）
   │   └─ 否 → init::create_database()       （全新初始化）
```

### 4.3 全新数据库初始化

[init.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs) 中的 `create_database()` 负责：

1. 创建所有核心集合（accounts, users, channels, messages, servers 等）
2. 创建安全相关集合：
   - `safety_reports`（[init.rs:L71-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L71-L73)）
   - `safety_snapshots`（[init.rs:L75-L77](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L75-L77)）
   - `safety_strikes`（[init.rs:L79-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L79-L81)）
3. 创建所有必要的数据库索引
4. **直接写入当前版本号**：向 `migrations` 集合写入 `{_id: 0, revision: LATEST_REVISION}`（[init.rs:L242-L248](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L242-L248)）

### 4.4 增量迁移脚本

[scripts.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs) 中 `migrate_database()` 流程：

1. 读取 `migrations` 集合中的当前 `revision`
2. 调用 `run_migrations(db, revision)` 执行所有未跑过的脚本
3. 将 revision 更新为最新值
4. 当前最新版本常量：`LATEST_REVISION = 50`（[scripts.rs:L28](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs#L28)）

### 4.5 安全报告相关迁移条目

| Revision | 日期 | 操作 | 代码位置 |
|----------|------|------|---------|
| 19 | 2023-02-27 | 创建 `safety_reports` 和 `safety_snapshots` 集合 | [scripts.rs:L659-L667](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs#L659-L667) |
| 20 | 2023-02-28 | 为 `safety_snapshots.report_id` 创建索引 | [scripts.rs:L669-L686](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs#L669-L686) |
| 21 | 2023-05-31 | 创建 `safety_strikes` 集合 | [scripts.rs:L688-L692](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs#L688-L692) |
| 22 | 2023-05-31 | 为 `safety_strikes` 所有文档补充 `moderator_id` 字段 | [scripts.rs:L694-L708](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs#L694-L708) |

---

## 五、环节四：异步任务执行

### 5.1 任务启动入口

delta 服务启动时在 [main.rs:L97](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/delta/src/main.rs#L97) 调用：

```rust
revolt_database::tasks::start_workers(db.clone(), amqp.clone());
```

### 5.2 任务工作线程

[crates/core/database/src/tasks/mod.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/tasks/mod.rs) 中定义：

- 常量 `WORKER_COUNT = 5`（[mod.rs:L8](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/tasks/mod.rs#L8)）
- 启动 4 类任务，每类 5 个并发线程：

| 任务类型 | 作用 | 代码 |
|----------|------|------|
| `authifier_relay` | 认证事件转发 | [tasks/authifier_relay.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/tasks/authifier_relay.rs) |
| `ack` | 消息已读处理（消费 AMQP 队列） | [tasks/ack.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/tasks/ack.rs) |
| `last_message_id` | 维护频道最新消息 ID | [tasks/last_message_id.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/tasks/last_message_id.rs) |
| `process_embeds` | 消息链接嵌入处理 | [tasks/process_embeds.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/tasks/process_embeds.rs) |

### 5.3 延迟批处理机制

`DelayedTask<T>` 结构体提供了任务合并/延迟能力：

- `SAVE_CONSTANT = 5s`：任务连续活跃时每 5 秒落库一次
- `EXPIRE_CONSTANT = 30s`：任务超过 30 秒未更新则强制落库
- `delay()` 方法推迟落库时间，用于合并短时间内的多次更新

### 5.4 AMQP 消息队列

[AMQP 模块](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/amqp/amqp.rs) 维护 8 个专用通道，核心消息流：

```
delta API 服务
   │
   ├─ amqp.ack_notification_message()  ──┐
   │                                      │  RabbitMQ
   ├─ amqp.process_ack()              ───┤  交换机 + 路由键
   │                                      │
   └─ amqp.message_sent()             ───┘
                                          │
                                          ▼
                                   pushd / crond 消费者
```

### 5.5 crond 定时任务服务

除 delta 内联线程外，还有独立的 [crond 服务](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/daemons/crond/src/main.rs)，使用 Tokio 运行 4 个长驻任务：

- `file_deletion`：文件清理
- `prune_dangling_files`：悬空文件修剪
- `prune_members`：成员数据修剪
- `acks`：已读事件批量处理（与 delta 内的 ack worker 对应）

### 5.6 bonfire WebSocket 事件分发

[bonfire 服务](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/main.rs) 作为事件分发层：

1. 客户端建立 WebSocket 连接并认证
2. 为每个客户端创建独立的 `State`，维护订阅的 Redis 通道集合
3. 报告创建时 `EventV1::ReportCreate` 通过 `global()` 发布到 Redis `"global"` 通道
4. bonfire 的 `listener_with_kill_signal` 任务监听 Redis，按客户端订阅列表进行过滤和推送
5. 客户端状态管理见 [events/state.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/bonfire/src/events/state.rs)

---

## 六、迁移任务重复执行的去重保障

迁移系统通过 **四层防线** 确保脚本的幂等性和重复执行安全：

### 第一层：Revision 版本号门控（核心机制）

在 [scripts.rs:run_migrations()](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/scripts.rs#L63-L1310) 中，**每个迁移脚本都被包裹在 `if revision <= N` 条件内**：

```rust
if revision <= 19 {
    // 创建 safety_reports 和 safety_snapshots
}

if revision <= 20 {
    // 创建 safety_snapshots.report_id 索引
}
// ... 以此类推
```

执行逻辑：
1. 从 `migrations` 集合读取当前 `revision` 值（如 18）
2. 从 revision=0 开始依次判断，所有 `N >= 当前revision` 的脚本都会执行
3. 执行完毕后，返回 `LATEST_REVISION.max(revision)`（即 50）
4. 将 `migrations` 集合的 revision 更新为最新值
5. **下次启动时**，revision=50，所有 `if revision <= N` 条件都为 false，全部跳过

### 第二层：全新库直接写入最高版本

对于首次部署（数据库不存在），[init.rs:L242-L248](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/ops/mongodb/init.rs#L242-L248) **不执行任何历史迁移脚本**，直接写入：

```javascript
{ "_id": 0, "revision": LATEST_REVISION }
```

这意味着新部署从一开始就是最新版本，避免了从零跑 50 个迁移脚本的冗余开销。

### 第三层：MongoDB 操作本身的幂等性

迁移脚本大量使用具备天然幂等性的数据库操作：

| 操作类型 | 幂等行为 | 示例 |
|----------|----------|------|
| `create_collection` | 集合已存在时返回 Ok（新版本 MongoDB 支持） | revision 19 创建集合 |
| `createIndexes` | 索引已存在时会跳过（同名称） | revision 20 创建索引 |
| `update_many` + `$set` | 重复写入相同字段值结果不变 | revision 22 补充字段 |
| `$unset` | 字段不存在时静默成功 | 各类字段清理迁移 |

> 注意：部分迁移（如 revision 13 权限重置、revision 23  discriminator 生成）包含非幂等逻辑，配合 revision 门控确保只执行一次。

### 第四层：单元测试双重执行验证

模型文件中的测试用例 [admin_migrations/model.rs:L15-L22](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/models/admin_migrations/model.rs#L15-L22) 显式验证了连续两次调用 `migrate_database()` 均不报错：

```rust
#[async_std::test]
async fn migrate() {
    database_test!(|db| async move {
        db.migrate_database().await.unwrap();  // 第一次
        db.migrate_database().await.unwrap();  // 第二次，必须也成功
    });
}
```

### 去重机制的整体流程

```
服务启动
   │
   ▼
migrate_database()
   │
   ├─ 读取 migrations.revision
   │
   ├─ revision == LATEST_REVISION？
   │   ├─ 是 → 直接返回，零操作
   │   └─ 否 → 进入 run_migrations()
   │             │
   │             ├─ for each if revision <= N:
   │             │   └─ 条件成立 → 执行迁移
   │             │   └─ 条件不成立 → 跳过
   │             │
   │             └─ 返回 LATEST_REVISION
   │
   └─ 更新 migrations.revision = LATEST_REVISION
```

---

## 七、补充：其他层级的去重机制

### AMQP 消息级去重

在 [amqp.rs:L280-L284](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/amqp/amqp.rs#L280-L284) 中，`ack_notification_message` 通过 RabbitMQ 消息头实现去重：

```rust
headers.insert(
    "x-deduplication-header".into(),
    AMQPValue::LongString(format!("{}-{}", &user_id, &channel_id).into()),
);
```

RabbitMQ 端基于该 header 在指定时间窗口内丢弃重复消息。

### HTTP 请求级幂等性

[util/idempotency.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/database/src/util/idempotency.rs) 提供了 `Idempotency-Key` HTTP 头支持：

- 基于 LRU Cache（容量 1000）缓存已处理的 key
- 重复请求返回 `DuplicateNonce` 错误（HTTP 409）
- 未提供 key 时自动生成 ULID 作为临时 key

### 任务合并级去重

`CoalescionService`（[coalesced/service.rs](file:///d:/fz/0601-1/solo-dogfeeding/code/90-backend/crates/core/coalesced/src/service.rs)）提供相同 ID 任务的合并执行：

- 同一 `Id` 已在运行时，后续请求直接等待已有结果
- 可选 LRU 缓存层缓存历史结果
- 可选排队机制（`queue_requests` feature）控制最大并发
