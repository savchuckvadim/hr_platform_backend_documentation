# Guards и Decorators - Защита маршрутов

## Обзор

Guards и Decorators обеспечивают защиту маршрутов и упрощают доступ к данным аутентифицированного пользователя.

## Guards

### JwtAuthGuard

Защищает маршруты, требующие валидный access token.

**Расположение:** `infrastructure/guards/jwt-auth.guard.ts`

**Реализация:**
```typescript
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }
}
```

**Использование:**
```typescript
@Controller('protected')
@UseGuards(JwtAuthGuard)
export class ProtectedController {
  @Get('profile')
  getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

**Принципы:**
- Использует JWT Strategy
- Проверяет валидность access token
- Загружает user и roleContext в request
- Бросает UnauthorizedException при ошибке

### RefreshAuthGuard

Защищает endpoint обновления токена. Использует Refresh Strategy.

**Расположение:** `infrastructure/guards/refresh-auth.guard.ts`

**Реализация:**
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class RefreshAuthGuard extends AuthGuard('refresh') {}
```

**Использование:**
```typescript
@Controller('auth')
export class AuthController {
  @Post('refresh')
  @UseGuards(RefreshAuthGuard)
  async refresh(@CurrentUser() user: User) {
    return this.authService.refreshAccessToken(user);
  }
}
```

**Принципы:**
- Используется только на `/auth/refresh`
- Валидирует refresh token из cookie/body
- Загружает user, roleContext и tokenRecord

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
    const userRole = request.user?.userRole;

    if (!roleContext || !userRole) {
      return false;
    }

    return requiredRoleNames.includes(userRole.name);
  }
}
```

**Использование:**
```typescript
@Controller('vacancies')
@UseGuards(JwtAuthGuard, RolesGuard)
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
- Проверяет userRole.name из roleContext
- Возвращает true если роль совпадает
- Возвращает false если роль не совпадает или отсутствует

## Decorators

### @CurrentUser()

Извлекает User entity из request.

**Расположение:** `infrastructure/decorators/current-user.decorator.ts`

**Реализация:**
```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { User } from '@user/domain/entities/user.entity';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): User => {
    const request = ctx.switchToHttp().getRequest();
    return request.user?.user;
  },
);
```

**Использование:**
```typescript
@Get('profile')
getProfile(@CurrentUser() user: User) {
  return user;
}
```

### @CurrentRole()

Извлекает RoleContext entity из request.

**Расположение:** `infrastructure/decorators/current-role.decorator.ts`

**Реализация:**
```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { RoleContext } from '@auth/domain/entities/role-context.entity';

export const CurrentRole = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): RoleContext => {
    const request = ctx.switchToHttp().getRequest();
    return request.user?.roleContext;
  },
);
```

**Использование:**
```typescript
@Get('dashboard')
getDashboard(
  @CurrentUser() user: User,
  @CurrentRole() role: RoleContext,
) {
  // Владелец компании работает через EMPLOYER роль
  // Проверка на HR_ADMIN для доступа к управлению компанией
  const userRole = request.user?.userRole;
  if (userRole?.name === 'EMPLOYER') {
    // EMPLOYER всегда имеет companyId
    const hrRole = request.user?.hrRole;
    return this.getEmployerDashboard(user, role.companyId, hrRole?.name || null);
  }
  return this.getCandidateDashboard(user);
}
```

### @CurrentCompany()

Извлекает companyId из role context (для EMPLOYER ролей).

**Расположение:** `infrastructure/decorators/current-company.decorator.ts`

**Реализация:**
```typescript
import { createParamDecorator, ExecutionContext, ForbiddenException } from '@nestjs/common';

export const CurrentCompany = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): string => {
    const request = ctx.switchToHttp().getRequest();
    const roleContext = request.user?.roleContext;

    if (!roleContext) {
      throw new ForbiddenException('Role context not found');
    }

    import { UserRoleName } from '@auth/enums/user-role-name.enum';

    const userRole = request.user?.userRole;
    if (userRole?.name === UserRoleName.EMPLOYER && !roleContext.companyId) {
      throw new ForbiddenException('EMPLOYER role must have companyId');
    }

    return roleContext.companyId;
  },
);
```

