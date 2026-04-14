# Franz-Go Examples 学习指南

本文档详细介绍了 franz-go Kafka 客户端库的示例代码，按照从入门到精通的顺序组织，帮助开发者系统性地掌握各个概念。

---

## 📚 学习路径概览

```
阶段 1: 基础入门 (无需外部 Kafka 集群)
├── testing_with_kfake      # 内存测试集群
├── admin_and_requests      # 管理操作
└── plugins                 # 日志与监控

阶段 2: 生产者基础
├── manual_flushing         # 手动刷新
├── partitioners            # 分区策略
└── record_formatter        # 记录格式化

阶段 3: 消费者基础
├── group_committing        # 5 种提交策略
└── goroutine_per_partition # 并发消费模式

阶段 4: 高级特性
├── consumer_runtime_control # 动态控制
├── dlq                      # 死信队列
├── consumer_group_lag       # 延迟监控
└── transactions             # 事务 (EOS)

阶段 5: 生产集成
├── sasl                     # 认证与 TLS
├── schema_registry          # Schema 管理
├── plugin_kotel             # OpenTelemetry
└── bench                    # 性能测试
```

---

## 阶段 1: 基础入门

### 1.1 testing_with_kfake - 内存测试集群

**学习目标**: 学习如何在无外部依赖的情况下测试 Kafka 应用

**核心概念**:
- `kfake.Cluster`: 内存中的假 Kafka 集群
- `ControlKey`: 拦截特定类型的请求
- `KeepControl`: 持久化控制器，用于观察
- `SleepControl`: 延迟响应，协调多连接

**代码示例**:
```go
// 创建单节点测试集群
c, err := kfake.NewCluster(
    kfake.NumBrokers(1),
    kfake.SeedTopics(1, "test-topic"),
)
defer c.Close()

// 方式 1: 一次性拦截 (注入错误)
c.ControlKey(int16(kmsg.Produce), func(kreq kmsg.Request) (kmsg.Response, error, bool) {
    // 构造错误响应
    resp := req.ResponseKind().(*kmsg.ProduceResponse)
    // ... 设置错误码
    return resp, nil, true  // true 表示已处理
})

// 方式 2: 持续观察 (KeepControl)
c.Control(func(kreq kmsg.Request) (kmsg.Response, error, bool) {
    c.KeepControl()  // 保持控制器活跃
    counts[kmsg.NameForKey(kreq.Key())]++
    return nil, nil, false  // false 表示未拦截，继续处理
})

// 方式 3: 协调多连接 (SleepControl)
c.ControlKey(int16(kmsg.Fetch), func(kmsg.Request) (kmsg.Response, error, bool) {
    c.SleepControl(func() {
        <-produced  // 等待 produce 信号
    })
    return nil, nil, false
})
```

**关键要点**:
- `kfake` 是单元测试的理想选择，无需 Docker 或真实 Kafka
- `ControlKey` 针对特定请求类型，`Control` 匹配所有请求
- 返回 `(nil, nil, false)` 表示不拦截，请求继续正常处理
- 使用 `SleepOutOfOrder()` 允许其他连接在处理时继续

---

### 1.2 admin_and_requests - 管理操作

**学习目标**: 掌握两种管理操作方式：高级 API 和原始协议

**核心概念**:
- `kadm.Client`: 高级管理客户端
- `kmsg`: 原始 Kafka 协议请求/响应
- `kerr`: Kafka 错误码处理

**方式一: kadm (推荐)**
```go
adm := kadm.NewClient(cl)

// 列出所有 Topic
topics, err := adm.ListTopics(ctx)

// 创建 Topic (使用默认分区数和副本数)
resp, err := adm.CreateTopic(ctx, -1, -1, nil, "my-topic")

// 其他常用操作
adm.DeleteTopics(ctx, "topic1", "topic2")
adm.DescribeGroups(ctx, "group1")
adm.ListOffsets(ctx, "topic", kadm.AfterMilli(time.Now().Add(-time.Hour)))
```

