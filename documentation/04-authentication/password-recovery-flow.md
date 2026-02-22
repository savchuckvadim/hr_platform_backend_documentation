# Password Recovery Flow - Восстановление пароля

## Обзор

Поток восстановления пароля через email. Пользователь запрашивает восстановление, получает письмо с токеном, и использует токен для установки нового пароля.

## Поток восстановления пароля

### Шаг 1: Запрос на восстановление

**Endpoint:**
```
POST /auth/forgot-password
```

**DTO:**
```typescript
// api/dto/forgot-password.dto.ts
export class ForgotPasswordDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;
}
```

**Controller:**
```typescript
@Post('forgot-password')
@Public()
@ApiOperation({ summary: 'Запрос на восстановление пароля' })
@ApiResponse({
  status: 200,
  description: 'Если email существует, письмо отправлено'
})
async forgotPassword(@Body() dto: ForgotPasswordDto) {
  await this.authService.requestPasswordReset(dto.email);
  // ✅ Всегда возвращаем одинаковый ответ для безопасности
  return {
    message: 'Если email существует, письмо для восстановления пароля отправлено'
  };
}
```

**Service:**
```typescript
async requestPasswordReset(email: string): Promise<void> {
  // Поиск пользователя
  const user = await this.userRepository.findByEmail(email);

  // ✅ Не раскрываем, существует ли пользователь
  if (!user) {
    return; // Всегда возвращаем успех
  }

  // Генерация токена восстановления
  const resetToken = crypto.randomBytes(32).toString('hex');
  const hashedToken = await bcrypt.hash(resetToken, 10);

  // Сохранение токена в БД с TTL (1 час)
  await this.passwordResetRepository.create({
    userId: user.id,
    token: hashedToken,
    expiresAt: new Date(Date.now() + 3600000), // 1 час
  });

  // Получение профиля для имени
  const profile = await this.getUserProfile(user.id);

  // ✅ Отправка письма через очередь (Mail Module)
  await this.mailService.sendPasswordReset(
    {
      id: user.id,
      email: user.email,
      firstName: profile?.firstName,
      lastName: profile?.lastName,
    },
    resetToken, // Отправляем незахешированный токен
    'ru', // или из настроек пользователя
  );
}
```

### Шаг 2: Получение письма

Пользователь получает письмо с ссылкой:
```
https://hr-platform.com/auth/recovery/{token}
```

### Шаг 3: Подтверждение сброса пароля

**Endpoint:**
```
POST /auth/reset-password
```

**DTO:**
```typescript
// api/dto/reset-password.dto.ts
export class ResetPasswordDto {
  @IsString()
  @IsNotEmpty()
  token: string;

  @IsPassword()
  @IsNotEmpty()
  newPassword: string;
}
```

**Controller:**
```typescript
@Post('reset-password')
@Public()
@ApiOperation({ summary: 'Сброс пароля по токену' })
@ApiResponse({ status: 200, description: 'Пароль успешно изменен' })
@ApiResponse({ status: 400, description: 'Неверный или истекший токен' })
async resetPassword(@Body() dto: ResetPasswordDto) {
  return this.authService.resetPassword(dto.token, dto.newPassword);
}
```

**Service:**
```typescript
async resetPassword(token: string, newPassword: string): Promise<void> {
  // Поиск токена в БД
  const resetToken = await this.passwordResetRepository.findByToken(token);

  if (!resetToken) {
    throw new BadRequestException('Invalid or expired token');
  }

  // Проверка срока действия
  if (resetToken.expiresAt < new Date()) {
    await this.passwordResetRepository.delete(resetToken.id);
    throw new BadRequestException('Token expired');
  }

  // Проверка токена (сравнение хэша)
  const isValid = await bcrypt.compare(token, resetToken.token);
  if (!isValid) {
    throw new BadRequestException('Invalid token');
  }

  // Обновление пароля
  const hashedPassword = await bcrypt.hash(newPassword, 10);
  await this.userRepository.update(resetToken.userId, {
    password: hashedPassword,
  });

  // ✅ Инвалидация всех refresh токенов пользователя (опционально)
  await this.tokenRepository.deleteAllByUserId(resetToken.userId);

  // Удаление использованного токена
  await this.passwordResetRepository.delete(resetToken.id);

  // ✅ Публикация события
  this.eventBus.emit(AppEvent.PASSWORD_RESET, {
    userId: resetToken.userId,
    timestamp: new Date(),
  });
}
```

