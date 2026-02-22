# Metrics Collection - Сбор метрик

## Обзор

Сбор метрик производительности, ошибок и бизнес-метрик в HR Platform.

## HTTP метрики

### Автоматический сбор

Собираются автоматически через `MetricsInterceptor`:

- `http_requests_total` - общее количество запросов
- `http_requests_errors_total` - количество ошибок
- `http_request_duration_seconds` - длительность запросов

## Бизнес-метрики

### Регистрация пользователей

```typescript
// auth/infrastructure/listeners/user-created.listener.ts
import { Injectable } from '@nestjs/common';
import { OnAppEvent } from '@core/events/event-decorators';
import { AppEvent } from '@core/events/events.types';
import { Counter } from 'prom-client';
import { InjectMetric } from '@willsoto/nestjs-prometheus';

@Injectable()
export class UserCreatedListener {
    constructor(
        @InjectMetric('users_registered_total')
        private readonly usersRegisteredCounter: Counter<string>,
    ) {}

    @OnAppEvent(AppEvent.USER_CREATED)
    async handleUserCreated(payload: { userId: string; roleType: RoleType }) {
        // ✅ Метрика регистрации пользователей
        this.usersRegisteredCounter
            .labels(payload.roleType)
            .inc();
    }
}
```

### Создание откликов

```typescript
// applications/infrastructure/listeners/application-created.listener.ts
@Injectable()
export class ApplicationCreatedListener {
    constructor(
        @InjectMetric('applications_created_total')
        private readonly applicationsCounter: Counter<string>,
    ) {}

    @OnAppEvent(AppEvent.APPLICATION_CREATED)
    async handleApplicationCreated(payload: { applicationId: string; status: ApplicationStatus }) {
        // ✅ Метрика создания откликов
        this.applicationsCounter
            .labels(payload.status)
            .inc();
    }
}
```

## Метрики очередей

См. [Event-based Metrics](./event-based-metrics.md)

## Best Practices

1. **Используйте labels** - для группировки метрик
2. **Собирайте метрики через события** - для бизнес-метрик
3. **Не перегружайте метриками** - только важные события
4. **Используйте правильные типы** - Counter, Histogram, Gauge
