# Employer Roles Guard - Проверка прав работодателя

## Обзор

Дополнительный guard для проверки прав работодателей (EMPLOYER). Различает обычных HR и HR_ADMIN внутри роли EMPLOYER.

## EmployerRolesGuard

Проверяет тип роли работодателя (HR или HR_ADMIN) для доступа к определённым операциям.

**Расположение:** `infrastructure/guards/employer-roles.guard.ts`

**Реализация:**
```typescript
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { EMPLOYER_ROLES_KEY } from '../decorators/employer-roles.decorator';
import { UserRoleName } from '../enums/user-role-name.enum';
import { HrRoleName } from '../enums/hr-role-name.enum';

@Injectable()
export class EmployerRolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredHrRoleNames = this.reflector.getAllAndOverride<HrRoleName[]>(
      EMPLOYER_ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredHrRoleNames) {
      return true; // Нет ограничений по HR ролям
    }

    const request = context.switchToHttp().getRequest();
    const roleContext = request.user?.roleContext;
    const userRole = request.user?.userRole;
    const hrRole = request.user?.hrRole;

    if (!roleContext || !userRole) {
      return false;
    }

    // Проверяем что это EMPLOYER роль
    if (userRole.name !== UserRoleName.EMPLOYER) {
      return false;
    }

    // Проверяем что EMPLOYER имеет HR роль
    if (!hrRole) {
      throw new ForbiddenException('EMPLOYER role must have hrRole');
    }

    // Сравниваем по enum значению (строка из БД должна совпадать с enum)
    return requiredHrRoleNames.some(role => role === hrRole.name);
  }
}
```

## @EmployerRoles() Decorator

Метаданные для EmployerRolesGuard. Определяет требуемый тип роли работодателя.

**Расположение:** `infrastructure/decorators/employer-roles.decorator.ts`

**Реализация:**
```typescript
import { SetMetadata } from '@nestjs/common';
import { HrRoleName } from '../enums/hr-role-name.enum';

export const EMPLOYER_ROLES_KEY = 'employerRoles';
export const EmployerRoles = (...hrRoleNames: HrRoleName[]) => SetMetadata(EMPLOYER_ROLES_KEY, hrRoleNames);
```

**Использование:**
```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

@Controller('hr')
@UseGuards(JwtAuthGuard, RolesGuard, EmployerRolesGuard)
@Roles(UserRoleName.EMPLOYER) // Только EMPLOYER роли
export class HrController {
  @Post('register')
  @EmployerRoles(HrRoleName.HR_ADMIN) // Только HR_ADMIN может регистрировать HR
  async registerHr(@Body() dto: RegisterHrDto) {
    return this.authService.registerHr(dto);
  }

  @Get('list')
  @EmployerRoles(HrRoleName.HR, HrRoleName.HR_ADMIN) // Все HR могут просматривать список
  async getHrList(@CurrentCompany() companyId: string) {
    return this.hrService.findByCompanyId(companyId);
  }

  @Delete(':hrId')
  @EmployerRoles(HrRoleName.HR_ADMIN) // Только HR_ADMIN может удалять HR
  async deleteHr(@Param('hrId') hrId: string) {
    return this.hrService.delete(hrId);
  }
}
```

## Комбинированное использование

### Пример: Регистрация HR

```typescript
@Controller('auth')
@UseGuards(JwtAuthGuard, RolesGuard, EmployerRolesGuard)
export class AuthController {
  @Post('register/hr')
  @Roles(UserRoleName.EMPLOYER) // Только EMPLOYER роли
  @EmployerRoles(HrRoleName.HR_ADMIN) // Только HR_ADMIN
  async registerHr(
    @CurrentUser() user: User,
    @CurrentRole() role: RoleContext,
    @Body() dto: RegisterHrDto,
  ) {
    // Проверка что HR_ADMIN той же компании
    if (role.companyId !== dto.companyId) {
      throw new ForbiddenException('Can only register HR in your own company');
    }

    return this.authService.registerHr(dto);
  }
}
```

### Пример: Управление HR

```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

@Controller('company/hr')
@UseGuards(JwtAuthGuard, RolesGuard, EmployerRolesGuard)
@Roles(UserRoleName.EMPLOYER)
export class CompanyHrController {
  @Get()
  @EmployerRoles(HrRoleName.HR, HrRoleName.HR_ADMIN)
  async getHrList(@CurrentCompany() companyId: string) {
    // Все HR могут просматривать список
    return this.hrService.findByCompanyId(companyId);
  }

  @Post()
  @EmployerRoles(HrRoleName.HR_ADMIN)
  async addHr(
    @CurrentCompany() companyId: string,
    @Body() dto: AddHrDto,
  ) {
    // Только HR_ADMIN может добавлять
    return this.hrService.addHr(companyId, dto);
  }

  @Delete(':hrId')
  @EmployerRoles(HrRoleName.HR_ADMIN)
  async removeHr(
    @CurrentCompany() companyId: string,
    @Param('hrId') hrId: string,
  ) {
    // Только HR_ADMIN может удалять
    return this.hrService.removeHr(companyId, hrId);
  }
}
```

## Проверка прав в сервисах

### Пример проверки HR_ADMIN

```typescript
@Injectable()
export class HrService {
  async addHr(companyId: string, dto: AddHrDto, currentRole: RoleContext, currentUserRole: UserRole, currentHrRole: HrRole) {
    import { UserRoleName } from '@auth/enums/user-role-name.enum';
    import { HrRoleName } from '@auth/enums/hr-role-name.enum';

    // Проверка что текущий пользователь - HR_ADMIN этой компании
    if (currentUserRole.name !== UserRoleName.EMPLOYER) {
      throw new ForbiddenException('Only EMPLOYER can add HR');
    }

    if (!currentHrRole || currentHrRole.name !== HrRoleName.HR_ADMIN) {
      throw new ForbiddenException('Only HR_ADMIN can add HR');
    }

    if (currentRole.companyId !== companyId) {
      throw new ForbiddenException('Can only add HR in your own company');
    }

    // Создание HR
    return this.createHr(companyId, dto);
  }
}
```

## Обработка ошибок

### Типичные ошибки

1. **ForbiddenException (403)**
   - `Only HR_ADMIN can perform this action`
   - `EMPLOYER role must have companyId`
   - `Can only perform actions in your own company`

## Тестирование

### Unit тесты

```typescript
describe('EmployerRolesGuard', () => {
  it('should allow HR_ADMIN', () => {
    const context = createMockContext({
      user: {
        roleContext: {
          userRole: { name: UserRoleName.EMPLOYER },
          hrRole: { name: HrRoleName.HR_ADMIN },
        },
      },
    });

    reflector.getAllAndOverride = jest.fn().mockReturnValue([HrRoleName.HR_ADMIN]);

    expect(guard.canActivate(context)).toBe(true);
  });

  it('should deny HR (not admin)', () => {
    const context = createMockContext({
      user: {
        roleContext: {
          userRole: { name: UserRoleName.EMPLOYER },
          hrRole: { name: HrRoleName.HR },
        },
      },
    });

    reflector.getAllAndOverride = jest.fn().mockReturnValue([HrRoleName.HR_ADMIN]);

    expect(guard.canActivate(context)).toBe(false);
  });
});
```

## Заключение

EmployerRolesGuard обеспечивает:

✅ Разделение прав между HR и HR_ADMIN внутри роли EMPLOYER
✅ Защиту операций управления HR (только HR_ADMIN)
✅ Проверку принадлежности к компании
✅ Гибкую систему прав доступа

Используется вместе с RolesGuard для многоуровневой защиты маршрутов.
