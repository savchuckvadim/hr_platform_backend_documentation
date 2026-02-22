# Email Confirmation - Подтверждение email

## Обзор

Отправка писем для подтверждения email адреса при регистрации пользователя.

## Интеграция с Auth Module

### Поток регистрации

```
User регистрируется
  ↓
AuthService.createUser()
  ↓
EventBus.emit(USER_CREATED)
  ↓
EmailConfirmationListener
  ↓
MailService.sendEmailVerification()
  ↓
queue.add(SEND_EMAIL)
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

### Listener для USER_CREATED

```typescript
// В Auth Module или отдельном Email Module
import { Injectable } from '@nestjs/common';
import { OnAppEvent } from '@core/events/event-decorators';
import { AppEvent } from '@core/events/events.types';
import { MailService } from '@mail/application/services/mail.service';
import { UserRepository } from '@user/infrastructure/repositories/user.repository';

@Injectable()
export class EmailConfirmationListener {
    constructor(
        private readonly mailService: MailService,
        private readonly userRepository: UserRepository,
    ) {}

    @OnAppEvent(AppEvent.USER_CREATED)
    async handleUserCreated(payload: {
        userId: string;
        email: string;
        roleType: RoleType;
    }) {
        // Получение пользователя для имени
        const user = await this.userRepository.findById(payload.userId);

        // Генерация токена подтверждения
        const token = await this.generateVerificationToken(payload.userId);

        // ✅ Отправка письма через очередь
        await this.mailService.sendEmailVerification(
            {
                id: payload.userId,
                email: payload.email,
                firstName: user.firstName,
                lastName: user.lastName,
            },
            token,
            'ru', // или из настроек пользователя
        );
    }

    private async generateVerificationToken(userId: string): Promise<string> {
        // Генерация токена (например, через crypto)
        // Сохранение в БД с TTL
        // ...
        return token;
    }
}
```

## Глобальное событие

```typescript
// core/events/events.types.ts
export enum AppEvent {
    USER_CREATED = 'user.created',
    // ...
}

export interface EventPayloadMap {
    [AppEvent.USER_CREATED]: {
        userId: string;
        email: string;
        roleType: RoleType;
    };
    // ...
}
```

## Шаблон письма

См. [React Email Templates](./react-email-templates.md)

## Best Practices

1. **Отправляйте сразу после регистрации** - через событие USER_CREATED
2. **Генерируйте токены с TTL** - для безопасности
3. **Используйте язык пользователя** - из настроек или по умолчанию
4. **Обрабатывайте ошибки** - логируйте неудачные отправки
