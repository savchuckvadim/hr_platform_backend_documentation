# Login и Refresh Flow - Потоки входа и обновления

## Обзор

Система входа поддерживает multi-role контекст, multi-device сессии и автоматическое обновление токенов.

## Login Flow

### Endpoint

```
POST /auth/login
```

### DTO

**Файл:** `api/dto/login.dto.ts`

```typescript
export class LoginDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @IsNotEmpty()
  password: string;

  @IsString()
  @IsOptional()
  roleContextId?: string; // Если у пользователя несколько ролей
}
```

### Поток входа

**Шаг 1: Валидация данных**

```typescript
@Post('login')
@Public()
async login(@Body() dto: LoginDto, @Req() request: Request) {
  return this.authService.login(dto, request);
}
```

**Шаг 2: Поиск пользователя**

```typescript
const user = await this.userRepository.findByEmail(dto.email);
if (!user) {
  throw new UnauthorizedException(AuthStatus.USER_NOT_FOUND);
}
```

**Шаг 3: Проверка активации**

```typescript
if (!user.isActivated) {
  throw new UnauthorizedException(AuthStatus.USER_NOT_ACTIVATED);
}
```

**Шаг 4: Проверка пароля**

```typescript
const isPasswordValid = await bcrypt.compare(dto.password, user.password);
if (!isPasswordValid) {
  throw new UnauthorizedException(AuthStatus.INVALID_PASSWORD);
}
```

**Шаг 5: Получение Role Context**

```typescript
const roleContexts = await this.roleContextRepository.findByUserId(user.id);

if (roleContexts.length === 0) {
  throw new UnauthorizedException(AuthStatus.ROLE_NOT_FOUND);
}

let roleContext: RoleContext;

if (roleContexts.length === 1) {
  // Одна роль - используем её
  roleContext = roleContexts[0];
} else {
  // Несколько ролей - требуется выбор
  if (!dto.roleContextId) {
    import { UserRoleName } from '@auth/enums/user-role-name.enum';
    import { HrRoleName } from '@auth/enums/hr-role-name.enum';

    // Загружаем userRole и hrRole для каждого roleContext
    const rolesWithDetails = await Promise.all(
      roleContexts.map(async (rc) => {
        const userRole = await rc.userRole;
        const hrRole = rc.hrRoleId ? await rc.hrRole : null;
        return {
          id: rc.id,
          userRoleName: userRole.name, // Значение из БД (для предустановленных совпадает с enum)
          companyId: rc.companyId,
          hrRoleName: hrRole?.name || null, // Для EMPLOYER ролей (для предустановленных совпадает с enum)
        };
      }),
    );

    // Возвращаем список ролей для выбора
    return {
      status: AuthStatus.MULTIPLE_ROLES,
      roles: rolesWithDetails,
    };
  }

  // Используем выбранную роль
  roleContext = roleContexts.find(rc => rc.id === dto.roleContextId);
  if (!roleContext) {
    throw new UnauthorizedException(AuthStatus.ROLE_NOT_FOUND);
  }
}
```

**Шаг 6: Генерация deviceId**

```typescript
const deviceId = this.generateDeviceId(request);
```

**Шаг 7: Проверка существующего токена**

```typescript
// Проверяем unique constraint (userId, roleContextId, deviceId)
const existingToken = await this.tokenRepository.findByUserRoleDevice(
  user.id,
  roleContext.id,
  deviceId,
);

if (existingToken) {
  // Удаляем старый токен (смена роли на устройстве)
  await this.tokenRepository.delete(existingToken.id);
}
```

**Шаг 8: Генерация токенов**

```typescript
const { accessToken, refreshToken } = await this.generateTokens(
  user,
  roleContext,
  deviceId,
  request,
);
```

**Шаг 9: Сохранение refresh token**

```typescript
await this.saveRefreshToken(
  user.id,
  roleContext.id,
  refreshToken,
  deviceId,
  request,
);
```

**Шаг 10: Возврат ответа**

```typescript
return {
  accessToken,
  refreshToken, // В production - только в HttpOnly cookie
  user: {
    id: user.id,
    email: user.email,
    userRoleName: roleContext.userRole.name,
    roleContextId: roleContext.id,
    companyId: roleContext.companyId, // null для CANDIDATE, обязателен для EMPLOYER
    hrRoleName: roleContext.hrRole?.name || null, // null для не-EMPLOYER ролей
  },
};
```

### Полная последовательность

```
1. POST /auth/login
   ↓
2. Валидация DTO
   ↓
3. Поиск User по email
   ↓
4. Проверка активации
   ↓
5. Проверка пароля
   ↓
6. Получение Role Contexts
   ↓
7. Выбор роли (если несколько)
   ↓
8. Генерация deviceId
   ↓
9. Проверка существующего токена
   ↓
10. Удаление старого токена (если есть)
   ↓
11. Генерация токенов
   ↓
12. Сохранение refresh token
   ↓
13. Возврат токенов
```