**方式二: kmsg (原始协议)**
```go
// 创建 Topic 请求
req := kmsg.NewPtrCreateTopicsRequest()
t := kmsg.NewCreateTopicsRequestTopic()
t.Topic = "my-topic"
t.NumPartitions = 1
t.ReplicationFactor = 1
req.Topics = append(req.Topics, t)

// 发送请求
res, err := req.RequestWith(ctx, cl)

// 检查响应错误
if err := kerr.ErrorForCode(res.Topics[0].ErrorCode); err != nil {
    // 处理错误
}
```

**对比**:

| 特性 | kadm | kmsg |
|------|------|------|
| 易用性 | 高 | 低 |
| 灵活性 | 中 | 高 |
| 适用场景 | 常规管理 | 特殊需求 |
| 学习成本 | 低 | 高 |

**重要提示**:
- 始终使用 `NewPtrXxxRequest()` 创建请求，确保前向兼容
- `MaxVersions` 限制协议版本，避免意外使用新字段

---

### 1.3 plugins - 日志与监控

**学习目标**: 配置日志和指标收集

**日志插件**:
```go
// 选项 1: slog (推荐)
sl := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelInfo}))
opts = append(opts, kgo.WithLogger(kslog.New(sl)))

// 选项 2: zap
zl, _ := zap.NewDevelopment()
opts = append(opts, kgo.WithLogger(kzap.New(zl)))

// 选项 3: 内置基础日志
opts = append(opts, kgo.WithLogger(kgo.BasicLogger(os.Stderr, kgo.LogLevelInfo, nil)))
```

**指标插件 (Prometheus)**:
```go
metrics := kprom.NewMetrics("kgo")
opts = append(opts, kgo.WithHooks(metrics))

// 暴露 HTTP 端点
http.Handle("/metrics", metrics.Handler())
log.Fatal(http.ListenAndServe(":9999", nil))
```

**日志级别**:
- `LogLevelError`: 仅错误
- `LogLevelWarn`: 警告及以上
- `LogLevelInfo`: 信息及以上 (推荐)
- `LogLevelDebug`: 调试信息

---

## 阶段 2: 生产者基础

### 2.1 manual_flushing - 手动刷新

**学习目标**: 控制批量发送时机

**核心概念**:
- `ManualFlushing()`: 禁用自动刷新
- `TryProduce`: 非阻塞生产
- `Flush`: 显式刷新缓冲区

**代码示例**:
```go
cl, err := kgo.NewClient(
    kgo.SeedBrokers(brokers...),
    kgo.DefaultProduceTopic(topic),
    kgo.ManualFlushing(),           // 关键：启用手动刷新
    kgo.MaxBufferedRecords(200),    // 缓冲区大小限制
)

// 缓冲一批记录
for i := 0; i < batchSize; i++ {
    r := kgo.StringRecord(fmt.Sprintf("batch=%d record=%d", batch, i))
    cl.TryProduce(ctx, r, func(r *kgo.Record, err error) {
        if err != nil {
            fmt.Printf("delivery error: %v\n", err)
        }
    })
}

// 显式刷新，等待所有记录发送完成
if err := cl.Flush(ctx); err != nil {
    log.Fatal("flush failed:", err)
}
```

**使用场景**:
- 需要精确控制批处理边界
- 定时任务，按时间窗口发送
- 事务性生产

**注意**: 如果缓冲区满，`TryProduce` 返回 `ErrMaxBuffered`，需要处理或增大缓冲区

---

### 2.2 partitioners - 分区策略

**学习目标**: 理解各种分区策略及其适用场景

**策略对比**:

| 策略 | 特点 | 适用场景 |
|------|------|---------|
| **Sticky** (默认) | 随机选分区，批次内保持 | 高吞吐，无需顺序 |
| **StickyKey** | 相同 Key 到同一分区 | 需要 Key 顺序 |
| **RoundRobin** | 均匀轮询 | 负载均衡优先 |
| **LeastBackup** | 选积压最少分区 | 故障恢复场景 |
| **Manual** | 显式指定分区 | 特殊路由需求 |
| **UniformBytes** | 按字节阈值分区 (KIP-794) | 统一批次大小 |

