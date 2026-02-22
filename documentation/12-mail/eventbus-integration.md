# EventBus Integration - Интеграция с EventBus

## Обзор

Публикация событий через EventBus после успешной отправки email. События публикуются из `@OnWorkerEvent('completed')` в MailProcessor.

## Принцип

```
MailProcessor.process()
  ↓
MailService.sendEmail()
  ↓
@OnWorkerEvent('completed')
  ↓
EventBus.emit(EMAIL_SENT)
  ↓
Listeners обрабатывают событие
```

## Реализация

### Публикация события

```typescript
// infrastructure/processors/mail.processor.ts
@OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
async onCompleted(job: Job<SendEmailJobData>) {
    const { to, subject, emailType } = job.data;

    // ✅ Логирование, метрики
    this.logger.log(`Email job completed: ${job.id}`);

    // ✅ Эмитим бизнес-событие после успешного завершения задачи
    this.eventBus.emit(AppEvent.EMAIL_SENT, {
        emailType: emailType || EmailType.OTHER,
        to: to,
        subject: subject,
    });
}
```

### Глобальное событие

```typescript
// core/events/events.types.ts
export enum AppEvent {
    EMAIL_SENT = 'email.sent',
    // ...
}

export interface EventPayloadMap {
    [AppEvent.EMAIL_SENT]: {
        emailType: EmailType;
        to: string[];
        subject: string;
    };
    // ...
}
```

## Listeners

### Пример listener для EMAIL_SENT

```typescript
// В другом модуле
import { Injectable } from '@nestjs/common';
import { OnAppEvent } from '@core/events/event-decorators';
import { AppEvent } from '@core/events/events.types';
import { TelegramService } from '@telegram/application/services/telegram.service';

@Injectable()
export class EmailNotificationListener {
    constructor(
        private readonly telegramService: TelegramService,
    ) {}

    @OnAppEvent(AppEvent.EMAIL_SENT)
    async handleEmailSent(payload: {
        emailType: EmailType;
        to: string[];
        subject: string;
    }) {
        // ✅ Обработка события (например, отправка в Telegram)
        if (payload.emailType === EmailType.VERIFICATION) {
            await this.telegramService.sendEmailVerificationNotification({
                email: payload.to[0],
                subject: payload.subject,
            });
        }
    }
}
```

## Best Practices

1. **Публикуйте из @OnWorkerEvent** - после успешного завершения задачи
2. **Типизируйте события** - через EventPayloadMap
3. **Разделяйте ответственность** - Worker публикует, Listeners обрабатывают
4. **Не смешивайте логику** - Worker только публикует, логика в Listeners
