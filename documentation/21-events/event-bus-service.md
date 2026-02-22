# Typed EventBus Service

## Описание

Типизированный EventBus сервис с автокомплитом и проверкой типов. Абстракция над EventEmitter2, которая может работать локально или через BullMQ/Redis/Kafka.

## Структура

### Файл: `src/core/events/event-bus.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { AppEvent, EventPayloadMap } from './events.types';

/**
 * Типизированный EventBus
 * Гарантирует типобезопасность при публикации и подписке на события
 */
@Injectable()
export class AppEventBus {
  constructor(
    private readonly emitter: EventEmitter2,
    @InjectQueue('events') private readonly eventsQueue?: Queue,
  ) {}

  /**
   * Публикация события
   * Тип события и payload строго проверяются
   */
  emit<E extends AppEvent>(
    event: E,
    payload: EventPayloadMap[E],
  ): void {
    // Локальная публикация (синхронная)
    this.emitter.emit(event, payload);

    // Опционально: отправка в очередь для асинхронной обработки
    if (this.shouldQueue(event)) {
      this.eventsQueue?.add(event, payload, {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000,
        },
      });
    }
  }

  /**
   * Подписка на событие
   * Handler получает строго типизированный payload
   */
  on<E extends AppEvent>(
    event: E,
    handler: (payload: EventPayloadMap[E]) => void | Promise<void>,
  ): void {
    this.emitter.on(event, handler);
  }

  /**
   * Одноразовая подписка
   */
  once<E extends AppEvent>(
    event: E,
    handler: (payload: EventPayloadMap[E]) => void | Promise<void>,
  ): void {
    this.emitter.once(event, handler);
  }

  /**
   * Отписка от события
   */
  off<E extends AppEvent>(
    event: E,
    handler: (payload: EventPayloadMap[E]) => void | Promise<void>,
  ): void {
    this.emitter.off(event, handler);
  }

  /**
   * Определяет, нужно ли отправлять событие в очередь
   * Можно настроить для определенных типов событий
   */
  private shouldQueue(event: AppEvent): boolean {
    // Пример: только определенные события идут в очередь
    const queuedEvents: AppEvent[] = [
      AppEvent.APPLICATION_CREATED,
      AppEvent.MESSAGE_CREATED,
    ];
    return queuedEvents.includes(event);
  }
}
```

## Использование

### Публикация события

```typescript
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';

@Injectable()
export class UserService {
  constructor(private readonly eventBus: AppEventBus) {}

  async createUser(data: CreateUserDto) {
    const user = await this.repository.create(data);

    // Тип события автокомплитится
    // Payload строго проверяется
    this.eventBus.emit(AppEvent.USER_CREATED, {
      userId: user.id,
      email: user.email,
      role: user.role,
    });

    return user;
  }
}
```

### Подписка на событие

Listeners могут выполнять логику сразу или отправлять в очередь для тяжелых операций.

**Пример 1: Выполнение сразу (быстрая операция)**

```typescript
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class ApplicationListener implements OnModuleInit {
  constructor(
    private readonly eventBus: AppEventBus,
    private readonly telegramService: TelegramService,
  ) {}

  onModuleInit() {
    // ✅ Выполнение сразу - быстрая операция (например, логирование)
    this.eventBus.on(AppEvent.APPLICATION_CREATED, (payload) => {
      this.telegramService.sendMessage(
        `New application created: ${payload.applicationId} for vacancy ${payload.vacancyId}`
      );
    });
  }
}
```

**Пример 2: Отправка в очередь (тяжелая операция)**

```typescript
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import { Injectable, OnModuleInit } from '@nestjs/common';
import { QueueDispatcherService } from '@core/queue/dispatch/queue-dispatcher.service';

@Injectable()
export class EmailListener implements OnModuleInit {
  constructor(
    private readonly eventBus: AppEventBus,
    private readonly queueDispatcher: QueueDispatcherService,
  ) {}

  onModuleInit() {
    // ✅ Отправка в очередь для тяжелой операции (отправка email)
    this.eventBus.on(AppEvent.USER_CREATED, (payload) => {
      this.queueDispatcher.dispatch(
        QueueNames.MAIL,
        JobNames.SEND_EMAIL,
        {
          to: [payload.email],
          subject: 'Welcome to HR Platform',
          html: '<h1>Welcome!</h1>',
          emailType: EmailType.WELCOME,
        },
      );
    });
  }
}
```

## Где публикуются глобальные события

Глобальные события могут публиковаться из разных мест в зависимости от типа операции:

### 1. Из сервиса (для легких операций)

```typescript
// application/services/user.service.ts
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly eventBus: AppEventBus,
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.create(dto);

    // ✅ Публикуем глобальное событие после успешного создания
    this.eventBus.emit(AppEvent.USER_CREATED, {
      userId: user.id,
      email: user.email,
      role: user.role,
    });

    return user;
  }
}
```

### 2. Из Use Case (рекомендуется для координации)

```typescript
// application/use-cases/create-application.use-case.ts
@Injectable()
export class CreateApplicationUseCase {
  constructor(
    private readonly applicationService: ApplicationService,
    private readonly eventBus: AppEventBus,
  ) {}

  async execute(dto: CreateApplicationDto, candidateId: string) {
    const application = await this.applicationService.create(dto, candidateId);

    // ✅ Публикуем глобальное событие
    this.eventBus.emit(AppEvent.APPLICATION_CREATED, {
      applicationId: application.id,
      candidateId,
      vacancyId: dto.vacancyId,
    });

    return application;
  }
}
```

### 3. Из Worker (после обработки очереди)

```typescript
// infrastructure/processors/mail.processor.ts
@OnWorkerEvent('completed')
async onCompleted(job: Job<SendEmailJobData>) {
  // Логирование, метрики
  this.logger.log(`Email job completed: ${job.id}`);

  // ✅ Публикуем глобальное событие после успешной отправки
  this.eventBus.emit(AppEvent.EMAIL_SENT, {
    emailType: job.data.emailType,
    to: job.data.to,
    subject: job.data.subject,
  });
}
```

**Важно:**
- Для легких операций (создание пользователя, обновление статуса) - публикуем сразу из сервиса/use case
- Для тяжелых операций (отправка email, обработка файлов) - публикуем из worker после успешного завершения

## Преимущества

1. **Типобезопасность**: Невозможно ошибиться с типом события или payload
2. **Автокомплит**: IDE подсказывает доступные события и поля payload
3. **Абстракция**: Бизнес-логика не знает про BullMQ/Redis
4. **Гибкость**: Можно легко переключиться на другую реализацию
5. **Централизация**: Все события в одном месте