**代码示例**:
```go
var partitioner kgo.Partitioner

switch strategy {
case "sticky":
    partitioner = kgo.StickyPartitioner()

case "sticky-key":
    // nil 使用默认 murmur2 哈希 (与 Java 兼容)
    partitioner = kgo.StickyKeyPartitioner(nil)

case "round-robin":
    partitioner = kgo.RoundRobinPartitioner()

case "least-backup":
    // 对慢 Broker 有弹性
    partitioner = kgo.LeastBackupPartitioner()

case "manual":
    partitioner = kgo.ManualPartitioner()
    // 需要设置 record.Partition

case "uniform-bytes":
    partitioner = kgo.UniformBytesPartitioner(
        16384,  // 16KB 阈值
        true,   // adaptive: 优先选择积压少的
        true,   // keys: 有 Key 时哈希
        nil,    // hasher
    )
}

cl, _ := kgo.NewClient(
    kgo.RecordPartitioner(partitioner),
)
```

**最佳实践**:
- 默认使用 `StickyPartitioner` 获得最佳吞吐
- 需要 Key 顺序时使用 `StickyKeyPartitioner`
- 生产环境避免 `RoundRobin` (小批次影响性能)

---

### 2.3 record_formatter - 记录格式化

**学习目标**: 自定义记录输入输出格式

**格式化动词**:
- `%t` - Topic 名称
- `%k` - Key
- `%v` - Value
- `%p` - 分区号
- `%o` - 偏移量
- `%T` - 时间戳
- `%h` - Headers

**使用示例**:
```go
// 读取记录时格式化输出
formatter := kgo.NewRecordFormatter("%t[%p]@%o: %k=%v\n")

fetches.EachRecord(func(r *kgo.Record) {
    output := formatter.Format(r)
    fmt.Print(output)
})
```

---

## 阶段 3: 消费者基础

### 3.1 group_committing - 偏移量提交策略

**学习目标**: 掌握 5 种不同的偏移量提交方式

**策略详解**:

#### 1. Autocommit (默认)
```go
opts := []kgo.Opt{
    kgo.ConsumerGroup(group),
    kgo.ConsumeTopics(topic),
    // 自动提交是默认行为
}

// 消费逻辑
fetches.EachRecord(func(r *kgo.Record) {
    process(r)
})
// 下次 poll 时自动提交上一次的偏移量
```

**特点**:
- ✅ 最简单，不会丢失已 poll 的记录
- ❌ 可能重复消费 (崩溃后上次 poll 的记录会重消费)

#### 2. Records (按记录提交)
```go
opts = append(opts, kgo.DisableAutoCommit())

// 消费后显式提交
var records []*kgo.Record
fetches.EachRecord(func(r *kgo.Record) {
    records = append(records, r)
})
cl.CommitRecords(ctx, records...)
```

**特点**:
- ✅ 精确控制
- ❌ 性能差 (每次提交都有网络开销)

#### 3. Uncommitted (推荐的手动模式)
```go
opts = append(opts, kgo.DisableAutoCommit())

// 消费后提交所有未提交偏移量
fetches.EachRecord(func(r *kgo.Record) {
    process(r)
})
cl.CommitUncommittedOffsets(ctx)
```

**特点**:
- ✅ 批量提交，性能好
- ✅ 推荐用于生产环境

