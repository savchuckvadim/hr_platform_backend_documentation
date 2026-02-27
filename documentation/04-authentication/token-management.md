# Token Management - Управление токенами

## Обзор

Система управления токенами включает автоматическую очистку истёкших токенов, продление сессий и мониторинг активных сессий. Токены хранятся в HttpOnly cookies для безопасности.

## CookieService

### Назначение

Сервис для работы с HttpOnly cookies, содержащими access и refresh токены.

**Расположение:** `core/cookie/cookie.service.ts`

**Методы:**

```typescript
@Injectable()
export class CookieService {
  // Установка access token в cookie (15 минут)
  setAccessToken(res: Response, token: string): void;

  // Установка refresh token в cookie (30 дней)
  setRefreshToken(res: Response, token: string): void;

  // Очистка обоих токенов из cookies
  clearAuthCookies(res: Response): void;

  // Извлечение refresh token из cookie
  getRefreshToken(req: Request): string | undefined;

  // Извлечение access token из cookie
  getAccessToken(req: Request): string | undefined;
}
```

**Настройки cookies:**
- `httpOnly: true` - защита от XSS
- `secure: true` - только HTTPS (production)
- `sameSite: 'none'` - для cross-origin (production)
- `sameSite: 'lax'` - для localhost (development)
- `domain` - настраивается через `CLIENT_DOMAIN` (production)
- `path: '/'` - доступно на всех путях

**Время жизни:**
- Access token: 15 минут
- Refresh token: 30 дней

## Cron очистка истёкших токенов

### Назначение

Автоматическое удаление истёкших refresh токенов из БД для освобождения места и поддержания чистоты данных.

### Реализация

**Файл:** `infrastructure/tasks/token-cleanup.task.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { TokenRepository } from '../repositories/token.repository';
import { Logger } from '@nestjs/common';

@Injectable()
export class TokenCleanupTask {
  private readonly logger = new Logger(TokenCleanupTask.name);

  constructor(private readonly tokenRepository: TokenRepository) {}

  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async cleanupExpiredTokens() {
    this.logger.log('Starting token cleanup task...');

    try {
      const deletedCount = await this.tokenRepository.deleteExpired();

      this.logger.log(
        `Token cleanup completed. Deleted ${deletedCount} expired tokens.`,
      );
    } catch (error) {
      this.logger.error('Error during token cleanup:', error);
    }
  }
}
```

### Расписание

**Выполнение:** Ежедневно в 3:00 AM

**Настройка:**

```typescript
// Использование CronExpression
@Cron(CronExpression.EVERY_DAY_AT_3AM)

// Или кастомное расписание
@Cron('0 3 * * *') // Каждый день в 3:00
```

### Логика очистки

**Метод в TokenRepository:**

```typescript
async deleteExpired(): Promise<number> {
  const result = await this.prisma.token.deleteMany({
    where: {
      expiresAt: {
        lt: new Date(), // expiresAt < NOW()
      },
    },
  });

  return result.count;
}
```

**Принципы:**
- Hard delete (физическое удаление)
- Удаляются только истёкшие токены
- Активные токены не затрагиваются
- Возвращается количество удалённых записей

### Регистрация задачи

**В AuthModule:**

```typescript
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [ScheduleModule.forRoot()],
  providers: [TokenCleanupTask],
})
export class AuthModule {}
```

### Мониторинг

**Логирование:**
- Начало задачи
- Количество удалённых токенов
- Ошибки при выполнении

**Метрики (опционально):**
```typescript
this.metrics.increment('tokens.cleaned', deletedCount);
this.metrics.gauge('tokens.active', activeCount);
```

## Sliding Session

### Принцип

Refresh token автоматически продлевается при активности пользователя, если осталось менее 50% времени до истечения.

### Реализация в Refresh Flow

