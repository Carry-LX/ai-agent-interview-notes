# Broker 队列：现状与 Redis 演进

> 面试问题：现在的队列怎么做？加 Redis 解决什么？
> 一句话：**现在是“任务落库、后台巡检抢活”；目标是“每个成员一个收件箱，消息直达、签收消费”。DB 仍是事实源，Redis 只负责唤醒和可靠投递。**

## 1. 现在的形态：公告栏 + 巡检

当前不是一个统一 MQ，而是几套机制拼起来：

| 部分 | 现在怎么工作 | 功能上的感觉 |
| --- | --- | --- |
| 团队任务 | `team_wake_events` 表，事件状态 pending / claimed；后台心跳 + 租约 + 巡检，过期事件重新派发 | 活写在公告栏，大家定时去看，谁先抢到谁干 |
| Host 任务 | 内存状态 + 轮询执行状态；投递状态 pending / sent / unknown / confirmed | 干完要不断问“好了吗”，没确认就记为未知 |
| Broker 连接 | WebSocket 长连接 + 进程内小队列 + 背压；断线重连换代 | 电话一直通着，但一次只能说这么多 |
| Work 执行 | 单进程 pending 集合 + checkpoint；失败/未知靠恢复 | 同一件事本进程不能重复开干 |
| 群消息扫描 | 定时扫时间窗口，带 overlap / lookback | 定时去翻消息，不是消息来找你 |

这套机制能工作，但功能上有几个明显特征：

- **唤醒有延迟**：要等下个巡检周期或轮询周期。
- **崩溃恢复慢**：要等心跳/租约超时，系统才敢把活重新派。
- **未知状态多**：`wake_result_unknown`、`delivery_unknown`、`release_unknown`，需要人工或后台对账。
- **多进程靠抢**：多个 worker 抢同一行，靠原子更新保证不重复；抢不到就空转。
- **突发靠背压**：连接层队列满了就拒绝/延迟，没有持久缓冲。
- **观测靠查表**：要自己统计积压、失败、重试。

## 2. 加 Redis 后的形态：收件箱 + 签收

```text
DB（事实源）
  消息日志 / 任务状态 / 回执 / outbox
    ↓
Redis Streams
  每个目标一个 inbox
  consumer group 分发
  XACK 签收
  XPENDING / XAUTOCLAIM 重投
  死信 stream
    ↓
Runtime / Manager
  拉取 → 校验 → 决定 → 执行 → 回写
```

功能上的感觉：

- 活直接进每个成员的收件箱，不用去公告栏找。
- 成员在线就取，处理完签收；没签收的自动回到收件箱。
- 群发就是往多个收件箱各投一份。
- 主管能看到每个收件箱积压多少、谁在处理、哪些进了死信。

## 3. 能规避哪些问题

| 现在的问题 | 加 Redis 后 |
| --- | --- |
| 唤醒有延迟 | 消息到达即唤醒，近实时 |
| 崩溃恢复慢 | 未签收消息自动回队/重投，不用等租约过期 |
| 结果 unknown 多 | ack / pending 归属明确；没签收就是没完成 |
| 多 worker 抢同一任务 | consumer group 天然分配，一条消息只给一个消费者 |
| 群发靠逐条写/扫 | 多 inbox fan-out |
| 突发流量被背压 | stream 缓冲，消费者按自己的速度取 |
| 顺序依赖 DB 时间 | 每个 target/thread 的 stream 内有序 |
| 重启后靠巡检 | 重放 pending，按 `operationId` 恢复 |
| 空转轮询 | 有事才唤醒，减少无效查询 |
| 观测靠查表 | 队列深度、pending、死信、consumer lag 直接可见 |

## 4. 不解决什么

- **远端不可用**：`BROKER_UNAVAILABLE` 这类错误，Redis 只能缓冲重试，不能替远端执行。
- **exactly-once**：仍然是至少一次，消费端必须按 `operationId` 幂等。
- **业务完成判定**：消息签收 ≠ 任务完成；完成仍由 Manager/验收决定。
- **大文件**：文件走对象存储，队列只带引用。
- **Redis 自身高可用**：需要 AOF、副本、哨兵/集群；否则只是换了一个单点。

## 5. 选型建议

| 阶段 | 推荐 | 理由 |
| --- | --- | --- |
| 本地单机 / MVP | SQLite/Postgres outbox + 轮询 | 不引入新组件，事务内写消息和 outbox |
| 云端共用版（默认） | **Postgres outbox + Redis Streams** | DB 保证持久，Redis 负责低延迟唤醒和 per-target inbox |
| 强投递 / 优先级 / DLQ | RabbitMQ quorum queues | ack/nack、死信、TTL、路由成熟 |
| 大规模 / 回放 / 多消费者 | Kafka / Redpanda | 持久日志、回放、分区扩展；现阶段偏重 |
| 云托管 | SQS / Pub/Sub / Service Bus | 托管省事；本地/私有化不适用 |