#### 4. Marks (标记提交)
```go
opts = append(opts,
    kgo.AutoCommitMarks(),          // 只提交标记的偏移量
    kgo.BlockRebalanceOnPoll(),     // poll 期间阻塞重平衡
    kgo.OnPartitionsRevoked(func(ctx context.Context, cl *kgo.Client, _ map[string][]int32) {
        // 分区被回收前提交标记的偏移量
        cl.CommitMarkedOffsets(ctx)
    }),
)

// 消费逻辑
marks := make(map[string]map[int32]kgo.EpochOffset)
fetches.EachPartition(func(p kgo.FetchTopicPartition) {
    if len(p.Records) == 0 {
        return
    }
    last := p.Records[len(p.Records)-1]
    marks[p.Topic][p.Partition] = kgo.EpochOffset{
        Epoch:  last.LeaderEpoch,
        Offset: last.Offset + 1,
    }
})
cl.MarkCommitOffsets(marks)
cl.AllowRebalance()  // 允许重平衡
```

**特点**:
- ✅ 精确控制哪些记录已处理
- ✅ 避免重平衡期间的重复消费
- ⚠️ 需要手动管理重平衡

#### 5. kadm (绕过消费组协议)
```go
// 使用 kadm 获取已提交偏移量
admCl := kadm.NewClient(adm)
os, _ := admCl.FetchOffsetsForTopics(ctx, group, topic)

// 直接分区分配消费
cl, _ := kgo.NewClient(
    kgo.ConsumePartitions(os.KOffsets()),
)

// 消费后使用 kadm 提交
admCl.CommitAllOffsets(ctx, group, kadm.OffsetsFromFetches(fs))
```

**特点**:
- ✅ 完全控制消费位置
- ⚠️ 失去消费组的自动分区分配

---

### 3.2 goroutine_per_partition_consuming - 并发消费

**学习目标**: 实现分区级别的并行处理

**架构模式**:

```
┌─────────────────────────────────────────┐
│           kgo.Client (Poll)             │
│                 │                       │
│         fetches.EachTopic               │
│                 │                       │
│    ┌────────────┼────────────┐         │
│    │            │            │         │
│  Topic1      Topic2      Topic3        │
│    │            │            │         │
│  Partition   Partition   Partition     │
│    │            │            │         │
│    ▼            ▼            ▼         │
│ ┌──────┐    ┌──────┐    ┌──────┐      │
│ │goroutine│  │goroutine│  │goroutine│   │
│ │  ch   │    │  ch   │    │  ch   │    │
│ └──────┘    └──────┘    └──────┘      │
└─────────────────────────────────────────┘
```

**三种实现方式对比**:

| 模式 | 自动提交 | 重平衡安全 | 复杂度 | 适用场景 |
|------|---------|-----------|--------|---------|
| autocommit_normal | ✅ | ❌ | ⭐ | 简单场景 |
| autocommit_marks | ✅ | ✅ | ⭐⭐⭐ | 推荐方案 |
| manual_commit | ❌ | ✅ | ⭐⭐⭐⭐ | 精确控制 |

#### 方式 1: Autocommit Normal (最简单)
```go
type splitConsume struct {
    mu        sync.Mutex
    consumers map[string]map[int32]pconsumer
}

type pconsumer struct {
    quit chan struct{}
    recs chan []*kgo.Record
}

func (s *splitConsume) assigned(_ context.Context, cl *kgo.Client, assigned map[string][]int32) {
    s.mu.Lock()
    defer s.mu.Unlock()
    for topic, partitions := range assigned {
        for _, partition := range partitions {
            pc := pconsumer{
                quit: make(chan struct{}),
                recs: make(chan []*kgo.Record, 10),
            }
            s.consumers[topic][partition] = pc
            go pc.consume(topic, partition)  // 启动 goroutine
        }
    }
}

func (s *splitConsume) lost(_ context.Context, cl *kgo.Client, lost map[string][]int32) {
    // 关闭对应 goroutine
    for topic, partitions := range lost {
        for _, partition := range partitions {
            pc := s.consumers[topic][partition]
            close(pc.quit)  // 通知 goroutine 退出
        }
    }
}

func (s *splitConsume) poll(cl *kgo.Client) {
    for {
        fetches := cl.PollFetches(context.Background())
        fetches.EachTopic(func(t kgo.FetchTopic) {
            s.mu.Lock()
            tconsumers := s.consumers[t.Topic]
            s.mu.Unlock()
            
            t.EachPartition(func(p kgo.FetchPartition) {
                pc := tconsumers[p.Partition]
                select {
                case pc.recs <- p.Records:
                case <-pc.quit:
                }
            })
        })
    }
}

// 初始化
opts := []kgo.Opt{
    kgo.ConsumerGroup(group),
    kgo.ConsumeTopics(topic),
    kgo.OnPartitionsAssigned(s.assigned),
    kgo.OnPartitionsRevoked(s.lost),
    kgo.OnPartitionsLost(s.lost),
}
```

