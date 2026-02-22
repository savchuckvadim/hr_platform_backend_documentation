# Event-based Metrics - Метрики через события

## Обзор

Сбор метрик через бизнес-события и события очередей в соответствии с best practices.

## Принципы

1. **Метрики из бизнес-событий** - через EventBus listeners
2. **Метрики из очередей** - через @OnWorkerEvent
3. **Типы метрик** - Counter, Histogram, Gauge
4. **Labels** - для группировки и фильтрации

## Метрики из бизнес-событий

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
        // ✅ Метрика регистрации
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
    async handleApplicationCreated(payload: {
        applicationId: string;
        status: ApplicationStatus;
        vacancyId: string;
    }) {
        // ✅ Метрика создания откликов
        this.applicationsCounter
            .labels(payload.status)
            .inc();
    }
}
```

### Отправка email

```typescript
// mail/infrastructure/listeners/email-sent.listener.ts
@Injectable()
export class EmailSentListener {
    constructor(
        @InjectMetric('emails_sent_total')
        private readonly emailsCounter: Counter<string>,
    ) {}

    @OnAppEvent(AppEvent.EMAIL_SENT)
    async handleEmailSent(payload: {
        emailType: EmailType;
        to: string[];
        status: 'success' | 'failed';
    }) {
        // ✅ Метрика отправки email
        this.emailsCounter
            .labels(payload.emailType, payload.status)
            .inc();
    }
}
```

## Метрики из очередей

### Mail Queue

```typescript
// mail/infrastructure/processors/mail.processor.ts
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Injectable } from '@nestjs/common';
import { Job } from 'bullmq';
import { Counter, Histogram, Gauge } from 'prom-client';
import { InjectMetric } from '@willsoto/nestjs-prometheus';
import { MAIL_QUEUE_NAME, MAIL_WORKER_EVENTS } from '../../events/mail-events.constants';

@Processor(MAIL_QUEUE_NAME)
@Injectable()
export class MailProcessor extends WorkerHost {
    constructor(
        private readonly mailService: MailService,
        @InjectMetric('queue_jobs_total')
        private readonly queueJobsCounter: Counter<string>,
        @InjectMetric('queue_job_duration_seconds')
        private readonly queueJobDuration: Histogram<string>,
        @InjectMetric('queue_jobs_active')
        private readonly queueJobsActive: Gauge<string>,
    ) {
        super();
    }

    async process(job: Job<SendEmailJobData>): Promise<void> {
        const start = Date.now();

        // ✅ Увеличиваем счетчик активных задач
        this.queueJobsActive
            .labels(MAIL_QUEUE_NAME)
            .inc();

        try {
            await this.mailService.sendEmail(job.data);
        } finally {
            const duration = (Date.now() - start) / 1000;

            // ✅ Уменьшаем счетчик активных задач
            this.queueJobsActive
                .labels(MAIL_QUEUE_NAME)
                .dec();
        }
    }

    @OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
    async onCompleted(job: Job<SendEmailJobData>) {
        const duration = (Date.now() - job.timestamp) / 1000;

        // ✅ Метрика успешных задач
        this.queueJobsCounter
            .labels(MAIL_QUEUE_NAME, job.name, 'completed')
            .inc();

        // ✅ Метрика длительности
        this.queueJobDuration
            .labels(MAIL_QUEUE_NAME, job.name)
            .observe(duration);
    }

    @OnWorkerEvent(MAIL_WORKER_EVENTS.FAILED)
    async onFailed(job: Job<SendEmailJobData>, error: Error) {
        // ✅ Метрика failed задач
        this.queueJobsCounter
            .labels(MAIL_QUEUE_NAME, job.name, 'failed')
            .inc();
    }

    @OnWorkerEvent(MAIL_WORKER_EVENTS.ACTIVE)
    async onActive(job: Job<SendEmailJobData>) {
        // ✅ Метрика активных задач
        this.queueJobsCounter
            .labels(MAIL_QUEUE_NAME, job.name, 'active')
            .inc();
    }
}
```

## Рекомендуемые метрики

### HTTP метрики

- `http_requests_total` - общее количество запросов
- `http_requests_errors_total` - количество ошибок
- `http_request_duration_seconds` - длительность запросов

### Бизнес-метрики

- `users_registered_total` - регистрации по ролям
- `users_logged_in_total` - входы пользователей
- `applications_created_total` - создание откликов по статусам
- `applications_status_changed_total` - изменение статусов откликов
- `vacancies_created_total` - создание вакансий
- `vacancies_published_total` - публикация вакансий
- `resumes_created_total` - создание резюме
- `messages_sent_total` - отправка сообщений в чате

### Email метрики

- `emails_sent_total` - отправка писем по типам и статусам
- `emails_failed_total` - неудачные отправки

### Queue метрики

- `queue_jobs_total` - количество задач по статусам
- `queue_job_duration_seconds` - длительность задач
- `queue_jobs_active` - активные задачи
- `queue_jobs_waiting` - задачи в ожидании
- `queue_jobs_failed` - неудачные задачи

### Database метрики

- `db_queries_total` - количество запросов к БД
- `db_query_duration_seconds` - длительность запросов
- `db_connections_active` - активные соединения

### Cache метрики

- `cache_hits_total` - попадания в кэш
- `cache_misses_total` - промахи кэша
- `cache_operations_total` - операции с кэшем

## Best Practices

1. **Используйте Counter** - для подсчета событий
2. **Используйте Histogram** - для длительности операций
3. **Используйте Gauge** - для текущих значений (активные задачи)
4. **Добавляйте labels** - для группировки метрик
5. **Не перегружайте метриками** - только важные события
6. **Собирайте метрики из событий** - для бизнес-метрик
7. **Собирайте метрики из очередей** - для инфраструктурных метрик
