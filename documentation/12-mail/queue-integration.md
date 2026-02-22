# Queue Integration - Интеграция с очередями

## Обзор

Обязательная отправка всех email через очередь задач (BullMQ). Обеспечивает асинхронную обработку, retry логику и масштабируемость.

## Расположение

```
mail/infrastructure/processors/mail.processor.ts
mail/events/mail-events.constants.ts
```

## Принцип

**Все письма отправляются через очередь - синхронная отправка запрещена.**

```
MailService.sendEmailVerification()
  ↓
queue.add(SEND_EMAIL job)
  ↓
MailProcessor.process()
  ↓
MailService.sendEmail()
  ↓
@OnWorkerEvent('completed')
  ↓
EventBus.emit(EMAIL_SENT)
```

## Реализация

### MailProcessor

```typescript
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Injectable, Logger } from '@nestjs/common';
import { Job } from 'bullmq';
import { MailService } from '../../application/services/mail.service';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import {
    MAIL_QUEUE_NAME,
    MAIL_WORKER_EVENTS,
    EmailType
} from '../../events/mail-events.constants';

export interface SendEmailJobData {
    to: string[];
    subject: string;
    html: string;
    context?: Record<string, any>;
    emailType?: EmailType;
    attachments?: Array<{
        filename: string;
        content: Buffer;
        cid?: string;
        contentType: string;
    }>;
}

@Processor(MAIL_QUEUE_NAME)
@Injectable()
export class MailProcessor extends WorkerHost {
    private readonly logger = new Logger(MailProcessor.name);

    constructor(
        private readonly mailService: MailService,
        private readonly eventBus: AppEventBus,
    ) {
        super();
    }

    /**
     * Обработка задачи из очереди
     * ✅ Только вызов сервиса - вся логика там
     */
    async process(job: Job<SendEmailJobData>): Promise<void> {
        const { to, subject, html, context, attachments } = job.data;

        try {
            // ✅ Только вызов сервиса для отправки email
            await this.mailService.sendEmail({
                subject,
                html,
                to,
                context: context || {},
                attachments,
            });

            this.logger.log(`📧 Email successfully sent to ${to.join(', ')}`);
        } catch (error) {
            this.logger.error(
                `❌ Error sending email to ${to.join(', ')}: ${error instanceof Error ? error.message : String(error)}`,
                error instanceof Error ? error.stack : undefined,
            );
            throw error; // Re-throw to mark job as failed
        }
    }

    /**
     * ✅ @OnWorkerEvent - инфраструктурные вещи + бизнес-события
     */
    @OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
    async onCompleted(job: Job<SendEmailJobData>) {
        const { to, subject, emailType } = job.data;

        // ✅ Логирование, метрики, мониторинг
        this.logger.log(`Email job completed: ${job.id}`);
        // this.metrics.increment('email.sent');

        // ✅ Эмитим бизнес-событие после успешного завершения задачи
        this.eventBus.emit(AppEvent.EMAIL_SENT, {
            emailType: emailType || EmailType.OTHER,
            to: to,
            subject: subject,
        });
    }

    @OnWorkerEvent(MAIL_WORKER_EVENTS.FAILED)
    async onFailed(job: Job<SendEmailJobData>, error: Error) {
        const { to } = job.data;

        // ✅ Логирование, метрики, алерты
        this.logger.error(
            `❌ Email job failed for ${to.join(', ')}: ${error.message}`,
        );
        // this.metrics.increment('email.failed');
        // this.sentry.captureException(error);
    }
}
```

## Константы

### Mail Events Constants

```typescript
// events/mail-events.constants.ts
export const MAIL_QUEUE_JOB_NAMES = {
    SEND_EMAIL: 'send-email',
} as const;

export const MAIL_QUEUE_NAME = 'mail' as const;

export const MAIL_WORKER_EVENTS = {
    COMPLETED: 'completed',
    FAILED: 'failed',
    ACTIVE: 'active',
    STALLED: 'stalled',
} as const;

export enum EmailType {
    VERIFICATION = 'verification',
    PASSWORD_RESET = 'password-reset',
    OTHER = 'other',
}
```

## Регистрация очереди

### MailModule

```typescript
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { MailService } from './application/services/mail.service';
import { MailProcessor } from './infrastructure/processors/mail.processor';
import { MAIL_QUEUE_NAME } from './events/mail-events.constants';

@Module({
    imports: [
        BullModule.registerQueue({
            name: MAIL_QUEUE_NAME,
        }),
    ],
    providers: [MailService, MailProcessor],
    exports: [MailService],
})
export class MailModule {}
```

## Job Options

```typescript
// consts/mail.constants.ts
export const JOB_OPTIONS = {
    REMOVE_ON_COMPLETE: true,  // Удалять успешные задачи
    REMOVE_ON_FAIL: false,        // Оставлять неудачные для анализа
} as const;
```

## Retry логика

Настраивается в конфигурации очереди:

```typescript
await this.queue.add(
    MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
    jobData,
    {
        attempts: 3,              // Количество попыток
        backoff: {
            type: 'exponential',
            delay: 2000,          // Начальная задержка
        },
        removeOnComplete: true,
        removeOnFail: false,
    },
);
```

## Best Practices

1. **Всегда через очередь** - синхронная отправка запрещена
2. **Worker только вызывает сервис** - вся логика в MailService
3. **Публикуйте события** - через EventBus после успешной отправки
4. **Логируйте операции** - для отладки и мониторинга
5. **Обрабатывайте ошибки** - в @OnWorkerEvent('failed')
