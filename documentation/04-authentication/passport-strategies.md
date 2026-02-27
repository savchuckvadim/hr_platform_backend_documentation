# Token Validation - Валидация токенов

## Обзор

Модуль использует `AccessTokenGuard` и `TokenService` для валидации токенов. Access token проверяется из заголовка Authorization или из cookie.

## AccessTokenGuard

### Назначение

Проверяет и валидирует access token на защищённых маршрутах. Извлекает токен из заголовка Authorization или из cookie.

### Расположение

`core/guards/access-token.guard.ts`

### Реализация

```typescript
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { Request } from 'express';
import { TokenPayloadDto, TokenService } from '@/modules/token';

@Injectable()
export class AccessTokenGuard implements CanActivate {
  constructor(
    private readonly tokenService: TokenService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest<Request & { user?: TokenPayloadDto }>();

    let accessToken: string | undefined;

    // 1) Берём из заголовка Authorization (приоритет)
    const authHeader = req.headers.authorization;
    if (authHeader?.startsWith('Bearer ')) {
      accessToken = authHeader.split(' ')[1];
    }

    // 2) Fallback: берём из Cookies
    if (!accessToken && req.cookies?.accessToken) {
      accessToken = req.cookies.accessToken;
    }

    if (!accessToken) {
      throw new UnauthorizedException('ACCESS_TOKEN_MISSING');
    }

    // Валидация токена через TokenService
    const user = await this.tokenService.validateAccessToken(accessToken);
    req.user = user;

    return true;
  }
}
```

**TokenPayloadDto интерфейс:**
```typescript
export interface TokenPayloadDto {
  userId: string;
  roleContextId?: string;
  userRoleName?: string;
  companyId?: string | null;
  hrRoleName?: string | null;
}
```

### Принципы работы

1. **Извлечение токена**
   - Приоритет: из заголовка `Authorization: Bearer <token>`
   - Fallback: из cookie `accessToken` через `req.cookies.accessToken`

2. **Валидация токена**
   - Проверка через `TokenService.validateAccessToken()`
   - Проверка подписи JWT через `JWT_SECRET`
   - Проверка срока действия (exp)
   - Проверка структуры payload

3. **Установка данных**
   - Устанавливает `request.user` с данными из токена (`TokenPayloadDto`)
   - Доступен через `@CurrentUser()` decorator

4. **Обработка ошибок**
   - Бросает `UnauthorizedException` при отсутствии или невалидности токена

### Использование

```typescript
@Controller('protected')
@UseGuards(AccessTokenGuard) // Проверяет токен из cookie или header
export class ProtectedController {
  @Get('profile')
  getProfile(@CurrentUser() user: TokenPayloadDto) {
    return user;
  }
}
```

## Refresh Token Validation

### Назначение

Валидирует refresh token при обновлении access token. Refresh token извлекается из HttpOnly cookie через `CookieService`.

### Расположение

Валидация происходит в `AuthService.refreshToken()` с использованием `TokenService.validateRefreshToken()`.

### Реализация в AuthService

```typescript
// В AuthService.refreshToken()
async refreshToken(refreshToken: string, res?: Response): Promise<AuthenticatedUserDto> {
  if (!refreshToken) {
    if (res) {
      this.cookieService.clearAuthCookies(res);
    }
    throw new UnauthorizedException('Refresh token not found');
  }

  // Валидация через TokenService
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

### Принципы работы Refresh Token

1. **Извлечение токена**
   - **Обязательно** из HttpOnly cookie через `CookieService.getRefreshToken()`
   - Передача в body не рекомендуется (fallback для обратной совместимости)

2. **Валидация в AuthService**
   - Валидация через `TokenService.validateRefreshToken()`
   - Проверка наличия в БД через `TokenService.findTokenByRefreshToken()`
   - Проверка соответствия userId
   - При ошибке - автоматическая очистка cookies

3. **Генерация новых токенов**
   - Генерация новых access и refresh токенов
   - Обновление refresh token в БД (удаление старого, сохранение нового)

4. **Установка в cookies**
   - Новые токены устанавливаются через `AuthCookieInterceptor`
   - Токены удаляются из response body

### Использование

```typescript
@Controller('auth')
export class AuthController {
  @Post('refresh')
  @Public()
  @SetAuthCookie() // Устанавливает новые токены в cookies
  async refresh(@Req() request: Request, @Res() response: Response) {
    // Извлекаем refresh token из cookie
    const refreshToken = this.cookieService.getRefreshToken(request);

    if (!refreshToken) {
      this.cookieService.clearAuthCookies(response);
      throw new UnauthorizedException('Refresh token not found');
    }

    return this.authService.refreshToken(refreshToken, response);
  }
}
```

## Регистрация модулей

### В AuthModule

```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AccessTokenGuard } from '@/core/guards/access-token.guard';
import { TokenService } from '@/modules/token';
import { CookieService } from '@/core/cookie';
import { AuthCookieInterceptor } from '@/core/interceptors/auth-cookie.interceptor';

