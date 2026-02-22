# Entity/DTO маппинг

## Общее описание

В проекте используется разделение между Entity (сущности базы данных) и DTO (Data Transfer Objects). Это обеспечивает:
- Изоляцию слоев (domain не зависит от API)
- Безопасность (скрытие внутренних полей)
- Гибкость (разные DTO для разных операций)
- Валидацию данных на границе API

## Принципы маппинга

### Entity (Domain слой)

Entity - это сущности из базы данных, полученные через Prisma:

```typescript
// domain/entity/user.entity.ts
export class User {
  id: string;
  email: string;
  password: string; // Внутреннее поле, не должно попадать в DTO
  isActivated: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### DTO (API слой)

DTO - объекты для передачи данных через API:

```typescript
// api/dto/user.dto.ts
export class UserResponseDto {
  id: string;
  email: string;
  isActivated: boolean;
  createdAt: Date;
}

export class CreateUserDto {
  email: string;
  password: string;
}
```

## Паттерны маппинга

### 1. Entity → DTO (для ответов)

Используется в контроллерах и сервисах для преобразования Entity в DTO:

```typescript
// application/services/user.service.ts
export class UserService {
  toDto(user: User): UserResponseDto {
    return {
      id: user.id,
      email: user.email,
      isActivated: user.isActivated,
      createdAt: user.createdAt,
      // password НЕ включается
    };
  }
  
  async getUser(id: string): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(id);
    return this.toDto(user);
  }
}
```

### 2. DTO → Entity (для создания/обновления)

Преобразование DTO в Entity для сохранения в БД:

```typescript
// application/services/user.service.ts
export class UserService {
  async createUser(dto: CreateUserDto): Promise<UserResponseDto> {
    const user = await this.userRepository.create({
      email: dto.email,
      password: await this.hashPassword(dto.password),
      isActivated: false,
    });
    
    return this.toDto(user);
  }
}
```

### 3. Частичное обновление (PATCH)

Для частичных обновлений используется `Partial<DTO>`:

```typescript
// api/dto/update-user.dto.ts
export class UpdateUserDto {
  @IsOptional()
  @IsEmail()
  email?: string;
  
  @IsOptional()
  @IsString()
  password?: string;
}

// application/services/user.service.ts
async updateUser(id: string, dto: UpdateUserDto): Promise<UserResponseDto> {
  const updateData: Partial<User> = {};
  
  if (dto.email) updateData.email = dto.email;
  if (dto.password) updateData.password = await this.hashPassword(dto.password);
  
  const user = await this.userRepository.update(id, updateData);
  return this.toDto(user);
}
```

## Работа со связями

### Вложенные Entity

При работе с связанными сущностями создаются вложенные DTO:

```typescript
// api/dto/user-with-profile.dto.ts
export class CandidateProfileDto {
  id: string;
  firstName: string;
  lastName: string;
}

export class UserWithProfileDto {
  id: string;
  email: string;
  profile: CandidateProfileDto;
}

// application/services/user.service.ts
async getUserWithProfile(id: string): Promise<UserWithProfileDto> {
  const user = await this.userRepository.findByIdWithProfile(id);
  
  return {
    id: user.id,
    email: user.email,
    profile: {
      id: user.candidateProfile.id,
      firstName: user.candidateProfile.firstName,
      lastName: user.candidateProfile.lastName,
    },
  };
}
```

### Маппинг массивов

```typescript
// application/services/user.service.ts
toDtoList(users: User[]): UserResponseDto[] {
  return users.map(user => this.toDto(user));
}
```

## Валидация DTO

Используется `class-validator` для валидации входящих данных:

```typescript
// api/dto/create-user.dto.ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;
  
  @IsString()
  @MinLength(8)
  password: string;
}
```

Валидация происходит автоматически через `ValidationPipe` в NestJS.

## Утилиты для маппинга

### Ручной маппинг (рекомендуется)

Явное преобразование обеспечивает контроль и понятность:

```typescript
toDto(user: User): UserResponseDto {
  return {
    id: user.id,
    email: user.email,
    isActivated: user.isActivated,
    createdAt: user.createdAt,
  };
}
```

### Автоматический маппинг (опционально)

Для сложных случаев можно использовать библиотеки типа `class-transformer`:

```typescript
import { plainToInstance } from 'class-transformer';

const dto = plainToInstance(UserResponseDto, user, {
  excludeExtraneousValues: true,
});
```

## Best Practices

1. **Всегда скрывайте чувствительные данные** (пароли, токены) в DTO
2. **Используйте разные DTO для разных операций** (Create, Update, Response)
3. **Валидируйте все входящие данные** через class-validator
4. **Документируйте DTO** через Swagger декораторы
5. **Избегайте циклических зависимостей** - DTO в `api/`, Entity в `domain/`

## Примеры

### Полный пример модуля

```typescript
// domain/entity/user.entity.ts
export class User {
  id: string;
  email: string;
  password: string;
  isActivated: boolean;
  createdAt: Date;
}

// api/dto/user.dto.ts
export class UserResponseDto {
  id: string;
  email: string;
  isActivated: boolean;
  createdAt: Date;
}

export class CreateUserDto {
  @IsEmail()
  email: string;
  
  @IsString()
  @MinLength(8)
  password: string;
}

// application/services/user.service.ts
export class UserService {
  constructor(private userRepository: IUserRepository) {}
  
  toDto(user: User): UserResponseDto {
    return {
      id: user.id,
      email: user.email,
      isActivated: user.isActivated,
      createdAt: user.createdAt,
    };
  }
  
  async createUser(dto: CreateUserDto): Promise<UserResponseDto> {
    const user = await this.userRepository.create({
      email: dto.email,
      password: await this.hashPassword(dto.password),
      isActivated: false,
    });
    
    return this.toDto(user);
  }
}

// api/controllers/user.controller.ts
@Controller('users')
export class UserController {
  constructor(private userService: UserService) {}
  
  @Post()
  async create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    return this.userService.createUser(dto);
  }
}
```

## Ссылки

- [Prisma Schema](./prisma-schema.md) - схема базы данных
- [Получение сущностей со связями](./relations-loading.md) - работа с relations в Prisma
- [Swagger документация](../07-swagger/dto-documentation.md) - документирование DTO