**注意**: 此方式在重平衡时可能重复消费

#### 方式 2: Autocommit Marks (推荐)
```go
opts = append(opts,
    kgo.AutoCommitMarks(),
    kgo.BlockRebalanceOnPoll(),
    kgo.OnPartitionsRevoked(func(ctx context.Context, cl *kgo.Client, _ map[string][]int32) {
        cl.CommitMarkedOffsets(ctx)
    }),
)

// 在 partition goroutine 中标记
func (pc *pconsumer) consume(topic string, partition int32, cl *kgo.Client) {
    for recs := range pc.recs {
        process(recs)
        // 标记最后一条记录
        last := recs[len(recs)-1]
        cl.MarkCommitRecords(last)
    }
}
```

#### 方式 3: Manual Commit
```go
// 在 goroutine 中同步提交
func (pc *pconsumer) consume(topic string, partition int32, cl *kgo.Client) {
    for recs := range pc.recs {
        process(recs)
        // 同步提交
        if err := cl.CommitRecords(ctx, recs[len(recs)-1]); err != nil {
            log.Printf("commit failed: %v", err)
        }
    }
}
```

---

## 阶段 4: 高级特性

### 4.1 dlq - 死信队列

**学习目标**: 实现可靠的消息处理失败处理

**模式说明**:
```
┌──────────────┐     处理失败     ┌──────────┐
│ Source Topic │ ──────────────▶ │ DLQ      │
└──────────────┘   重试 N 次后    └──────────┘
       │
       ▼ 处理成功
┌──────────────┐
│ 正常业务逻辑  │
└──────────────┘
```

**代码实现**:
```go
const (
    maxRecRetryCount = 3
    dlqTopic         = "dlq"
)

func (k *kafka) run(ctx context.Context) {
    for {
        fetches := k.consumer.PollFetches(ctx)
        fetches.EachPartition(func(p kgo.FetchTopicPartition) {
            for _, rec := range p.Records {
                if err := k.process(rec); err != nil {
                    // 重试机制
                    if err = k.retry(rec); err != nil {
                        // 发送到 DLQ
                        k.sendToDLQ(ctx, rec, err)
                    }
                }
            }
        })
    }
}

func (k *kafka) retry(r *kgo.Record) error {
    var err error
    for i := 1; i <= maxRecRetryCount; i++ {
        if err = k.process(r); err != nil {
            // 指数退避
            time.Sleep(time.Duration(i*2) * time.Second)
        } else {
            return nil
        }
    }
    return err
}

func (k *kafka) sendToDLQ(ctx context.Context, rec *kgo.Record, err error) {
    failed, _ := json.Marshal(&message{
        Topic:       rec.Topic,
        Key:         rec.Key,
        Value:       rec.Value,
        Offset:      rec.Offset,
        Partition:   rec.Partition,
        // ...
    })
    
    k.producer.Produce(ctx, &kgo.Record{
        Topic: dlqTopic,
        Key:   rec.Key,
        Value: failed,
        Headers: []kgo.RecordHeader{
            {Key: "status", Value: []byte("review")},
            {Key: "error", Value: []byte(err.Error())},
        },
    }, nil)
}
```

**关键点**:
- 重试应有退避策略 (backoff)
- DLQ 消息应包含原始信息和错误详情
- 考虑幂等性处理

---

### 4.2 transactions - Kafka 事务

