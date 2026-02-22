# Core Decorators - Переиспользуемые декораторы

## Описание

Переиспользуемые декораторы для упрощения кода и стандартизации паттернов. Включает декораторы для API документации, аутентификации, ролей и DTO валидации.

**Ключевые особенности:**
- Декораторы для получения текущего пользователя, роли и компании
- Декораторы для ограничения доступа по ролям
- Декораторы для валидации DTO
- Декораторы для API документации (Swagger)

## Расположение

```
core/decorators/
├── api/
│   └── api-response.documentation.decorator.ts
├── auth/
│   ├── current-user.decorator.ts
│   ├── current-role.decorator.ts
│   ├── current-company.decorator.ts
│   ├── roles.decorator.ts
│   └── employer-roles.decorator.ts
└── dto/
    ├── password.decorator.ts
    ├── number-or-string.decorator.ts
    └── object-or-string.decorator.ts
```

## Auth Decorators

### @CurrentUser()

Извлекает User entity из request. Добавляется в request через JwtAuthGuard.

**Реализация:**
```typescript
// core/decorators/auth/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { User } from '@prisma/client';

export const CurrentUser = createParamDecorator(
    (data: unknown, ctx: ExecutionContext): User => {
        const request = ctx.switchToHttp().getRequest();
        return request.user?.user;
    },
);
```

**Использование:**
```typescript
import { CurrentUser } from '@core/decorators/auth/current-user.decorator';
import { User } from '@prisma/client';

@Get('profile')
@UseGuards(JwtAuthGuard)
async getProfile(@CurrentUser() user: User) {
    return user;
}
```

**Особенности:**
- Требует `JwtAuthGuard` для работы
- Возвращает полную User entity
- Типобезопасен через TypeScript

