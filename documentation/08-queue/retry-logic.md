# Retry Logic - Логика повторных попыток

## Обзор

Настройка логики повторных попыток для failed задач. BullMQ поддерживает различные стратегии retry с настраиваемыми задержками.

## Стратегии retry

### Exponential Backoff (экспоненциальная задержка)

**Рекомендуется для большинства случаев.**

```typescript
await this.queue.add('send-email', data, {
    attempts: 3,
    backoff: {
        type: 'exponential',
        delay: 2000, // Начальная задержка 2 секунды
    },
});
```

**Задержки:**
- Попытка 1: сразу
- Попытка 2: через 2 секунды
- Попытка 3: через 4 секунды
- Попытка 4: через 8 секунд

### Fixed Backoff (фиксированная задержка)

```typescript
await this.queue.add('send-email', data, {
    attempts: 5,
    backoff: {
        type: 'fixed',
        delay: 5000, // Фиксированная задержка 5 секунд
    },
});
```

**Задержки:**
- Попытка 1: сразу
- Попытка 2: через 5 секунд
- Попытка 3: через 5 секунд
- Попытка 4: через 5 секунд

## Настройка attempts

### Разные значения для разных типов задач

```typescript
// Email - 3 попытки
await this.mailQueue.add('send-email', data, {
    attempts: 3,
    backoff: {
        type: 'exponential',
        delay: 2000,
    },
});

// File processing - 5 попыток
await this.fileQueue.add('process-file', data, {
    attempts: 5,
    backoff: {
        type: 'exponential',
        delay: 5000,
    },
});

// External API - 10 попыток
await this.apiQueue.add('call-external-api', data, {
    attempts: 10,
    backoff: {
        type: 'exponential',
        delay: 1000,
    },
});
```

## Обработка failed задач

### @OnWorkerEvent('failed')

```typescript
@Processor(MAIL_QUEUE_NAME)
export class MailProcessor extends WorkerHost {
    @OnWorkerEvent('failed')
    async onFailed(job: Job<SendEmailJobData>, error: Error) {
        const { to, emailType } = job.data;

        // ✅ Логирование
        this.logger.error(
            `Email job failed for ${to.join(', ')}: ${error.message}`,
            error.stack,
        );

        // ✅ Метрики
        // this.metrics.increment('email.failed');

        // ✅ Алерты
        // this.sentry.captureException(error);

        // ✅ Может триггерить бизнес-событие для обработки ошибок
        this.eventBus.emit(AppEvent.EMAIL_SEND_FAILED, {
            email: to[0],
            error: error.message,
            jobId: job.id,
            attemptsMade: job.attemptsMade,
        });
    }
}
```

### Проверка количества попыток

```typescript
@OnWorkerEvent('failed')
async onFailed(job: Job<SendEmailJobData>, error: Error) {
    // Проверка, были ли исчерпаны все попытки
    if (job.attemptsMade >= job.opts.attempts) {
        // Все попытки исчерпаны - критическая ошибка
        this.logger.error(
            `Email job failed after ${job.attemptsMade} attempts`,
            error.stack,
        );

        // Отправка в систему мониторинга
        // this.sentry.captureException(error);
    } else {
        // Еще будут попытки - просто логируем
        this.logger.warn(
            `Email job failed, will retry (attempt ${job.attemptsMade}/${job.opts.attempts})`,
        );
    }
}
```

## Условный retry

### Retry только для определенных ошибок

```typescript
@Processor(MAIL_QUEUE_NAME)
export class MailProcessor extends WorkerHost {
    async process(job: Job<SendEmailJobData>): Promise<void> {
        try {
            await this.mailService.sendEmail(job.data);
        } catch (error) {
            // ✅ Не retry для определенных ошибок
            if (error instanceof InvalidEmailError) {
                // Не повторяем для невалидных email
                this.logger.error(`Invalid email: ${job.data.to[0]}`);
                return; // Задача завершается без retry
            }

            // Для других ошибок - пробрасываем для retry
            throw error;
        }
    }
}
```

## Best Practices

1. **Используйте exponential backoff** - для большинства случаев
2. **Настраивайте attempts** - в зависимости от типа задачи
3. **Логируйте failed задачи** - для анализа
4. **Обрабатывайте критические ошибки** - после исчерпания попыток
5. **Условный retry** - не retry для некритичных ошибок