**学习目标**: 实现 Exactly-Once Semantics (EOS)

**两种模式**:

#### 模式 1: Standalone Producer (独立事务生产者)
```go
cl, _ := kgo.NewClient(
    kgo.TransactionalID("my-txn-id"),
)

for {
    // 开启事务
    if err := cl.BeginTransaction(); err != nil {
        log.Fatal(err)
    }
    
    // 生产一批记录
    e := kgo.AbortingFirstErrPromise(cl)
    for i := 0; i < 10; i++ {
        cl.Produce(ctx, kgo.StringRecord(msg), e.Promise())
    }
    
    // 根据是否有错误决定提交或中止
    commit := kgo.TransactionEndTry(doCommit && e.Err() == nil)
    
    switch err := cl.EndTransaction(ctx, commit); err {
    case nil:
        fmt.Println("transaction committed")
    case kerr.OperationNotAttempted:
        // 需要中止
        cl.EndTransaction(ctx, kgo.TryAbort)
    default:
        log.Fatal("transaction failed:", err)
    }
}
```

#### 模式 2: EOS (Exactly-Once Consume-Transform-Produce)
```go
sess, _ := kgo.NewGroupTransactSession(
    kgo.TransactionalID("eos-txn-id"),
    kgo.FetchIsolationLevel(kgo.ReadCommitted()), // 只读已提交
    kgo.ConsumerGroup(group),
    kgo.ConsumeTopics(inputTopic),
    kgo.RequireStableFetchOffsets(),
)
defer sess.Close()

for {
    fetches := sess.PollFetches(ctx)
    
    if err := sess.Begin(); err != nil {
        log.Fatal(err)
    }
    
    e := kgo.AbortingFirstErrPromise(sess.Client())
    fetches.EachRecord(func(r *kgo.Record) {
        // 转换并生产
        transformed := transform(r)
        sess.Produce(ctx, transformed, e.Promise())
    })
    
    // End 自动提交消费偏移量和生产的数据
    committed, err := sess.End(ctx, e.Err() == nil)
    if !committed {
        log.Fatal("EOS commit failed:", err)
    }
}
```

**关键概念**:
- `TransactionalID`: 事务标识符，用于恢复未完成事务
- `ReadCommitted`: 隔离级别，防止读取未提交数据
- `RequireStableFetchOffsets`: 确保获取稳定偏移量
- 事务超时默认 1 分钟，可通过 `TransactionTimeout` 配置

---

### 4.3 consumer_runtime_control - 动态控制

**学习目标**: 运行时动态调整消费行为

**支持的操作**:
```go
// 1. 动态添加 Topic
cl.AddConsumeTopics("new-topic")

// 2. 动态添加分区
cl.AddConsumePartitions(map[string][]int32{
    "topic": {0, 1, 2},
})

// 3. 暂停消费
cl.PauseFetchTopics("topic1", "topic2")
cl.PauseFetchPartitions(map[string][]int32{
    "topic": {0, 1},
})

// 4. 恢复消费
cl.ResumeFetchTopics("topic1")
cl.ResumeFetchPartitions(map[string][]int32{
    "topic": {0},
})
```

**使用场景**:
- 根据业务负载动态调整
- 故障隔离 (暂停问题分区)
- 灰度发布

---

### 4.4 consumer_group_lag - 消费延迟监控

**学习目标**: 监控消费组健康状况
```go
adm := kadm.NewClient(cl)

// 获取延迟信息
lag, err := adm.Lag(ctx, groups...)

// lag 包含：
// - Committed: 已提交偏移量
// - End: 最新偏移量
// - Lag: 延迟数量
```

---

## 阶段 5: 生产集成

### 5.1 sasl - 认证与 TLS

