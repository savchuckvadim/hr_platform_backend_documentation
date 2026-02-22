# Logging and Metrics - Логирование и метрики

## Описание

Система логирования и метрик для HR Platform с использованием Grafana Loki, Promtail и Prometheus. Обеспечивает централизованное хранение логов, сбор метрик производительности и визуализацию в Grafana.

## Архитектурные принципы

### Ключевые принципы

1. **Структурированное логирование** - JSON формат для всех логов
2. **Централизованное хранение** - все логи в Grafana Loki
3. **Автоматический сбор метрик** - через interceptors и middleware
4. **Метрики через события** - сбор метрик из бизнес-событий и очередей
5. **Визуализация** - дашборды в Grafana для мониторинга

## Структура модуля

### Core Metrics Module

```
core/metrics/
├── metrics.module.ts              # Модуль метрик
├── metrics.service.ts             # Сервис для работы с метриками
└── consts/
    └── metrics.constants.ts       # Константы метрик
```

### Core Interceptors

```
core/interceptors/
├── metrics.interceptor.ts        # Сбор HTTP метрик
└── response.interceptor.ts        # Стандартизация ответов
```

### Core Middleware

```
core/middleware/
└── logger.middleware.ts          # Логирование запросов
```

### Config

```
config/
├── prometheus/
│   └── prometheus.yml            # Конфигурация Prometheus
├── loki/
│   └── config.yaml               # Конфигурация Loki
└── promtail/
    └── config.yaml               # Конфигурация Promtail
```

## Задачи

### ✅ Prometheus Setup
**Статус**: Завершено
**Файл**: [prometheus-setup.md](./prometheus-setup.md)
**Описание**: Настройка Prometheus для сбора метрик

### ✅ Loki Setup
**Статус**: Завершено
**Файл**: [loki-setup.md](./loki-setup.md)
**Описание**: Настройка Grafana Loki для сбора логов

### ✅ Promtail Configuration
**Статус**: Завершено
**Файл**: [promtail-config.md](./promtail-config.md)
**Описание**: Настройка Promtail для отправки логов в Loki

### ✅ Structured Logging
**Статус**: Завершено
**Файл**: [structured-logging.md](./structured-logging.md)
**Описание**: Структурированное логирование в приложении

### ✅ Metrics Collection
**Статус**: Завершено
**Файл**: [metrics-collection.md](./metrics-collection.md)
**Описание**: Сбор метрик приложения (performance, errors, etc.)

### ✅ Integration
**Статус**: Завершено
**Файл**: [integration.md](./integration.md)
**Описание**: Интеграция в проект (interceptors, middleware)

### ✅ Docker Compose
**Статус**: Завершено
**Файл**: [docker-compose.md](./docker-compose.md)
**Описание**: Конфигурация Docker Compose для мониторинга

### ✅ Event-based Metrics
**Статус**: Завершено
**Файл**: [event-based-metrics.md](./event-based-metrics.md)
**Описание**: Сбор метрик через события и очереди

## Ключевые концепции

- **Grafana Loki** - централизованное хранение логов
- **Promtail** - сбор и отправка логов в Loki
- **Prometheus** - сбор и хранение метрик
- **Grafana** - визуализация логов и метрик
- **Структурированное логирование** - JSON формат
- **Уровни логирования** - debug, info, warn, error
- **Контекстное логирование** - request ID, user ID, trace ID
- **HTTP метрики** - requests, duration, errors
- **Бизнес-метрики** - через события и очереди

## Ссылки

- [Prometheus Setup](./prometheus-setup.md) - настройка Prometheus
- [Loki Setup](./loki-setup.md) - настройка Grafana Loki
- [Promtail Configuration](./promtail-config.md) - настройка Promtail
- [Structured Logging](./structured-logging.md) - структурированное логирование
- [Metrics Collection](./metrics-collection.md) - сбор метрик
- [Integration](./integration.md) - интеграция в проект
- [Docker Compose](./docker-compose.md) - конфигурация Docker
- [Event-based Metrics](./event-based-metrics.md) - метрики через события
