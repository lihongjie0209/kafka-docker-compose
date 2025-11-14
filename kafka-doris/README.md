# Kafka + Doris 集成环境

这个配置启动一个包含 Kafka 和 Apache Doris 的集成环境，适用于实时数据流处理和分析场景。

## 服务说明

- **Kafka**: 消息队列服务（KRaft 模式）
  - 内部端口: 9092
  - 外部端口: 9093 (localhost:9093)
- **Doris FE** (Frontend): 查询和元数据服务
  - HTTP 端口: 8030
  - MySQL 协议端口: 9030
- **Doris BE** (Backend): 数据存储和计算服务
  - HTTP 端口: 8040
  - Heartbeat 端口: 9060
- **Kafka UI**: Web 管理界面
  - HTTP 端口: 8080

## 快速启动

```bash
docker compose up -d
```

等待所有服务启动（大约 1-2 分钟）：

```bash
docker compose ps
```

## 访问服务

### Kafka UI
- URL: http://localhost:8080
- 用途: 管理 Kafka topics、查看消息

### Doris Web UI
- URL: http://localhost:8030
- 默认账号: root
- 默认密码: (空)

### Doris MySQL 客户端连接
```bash
mysql -h 127.0.0.1 -P 9030 -u root
```

## 使用示例

### 1. 创建 Kafka Topic

```bash
docker exec kafka kafka-topics --create \
  --topic user_events \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1
```

### 2. 发送测试消息到 Kafka

```bash
docker exec -it kafka kafka-console-producer \
  --topic user_events \
  --bootstrap-server localhost:9092
```

输入 JSON 格式数据：
```json
{"user_id": 1, "event": "login", "timestamp": "2025-11-14 10:00:00"}
{"user_id": 2, "event": "purchase", "timestamp": "2025-11-14 10:05:00"}
```

按 `Ctrl+C` 退出。

### 3. 在 Doris 中创建数据库和表

连接到 Doris：
```bash
mysql -h 127.0.0.1 -P 9030 -u root
```

创建数据库：
```sql
CREATE DATABASE IF NOT EXISTS kafka_demo;
USE kafka_demo;
```

创建表：
```sql
CREATE TABLE user_events (
    user_id INT,
    event VARCHAR(50),
    timestamp DATETIME
)
DUPLICATE KEY(user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 3
PROPERTIES (
    "replication_num" = "1"
);
```

### 4. 创建 Routine Load 任务（从 Kafka 导入）

```sql
CREATE ROUTINE LOAD kafka_demo.load_user_events ON user_events
COLUMNS(user_id, event, timestamp)
PROPERTIES
(
    "desired_concurrent_number" = "1",
    "max_batch_interval" = "10",
    "max_batch_rows" = "1000",
    "format" = "json"
)
FROM KAFKA
(
    "kafka_broker_list" = "kafka:9092",
    "kafka_topic" = "user_events",
    "property.group.id" = "doris_consumer_group",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
```

### 5. 查看导入任务状态

```sql
SHOW ROUTINE LOAD FOR kafka_demo.load_user_events\G
```

### 6. 查询导入的数据

```sql
SELECT * FROM kafka_demo.user_events ORDER BY timestamp DESC LIMIT 10;
```

### 7. 停止导入任务

```sql
STOP ROUTINE LOAD FOR kafka_demo.load_user_events;
```

## 数据流程

```
生产者 → Kafka Topic → Routine Load → Doris 表 → SQL 查询
```

## 性能调优建议

### Kafka 配置
- 增加分区数以提高并行度
- 调整保留策略：`retention.ms`
- 配置压缩：`compression.type=snappy`

### Doris Routine Load 配置
```sql
PROPERTIES
(
    "desired_concurrent_number" = "3",  -- 并发消费任务数
    "max_batch_interval" = "10",        -- 最大批次间隔（秒）
    "max_batch_rows" = "10000",         -- 最大批次行数
    "max_error_number" = "100"          -- 允许的最大错误数
)
```

### Doris 表配置
```sql
PROPERTIES (
    "replication_num" = "1",            -- 副本数（单节点设为1）
    "storage_medium" = "SSD",           -- 存储介质
    "compression" = "LZ4"               -- 压缩算法
);
```

## 监控和运维

### 查看 Kafka 消息积压
```bash
docker exec kafka kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group doris_consumer_group \
  --describe
```

### 查看 Doris 集群状态
```sql
SHOW PROC '/frontends';
SHOW PROC '/backends';
```

### 查看导入任务错误日志
```sql
SHOW ROUTINE LOAD TASK WHERE JobName = 'load_user_events';
```

### 查看表数据量
```sql
SELECT TABLE_NAME, TABLE_ROWS, DATA_LENGTH 
FROM information_schema.tables 
WHERE TABLE_SCHEMA = 'kafka_demo';
```

## 常见问题

### Q: Doris BE 启动失败
A: 检查内存是否足够（建议至少 4GB）：
```bash
docker stats doris-be
```

### Q: Routine Load 任务卡住
A: 检查 Kafka 连接和 topic 是否存在：
```bash
docker exec kafka kafka-topics --list --bootstrap-server localhost:9092
```

### Q: 数据导入延迟高
A: 调整 `max_batch_interval` 和 `desired_concurrent_number`：
```sql
ALTER ROUTINE LOAD FOR kafka_demo.load_user_events
PROPERTIES
(
    "desired_concurrent_number" = "3",
    "max_batch_interval" = "5"
);
```

### Q: 如何重置消费位置
A: 停止任务后重新创建，或使用 `property.kafka_default_offsets`：
```sql
-- OFFSET_BEGINNING: 从头开始
-- OFFSET_END: 从最新开始
-- timestamp: 从指定时间戳开始
```

## 停止服务

```bash
docker compose down
```

## 停止服务并清除数据

```bash
docker compose down -v
```

## 数据持久化

数据存储在以下 Docker volumes：
- `doris-fe-meta`: FE 元数据
- `doris-fe-log`: FE 日志
- `doris-be-storage`: BE 数据存储
- `doris-be-log`: BE 日志

## 使用场景

1. **实时数据分析**: Kafka 收集日志 → Doris 实时分析
2. **用户行为分析**: 用户事件流 → Doris OLAP 查询
3. **监控告警**: 指标数据流 → Doris 聚合计算
4. **ETL 管道**: Kafka 作为中间层 → Doris 作为数仓

## 参考资料

- [Apache Doris 官方文档](https://doris.apache.org/docs/data-operate/import/import-way/routine-load-manual)
- [Doris Routine Load 指南](https://doris.apache.org/docs/data-operate/import/import-way/routine-load-manual)
- [Kafka 官方文档](https://kafka.apache.org/documentation/)

## 注意事项

- 本配置适用于开发和测试环境
- 生产环境建议使用多副本配置
- Doris 需要较多内存（建议 4GB+）
- 首次启动 Doris 可能需要 1-2 分钟