### Обработка множественных ролей

Если у пользователя несколько RoleContext, требуется выбор роли:

**Ответ при множественных ролях:**

```typescript
{
  status: 'MULTIPLE_ROLES',
  roles: [
    {
      id: 'rc-1',
      userRoleName: 'CANDIDATE',
      companyId: null,
      hrRoleName: null,
    },
    {
      id: 'rc-2',
      userRoleName: 'EMPLOYER',
      companyId: 'company-id',
      hrRoleName: 'HR_ADMIN',
    },
  ],
}
```

**Клиент отправляет повторный запрос с выбранной ролью:**

```typescript
POST /auth/login
{
  email: 'user@example.com',
  password: 'password',
  roleContextId: 'rc-1', // Выбранная роль
}
```

## Refresh Flow

### Endpoint

```
POST /auth/refresh
```

### DTO

**Файл:** `api/dto/refresh.dto.ts`

```typescript
export class RefreshDto {
  @IsString()
  @IsOptional()
  refreshToken?: string; // Если не в cookie
}
```

### Поток обновления

**Шаг 1: Валидация refresh token**

```typescript
@Post('refresh')
@UseGuards(RefreshAuthGuard)
@Public()
async refresh(
  @CurrentUser() user: User,
  @CurrentRole() roleContext: RoleContext,
  @Req() request: Request,
) {
  return this.authService.refreshAccessToken(user, roleContext, request);
}
```

RefreshAuthGuard уже проверил:
- Наличие refresh token
- Валидность токена в БД
- Срок действия
- Hash сравнение

**Шаг 2: Генерация нового access token**

```typescript
const accessToken = await this.generateAccessToken(user, roleContext);
```

**Шаг 3: Sliding Session (опционально)**

```typescript
// Продление refresh token если осталось < 50% времени
const tokenRecord = request.user.tokenRecord;
const now = new Date();
const expiresAt = new Date(tokenRecord.expiresAt);
const timeLeft = expiresAt.getTime() - now.getTime();
const totalTime = expiresAt.getTime() - tokenRecord.createdAt.getTime();

if (timeLeft / totalTime < 0.5) {
  // Продлеваем refresh token
  const newExpiresAt = new Date();
  newExpiresAt.setDate(newExpiresAt.getDate() + 7); // +7 дней

  await this.tokenRepository.update(tokenRecord.id, {
    expiresAt: newExpiresAt,
  });
}
```

**Шаг 4: Возврат нового access token**

```typescript
return {
  accessToken,
  // refreshToken не возвращается (остаётся в cookie)
};
```

### Полная последовательность

```
1. POST /auth/refresh
   ↓
2. RefreshAuthGuard валидирует refresh token
   ↓
3. Генерация нового access token
   ↓
4. Проверка времени до истечения refresh token
   ↓
5. Продление refresh token (sliding session)
   ↓
6. Возврат нового access token
```

## Logout Flow

### Logout одного устройства

**Endpoint:**

```
POST /auth/logout
```

**Поток:**

```typescript
@Post('logout')
@UseGuards(JwtAuthGuard)
async logout(
  @CurrentUser() user: User,
  @CurrentRole() roleContext: RoleContext,
  @Req() request: Request,
) {
  const deviceId = this.extractDeviceId(request);

  await this.tokenRepository.deleteByUserRoleDevice(
    user.id,
    roleContext.id,
    deviceId,
  );

  return { message: 'Logged out successfully' };
}
```

**Принципы:**
- Удаляется только токен текущего устройства
- Другие устройства остаются залогинены
- Используется deviceId из request или JWT

### Logout со всех устройств

**Endpoint:**

```
POST /auth/logout-all
```

**Поток:**

```typescript
@Post('logout-all')
@UseGuards(JwtAuthGuard)
async logoutAll(@CurrentUser() user: User) {
  await this.tokenRepository.deleteAllByUserId(user.id);

  return { message: 'Logged out from all devices' };
}
```

**Принципы:**
- Удаляются все токены пользователя
- Независимо от роли и устройства
- Используется для смены пароля, безопасности и т.д.

## Sliding Session

### Принцип

Access token автоматически продлевается при активности пользователя.

### Реализация через Interceptor

**Файл:** `infrastructure/interceptors/token-refresh.interceptor.ts`