```typescript
async refreshToken(refreshToken: string, res?: Response): Promise<AuthenticatedUserDto> {
  // Валидация refresh token из cookie
  const userData = await this.tokenService.validateRefreshToken(refreshToken);

  // Проверка наличия в БД
  const tokenFromDb = await this.tokenService.findTokenByRefreshToken(refreshToken);
  if (!tokenFromDb || !userData?.userId) {
    if (res) {
      this.cookieService.clearAuthCookies(res);
    }
    throw new UnauthorizedException('Invalid refresh token');
  }

  // Генерация новых токенов
  const user = await this.userService.getUser(userData.userId);
  const { accessToken, refreshToken: newRefreshToken } = await this.generateTokens(user);

  // Обновление refresh token в БД
  await this.tokenService.removeToken(refreshToken);
  await this.tokenService.saveToken(user.id, newRefreshToken);

  return {
    tokens: {
      accessToken,
      refreshToken: newRefreshToken,
    },
    user,
  };
  // AuthCookieInterceptor автоматически установит токены в cookies
}
```

**Примечание:** Sliding session (продление refresh token) может быть реализовано при необходимости, но в текущей реализации refresh token обновляется при каждом refresh запросе.

### Параметры

**Порог продления:** 50% времени до истечения

**Время продления:** +7 дней (настраивается)

**Пример:**

```
Refresh token создан: 2024-01-01
Expires at: 2024-01-08 (7 дней)
Текущая дата: 2024-01-05 (осталось 3 дня = 43%)

43% < 50% → продлеваем до 2024-01-12
```

## Управление сессиями

### Получение активных сессий

**Endpoint:**

```
GET /auth/sessions
```

**Реализация:**

```typescript
@Get('sessions')
@UseGuards(AccessTokenGuard)
async getSessions(@CurrentUser() user: User) {
  const tokens = await this.tokenRepository.findByUserId(user.id);

  return tokens.map(token => ({
    id: token.id,
    deviceId: token.deviceId,
    deviceName: token.deviceName,
    userRoleName: token.roleContext.userRole, // Enum значение (UserRole)
    ipAddress: token.ipAddress,
    createdAt: token.createdAt,
    expiresAt: token.expiresAt,
    isCurrent: token.deviceId === this.extractDeviceId(request),
  }));
}
```

### Удаление конкретной сессии

**Endpoint:**

```
DELETE /auth/sessions/:sessionId
```

**Реализация:**

```typescript
@Delete('sessions/:sessionId')
@UseGuards(AccessTokenGuard)
async deleteSession(
  @CurrentUser() user: User,
  @Param('sessionId') sessionId: string,
) {
  const token = await this.tokenRepository.findById(sessionId);

  if (!token || token.userId !== user.id) {
    throw new NotFoundException('Session not found');
  }

  await this.tokenRepository.delete(sessionId);

  return { message: 'Session deleted successfully' };
}
```

## Безопасность

### Хранение токенов

**Текущая реализация:**

1. **Access Token - HttpOnly Cookie (обязательно)**
   - Устанавливается через `CookieService.setAccessToken()`
   - Время жизни: 15 минут
   - Настройки: `httpOnly: true`, `secure: true` (production), `sameSite: 'none'` (production)

2. **Refresh Token - HttpOnly Cookie (обязательно)**
   - Устанавливается через `CookieService.setRefreshToken()`
   - Время жизни: 30 дней
   - Настройки: `httpOnly: true`, `secure: true` (production), `sameSite: 'none'` (production)
   - Извлекается через `CookieService.getRefreshToken()`

3. **Fallback для Access Token:**
   - Может быть передан в заголовке `Authorization: Bearer <token>`
   - `AccessTokenGuard` проверяет сначала заголовок, затем cookie

**Никогда не хранить в:**
- localStorage
- sessionStorage
- URL параметрах
- Публичных переменных
- Response body (удаляется через `AuthCookieInterceptor`)

### Хэширование refresh token

**Принципы:**

1. **Хранение в БД:**
   ```typescript
   const hashedToken = await bcrypt.hash(refreshToken, 10);
   await tokenRepository.create({
     refreshToken: hashedToken,
     // ...
   });
   ```

2. **Сравнение:**
   ```typescript
   const isValid = await bcrypt.compare(
     providedToken,
     storedHashedToken,
   );
   ```

3. **Безопасность:**
   - Используется bcrypt с salt rounds = 10
   - Даже при утечке БД токены не могут быть использованы
   - Сравнение через hash comparison

### Отслеживание устройств

**Метрики:**

