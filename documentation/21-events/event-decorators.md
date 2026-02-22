# Event Decorators - Автоматическая подписка

## Описание

Декораторы для автоматической подписки на события. Упрощают код и делают подписку более декларативной.

## Реализация декоратора

### Файл: `src/core/events/decorators/on-app-event.decorator.ts`

```typescript
import { SetMetadata } from '@nestjs/common';
import { AppEvent } from '../events.types';

export const ON_APP_EVENT_KEY = 'onAppEvent';

export const OnAppEvent = (event: AppEvent) =>
  SetMetadata(ON_APP_EVENT_KEY, event);
```

### Обработчик декоратора

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { AppEventBus } from '../event-bus.service';
import { AppEvent, EventPayloadMap } from '../events.types';
import { Reflector } from '@nestjs/core';
import { ON_APP_EVENT_KEY } from './decorators/on-app-event.decorator';

@Injectable()
export class EventSubscriberService implements OnModuleInit {
  constructor(
    private readonly eventBus: AppEventBus,
    private readonly reflector: Reflector,
  ) {}

  onModuleInit() {
    // Автоматическая регистрация всех обработчиков с декоратором
    // Реализация через метаданные NestJS
  }
}
```

## Использование

### Простой вариант (без декоратора)

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';

@Injectable()
export class UserCreatedListener implements OnModuleInit {
  constructor(
    private readonly eventBus: AppEventBus,
    private readonly telegramService: TelegramService,
  ) {}

  onModuleInit() {
    this.eventBus.on(AppEvent.USER_CREATED, (payload) => {
      this.telegramService.sendMessage(`New user created: ${payload.userId}`);
    });
  }
}
```

### С декоратором (будущая реализация)

```typescript
import { Injectable } from '@nestjs/common';
import { OnAppEvent } from '@core/events/decorators/on-app-event.decorator';
import { AppEvent, EventPayloadMap } from '@core/events/events.types';

@Injectable()
export class ApplicationListener {
  constructor(
    private readonly mailQueue: Queue,
  ) {}

  @OnAppEvent(AppEvent.APPLICATION_CREATED)
  async handleApplicationCreated(
    payload: EventPayloadMap[AppEvent.APPLICATION_CREATED],
  ) {
    // ✅ Отправка в очередь для тяжелой операции (отправка email)
    await this.mailQueue.add(MAIL_QUEUE_JOB_NAMES.SEND_EMAIL, {
      to: [payload.employerEmail],
      subject: 'New Application',
      html: await this.renderApplicationEmail(payload),
      emailType: EmailType.APPLICATION_NEW,
    });
  }
}
```

## Преимущества декораторов

1. **Декларативность**: Обработчики явно помечены декоратором
2. **Автоматизация**: Не нужно вручную регистрировать в `onModuleInit`
3. **Типизация**: Payload автоматически типизирован
4. **Читаемость**: Легко понять, на какие события подписан класс

## Примечание

Декораторы - это опциональная функциональность для упрощения кода. Базовый подход через `onModuleInit` также полностью поддерживается и является более явным.
