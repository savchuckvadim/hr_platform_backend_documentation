# Adding Jobs to Queue - Добавление задач в очередь

## Обзор

Добавление задач в очередь напрямую из сервисов через `queue.add()`. **Важно:** Dispatcher не используется - сервисы напрямую работают с очередями.

## Принцип

**Сервисы напрямую используют `@InjectQueue()` и вызывают `queue.add()`.**

## Реализация

### Инъекция очереди в сервис

```typescript
// mail/application/services/mail.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { MAIL_QUEUE_NAME, MAIL_QUEUE_JOB_NAMES } from '../../events/mail-events.constants';

@Injectable()
export class MailService {
    private readonly logger = new Logger(MailService.name);

    constructor(
        @InjectQueue(MAIL_QUEUE_NAME) private readonly queue: Queue,
    ) {}

    async sendEmailVerification(user: User, token: string) {
        // Подготовка данных
        const html = await this.renderVerificationEmail(user, token);

        // ✅ Прямое добавление задачи в очередь
        await this.queue.add(
            MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
            {
                to: [user.email],
                subject: 'Email Verification',
                html,
                emailType: EmailType.VERIFICATION,
            },
            {
                // Опции задачи (опционально, можно переопределить defaults)
                attempts: 3,
                backoff: {
                    type: 'exponential',
                    delay: 2000,
                },
                removeOnComplete: true,
                removeOnFail: false,
            },
        );

        this.logger.log(`Email verification queued for ${user.email}`);
    }
}
```

## Методы добавления задач

### queue.add() - Базовый метод

```typescript
await queue.add(
    jobName: string,        // Название задачи
    data: any,             // Данные задачи
    options?: JobOptions,   // Опции задачи (опционально)
): Promise<Job>
```

**Пример:**
```typescript
const job = await this.queue.add(
    'send-email',
    {
        to: ['user@example.com'],
        subject: 'Test',
        html: '<h1>Test</h1>',
    },
    {
        attempts: 3,
        delay: 5000, // Задержка перед выполнением (5 секунд)
    },
);
```

### queue.addBulk() - Массовое добавление

```typescript
await queue.addBulk([
    {
        name: 'send-email',
        data: { to: ['user1@example.com'], subject: 'Email 1', html: '...' },
    },
    {
        name: 'send-email',
        data: { to: ['user2@example.com'], subject: 'Email 2', html: '...' },
    },
]);
```

## Job Options

### Основные опции

```typescript
interface JobOptions {
    // Идентификатор задачи (для предотвращения дубликатов)
    jobId?: string;

    // Количество попыток
    attempts?: number;

    // Задержка перед выполнением (миллисекунды)
    delay?: number;

    // Стратегия повторных попыток
    backoff?: {
        type: 'fixed' | 'exponential';
        delay: number;
    };

    // Приоритет (меньше = выше приоритет)
    priority?: number;

    // Удаление успешных задач
    removeOnComplete?: boolean | number | { age?: number; count?: number };

    // Удаление неудачных задач
    removeOnFail?: boolean | number | { age?: number; count?: number };

    // Повторять задачу через определенный интервал
    repeat?: {
        every?: number;      // Интервал в миллисекундах
        cron?: string;       // Cron выражение
        tz?: string;         // Timezone
    };
}
```

### Примеры опций

#### С задержкой

```typescript
await this.queue.add('send-email', data, {
    delay: 10000, // Выполнить через 10 секунд
});
```

#### С приоритетом

```typescript
await this.queue.add('send-email', data, {
    priority: 1, // Высокий приоритет (меньше = выше)
});
```

#### С повторением (cron)

```typescript
await this.queue.add('send-daily-report', data, {
    repeat: {
        cron: '0 9 * * *', // Каждый день в 9:00
        tz: 'Europe/Moscow',
    },
});
```

#### С уникальным jobId (предотвращение дубликатов)

```typescript
await this.queue.add('send-email', data, {
    jobId: `email-${userId}-${emailType}`, // Уникальный ID
});
```

## Использование в разных модулях

### Mail Module

```typescript
// mail/application/services/mail.service.ts
@Injectable()
export class MailService {
    constructor(
        @InjectQueue(MAIL_QUEUE_NAME) private readonly queue: Queue,
    ) {}

    async sendEmailVerification(user: User, token: string) {
        await this.queue.add(MAIL_QUEUE_JOB_NAMES.SEND_EMAIL, {
            to: [user.email],
            subject: 'Email Verification',
            html: await this.renderEmail(user, token),
            emailType: EmailType.VERIFICATION,
        });
    }
}
```

### File Processing Module

```typescript
// file-processing/application/services/file-processing.service.ts
@Injectable()
export class FileProcessingService {
    constructor(
        @InjectQueue(FILE_PROCESSING_QUEUE_NAME) private readonly queue: Queue,
    ) {}

    async processImage(fileId: string) {
        await this.queue.add('process-image', {
            fileId,
            operations: ['resize', 'optimize'],
        }, {
            attempts: 5, // Больше попыток для обработки файлов
            backoff: {
                type: 'exponential',
                delay: 5000,
            },
        });
    }
}
```

### Reports Module

```typescript
// reports/application/services/report.service.ts
@Injectable()
export class ReportService {
    constructor(
        @InjectQueue(REPORTS_QUEUE_NAME) private readonly queue: Queue,
    ) {}

    async generateReport(userId: string, reportType: string) {
        await this.queue.add('generate-report', {
            userId,
            reportType,
        }, {
            priority: 1, // Высокий приоритет для отчетов
            attempts: 2,
        });
    }
}
```

## Добавление задач из Listeners

### Когда listener отправляет в очередь

Listeners на глобальные события могут отправлять задачи в очередь для **тяжелых операций**:

```typescript
// mail/infrastructure/listeners/application-email.listener.ts
@Injectable()
export class ApplicationEmailListener {
    constructor(
        @InjectQueue(MAIL_QUEUE_NAME) private readonly mailQueue: Queue,
    ) {}

    @OnAppEvent(AppEvent.APPLICATION_CREATED)
    async handleApplicationCreated(payload: ApplicationCreatedPayload) {
        // ✅ Тяжелая операция - отправка в очередь
        await this.mailQueue.add(MAIL_QUEUE_JOB_NAMES.SEND_EMAIL, {
            to: [payload.employerEmail],
            subject: 'New Application',
            html: await this.renderApplicationEmail(payload),
            emailType: EmailType.APPLICATION_NEW,
        });
    }
}
```

## Типизация данных задачи

### Интерфейс данных задачи

```typescript
// mail/events/mail-events.constants.ts
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
```

### Использование в Processor

```typescript
// mail/infrastructure/processors/mail.processor.ts
@Processor(MAIL_QUEUE_NAME)
export class MailProcessor extends WorkerHost {
    async process(job: Job<SendEmailJobData>): Promise<void> {
        // ✅ job.data типизирован
        const { to, subject, html } = job.data;
        // ...
    }
}
```

## Best Practices

1. **Используйте константы** - для названий очередей и задач
2. **Типизируйте данные** - создавайте интерфейсы для job data
3. **Настраивайте опции** - для retry, priority, delay
4. **Логируйте добавление** - для отладки
5. **Используйте jobId** - для предотвращения дубликатов
6. **Не используйте dispatcher** - работайте напрямую с queue.add()