Redis 里选 **Streams + consumer group**，不要用 Pub/Sub（fire-and-forget）或裸 List（没有 ack/pending）。BullMQ 可以用，但它是 job 语义，不是消息日志；核心事实仍要落 DB。

## 6. 推荐接法

```text
1. 同一个 DB 事务：
   - 写消息日志
   - 写 outbox(target, thread_id, operation_id, payload_ref, status=pending)
2. Dispatcher：outbox → XADD 到目标 stream
3. Consumer group：每条消息只给一个 runtime
4. 处理完：更新 DB 状态 → XACK
5. 失败/超时：留在 pending → XAUTOCLAIM 重投
6. 超过次数：进 dead-letter stream，升级给人
7. 重启：按 operationId 查 DB + reclaim pending
```

语义：

- **per-target inbox**：B 一个 stream，不按发送方分叉。
- **thread 内有序**：同一 `threadId` 在一个 stream/分区内有序；不要全局有序。
- **至少一次 + 幂等**：`operationId` 去重。
- **优先级**：审批、取消、停止走单独 stream 或优先级字段。
- **批量唤醒**：Broker 只发 wake；ManagerRun 拉一批处理 0/1/N 个 WorkCase。
- **公平性**：按发送方/租户限流，避免一个 A 打满 B。

## 7. 部署形态

| 形态 | 队列 |
| --- | --- |
| 本地单机 | 不引入 Redis；SQLite/Postgres outbox + 轮询 |
| 云端共用 | 服务端 Redis（AOF + 副本 + 哨兵/集群）+ Postgres + 对象存储 |
| 私有化 | 同上，自建 Redis / Postgres / 对象存储 |

Broker 可以和协作存储同集群，但建议独立进程/独立存储；Redis 是 Broker 的投递层，不是群存储。

## 8. 真实案例对照

「时间4」团队：

- 本地数据分析成员成功，没走 Broker。
- 外部同事 A的数字分身是远端成员，连续 `wake_result_unknown`，最后 `422 BROKER_UNAVAILABLE` 升级。
- 加 Redis 后：唤醒会更快，未签收会重投，pending 归属明确；但 **422 仍然要升级**，因为远端服务不可用不是队列能修的。

外部单聊审批案例：

- 现在：请求进入、审批、恢复，靠 DB/WS 状态推进。
- 加 Redis 后：审批/恢复事件可以更快唤醒 Manager；审批不扩权、业务验收不变。

## 9. 面试口径

**30 秒版**

> 现在的队列是数据库行加租约和巡检：任务落库，后台 worker 定时找活，成员崩溃要等心跳/租约过期才恢复，结果不确定时记 unknown。加 Redis Streams 后，每个成员一个收件箱，消息直达，consumer group 分发，处理完签收，没签收自动重投，积压和死信可见。DB 仍是事实源，Redis 只负责唤醒和可靠投递。

**2 分钟版**

> 我们不是缺一个 MQ，而是现在的投递是“收件人定时去看”。团队任务用 pending/claimed + 心跳 + 租约 + 巡检；Host 任务用轮询；Broker 连接用 WebSocket + 进程内背压。功能上就是公告栏加巡检：唤醒有延迟，崩溃恢复慢，unknown 多，多进程靠抢，突发靠背压。目标形态是邮局加签收：DB outbox 保证事实，Redis Streams 每个目标一个 inbox，consumer group 分发，XACK 签收，XPENDING/XAUTOCLAIM 重投，死信升级。这样唤醒、恢复、群发、顺序、观测都会更自然；但远端不可用仍然要升级，幂等仍要按 operationId 做。

**追问**

- **“为什么不用 Kafka？”** 当前是每目标 inbox、消息量中等、要低延迟唤醒；Kafka 的分区/消费者组/运维偏重，留到大规模和回放时再上。
- **“Redis 丢了怎么办？”** DB outbox 是事实源，Redis 丢了从 outbox 重放；Redis 配 AOF + 副本。
- **“是不是 exactly-once？”** 不是；至少一次 + `operationId` 幂等。
- **“远端 422 呢？”** Redis 只能缓冲重试，不能替远端执行；最终仍要升级给人。

## 10. 证据

- [11 存算分离](11-存算分离怎么实现.md)
- [12 本地版到共用版](12-本地版到共用版的差距.md)
- 11-09 跨账号群聊
- 11-10 本地服务恢复
- 代码参考：`vendor/org-platform-runtime/backend/app/teams/wakeup.py`、`products/digital-twin/plugins/digital-twin/application/host-tasks.mjs`、`adapters/broker/host-connection.mjs`、`application/work-runtime.mjs`
