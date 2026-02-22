# Promtail Configuration - Настройка Promtail

## Обзор

Настройка Promtail для сбора логов из Docker контейнеров и отправки в Grafana Loki.

## Конфигурация

### Promtail Config

```yaml
# config/promtail/config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: hr-platform-api
          __path__: /var/lib/docker/containers/*/*.log

    pipeline_stages:
      # Парсинг Docker логов
      - docker: {}

      # Извлечение labels из контейнера
      - labels:
          container_name:
          container_id:

      # Парсинг JSON логов
      - match:
          selector: '{container_name="hr-platform-api"}'
          stages:
            - json:
                expressions:
                  timestamp: timestamp
                  level: level
                  message: message
                  context: context
                  trace: trace
                  userId: userId
                  requestId: requestId

            # Добавление labels из JSON
            - labels:
                level:
                context:

            # Форматирование вывода
            - output:
                source: message
```

## Docker Compose

См. [Docker Compose](./docker-compose.md)

## Структура логов

### Формат JSON логов

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "info",
  "message": "Request completed",
  "context": "LoggerMiddleware",
  "method": "POST",
  "url": "/api/auth/login",
  "statusCode": 200,
  "duration": "45ms",
  "userId": "user-123",
  "requestId": "req-456",
  "trace": "trace-789"
}
```

## Best Practices

1. **Используйте JSON формат** - для структурированного логирования
2. **Добавляйте labels** - для эффективного поиска
3. **Извлекайте контекст** - userId, requestId, trace
4. **Фильтруйте логи** - на уровне Promtail