**См. также:** [Authentication Module - Guards & Decorators](./04-authentication/guards-decorators.md#currentuser)

### @CurrentRole()

Извлекает RoleContext entity из request. Включает связанные UserRole и HrRole.

**Реализация:**
```typescript
// core/decorators/auth/current-role.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { RoleContext } from '@prisma/client';

export const CurrentRole = createParamDecorator(
    (data: unknown, ctx: ExecutionContext): RoleContext => {
        const request = ctx.switchToHttp().getRequest();
        return request.user?.roleContext;
    },
);
```

**Использование:**
```typescript
import { CurrentRole } from '@core/decorators/auth/current-role.decorator';
import { RoleContext } from '@prisma/client';

@Get('dashboard')
@UseGuards(JwtAuthGuard)
async getDashboard(
    @CurrentUser() user: User,
    @CurrentRole() role: RoleContext,
) {
    // role содержит userRole, hrRole, companyId
    const userRole = role.userRole;
    const hrRole = role.hrRole;

    if (userRole.name === UserRoleName.EMPLOYER) {
        return this.getEmployerDashboard(user, role.companyId, hrRole?.name);
    }
    return this.getCandidateDashboard(user);
}
```

**Особенности:**
- Требует `JwtAuthGuard` для работы
- Возвращает RoleContext с загруженными связанными сущностями (userRole, hrRole)
- Позволяет получить доступ к companyId для EMPLOYER ролей

**См. также:** [Authentication Module - Guards & Decorators](./04-authentication/guards-decorators.md#currentrole)

### @CurrentCompany()

Извлекает companyId из role context. Работает только для EMPLOYER ролей.

**Реализация:**
```typescript
// core/decorators/auth/current-company.decorator.ts
import {
    createParamDecorator,
    ExecutionContext,
    ForbiddenException
} from '@nestjs/common';
import { UserRoleName } from '@auth/enums/user-role-name.enum';

export const CurrentCompany = createParamDecorator(
    (data: unknown, ctx: ExecutionContext): string => {
        const request = ctx.switchToHttp().getRequest();
        const roleContext = request.user?.roleContext;

        if (!roleContext) {
            throw new ForbiddenException('Role context not found');
        }

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
import { CurrentCompany } from '@core/decorators/auth/current-company.decorator';
import { UserRoleName } from '@auth/enums/user-role-name.enum';

@Get('company/vacancies')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRoleName.EMPLOYER)
async getCompanyVacancies(@CurrentCompany() companyId: string) {
    // companyId автоматически извлекается из role context
    return this.vacancyService.findByCompanyId(companyId);
}
```

**Особенности:**
- Работает только для EMPLOYER ролей
- Выбрасывает `ForbiddenException` если role context не найден или companyId отсутствует
- Упрощает получение companyId без ручной проверки

**См. также:** [Authentication Module - Guards & Decorators](./04-authentication/guards-decorators.md#currentcompany)

### @Roles()

Метаданные для RolesGuard. Определяет разрешённые системные роли для маршрута.

**Реализация:**
```typescript
// core/decorators/auth/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { UserRoleName } from '@auth/enums/user-role-name.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roleNames: UserRoleName[]) =>
    SetMetadata(ROLES_KEY, roleNames);
```

**Использование:**
```typescript
import { Roles } from '@core/decorators/auth/roles.decorator';
import { UserRoleName } from '@auth/enums/user-role-name.enum';

@Post('applications')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRoleName.CANDIDATE)
async createApplication(@CurrentUser() user: User) {
    // Только кандидаты могут создавать отклики
    return this.applicationService.create(user.id, dto);
}

@Get('applications')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRoleName.EMPLOYER)
async getApplications(@CurrentUser() user: User) {
    // EMPLOYER могут просматривать отклики
    return this.applicationService.findByCompany(user.companyId);
}
```

**Особенности:**
- Используется вместе с `RolesGuard`
- Принимает массив ролей (можно указать несколько)
- Типобезопасен через `UserRoleName` enum
- Проверяет системную роль (CANDIDATE, EMPLOYER, ADMIN)

**См. также:** [Authentication Module - Guards & Decorators](./04-authentication/guards-decorators.md#roles)

### @EmployerRoles()

Метаданные для EmployerRolesGuard. Определяет разрешённые HR роли для маршрута (только для EMPLOYER).

**Реализация:**
```typescript
// core/decorators/auth/employer-roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

export const EMPLOYER_ROLES_KEY = 'employerRoles';
export const EmployerRoles = (...hrRoleNames: HrRoleName[]) =>
    SetMetadata(EMPLOYER_ROLES_KEY, hrRoleNames);
```

**Использование:**
```typescript
import { EmployerRoles } from '@core/decorators/auth/employer-roles.decorator';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';
import { UserRoleName } from '@auth/enums/user-role-name.enum';

@Post('company/hr')
@UseGuards(JwtAuthGuard, RolesGuard, EmployerRolesGuard)
@Roles(UserRoleName.EMPLOYER)
@EmployerRoles(HrRoleName.HR_ADMIN)
async addHrToCompany(
    @CurrentCompany() companyId: string,
    @Body() dto: AddHrDto,
) {
    // Только HR_ADMIN может добавлять HR в компанию
    return this.companyService.addHr(companyId, dto);
}
```

**Особенности:**
- Используется вместе с `EmployerRolesGuard`
- Работает только для EMPLOYER ролей
- Проверяет HR роль внутри компании (HR, HR_ADMIN)
- Типобезопасен через `HrRoleName` enum

**См. также:** [Authentication Module - HR Roles Guard](./04-authentication/hr-roles-guard.md)

## API Decorators

### ApiResponseDocumentation

Декоратор для документирования формата ответов в Swagger.

**См. подробное описание:** [Swagger Custom Decorators](./07-swagger/custom-decorators.md)

## DTO Decorators

### @IsPassword()

Декоратор для валидации пароля. Проверяет минимальную длину и наличие букв и цифр.

**Реализация:**
```typescript
// core/decorators/dto/password.decorator.ts
import { registerDecorator, ValidationOptions } from 'class-validator';

export function IsPassword(validationOptions?: ValidationOptions) {
    return function (object: Object, propertyName: string) {
        registerDecorator({
            name: 'isPassword',
            target: object.constructor,
            propertyName: propertyName,
            options: validationOptions,
            validator: {
                validate(value: any) {
                    // Минимум 8 символов, хотя бы одна буква и одна цифра
                    return (
                        typeof value === 'string' &&
                        value.length >= 8 &&
                        /[A-Za-z]/.test(value) &&
                        /[0-9]/.test(value)
                    );
                },
                defaultMessage() {
                    return 'Password must be at least 8 characters long and contain at least one letter and one number';
                },
            },
        });
    };
}
```

**Использование:**
```typescript
import { IsPassword } from '@core/decorators/dto/password.decorator';

export class CreateUserDto {
    @IsEmail()
    email: string;

    @IsPassword()
    password: string;
}
```

**Особенности:**
- Минимум 8 символов
- Хотя бы одна буква (латиница)
- Хотя бы одна цифра
- Кастомное сообщение об ошибке

### @IsNumberOrString()

Декоратор для валидации значения, которое может быть числом или строкой.

**Реализация:**
```typescript
// core/decorators/dto/number-or-string.decorator.ts
import { registerDecorator, ValidationOptions } from 'class-validator';

export function IsNumberOrString(validationOptions?: ValidationOptions) {
    return function (object: Object, propertyName: string) {
        registerDecorator({
            name: 'isNumberOrString',
            target: object.constructor,
            propertyName: propertyName,
            options: validationOptions,
            validator: {
                validate(value: any) {
                    return typeof value === 'number' || typeof value === 'string';
                },
                defaultMessage() {
                    return 'Value must be a number or string';
                },
            },
        });
    };
}
```

**Использование:**
```typescript
import { IsNumberOrString } from '@core/decorators/dto/number-or-string.decorator';

export class SearchDto {
    @IsNumberOrString()
    id: number | string;
}
```

### @IsObjectOrString()

Декоратор для валидации значения, которое может быть объектом или строкой.

**Реализация:**
```typescript
// core/decorators/dto/object-or-string.decorator.ts
import { registerDecorator, ValidationOptions } from 'class-validator';

export function IsObjectOrString(validationOptions?: ValidationOptions) {
    return function (object: Object, propertyName: string) {
        registerDecorator({
            name: 'isObjectOrString',
            target: object.constructor,
            propertyName: propertyName,
            options: validationOptions,
            validator: {
                validate(value: any) {
                    return typeof value === 'object' || typeof value === 'string';
                },
                defaultMessage() {
                    return 'Value must be an object or string';
                },
            },
        });
    };
}
```

**Использование:**
```typescript
import { IsObjectOrString } from '@core/decorators/dto/object-or-string.decorator';

export class FilterDto {
    @IsObjectOrString()
    filter: object | string;
}
```

## Best Practices

1. **Используйте декораторы из core** - для консистентности
   ```typescript
   // ✅ Хорошо
   import { CurrentUser } from '@core/decorators/auth/current-user.decorator';

   // ❌ Плохо
   // Создание локального декоратора в каждом модуле
   ```

2. **Комбинируйте декораторы** - для упрощения кода
   ```typescript
   @Get('company/vacancies')
   @UseGuards(JwtAuthGuard, RolesGuard)
   @Roles(UserRoleName.EMPLOYER)
   async getVacancies(
       @CurrentUser() user: User,
       @CurrentCompany() companyId: string,
   ) {
       // Используем несколько декораторов для получения данных
   }
   ```

3. **Используйте типизацию** - для безопасности типов
   ```typescript
   @CurrentUser() user: User  // ✅ Типобезопасно
   @CurrentUser() user        // ❌ Без типизации
   ```

4. **Документируйте декораторы** - для понимания их работы
   ```typescript
   /**
    * Извлекает User entity из request
    * Требует JwtAuthGuard для работы
    */
   export const CurrentUser = createParamDecorator(...)
   ```

5. **Создавайте кастомные декораторы** - для специфичных валидаций
   ```typescript
   // Для специфичных валидаций создавайте в core/decorators/dto/
   @IsPhoneNumber()
   phone: string;
   ```

## Связи с другими модулями

- **[Authentication Module](./04-authentication/guards-decorators.md)** - детальное описание всех auth декораторов и guards
- **[Swagger документация](./07-swagger/custom-decorators.md)** - декораторы для API документации
- **[Response Filters, Interceptors & Middleware](./03-response-filters/index.md)** - использование декораторов в контроллерах

## Структура использования в проекте

```
core/decorators/               # Переиспользуемые декораторы
├── auth/
│   ├── current-user.decorator.ts
│   ├── current-role.decorator.ts
│   ├── current-company.decorator.ts
│   ├── roles.decorator.ts
│   └── employer-roles.decorator.ts
├── dto/
│   ├── password.decorator.ts
│   ├── number-or-string.decorator.ts
│   └── object-or-string.decorator.ts
└── api/
    └── api-response.documentation.decorator.ts

modules/*/api/controllers/     # Использование декораторов
└── *.controller.ts
```

## Примечания

- Все auth декораторы требуют `JwtAuthGuard` для работы
- Декораторы извлекают данные из `request.user`, который устанавливается через Passport strategies
- Для получения companyId используйте `@CurrentCompany()` вместо ручного извлечения из role context
- Декораторы ролей (`@Roles`, `@EmployerRoles`) работают только с соответствующими guards
- DTO декораторы используют `class-validator` для валидации
