# Usage Examples - Примеры использования

## Обзор

Практические примеры использования очередей в различных модулях проекта.

## Пример 1: Mail Module

### Структура

```
mail/
├── application/
│   └── services/
│       └── mail.service.ts
├── infrastructure/
│   └── processors/
│       └── mail.processor.ts
├── events/
│   └── mail-events.constants.ts
└── mail.module.ts
```

### Константы

```typescript
// events/mail-events.constants.ts
export const MAIL_QUEUE_NAME = 'mail' as const;

export const MAIL_QUEUE_JOB_NAMES = {
    SEND_EMAIL: 'send-email',
} as const;

export interface SendEmailJobData {
    to: string[];
    subject: string;
    html: string;
    context?: Record<string, any>;
    emailType?: EmailType;
}
```

### Service

```typescript
// application/services/mail.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { MAIL_QUEUE_NAME, MAIL_QUEUE_JOB_NAMES, SendEmailJobData } from '../../events/mail-events.constants';

@Injectable()
export class MailService {
    private readonly logger = new Logger(MailService.name);

    constructor(
        @InjectQueue(MAIL_QUEUE_NAME) private readonly queue: Queue,
    ) {}

    async sendEmailVerification(user: User, token: string) {
        const html = await this.renderVerificationEmail(user, token);

        // ✅ Прямое добавление в очередь
        await this.queue.add(
            MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
            {
                to: [user.email],
                subject: 'Email Verification',
                html,
                emailType: EmailType.VERIFICATION,
            } as SendEmailJobData,
        );

        this.logger.log(`Email verification queued for ${user.email}`);
    }

    // Низкоуровневый метод (вызывается из Processor)
    async sendEmail(data: SendEmailJobData) {
        // Реальная отправка через SMTP
        await this.mailerService.sendMail({
            to: data.to,
            subject: data.subject,
            html: data.html,
        });
    }
}
```

### Processor

```typescript
// infrastructure/processors/mail.processor.ts
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Injectable, Logger } from '@nestjs/common';
import { Job } from 'bullmq';
import { MailService } from '../../application/services/mail.service';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import { MAIL_QUEUE_NAME, MAIL_WORKER_EVENTS, SendEmailJobData } from '../../events/mail-events.constants';

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

    async process(job: Job<SendEmailJobData>): Promise<void> {
        // ✅ Только вызов сервиса
        await this.mailService.sendEmail(job.data);
    }

    @OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
    async onCompleted(job: Job<SendEmailJobData>) {
        // ✅ Логирование, метрики
        this.logger.log(`Email job completed: ${job.id}`);

        // ✅ Бизнес-событие
        this.eventBus.emit(AppEvent.EMAIL_SENT, {
            emailType: job.data.emailType,
            to: job.data.to,
            subject: job.data.subject,
        });
    }

    @OnWorkerEvent(MAIL_WORKER_EVENTS.FAILED)
    async onFailed(job: Job<SendEmailJobData>, error: Error) {
        this.logger.error(`Email job failed: ${job.id}`, error.stack);
    }
}
```

### Module

```typescript
// mail.module.ts
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

## Пример 2: File Processing Module

### Service

```typescript
// file-processing/application/services/file-processing.service.ts
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { FILE_PROCESSING_QUEUE_NAME } from '../../events/file-processing-events.constants';

@Injectable()
export class FileProcessingService {
    constructor(
        @InjectQueue(FILE_PROCESSING_QUEUE_NAME) private readonly queue: Queue,
    ) {}

    async processImage(fileId: string, operations: string[]) {
        await this.queue.add(
            'process-image',
            {
                fileId,
                operations,
            },
            {
                attempts: 5, // Больше попыток для обработки файлов
                backoff: {
                    type: 'exponential',
                    delay: 5000,
                },
            },
        );
    }

    // Низкоуровневый метод (вызывается из Processor)
    async processImageFile(fileId: string, operations: string[]) {
        // Реальная обработка файла
        // ...
    }
}
```

### Processor

```typescript
// infrastructure/processors/file-processing.processor.ts
@Processor(FILE_PROCESSING_QUEUE_NAME)
export class FileProcessingProcessor extends WorkerHost {
    constructor(
        private readonly fileProcessingService: FileProcessingService,
        private readonly eventBus: AppEventBus,
    ) {
        super();
    }

    async process(job: Job<ProcessImageJobData>): Promise<void> {
        await this.fileProcessingService.processImageFile(
            job.data.fileId,
            job.data.operations,
        );
    }

    @OnWorkerEvent('completed')
    async onCompleted(job: Job<ProcessImageJobData>) {
        this.eventBus.emit(AppEvent.FILE_PROCESSED, {
            fileId: job.data.fileId,
        });
    }
}
```

## Пример 3: Добавление задачи из Listener

### Listener на событие

```typescript
// mail/infrastructure/listeners/application-email.listener.ts
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { OnAppEvent } from '@core/events/event-decorators';
import { AppEvent } from '@core/events/events.types';
import { MAIL_QUEUE_NAME, MAIL_QUEUE_JOB_NAMES } from '@mail/events/mail-events.constants';

@Injectable()
export class ApplicationEmailListener {
    constructor(
        @InjectQueue(MAIL_QUEUE_NAME) private readonly mailQueue: Queue,
    ) {}

    @OnAppEvent(AppEvent.APPLICATION_CREATED)
    async handleApplicationCreated(payload: {
        applicationId: string;
        employerId: string;
        employerEmail: string;
        vacancyId: string;
    }) {
        // ✅ Тяжелая операция - отправка в очередь
        await this.mailQueue.add(
            MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
            {
                to: [payload.employerEmail],
                subject: 'New Application',
                html: await this.renderApplicationEmail(payload),
                emailType: EmailType.APPLICATION_NEW,
            },
        );
    }
}
```

## Пример 4: Периодические задачи (Cron)

### Добавление повторяющейся задачи

```typescript
// reports/application/services/report.service.ts
@Injectable()
export class ReportService {
    constructor(
        @InjectQueue(REPORTS_QUEUE_NAME) private readonly queue: Queue,
    ) {}

    async scheduleDailyReport() {
        await this.queue.add(
            'generate-daily-report',
            {
                reportType: 'daily',
            },
            {
                repeat: {
                    cron: '0 9 * * *', // Каждый день в 9:00
                    tz: 'Europe/Moscow',
                },
            },
        );
    }
}
```

## Пример 5: Задачи с приоритетом

```typescript
// Отправка срочного email с высоким приоритетом
await this.mailQueue.add(
    MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
    {
        to: ['urgent@example.com'],
        subject: 'Urgent',
        html: '...',
    },
    {
        priority: 1, // Высокий приоритет (меньше = выше)
    },
);
```

## Пример 6: Задачи с задержкой

```typescript
// Отправка email через 1 час
await this.mailQueue.add(
    MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
    {
        to: ['user@example.com'],
        subject: 'Reminder',
        html: '...',
    },
    {
        delay: 3600000, // 1 час в миллисекундах
    },
);
```

## Best Practices

1. **Используйте константы** - для названий очередей и задач
2. **Типизируйте данные** - создавайте интерфейсы для job data
3. **Разделяйте ответственность** - Service добавляет, Processor обрабатывает
4. **Публикуйте события** - из @OnWorkerEvent('completed')
5. **Логируйте операции** - для отладки и мониторинга
