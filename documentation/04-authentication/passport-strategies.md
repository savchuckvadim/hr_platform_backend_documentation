# Passport Strategies - Стратегии аутентификации

## Обзор

Модуль использует NestJS Passport для реализации стратегий аутентификации. Две основные стратегии: JWT (для access token) и Refresh (для refresh token).

## JWT Strategy

### Назначение

Проверяет и валидирует access token на защищённых маршрутах.

### Расположение

`infrastructure/strategies/jwt.strategy.ts`

### Реализация

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';
import { UserRepository } from '@user/infrastructure/repositories/user.repository';
import { RoleContextRepository } from '@auth/infrastructure/repositories/role-context.repository';

import { UserRoleName } from '@auth/enums/user-role-name.enum';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

export interface JwtPayload {
  sub: string;              // userId
  roleContextId: string;    // active role context ID
  userRoleName: string;     // Название системной роли из user_roles.name (для предустановленных используется UserRoleName enum)
  companyId: string | null; // ID компании (для EMPLOYER роли - обязательно)
  hrRoleName: string | null; // Название HR роли из hr_roles.name (для предустановленных используется HrRoleName enum, только для EMPLOYER роли)
  iat: number;
  exp: number;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    private readonly configService: ConfigService,
    private readonly userRepository: UserRepository,
    private readonly roleContextRepository: RoleContextRepository,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.getOrThrow<string>('JWT_SECRET'),
    });
  }

  async validate(payload: JwtPayload) {
    // Проверка существования пользователя
    const user = await this.userRepository.findById(payload.sub);
    if (!user) {
      throw new UnauthorizedException('User not found');
    }

    // Проверка активации
    if (!user.isActivated) {
      throw new UnauthorizedException('User not activated');
    }

    // Загрузка role context
    const roleContext = await this.roleContextRepository.findById(
      payload.roleContextId,
    );
    if (!roleContext) {
      throw new UnauthorizedException('Role context not found');
    }

    // Загрузка userRole и hrRole
    const userRole = await roleContext.userRole;
    if (!userRole) {
      throw new UnauthorizedException('User role not found');
    }

    // Проверка соответствия роли
    if (userRole.name !== payload.userRoleName) {
      throw new UnauthorizedException('Role mismatch');
    }

    import { UserRoleName } from '@auth/enums/user-role-name.enum';

    // Проверка companyId для EMPLOYER роли
    if (userRole.name === UserRoleName.EMPLOYER && !roleContext.companyId) {
      throw new UnauthorizedException('EMPLOYER role must have companyId');
    }

    // Проверка соответствия companyId
    if (payload.companyId && roleContext.companyId !== payload.companyId) {
      throw new UnauthorizedException('Company mismatch');
    }

    import { HrRoleName } from '@auth/enums/hr-role-name.enum';

    // Проверка hrRole для EMPLOYER роли
    let hrRole = null;
    if (userRole.name === UserRoleName.EMPLOYER && roleContext.hrRoleId) {
      hrRole = await roleContext.hrRole;
      if (hrRole && hrRole.name !== payload.hrRoleName) {
        throw new UnauthorizedException('HR role mismatch');
      }
    }

    // Возвращаем данные для request.user
    return {
      user,
      roleContext,
      userRole,
      hrRole,
      companyId: roleContext.companyId,
      hrRoleName: hrRole?.name || null,
    };
  }
}
```

### Принципы работы

1. **Извлечение токена**
   - Из заголовка `Authorization: Bearer <token>`
   - Использует `ExtractJwt.fromAuthHeaderAsBearerToken()`

2. **Валидация токена**
   - Проверка подписи через `JWT_SECRET`
   - Проверка срока действия (exp)
   - Проверка структуры payload

3. **Загрузка данных**
   - Загрузка User из БД
   - Загрузка RoleContext из БД
   - Проверка активации пользователя
   - Проверка соответствия роли

4. **Возврат данных**
   - Возвращает объект `{ user, roleContext }`
   - Доступен через `@CurrentUser()` и `@CurrentRole()` decorators

### Использование

```typescript
@Controller('protected')
@UseGuards(JwtAuthGuard)  // Использует JwtStrategy
export class ProtectedController {
  @Get('profile')
  getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

## Refresh Strategy

### Назначение

Валидирует refresh token при обновлении access token.

### Расположение

`infrastructure/strategies/refresh.strategy.ts`

### Реализация

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-custom';
import { Request } from 'express';
import { TokenRepository } from '@auth/infrastructure/repositories/token.repository';
import { UserRepository } from '@user/infrastructure/repositories/user.repository';
import { RoleContextRepository } from '@auth/infrastructure/repositories/role-context.repository';
import * as bcrypt from 'bcrypt';

@Injectable()
export class RefreshStrategy extends PassportStrategy(Strategy, 'refresh') {
  constructor(
    private readonly tokenRepository: TokenRepository,
    private readonly userRepository: UserRepository,
    private readonly roleContextRepository: RoleContextRepository,
  ) {
    super();
  }

