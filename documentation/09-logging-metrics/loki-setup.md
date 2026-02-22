# Loki Setup - Настройка Grafana Loki

## Обзор

Настройка Grafana Loki для централизованного хранения логов HR Platform.

## Конфигурация

### Loki Config

```yaml
# config/loki/config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

# Ingester конфигурация
ingester:
  wal:
    enabled: true
    dir: /loki/wal
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1

# Схема хранения
schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

# Конфигурация хранилища
storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/boltdb-cache
  filesystem:
    directory: /loki/chunks

# Compactor для оптимизации
compactor:
  shared_store: filesystem
  working_directory: /loki/compactor

# Лимиты
limits_config:
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_query_length: 721h
  max_query_parallelism: 32
  max_streams_per_user: 10000
  max_line_size: 256KB

# Retention (опционально)
table_manager:
  retention_deletes_enabled: true
  retention_period: 720h  # 30 дней
```

## Docker Compose

См. [Docker Compose](./docker-compose.md)

## Запросы LogQL

### Примеры запросов

```logql
# Все логи за последний час
{job="hr-platform-api"}

# Логи уровня error
{job="hr-platform-api"} |= "error"

# Логи по конкретному URL
{job="hr-platform-api"} | json | url="/api/auth/login"

# Количество логов по уровням
sum(count_over_time({job="hr-platform-api"}[1m])) by (level)

# Логи с определенным user_id
{job="hr-platform-api"} | json | userId="user-123"
```

## Best Practices

1. **Настройте retention** - для управления размером хранилища
2. **Используйте labels** - для эффективного поиска
3. **Ограничьте размер логов** - max_line_size
4. **Настройте ingestion_rate** - для контроля нагрузки
