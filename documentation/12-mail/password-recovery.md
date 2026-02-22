# Password Recovery - Восстановление пароля

## Обзор

Отправка писем для восстановления пароля. **Важно:** Функционал восстановления пароля должен быть реализован в Auth Module.

## Интеграция с Auth Module

### Поток восстановления пароля

```
User запрашивает восстановление пароля
  ↓
POST /auth/forgot-password
  ↓
AuthService.requestPasswordReset()
  ↓
Генерация reset token
  ↓
Сохранение token в БД с TTL
  ↓
MailService.sendPasswordReset()
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

## Реализация в Auth Module

### DTO

```typescript
// auth/api/dto/forgot-password.dto.ts
import { IsEmail } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class ForgotPasswordDto {
    @ApiProperty({ description: 'Email пользователя', example: 'user@example.com' })
    @IsEmail()
    email: string;
}
```

### Controller

```typescript
// auth/api/controllers/auth.controller.ts
@Post('forgot-password')
@ApiOperation({ summary: 'Запрос на восстановление пароля' })
@ApiResponse({ status: 200, description: 'Письмо отправлено' })
async forgotPassword(@Body() dto: ForgotPasswordDto) {
    await this.authService.requestPasswordReset(dto.email);
    return { message: 'Если email существует, письмо отправлено' };
}
```

### Service

```typescript
// auth/application/services/auth.service.ts
async requestPasswordReset(email: string): Promise<void> {
    // Поиск пользователя
    const user = await this.userRepository.findByEmail(email);

    if (!user) {
        // Не раскрываем, существует ли пользователь
        return;
    }

    // Генерация токена восстановления
    const resetToken = crypto.randomBytes(32).toString('hex');
    const hashedToken = await bcrypt.hash(resetToken, 10);

    // Сохранение токена в БД с TTL (например, 1 час)
    await this.passwordResetRepository.create({
        userId: user.id,
        token: hashedToken,
        expiresAt: new Date(Date.now() + 3600000), // 1 час
    });

    // Получение профиля для имени
    const profile = await this.getUserProfile(user.id);

    // ✅ Отправка письма через очередь
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

### Подтверждение сброса пароля

```typescript
// auth/api/dto/reset-password.dto.ts
export class ResetPasswordDto {
    @ApiProperty({ description: 'Токен восстановления' })
    @IsString()
    token: string;

    @ApiProperty({ description: 'Новый пароль' })
    @IsPassword()
    newPassword: string;
}
```

```typescript
@Post('reset-password')
async resetPassword(@Body() dto: ResetPasswordDto) {
    // Поиск токена в БД
    const resetToken = await this.passwordResetRepository.findByToken(dto.token);

    if (!resetToken || resetToken.expiresAt < new Date()) {
        throw new BadRequestException('Invalid or expired token');
    }

    // Проверка токена
    const isValid = await bcrypt.compare(dto.token, resetToken.token);
    if (!isValid) {
        throw new BadRequestException('Invalid token');
    }

    // Обновление пароля
    const hashedPassword = await bcrypt.hash(dto.newPassword, 10);
    await this.userRepository.update(resetToken.userId, {
        password: hashedPassword,
    });

    // Удаление использованного токена
    await this.passwordResetRepository.delete(resetToken.id);

    return { message: 'Password reset successfully' };
}
```

## Модель БД для токенов восстановления

```prisma
model PasswordResetToken {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  token     String   // Хэшированный токен
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("password_reset_tokens")
  @@index([token])
  @@index([userId])
  @@index([expiresAt])
}
```

## Шаблон письма

См. [React Email Templates](./react-email-templates.md)

## Best Practices

1. **Не раскрывайте существование email** - одинаковый ответ для существующих и несуществующих
2. **Используйте TTL для токенов** - ограничение времени действия
3. **Хэшируйте токены** - для безопасности
4. **Удаляйте использованные токены** - после сброса пароля
5. **Ограничивайте частоту запросов** - rate limiting
