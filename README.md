# Kafka Docker Compose

使用 Docker Compose 快速启动 Kafka 环境。

## 单节点配置

位于 `single-node/` 目录，使用 KRaft 模式（无需 Zookeeper）。

### 快速启动

```bash
cd single-node
docker compose up -d
```

### 访问

- **Kafka UI**: http://localhost:8080
- **Kafka Broker**: localhost:9093

详细说明请查看 [single-node/README.md](single-node/README.md)

## 三节点集群配置

位于 `three-nodes/` 目录，使用 KRaft 模式的三节点高可用集群。

### 快速启动

```bash
cd three-nodes
docker compose up -d
```

### 访问

- **Kafka UI**: http://localhost:8080
- **Kafka Brokers**: 
  - Node 1: localhost:19093
  - Node 2: localhost:29093
  - Node 3: localhost:39093

详细说明请查看 [three-nodes/README.md](three-nodes/README.md)

## Kafka + Doris 集成环境

位于 `kafka-doris/` 目录，包含 Kafka 和 Apache Doris 数据库的集成环境。

### 快速启动

```bash
cd kafka-doris
docker compose up -d
```

### 访问

- **Kafka UI**: http://localhost:8080
- **Doris Web UI**: http://localhost:8030
- **Doris MySQL**: localhost:9030 (用户名: root, 密码: 空)

详细说明请查看 [kafka-doris/README.md](kafka-doris/README.md)

## 特性

- ✅ KRaft 模式（Kafka 3.0+）
- ✅ 无需 Zookeeper
- ✅ Web UI 管理界面
- ✅ 开箱即用

## 停止服务

```bash
docker compose down
```

## License

MIT