**Использование:**
```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';

@Get('company/vacancies')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRoleName.EMPLOYER)
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
export const Roles = (...roleNames: UserRoleName[]) => SetMetadata(ROLES_KEY, roleNames);
```

**Использование:**
```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';

@Post('applications')
@Roles(UserRoleName.CANDIDATE)
createApplication(@CurrentUser() user: User) {
  // Только кандидаты могут создавать отклики
}

@Get('applications')
@Roles(UserRoleName.EMPLOYER)
getApplications(@CurrentUser() user: User) {
  // EMPLOYER могут просматривать отклики
}
```

## Роли

### Системные роли (UserRole)

Роли хранятся в таблице `user_roles` для расширяемости. Предустановленные роли определены в enum `UserRoleName`.

**Enum для предустановленных ролей:**

**Расположение:** `enums/user-role-name.enum.ts`

```typescript
export enum UserRoleName {
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
- Предустановленные роли обязательно создаются в БД через seed данные
- Для предустановленных ролей используется enum `UserRoleName` для типобезопасности
- Новые роли можно добавлять в БД, но для них enum не требуется

**Использование:**
```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';

// Получение роли из БД (используем enum для предустановленных)
const userRole = await this.userRoleRepository.findByName(UserRoleName.EMPLOYER);

// Использование в декораторах (используем enum)
@Roles(UserRoleName.EMPLOYER) // Типобезопасно, автокомплит работает
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
@UseGuards(JwtAuthGuard)
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
@UseGuards(JwtAuthGuard, RolesGuard)
export class VacancyController {
  @Post()
  @Roles(UserRoleName.EMPLOYER)
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
@UseGuards(JwtAuthGuard)
export class DashboardController {
  @Get()
  getDashboard(
    @CurrentUser() user: User,
    @CurrentRole() role: RoleContext,
  ) {
    import { UserRoleName } from '@auth/enums/user-role-name.enum';

    const userRole = request.user?.userRole;
    if (!userRole) {
      throw new ForbiddenException('User role not found');
    }

    switch (userRole.name) {
      case UserRoleName.CANDIDATE:
        return this.getCandidateDashboard(user);
      // Владелец компании работает через EMPLOYER роль
      // Используем hrRole для проверки прав
      case UserRoleName.EMPLOYER:
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

### Регистрация глобального JwtAuthGuard

```typescript
// main.ts или app.module.ts
import { APP_GUARD } from '@nestjs/core';
import { JwtAuthGuard } from '@auth/infrastructure/guards/jwt-auth.guard';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
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

**Обновление JwtAuthGuard:**

```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    return super.canActivate(context);
  }
}
```

**Использование:**

```typescript
@Controller('auth')
export class AuthController {
  @Post('register')
  @Public()
  register(@Body() dto: RegisterDto) {
    // Публичный endpoint
  }

  @Post('login')
  @Public()
  login(@Body() dto: LoginDto) {
    // Публичный endpoint
  }
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
    reflector.getAllAndOverride = jest.fn().mockReturnValue([UserRoleName.EMPLOYER]);
    const context = createMockContext({
      user: { roleContext: { userRole: { name: UserRoleName.EMPLOYER } } },
    });
    expect(guard.canActivate(context)).toBe(true);
  });

  it('should deny access if role does not match', () => {
    reflector.getAllAndOverride = jest.fn().mockReturnValue([UserRoleName.EMPLOYER]);
    const context = createMockContext({
      user: { roleContext: { userRole: { name: UserRoleName.CANDIDATE } } },
    });
    expect(guard.canActivate(context)).toBe(false);
  });
});
```

## Заключение

Guards и Decorators обеспечивают:

✅ Защиту маршрутов через JwtAuthGuard
✅ Валидацию refresh token через RefreshAuthGuard
✅ Ограничение доступа по ролям через RolesGuard
✅ Упрощённый доступ к user и roleContext через decorators
✅ Гибкую систему прав доступа
✅ Публичные маршруты через @Public()

Все компоненты работают вместе для обеспечения безопасной и удобной системы аутентификации.
