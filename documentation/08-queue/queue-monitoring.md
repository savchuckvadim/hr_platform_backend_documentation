# Queue Monitoring - Мониторинг очередей

## Обзор

Мониторинг очередей и задач для отслеживания производительности, ошибок и состояния системы.

## Метрики

### Сбор метрик в @OnWorkerEvent

```typescript
@Processor(MAIL_QUEUE_NAME)
export class MailProcessor extends WorkerHost {
    constructor(
        private readonly mailService: MailService,
        private readonly eventBus: AppEventBus,
        // private readonly metrics: MetricsService, // Prometheus metrics
    ) {
        super();
    }

    @OnWorkerEvent('completed')
    async onCompleted(job: Job<SendEmailJobData>) {
        // ✅ Метрики успешных задач
        // this.metrics.increment('email.sent', {
        //     type: job.data.emailType || 'other',
        // });

        // ✅ Метрики времени выполнения
        const duration = Date.now() - job.timestamp;
        // this.metrics.histogram('email.duration', duration);

        this.logger.log(`Email job completed: ${job.id} (${duration}ms)`);
    }

    @OnWorkerEvent('failed')
    async onFailed(job: Job<SendEmailJobData>, error: Error) {
        // ✅ Метрики failed задач
        // this.metrics.increment('email.failed', {
        //     type: job.data.emailType || 'other',
        //     error: error.name,
        // });

        this.logger.error(`Email job failed: ${job.id}`, error.stack);
    }

    @OnWorkerEvent('active')
    async onActive(job: Job<SendEmailJobData>) {
        // ✅ Метрики активных задач
        // this.metrics.increment('email.active');

        this.logger.debug(`Email job started: ${job.id}`);
    }
}
```

## Логирование

### Структурированное логирование

```typescript
@OnWorkerEvent('completed')
async onCompleted(job: Job<SendEmailJobData>) {
    this.logger.log({
        event: 'email.completed',
        jobId: job.id,
        email: job.data.to[0],
        emailType: job.data.emailType,
        duration: Date.now() - job.timestamp,
    });
}

@OnWorkerEvent('failed')
async onFailed(job: Job<SendEmailJobData>, error: Error) {
    this.logger.error({
        event: 'email.failed',
        jobId: job.id,
        email: job.data.to[0],
        emailType: job.data.emailType,
        error: error.message,
        stack: error.stack,
        attemptsMade: job.attemptsMade,
    });
}
```

## Получение статистики очереди

### Методы Queue API

```typescript
// Получение количества задач в очереди
const waiting = await queue.getWaitingCount();
const active = await queue.getActiveCount();
const completed = await queue.getCompletedCount();
const failed = await queue.getFailedCount();
const delayed = await queue.getDelayedCount();

// Получение задач
const waitingJobs = await queue.getWaiting();
const activeJobs = await queue.getActive();
const completedJobs = await queue.getCompleted(0, 10); // Последние 10
const failedJobs = await queue.getFailed(0, 10); // Последние 10
```

### Endpoint для мониторинга

```typescript
// api/controllers/queue-monitoring.controller.ts
@Controller('admin/queues')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(RoleType.ADMIN)
export class QueueMonitoringController {
    constructor(
        @InjectQueue(MAIL_QUEUE_NAME) private readonly mailQueue: Queue,
    ) {}

    @Get('mail/stats')
    async getMailQueueStats() {
        return {
            waiting: await this.mailQueue.getWaitingCount(),
            active: await this.mailQueue.getActiveCount(),
            completed: await this.mailQueue.getCompletedCount(),
            failed: await this.mailQueue.getFailedCount(),
            delayed: await this.mailQueue.getDelayedCount(),
        };
    }

    @Get('mail/failed')
    async getFailedJobs(@Query('limit') limit = 10) {
        const jobs = await this.mailQueue.getFailed(0, limit);
        return jobs.map(job => ({
            id: job.id,
            data: job.data,
            error: job.failedReason,
            attemptsMade: job.attemptsMade,
            timestamp: job.timestamp,
        }));
    }
}
```

## Алерты

### Отправка алертов при критических ошибках

```typescript
@OnWorkerEvent('failed')
async onFailed(job: Job<SendEmailJobData>, error: Error) {
    // Проверка критичности ошибки
    if (job.attemptsMade >= job.opts.attempts) {
        // Все попытки исчерпаны - критическая ошибка
        await this.sendAlert({
            type: 'critical',
            message: `Email job failed after ${job.attemptsMade} attempts`,
            jobId: job.id,
            error: error.message,
        });
    }
}
```

## Best Practices

1. **Собирайте метрики** - для всех событий (completed, failed, active)
2. **Логируйте структурированно** - для анализа
3. **Мониторьте размер очереди** - для предотвращения переполнения
4. **Алерты для критических ошибок** - после исчерпания попыток
5. **Endpoint для мониторинга** - для администраторов
