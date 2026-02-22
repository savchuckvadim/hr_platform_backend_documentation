# Prometheus Setup - Настройка Prometheus

## Обзор

Настройка Prometheus для сбора метрик из HR Platform приложения.

## Конфигурация

### Prometheus Config

```yaml
# config/prometheus/prometheus.yml
global:
  scrape_interval: 15s      # Интервал сбора метрик
  evaluation_interval: 15s  # Интервал оценки правил
  external_labels:
    cluster: 'hr-platform'
    environment: 'production'

# Правила алертов (опционально)
rule_files:
  # - "alerts.yml"

# Конфигурация scrape targets
scrape_configs:
  # HR Platform API
  - job_name: 'hr-platform-api'
    metrics_path: '/api/metrics'
    static_configs:
      - targets: ['api:3000']
        labels:
          service: 'api'
          environment: 'production'

  # Redis (если экспортирует метрики)
  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']
        labels:
          service: 'redis'

  # PostgreSQL (если используется postgres_exporter)
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
        labels:
          service: 'postgres'

  # Node Exporter (системные метрики)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          service: 'node'
```

## Docker Compose

См. [Docker Compose](./docker-compose.md)

## Метрики приложения

### HTTP метрики

- `http_requests_total` - общее количество HTTP запросов
- `http_requests_errors_total` - количество ошибок
- `http_request_duration_seconds` - длительность запросов

### Бизнес-метрики

- `users_registered_total` - количество зарегистрированных пользователей
- `applications_created_total` - количество созданных откликов
- `emails_sent_total` - количество отправленных писем
- `queue_jobs_total` - количество задач в очереди
- `queue_job_duration_seconds` - длительность задач
- `queue_jobs_active` - активные задачи

## Запросы PromQL

### Примеры запросов

```promql
# Количество запросов в секунду
rate(http_requests_total[5m])

# Процент ошибок
sum(rate(http_requests_errors_total[5m])) / sum(rate(http_requests_total[5m])) * 100

# Средняя длительность запросов
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Количество зарегистрированных пользователей
sum(users_registered_total)

# Количество активных задач в очереди
sum(queue_jobs_active)
```

## Best Practices

1. **Настройте scrape_interval** - в зависимости от нагрузки
2. **Используйте labels** - для группировки метрик
3. **Настройте retention** - для хранения метрик
4. **Создайте алерты** - для критичных метрик