```typescript
// При создании токена
this.metrics.increment('tokens.created', 1, {
  device: tokenRecord.deviceName || 'unknown',
    userRoleName: roleContext.userRole, // Enum значение
});

// При удалении токена
this.metrics.increment('tokens.deleted', 1, {
  reason: 'logout' | 'expired' | 'cleanup',
});
```

**Логирование подозрительной активности:**

```typescript
// Проверка IP адреса
if (tokenRecord.ipAddress !== request.ip) {
  this.logger.warn(
    `IP address changed for token ${tokenRecord.id}: ${tokenRecord.ipAddress} -> ${request.ip}`,
  );
}
```

## Мониторинг

### Метрики

**Ключевые метрики:**

1. **Количество активных токенов**
   ```typescript
   const activeCount = await tokenRepository.countActive();
   this.metrics.gauge('tokens.active', activeCount);
   ```

2. **Количество истёкших токенов**
   ```typescript
   const expiredCount = await tokenRepository.countExpired();
   this.metrics.gauge('tokens.expired', expiredCount);
   ```

3. **Количество токенов по ролям**
   ```typescript
   const byRole = await tokenRepository.countByRole();
   this.metrics.gauge('tokens.by_role.candidate', byRole.CANDIDATE);
   // Убрано: роль EMPLOYER избыточна
   ```

4. **Среднее время жизни токена**
   ```typescript
   const avgLifetime = await tokenRepository.avgLifetime();
   this.metrics.gauge('tokens.avg_lifetime_hours', avgLifetime);
   ```

### Алерты

**Критические алерты:**

1. **Большое количество истёкших токенов**
   - Порог: > 10,000
   - Действие: Проверить cron задачу

2. **Ошибки при очистке**
   - Действие: Проверить БД и логи

3. **Подозрительная активность**
   - Множественные токены с одного IP
   - Быстрая смена устройств
   - Действие: Блокировка или уведомление

## Оптимизация

### Индексы БД

**Критичные индексы:**

```sql
-- Для поиска токена
CREATE INDEX idx_tokens_refresh_token ON tokens(refresh_token);

-- Для поиска по пользователю
CREATE INDEX idx_tokens_user_id ON tokens(user_id);

-- Для cron очистки
CREATE INDEX idx_tokens_expires_at ON tokens(expires_at);

-- Для unique constraint
CREATE UNIQUE INDEX idx_tokens_user_role_device
ON tokens(user_id, role_context_id, device_id);
```

### Партиционирование (опционально)

Для больших объёмов данных можно использовать партиционирование по дате:

```sql
-- Партиционирование по expires_at
CREATE TABLE tokens_2024_01 PARTITION OF tokens
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

## Тестирование

### Unit тесты

```typescript
describe('TokenCleanupTask', () => {
  it('should delete expired tokens', async () => {
    // Создаём истёкший токен
    await tokenRepository.create({
      expiresAt: new Date('2024-01-01'), // Прошлая дата
      // ...
    });

    await task.cleanupExpiredTokens();

    const count = await tokenRepository.count();
    expect(count).toBe(0);
  });

  it('should not delete active tokens', async () => {
    // Создаём активный токен
    await tokenRepository.create({
      expiresAt: new Date('2025-01-01'), // Будущая дата
      // ...
    });

    await task.cleanupExpiredTokens();

    const count = await tokenRepository.count();
    expect(count).toBe(1);
  });
});
```

### Integration тесты

```typescript
describe('Token Management E2E', () => {
  it('should cleanup expired tokens via cron', async () => {
    // Setup: создаём истёкшие токены
    // ...

    // Запускаем cron задачу
    await tokenCleanupTask.cleanupExpiredTokens();

    // Проверяем результат
    const expired = await tokenRepository.findExpired();
    expect(expired).toHaveLength(0);
  });
});
```

## Заключение

Система управления токенами обеспечивает:

✅ Автоматическую очистку истёкших токенов через cron
✅ Sliding session для продления активных сессий
✅ Управление активными сессиями пользователя
✅ Безопасное хранение и сравнение токенов
✅ Мониторинг и метрики
✅ Оптимизацию производительности

Все компоненты работают вместе для обеспечения надёжной и безопасной системы управления токенами.