**支持的方法**:
```go
// PLAIN
kgo.SASL(kgo.PlainSASL(user, pass))

// SCRAM-SHA-256
kgo.SASL(kgo.ScramSASL(user, pass, kgo.ScramSha256))

// SCRAM-SHA-512
kgo.SASL(kgo.ScramSASL(user, pass, kgo.ScramSha512))

// AWS MSK IAM
kgo.SASL(aws.NewSASL(aws.Credentials{
    AccessKey:    "...",
    SecretKey:    "...",
    SessionToken: "...",
}))

// TLS
kgo.DialTLSConfig(&tls.Config{
    InsecureSkipVerify: false,
    // ...
})
```

---

### 5.2 schema_registry - Schema 管理

**学习目标**: 使用 Avro 等 Schema
```go
// 创建 Schema Registry 客户端
rcl, _ := sr.NewClient(sr.URLs("http://localhost:8081"))

// 注册 Schema
schema, _ := rcl.CreateSchema(ctx, "subject", sr.Schema{
    Schema: avroSchema,
    Type:   sr.TypeAvro,
})

// 使用 Serde 序列化/反序列化
var serde sr.Serde
serde.Register(
    schema.ID,
    MyType{},
    sr.EncodeFn(func(v any) ([]byte, error) { /* ... */ }),
    sr.DecodeFn(func(b []byte, v any) error { /* ... */ }),
)

// 编码并生产
b, _ := serde.Encode(value)
cl.Produce(ctx, &kgo.Record{Value: b}, nil)

// 消费并解码
fetches.EachRecord(func(r *kgo.Record) {
    var v MyType
    serde.Decode(r.Value, &v)
})
```

---

### 5.3 plugin_kotel - OpenTelemetry

**学习目标**: 分布式链路追踪
```go
// 创建 kotel 插件
kotelService := kotel.NewKotel(
    kotel.WithTracer(kotel.NewTracer()),
    kotel.WithMeter(kotel.NewMeter()),
)

// 配置客户端
cl, _ := kgo.NewClient(
    kgo.WithHooks(kotelService.Hooks()),
)

// 生产时携带 trace 上下文
ctx, span := tracer.Start(ctx, "produce")
cl.Produce(ctx, record, nil)
span.End()

// 消费时提取 trace 上下文
fetches.EachRecord(func(r *kgo.Record) {
    ctx := r.Context  // 包含 trace 信息
    _, span := tracer.Start(ctx, "consume")
    defer span.End()
    process(r)
})
```

---

### 5.4 bench - 性能测试

**学习目标**: 性能调优和对比

**关键配置**:
```go
kgo.Linger(100 * time.Millisecond)        // 批处理延迟
kgo.RecordBatchCompression(kgo.ZstdCompression())  // 压缩
kgo.MaxBufferedRecords(10000)             // 缓冲区大小
kgo.MaxConcurrentFetches(12)              // 并发 fetch
```

---

## 📖 总结

### 快速参考表

| 概念 | 相关示例 | 关键 API |
|------|---------|---------|
| 测试 | testing_with_kfake | `kfake.NewCluster`, `ControlKey` |
| 管理 | admin_and_requests | `kadm.Client`, `kmsg` |
| 日志 | plugins | `kgo.WithLogger`, `kslog.New` |
| 手动刷新 | manual_flushing | `ManualFlushing`, `Flush` |
| 分区 | partitioners | `RecordPartitioner` |
| 提交 | group_committing | `CommitRecords`, `MarkCommitOffsets` |
| 并发 | goroutine_per_partition | `OnPartitionsAssigned` |
| 事务 | transactions | `NewGroupTransactSession` |
| DLQ | dlq | 应用层实现 |
| 认证 | sasl | `SASL`, `DialTLSConfig` |
| Schema | schema_registry | `sr.Client`, `Serde` |
| 可观测 | plugin_kotel | `kotel.NewKotel` |

### 学习建议

1. **从简单开始**: 先跑通 `testing_with_kfake` 和 `group_committing`
2. **理解概念**: 深入理解偏移量提交和重平衡
3. **实践并发**: 掌握 `goroutine_per_partition` 模式
4. **生产准备**: 学习事务、DLQ、监控等高级特性

---

*Generated for franz-go examples study*