```typescript
@Injectable()
export class TokenRefreshInterceptor implements NestInterceptor {
  constructor(
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      return next.handle();
    }

    // Проверяем время до истечения access token
    const token = this.extractToken(request);
    if (token) {
      const decoded = this.jwtService.decode(token) as JwtPayload;
      const now = Math.floor(Date.now() / 1000);
      const expiresAt = decoded.exp;
      const timeLeft = expiresAt - now;
      const totalTime = expiresAt - decoded.iat;

      // Если осталось < 50% времени - продлеваем
      if (timeLeft / totalTime < 0.5) {
        const newAccessToken = await this.generateAccessToken(
          user.user,
          user.roleContext,
        );

        // Добавляем в response header
        const response = context.switchToHttp().getResponse();
        response.setHeader('X-New-Access-Token', newAccessToken);
      }
    }

    return next.handle();
  }

  private extractToken(request: Request): string | null {
    const authHeader = request.headers.authorization;
    if (!authHeader) {
      return null;
    }
    return authHeader.replace('Bearer ', '');
  }
}
```

**Регистрация:**

```typescript
@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: TokenRefreshInterceptor,
    },
  ],
})
export class AuthModule {}
```

## Обработка ошибок

### Типичные ошибки Login

1. **UnauthorizedException (401)**
   - `USER_NOT_FOUND` - пользователь не найден
   - `INVALID_PASSWORD` - неверный пароль
   - `USER_NOT_ACTIVATED` - email не активирован
   - `ROLE_NOT_FOUND` - роль не найдена

2. **BadRequestException (400)**
   - `MULTIPLE_ROLES` - требуется выбор роли (не ошибка, но требует действия)

### Типичные ошибки Refresh

1. **UnauthorizedException (401)**
   - `TOKEN_NOT_PROVIDED` - токен отсутствует
   - `TOKEN_INVALID` - невалидный токен
   - `TOKEN_EXPIRED` - токен истёк
   - `USER_NOT_FOUND` - пользователь не найден
   - `USER_NOT_ACTIVATED` - пользователь не активирован

## Примеры использования

### Пример 1: Login с одной ролью

```typescript
// Request
POST /auth/login
{
  "email": "candidate@example.com",
  "password": "Password123"
}

// Response
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "abc123...",
  "user": {
    "id": "user-id",
    "email": "candidate@example.com",
    "userRoleName": "CANDIDATE", // Значение из БД (совпадает с UserRoleName.CANDIDATE)
    "roleContextId": "rc-1"
  }
}
```

### Пример 2: Login с множественными ролями

```typescript
// Request 1
POST /auth/login
{
  "email": "user@example.com",
  "password": "Password123"
}

// Response 1
{
  "status": "MULTIPLE_ROLES",
  "roles": [
    {
      "id": "rc-1",
      "userRoleName": "CANDIDATE",
      "companyId": null,
      "hrRoleName": null
    },
    {
      "id": "rc-2",
      "userRoleName": "EMPLOYER",
      "companyId": "company-id",
      "hrRoleName": "HR_ADMIN"
    },
    {
      "id": "rc-3",
      "userRoleName": "EMPLOYER",
      "companyId": "company-id",
      "hrRoleName": "HR_ADMIN"
    }
  ]
}

// Request 2 (клиент выбирает роль)
POST /auth/login
{
  "email": "user@example.com",
  "password": "Password123",
  "roleContextId": "rc-2"
}

// Response 2
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "abc123...",
  "user": {
    "id": "user-id",
    "email": "user@example.com",
    "userRoleName": "EMPLOYER", // Значение из БД (совпадает с UserRoleName.EMPLOYER)
    "roleContextId": "rc-2"
  }
}
```

### Пример 3: Refresh

```typescript
// Request
POST /auth/refresh
Cookie: refreshToken=abc123...

// Response
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs..." // Новый токен
}
```

### Пример 4: Logout

```typescript
// Request
POST /auth/logout
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

// Response
{
  "message": "Logged out successfully"
}
```

## Тестирование

### Unit тесты

```typescript
describe('AuthService - login', () => {
  it('should login user with single role', async () => {
    const dto = {
      email: 'test@example.com',
      password: 'Password123',
    };

    const result = await authService.login(dto, mockRequest);

    expect(result).toHaveProperty('accessToken');
    expect(result).toHaveProperty('refreshToken');
    expect(result.user.email).toBe(dto.email);
  });

  it('should return multiple roles if user has several', async () => {
    // Setup: user with 2 roles
    const result = await authService.login(dto, mockRequest);

    expect(result.status).toBe(AuthStatus.MULTIPLE_ROLES);
    expect(result.roles).toHaveLength(2);
  });

  it('should throw if password invalid', async () => {
    await expect(
      authService.login({ email: 'test@example.com', password: 'wrong' }, mockRequest),
    ).rejects.toThrow(UnauthorizedException);
  });
});
```

## Заключение

Потоки входа и обновления обеспечивают:

✅ Безопасный вход с проверкой пароля
✅ Поддержку множественных ролей
✅ Автоматическое обновление access token
✅ Sliding session для продления сессий
✅ Logout одного устройства
✅ Logout со всех устройств
✅ Валидацию всех данных
✅ Обработку ошибок

Все потоки интегрированы с Guards и Strategies для обеспечения безопасности.