@Module({
  imports: [
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        secret: configService.getOrThrow<string>('JWT_SECRET'),
        signOptions: {
          expiresIn: configService.getOrThrow<string>('JWT_EXPIRES_IN', '15m'),
        },
      }),
      inject: [ConfigService],
    }),
  ],
  providers: [
    AccessTokenGuard,
    TokenService,
    CookieService,
    AuthCookieInterceptor,
  ],
  exports: [JwtModule, TokenService, CookieService],
})
export class AuthModule {}
```

## Конфигурация

### Environment Variables

```env
JWT_SECRET=your-secret-key-here
JWT_EXPIRES_IN=15m
REFRESH_TOKEN_EXPIRES_IN=7d
```

### Рекомендации

1. **JWT_SECRET**
   - Минимум 32 символа
   - Случайная строка
   - Хранится в secrets manager (не в коде)

2. **JWT_EXPIRES_IN**
   - 15-30 минут для access token
   - Короткое время для безопасности

3. **REFRESH_TOKEN_EXPIRES_IN**
   - 7-30 дней для refresh token
   - Долгое время для UX

## Обработка ошибок

### Типичные ошибки

1. **Token not provided**
   - Токен отсутствует в запросе
   - HTTP 401 Unauthorized

2. **Invalid token**
   - Неверная подпись
   - Неверный формат
   - HTTP 401 Unauthorized

3. **Token expired**
   - Истёк срок действия
   - HTTP 401 Unauthorized

4. **User not found**
   - Пользователь удалён
   - HTTP 401 Unauthorized

5. **User not activated**
   - Email не подтверждён
   - HTTP 401 Unauthorized

6. **Role context not found**
   - Role context удалён
   - HTTP 401 Unauthorized

7. **Role mismatch**
   - Роль в токене не соответствует БД
   - HTTP 401 Unauthorized

## Тестирование

### Unit тесты

```typescript
describe('AccessTokenGuard', () => {
  let guard: AccessTokenGuard;
  let tokenService: jest.Mocked<TokenService>;

  beforeEach(() => {
    // Setup mocks
  });

  it('should validate valid token from header', async () => {
    const mockRequest = {
      headers: {
        authorization: 'Bearer valid-token',
      },
      cookies: {},
      user: undefined,
    };

    tokenService.validateAccessToken.mockResolvedValue({
      userId: 'user-id',
      roleContextId: 'role-id',
      userRoleName: UserRole.CANDIDATE,
    });

    const result = await guard.canActivate({
      switchToHttp: () => ({
        getRequest: () => mockRequest,
      }),
    } as ExecutionContext);

    expect(result).toEqual({
      user: mockUser,
      roleContext: mockRoleContext,
    });
  });

  it('should throw if user not found', async () => {
    userRepository.findById.mockResolvedValue(null);

    await expect(
      strategy.validate({ sub: 'invalid-id', ... }),
    ).rejects.toThrow(UnauthorizedException);
  });
});
```

## Заключение

Система валидации токенов обеспечивает:

✅ Валидацию access token через `AccessTokenGuard` и `TokenService`
✅ Извлечение access token из cookie или заголовка Authorization
✅ Валидацию refresh token через `TokenService.validateRefreshToken()`
✅ Извлечение refresh token из HttpOnly cookie через `CookieService`
✅ Автоматическую установку токенов в cookies через `AuthCookieInterceptor`
✅ Автоматическую очистку cookies при ошибках
✅ Безопасную обработку ошибок

Все компоненты интегрированы с Guards и Interceptors для защиты маршрутов и управления токенами.