## Модель БД

### PasswordResetToken

```prisma
model PasswordResetToken {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  token     String   @db.VarChar(500) // Хэшированный токен
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("password_reset_tokens")
  @@index([token])
  @@index([userId])
  @@index([expiresAt])
}
```

### Обновление User модели

```prisma
model User {
  // ... существующие поля

  passwordResetTokens PasswordResetToken[]
}
```

## Repository

### PasswordResetTokenRepository

```typescript
// infrastructure/repositories/password-reset-token.repository.ts
export abstract class PasswordResetTokenRepository {
  abstract create(data: {
    userId: string;
    token: string;
    expiresAt: Date;
  }): Promise<PasswordResetToken>;

  abstract findByToken(token: string): Promise<PasswordResetToken | null>;

  abstract delete(id: string): Promise<void>;

  abstract deleteExpired(): Promise<number>; // Для cron очистки
}
```

## Cron очистка истекших токенов

```typescript
// infrastructure/cron/password-reset-cleanup.cron.ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { PasswordResetTokenRepository } from '../repositories/password-reset-token.repository';

@Injectable()
export class PasswordResetCleanupCron {
  constructor(
    private readonly passwordResetRepository: PasswordResetTokenRepository,
  ) {}

  /**
   * Удаление истекших токенов восстановления пароля
   * Запускается каждый час
   */
  @Cron(CronExpression.EVERY_HOUR)
  async cleanupExpiredTokens() {
    const deleted = await this.passwordResetRepository.deleteExpired();
    this.logger.log(`Cleaned up ${deleted} expired password reset tokens`);
  }
}
```

## EventBus интеграция

### Событие PASSWORD_RESET

```typescript
// core/events/events.types.ts
export enum AppEvent {
  PASSWORD_RESET = 'password.reset',
  // ...
}

export interface EventPayloadMap {
  [AppEvent.PASSWORD_RESET]: {
    userId: string;
    timestamp: Date;
  };
  // ...
}
```

### Listener (опционально)

```typescript
// В другом модуле
@OnAppEvent(AppEvent.PASSWORD_RESET)
async handlePasswordReset(payload: { userId: string; timestamp: Date }) {
  // Например, отправка уведомления в Telegram
  // или логирование события
}
```

## Безопасность

### Принципы

1. **Не раскрывать существование email** - одинаковый ответ для всех запросов
2. **Хэшировать токены** - хранить в БД только хэш
3. **TTL для токенов** - ограничение времени действия (1 час)
4. **Одноразовые токены** - удаление после использования
5. **Rate limiting** - ограничение частоты запросов
6. **Инвалидация сессий** - опционально, после сброса пароля

### Rate Limiting

```typescript
@Post('forgot-password')
@Throttle(1, 60) // 1 запрос в минуту
async forgotPassword(@Body() dto: ForgotPasswordDto) {
  // ...
}
```

## Интеграция с Mail Module

См. [Password Recovery в Mail Module](../12-mail/password-recovery.md)

## Best Practices

1. **Всегда одинаковый ответ** - не раскрывать существование email
2. **Хэшировать токены** - для безопасности
3. **Использовать TTL** - ограничение времени действия
4. **Удалять использованные токены** - после сброса пароля
5. **Rate limiting** - ограничение частоты запросов
6. **Очистка истекших токенов** - через cron
