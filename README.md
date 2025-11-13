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