  async validate(request: Request) {
    // Извлечение refresh token из cookie или body
    const refreshToken = this.extractRefreshToken(request);
    if (!refreshToken) {
      throw new UnauthorizedException('Refresh token not provided');
    }

    // Поиск токена в БД
    const tokenRecord = await this.tokenRepository.findByRefreshToken(
      refreshToken,
    );
    if (!tokenRecord) {
      throw new UnauthorizedException('Invalid refresh token');
    }

    // Проверка срока действия
    if (tokenRecord.expiresAt < new Date()) {
      throw new UnauthorizedException('Refresh token expired');
    }

    // Сравнение hash
    const isValid = await bcrypt.compare(
      refreshToken,
      tokenRecord.refreshToken,
    );
    if (!isValid) {
      throw new UnauthorizedException('Invalid refresh token');
    }

    // Загрузка пользователя
    const user = await this.userRepository.findById(tokenRecord.userId);
    if (!user) {
      throw new UnauthorizedException('User not found');
    }

    // Проверка активации
    if (!user.isActivated) {
      throw new UnauthorizedException('User not activated');
    }

    // Загрузка role context
    const roleContext = await this.roleContextRepository.findById(
      tokenRecord.roleContextId,
    );
    if (!roleContext) {
      throw new UnauthorizedException('Role context not found');
    }

    // Возвращаем данные
    return {
      user,
      roleContext,
      tokenRecord,
    };
  }

  private extractRefreshToken(request: Request): string | null {
    // Из cookie (предпочтительно)
    if (request.cookies?.refreshToken) {
      return request.cookies.refreshToken;
    }

    // Из body (fallback)
    if (request.body?.refreshToken) {
      return request.body.refreshToken;
    }

    return null;
  }
}
```

### Принципы работы

1. **Извлечение токена**
   - Из HttpOnly cookie (предпочтительно)
   - Или из body запроса (fallback)

2. **Поиск в БД**
   - Поиск записи по refresh token (hash lookup)
   - Проверка существования записи

3. **Валидация**
   - Проверка срока действия (expiresAt)
   - Сравнение hash токена
   - Проверка активации пользователя

4. **Возврат данных**
   - Возвращает `{ user, roleContext, tokenRecord }`
   - `tokenRecord` используется для обновления expiresAt (sliding session)

### Использование

```typescript
@Controller('auth')
export class AuthController {
  @Post('refresh')
  @UseGuards(RefreshAuthGuard)  // Использует RefreshStrategy
  async refresh(@CurrentUser() user: User, @CurrentRole() role: RoleContext) {
    // Генерация нового access token
    return this.authService.refreshAccessToken(user, role);
  }
}
```

## Регистрация стратегий

### В AuthModule

```typescript
import { Module } from '@nestjs/common';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtStrategy } from './infrastructure/strategies/jwt.strategy';
import { RefreshStrategy } from './infrastructure/strategies/refresh.strategy';

@Module({
  imports: [
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        secret: configService.getOrThrow<string>('JWT_SECRET'),
        signOptions: {
          expiresIn: configService.getOrThrow<string>('JWT_EXPIRES_IN'),
        },
      }),
      inject: [ConfigService],
    }),
  ],
  providers: [JwtStrategy, RefreshStrategy],
  exports: [PassportModule, JwtModule],
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
describe('JwtStrategy', () => {
  let strategy: JwtStrategy;
  let userRepository: jest.Mocked<UserRepository>;
  let roleContextRepository: jest.Mocked<RoleContextRepository>;

  beforeEach(() => {
    // Setup mocks
  });

  it('should validate valid token', async () => {
    const payload = {
      sub: 'user-id',
      roleContextId: 'role-id',
      userRoleName: UserRoleName.CANDIDATE,
      companyId: null,
      hrRoleName: null,
      iat: Date.now(),
      exp: Date.now() + 3600,
    };

    userRepository.findById.mockResolvedValue(mockUser);
    roleContextRepository.findById.mockResolvedValue({
      ...mockRoleContext,
      userRole: { name: UserRoleName.CANDIDATE },
      hrRole: null,
    });

    const result = await strategy.validate(payload);

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

Passport стратегии обеспечивают:

✅ Валидацию access token через JWT Strategy
✅ Валидацию refresh token через Refresh Strategy
✅ Загрузку пользователя и role context
✅ Проверку активации и прав доступа
✅ Безопасную обработку ошибок

Все стратегии интегрированы с Guards для защиты маршрутов.
