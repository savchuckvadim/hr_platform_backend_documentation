# Queue Setup - Настройка очередей

## Обзор

Настройка BullMQ и Redis для работы с очередями задач. Глобальный модуль очередей регистрируется в `core/queue` и используется всеми бизнес-модулями.

## Расположение

```
core/queue/
├── queue.module.ts
├── consts/
│   ├── queue-names.enum.ts
│   └── job-names.enum.ts
└── index.ts
```

## Установка

```bash
npm install @nestjs/bullmq bullmq ioredis
```

## Реализация

### Queue Module

```typescript
// core/queue/queue.module.ts
import { Global, Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { createRedisOptions } from '../redis/redis.config';

@Global()
@Module({
    imports: [
        BullModule.forRootAsync({
            imports: [ConfigModule],
            useFactory: (configService: ConfigService) => {
                const redisOptions = createRedisOptions(configService);

                return {
                    connection: redisOptions.url
                        ? { url: redisOptions.url }
                        : {
                            host: redisOptions.host,
                            port: redisOptions.port,
                        },
                    defaultJobOptions: {
                        attempts: 3,
                        backoff: {
                            type: 'exponential',
                            delay: 2000,
                        },
                        removeOnComplete: {
                            age: 3600, // Удалять успешные задачи старше 1 часа
                            count: 1000, // Или оставлять последние 1000
                        },
                        removeOnFail: {
                            age: 86400, // Удалять неудачные задачи старше 24 часов
                        },
                    },
                };
            },
            inject: [ConfigService],
        }),
    ],
    exports: [BullModule],
})
export class QueueModule {}
```

### Queue Names

```typescript
// core/queue/consts/queue-names.enum.ts
export enum QueueNames {
    MAIL = 'mail',
    FILE_PROCESSING = 'file-processing',
    REPORTS = 'reports',
}
```

### Job Names (опционально)

```typescript
// core/queue/consts/job-names.enum.ts
export enum JobNames {
    // Mail jobs
    SEND_EMAIL = 'send-email',


    // File processing jobs
    PROCESS_IMAGE = 'process-image',
    GENERATE_REPORT = 'generate-report',
}
```

## Регистрация очереди в модуле

### Mail Module

```typescript
// mail/mail.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bullmq';
import { MailService } from './application/services/mail.service';
import { MailProcessor } from './infrastructure/processors/mail.processor';
import { MAIL_QUEUE_NAME } from './events/mail-events.constants';

@Module({
    imports: [
        // ✅ Регистрация очереди для модуля
        BullModule.registerQueue({
            name: MAIL_QUEUE_NAME,
        }),
    ],
    providers: [MailService, MailProcessor],
    exports: [MailService],
})
export class MailModule {}
```

### Константы очереди в модуле

```typescript
// mail/events/mail-events.constants.ts
export const MAIL_QUEUE_NAME = 'mail' as const;

export const MAIL_QUEUE_JOB_NAMES = {
    SEND_EMAIL: 'send-email',
} as const;
```

## Environment Variables

```env
# Redis для очередей (используется тот же Redis что и для кэша)
REDIS_URL=redis://localhost:6379
# или
REDIS_HOST=localhost
REDIS_PORT=6379
```

## Default Job Options

Настройки по умолчанию для всех задач:

```typescript
defaultJobOptions: {
    attempts: 3,                    // Количество попыток
    backoff: {
        type: 'exponential',        // Экспоненциальная задержка
        delay: 2000,                // Начальная задержка 2 секунды
    },
    removeOnComplete: {
        age: 3600,                  // Удалять успешные задачи старше 1 часа
        count: 1000,                // Или оставлять последние 1000
    },
    removeOnFail: {
        age: 86400,                 // Удалять неудачные задачи старше 24 часов
    },
}
```

Эти настройки можно переопределить при добавлении задачи.

## Best Practices

1. **Глобальный модуль** - QueueModule должен быть глобальным
2. **Разделение очередей** - каждая очередь для своего типа задач
3. **Константы** - используйте константы для названий очередей и задач
4. **Default options** - настройте разумные значения по умолчанию
5. **Очистка задач** - настройте автоматическую очистку старых задач
