# Event Module Setup

## Описание

Настройка модуля событий и интеграция с EventEmitter2 и BullMQ.

## Структура модуля

### Файл: `src/core/events/event.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';
import { BullModule } from '@nestjs/bullmq';
import { AppEventBus } from './event-bus.service';

@Module({
  imports: [
    // EventEmitter для локальных событий
    EventEmitterModule.forRoot({
      wildcard: false,
      delimiter: '.',
      maxListeners: 10,
      verboseMemoryLeak: false,
      ignoreErrors: false,
    }),

    // BullMQ для асинхронных событий через очередь
    BullModule.registerQueue({
      name: 'events',
      defaultJobOptions: {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000,
        },
      },
    }),
  ],
  providers: [AppEventBus],
  exports: [AppEventBus],
})
export class EventModule {}
```

## Использование в модулях

### Импорт в модуле

```typescript
import { Module } from '@nestjs/common';
import { EventModule } from '@core/events/event.module';
import { UserService } from './application/services/user.service';
import { UserCreatedListener } from './infrastructure/listeners/user-created.listener';

@Module({
  imports: [EventModule],
  providers: [UserService, UserCreatedListener],
})
export class UserModule {}
```

### Использование в сервисе

```typescript
import { Injectable } from '@nestjs/common';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';

@Injectable()
export class UserService {
  constructor(private readonly eventBus: AppEventBus) {}

  async createUser(data: CreateUserDto) {
    const user = await this.repository.create(data);

    // Публикация события
    this.eventBus.emit(AppEvent.USER_CREATED, {
      userId: user.id,
      email: user.email,
      role: user.role,
    });

    return user;
  }
}
```

## Интеграция с BullMQ (опционально)

EventBus может быть расширен для интеграции с BullMQ, но по умолчанию работает синхронно через EventEmitter2.

**Важно:** EventBus не должен автоматически отправлять события в очередь. Вместо этого listeners сами решают - выполнить логику сразу или отправить в очередь для тяжелых операций.

**Пример listener который отправляет в очередь:**

```typescript
@Injectable()
export class UserCreatedListener {
  constructor(
    private readonly eventBus: AppEventBus,
    private readonly queueDispatcher: QueueDispatcherService,
  ) {}

  @OnAppEvent(AppEvent.USER_CREATED)
  async handleUserCreated(payload: EventPayloadMap[AppEvent.USER_CREATED]) {
    // ✅ Listener сам решает отправить в очередь для тяжелой операции
    await this.queueDispatcher.dispatch(
      QueueNames.MAIL,
      JobNames.SEND_EMAIL,
      {
        to: [payload.email],
        subject: 'Welcome!',
        html: '<h1>Welcome</h1>',
        emailType: EmailType.WELCOME,
      },
    );
  }
}
```

**Пример listener который выполняет сразу:**

```typescript
@Injectable()
export class ApplicationCreatedListener {
  constructor(
    private readonly telegramService: TelegramService,
  ) {}

  @OnAppEvent(AppEvent.APPLICATION_CREATED)
  async handleApplicationCreated(
    payload: EventPayloadMap[AppEvent.APPLICATION_CREATED],
  ) {
    // ✅ Выполнение сразу - быстрая операция (логирование)
    await this.telegramService.sendMessage(
      `New application created: ${payload.applicationId} for vacancy ${payload.vacancyId}`
    );
  }
}
```

См. подробнее: [Queue Processors](../../08-queue/queue-processors.md) - раздел "Listeners и очереди"
