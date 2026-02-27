# Guards и Decorators - Защита маршрутов

## Обзор

Guards и Decorators обеспечивают защиту маршрутов и упрощают доступ к данным аутентифицированного пользователя.

## Guards

### AccessTokenGuard

Защищает маршруты, требующие валидный access token. Проверяет токен из заголовка Authorization или из cookie.

**Расположение:** `core/guards/access-token.guard.ts`

**Реализация:**
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

    const user = await this.tokenService.validateAccessToken(accessToken);
    req.user = user;

    return true;
  }
}
```

**Использование:**
```typescript
@Controller('protected')
@UseGuards(AccessTokenGuard)
export class ProtectedController {
  @Get('profile')
  getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

**Принципы:**
- Проверяет access token из заголовка `Authorization: Bearer <token>` (приоритет)
- Fallback: проверяет access token из cookie `accessToken`
- Валидирует токен через `TokenService.validateAccessToken()`
- Устанавливает `request.user` с данными из токена
- Бросает UnauthorizedException при отсутствии или невалидности токена

### Refresh Token Flow

Refresh token извлекается из cookie и валидируется в AuthService.

**Принципы:**
- Refresh token **обязательно** извлекается из HttpOnly cookie через `CookieService.getRefreshToken()`
- Валидация происходит в `AuthService.refreshToken()`
- При ошибке - куки автоматически очищаются через `CookieService.clearAuthCookies()`
- Не требует отдельного guard, используется `@Public()` на endpoint

### RolesGuard

Проверяет роль пользователя на основе role context.

**Расположение:** `infrastructure/guards/roles.guard.ts`

**Реализация:**
```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoleNames = this.reflector.getAllAndOverride<string[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoleNames) {
      return true; // Нет ограничений по ролям
    }

    const request = context.switchToHttp().getRequest();
    const roleContext = request.user?.roleContext;

    if (!roleContext) {
      return false;
    }

    return requiredRoleNames.includes(roleContext.userRole);
  }
}
```

**Использование:**
```typescript
@Controller('vacancies')
@UseGuards(AccessTokenGuard, RolesGuard)
export class VacancyController {
  @Post()
  @Roles('EMPLOYER')
  createVacancy(@CurrentUser() user: User) {
    // Только EMPLOYER могут создавать вакансии
  }

  @Get()
  findAll() {
    // Доступно всем аутентифицированным
  }
}
```

**Принципы:**
- Работает вместе с `@Roles()` decorator
- Проверяет userRole (enum) из roleContext
- Возвращает true если роль совпадает
- Возвращает false если роль не совпадает или отсутствует

## Decorators

### @CurrentUser()

Извлекает данные пользователя из request. Устанавливается через `AccessTokenGuard`.

**Расположение:** `core/decorators/auth/current-user.decorator.ts`

**Реализация:**
```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { TokenPayloadDto } from '@/modules/token';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user; // TokenPayloadDto из AccessTokenGuard

    return data ? user?.[data] : user;
  },
);
```

**Использование:**
```typescript
@Get('profile')
@UseGuards(AccessTokenGuard)
getProfile(@CurrentUser() user: TokenPayloadDto) {
  // user содержит: userId, roleContextId, userRoleName, companyId, hrRoleName
  return user;
}

// Или получить конкретное поле
@Get('profile')
getProfile(@CurrentUser('userId') userId: string) {
  return userId;
}
```

### @CurrentRole()

Извлекает RoleContext из request. В текущей реализации `AccessTokenGuard` устанавливает только `TokenPayloadDto` в `request.user`, поэтому для получения полного `RoleContext` с связанными данными (company, hrRole) требуется дополнительная загрузка из БД.

**Расположение:** `core/decorators/auth/current-role.decorator.ts`

**Реализация:**
```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentRole = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    // В базовой реализации возвращает только данные из TokenPayloadDto
    // Для получения полного RoleContext с relations нужно загрузить из БД
    return request.user?.roleContext;
  },
);
```

**Примечание:** 
- `AccessTokenGuard` устанавливает только `TokenPayloadDto` в `request.user` (userId, roleContextId, userRoleName, companyId, hrRoleName)
- Для получения полного `RoleContext` entity с связанными данными (company, hrRole) нужно загрузить его из БД по `roleContextId` в сервисе
- Используйте `@CurrentUser('roleContextId')` для получения ID, затем загрузите полный RoleContext из репозитория

**Использование:**
```typescript
@Get('dashboard')
getDashboard(
  @CurrentUser() user: User,
  @CurrentRole() role: RoleContext,
) {
  // Владелец компании работает через EMPLOYER роль
  // Проверка на HR_ADMIN для доступа к управлению компанией
  const roleContext = request.user?.roleContext;
  if (roleContext?.userRole === UserRole.EMPLOYER) {
    // EMPLOYER всегда имеет companyId
    const hrRole = request.user?.hrRole;
    return this.getEmployerDashboard(user, role.companyId, hrRole?.name || null);
  }
  return this.getCandidateDashboard(user);
}
```

### @CurrentCompany()

Извлекает companyId из `TokenPayloadDto` (для EMPLOYER ролей).

**Расположение:** `core/decorators/auth/current-company.decorator.ts`

**Реализация:**
```typescript
import { createParamDecorator, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { UserRole } from '@auth/enums/user-role.enum';

export const CurrentCompany = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): string => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user; // TokenPayloadDto

    if (!user) {
      throw new ForbiddenException('User not found');
    }

    if (user.userRoleName !== UserRole.EMPLOYER) {
      throw new ForbiddenException('Only EMPLOYER roles have companyId');
    }

    if (!user.companyId) {
      throw new ForbiddenException('EMPLOYER role must have companyId');
    }

    return user.companyId;
  },
);
```

**Использование:**
```typescript
import { UserRole } from '@auth/enums/user-role.enum';

@Get('company/vacancies')
@UseGuards(AccessTokenGuard, RolesGuard)
@Roles(UserRole.EMPLOYER)
getCompanyVacancies(@CurrentCompany() companyId: string) {
  // companyId автоматически извлекается из role context
  return this.vacancyService.findByCompanyId(companyId);
}
```

### @Roles()

Метаданные для RolesGuard. Определяет разрешённые роли для маршрута.

**Расположение:** `infrastructure/decorators/roles.decorator.ts`

**Реализация:**
```typescript
import { SetMetadata } from '@nestjs/common';
import { UserRoleName } from '../enums/user-role-name.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roleNames: UserRole[]) => SetMetadata(ROLES_KEY, roleNames);
```

**Использование:**
```typescript
import { UserRole } from '@auth/enums/user-role.enum';

@Post('applications')
@Roles(UserRole.CANDIDATE)
createApplication(@CurrentUser() user: User) {
  // Только кандидаты могут создавать отклики
}

@Get('applications')
@Roles(UserRole.EMPLOYER)
getApplications(@CurrentUser() user: User) {
  // EMPLOYER могут просматривать отклики
}
```

## Роли

### Системные роли (UserRole)

Роли определяются через enum `UserRole` для типобезопасности.

**Enum для предустановленных ролей:**

**Расположение:** `enums/user-role-name.enum.ts`

```typescript
export enum UserRole {
  CANDIDATE = 'CANDIDATE',
  EMPLOYER = 'EMPLOYER',
  ADMIN = 'ADMIN',
}
```

**Предустановленные роли:**
- `CANDIDATE` - соискатель
- `EMPLOYER` - работодатель
- `ADMIN` - администратор системы

**Принципы:**
- Роли определяются через enum `UserRole` в БД
- Enum обеспечивает типобезопасность на уровне БД и приложения
- Три предустановленные роли: CANDIDATE, EMPLOYER, ADMIN

**Использование:**
```typescript
import { UserRole } from '@auth/enums/user-role.enum';

// Использование в декораторах (используем enum)
@Roles(UserRole.EMPLOYER) // Типобезопасно, автокомплит работает
```

### HR роли (HrRole)

HR роли хранятся в таблице `hr_roles` для расширяемости. Предустановленные роли определены в enum `HrRoleName`.

**Enum для предустановленных ролей:**

**Расположение:** `enums/hr-role-name.enum.ts`

```typescript
export enum HrRoleName {
  HR = 'HR',
  HR_ADMIN = 'HR_ADMIN',
}
```

**Предустановленные роли:**
- `HR` - обычный HR сотрудник
- `HR_ADMIN` - HR администратор

**Принципы:**
- Предустановленные роли обязательно создаются в БД через seed данные
- Для предустановленных ролей используется enum `HrRoleName` для типобезопасности
- Новые HR роли можно добавлять в БД, но для них enum не требуется

**Использование:**
```typescript
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

// Получение HR роли из БД (используем enum для предустановленных)
const hrRole = await this.hrRoleRepository.findByName(HrRoleName.HR_ADMIN);

// Использование в EmployerRolesGuard (используем enum)
@EmployerRoles(HrRoleName.HR_ADMIN) // Типобезопасно, автокомплит работает
```

## Enums

### AuthStatus

### AuthStatus

Статусы ответов аутентификации.

**Расположение:** `enums/auth-status.enum.ts`

```typescript
export enum AuthStatus {
  SUCCESS = 'SUCCESS',
  USER_NOT_FOUND = 'USER_NOT_FOUND',
  INVALID_PASSWORD = 'INVALID_PASSWORD',
  USER_NOT_ACTIVATED = 'USER_NOT_ACTIVATED',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  TOKEN_INVALID = 'TOKEN_INVALID',
  ROLE_NOT_FOUND = 'ROLE_NOT_FOUND',
  MULTIPLE_ROLES = 'MULTIPLE_ROLES', // Требуется выбор роли
}
```

**Использование:**
```typescript
import { AuthStatus } from '@auth/enums/auth-status.enum';

throw new UnauthorizedException(AuthStatus.INVALID_PASSWORD);
```

## Комбинированное использование

### Пример 1: Защищённый маршрут

```typescript
@Controller('applications')
@UseGuards(AccessTokenGuard) // Проверяет токен из cookie или header
export class ApplicationController {
  @Get()
  findAll(@CurrentUser() user: User) {
    // Доступно всем аутентифицированным
  }
}
```

### Пример 2: Маршрут с ограничением по роли

```typescript
@Controller('vacancies')
@UseGuards(AccessTokenGuard, RolesGuard)
export class VacancyController {
  @Post()
  @Roles(UserRole.EMPLOYER)
  create(@CurrentUser() user: User, @Body() dto: CreateVacancyDto) {
    // Только EMPLOYER могут создавать вакансии
  }

  @Get()
  findAll() {
    // Доступно всем (нет @Roles)
  }
}
```

### Пример 3: Разная логика по ролям

```typescript
@Controller('dashboard')
@UseGuards(AccessTokenGuard)
export class DashboardController {
  @Get()
  getDashboard(
    @CurrentUser() user: User,
    @CurrentRole() role: RoleContext,
  ) {
    import { UserRole } from '@auth/enums/user-role.enum';

    if (!role) {
      throw new ForbiddenException('Role context not found');
    }

    switch (role.userRole) {
      case UserRole.CANDIDATE:
        return this.getCandidateDashboard(user);
      // Владелец компании работает через EMPLOYER роль
      // Используем hrRole для проверки прав
      case UserRole.EMPLOYER:
        if (!role.companyId) {
          throw new ForbiddenException('EMPLOYER role must have companyId');
        }
        const hrRole = request.user?.hrRole;
        return this.getEmployerDashboard(user, role.companyId, hrRole?.name || null);
      default:
        throw new ForbiddenException();
    }
  }
}
```

## Глобальные Guards

### Регистрация глобального AccessTokenGuard

```typescript
// main.ts или app.module.ts
import { APP_GUARD } from '@nestjs/core';
import { AccessTokenGuard } from '@/core/guards/access-token.guard';
import { Reflector } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: AccessTokenGuard,
    },
    Reflector,
  ],
})
export class AppModule {}
```

**Примечание:** Глобальный guard применяется ко всем маршрутам. Для публичных маршрутов используйте `@Public()` decorator.

### @Public() Decorator

Помечает маршрут как публичный (игнорирует глобальный guard).

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

**Обновление AccessTokenGuard для поддержки @Public():**

```typescript
@Injectable()
export class AccessTokenGuard implements CanActivate {
  constructor(
    private readonly tokenService: TokenService,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Проверяем @Public() декоратор
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    // Обычная проверка токена
    const req = context.switchToHttp().getRequest<Request & { user?: TokenPayloadDto }>();
    // ... остальная логика проверки токена
  }
}
```

**Использование:**

```typescript
@Controller('auth')
export class AuthController {
  @Post('register')
  @Public()
  @SetAuthCookie() // Устанавливает токены в cookies
  register(@Body() dto: RegisterDto) {
    // Публичный endpoint
  }

  @Post('login')
  @Public()
  @SetAuthCookie() // Устанавливает токены в cookies
  login(@Body() dto: LoginDto) {
    // Публичный endpoint
  }
}
```

### @SetAuthCookie() Decorator

Автоматически применяет `AuthCookieInterceptor` для установки токенов в HttpOnly cookies.

**Расположение:** `core/decorators/auth/set-auth-cookie.decorator.ts`

**Реализация:**
```typescript
import { applyDecorators, UseInterceptors } from '@nestjs/common';
import { AuthCookieInterceptor } from '@/core/interceptors/auth-cookie.interceptor';

export function SetAuthCookie() {
  return applyDecorators(
    UseInterceptors(AuthCookieInterceptor),
  );
}
```

**Принцип работы:**
1. Endpoint возвращает объект с полем `tokens: { accessToken, refreshToken }`
2. `AuthCookieInterceptor` перехватывает response
3. Устанавливает токены в HttpOnly cookies через `CookieService`
4. Удаляет поле `tokens` из response body
5. Клиент получает только данные пользователя, токены в cookies

**Использование:**
```typescript
@Post('login')
@Public()
@SetAuthCookie() // Автоматически устанавливает токены в cookies
async login(@Body() dto: LoginDto) {
  return {
    tokens: {
      accessToken: '...',
      refreshToken: '...',
    },
    user: { ... },
  };
  // Токены автоматически устанавливаются в cookies, удаляются из body
}
```

## Обработка ошибок

### Типичные ошибки Guards

1. **UnauthorizedException**
   - Невалидный токен
   - Токен отсутствует
   - Пользователь не найден
   - HTTP 401

2. **ForbiddenException**
   - Недостаточно прав (роль не совпадает)
   - HTTP 403

### Кастомные исключения

```typescript
import { UnauthorizedException } from '@nestjs/common';
import { AuthStatus } from '../enums/auth-status.enum';

throw new UnauthorizedException({
  status: AuthStatus.USER_NOT_ACTIVATED,
  message: 'Please activate your email',
});
```

## Тестирование

### Unit тесты Guards

```typescript
describe('RolesGuard', () => {
  let guard: RolesGuard;
  let reflector: Reflector;

  beforeEach(() => {
    reflector = new Reflector();
    guard = new RolesGuard(reflector);
  });

  it('should allow access if no roles required', () => {
    const context = createMockContext();
    expect(guard.canActivate(context)).toBe(true);
  });

  it('should allow access if role matches', () => {
    reflector.getAllAndOverride = jest.fn().mockReturnValue([UserRole.EMPLOYER]);
    const context = createMockContext({
      user: { roleContext: { userRole: UserRole.EMPLOYER } },
    });
    expect(guard.canActivate(context)).toBe(true);
  });

  it('should deny access if role does not match', () => {
    reflector.getAllAndOverride = jest.fn().mockReturnValue([UserRole.EMPLOYER]);
    const context = createMockContext({
      user: { roleContext: { userRole: UserRole.CANDIDATE } },
    });
    expect(guard.canActivate(context)).toBe(false);
  });
});
```

## Заключение

Guards и Decorators обеспечивают:

✅ Защиту маршрутов через AccessTokenGuard
✅ Валидацию refresh token через RefreshAuthGuard
✅ Ограничение доступа по ролям через RolesGuard
✅ Упрощённый доступ к user и roleContext через decorators
✅ Гибкую систему прав доступа
✅ Публичные маршруты через @Public()

Все компоненты работают вместе для обеспечения безопасной и удобной системы аутентификации.
