# DTO Documentation - Документирование DTO

## Обзор

Автоматическое документирование DTO через декораторы `@ApiProperty()` для генерации Swagger схемы.

## Базовое использование

### Простой DTO

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateUserDto {
    @ApiProperty({
        description: 'Email пользователя',
        example: 'user@example.com',
    })
    @IsEmail()
    email: string;

    @ApiProperty({
        description: 'Пароль пользователя',
        example: 'SecurePassword123!',
        minLength: 8,
    })
    @IsString()
    @MinLength(8)
    password: string;
}
```

### DTO с вложенными объектами

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import { ValidateNested } from 'class-validator';

export class AddressDto {
    @ApiProperty({ description: 'Город', example: 'Москва' })
    city: string;

    @ApiProperty({ description: 'Улица', example: 'Ленина, 1' })
    street: string;
}

export class CreateCandidateDto {
    @ApiProperty({ description: 'Имя', example: 'Иван' })
    firstName: string;

    @ApiProperty({ description: 'Фамилия', example: 'Иванов' })
    lastName: string;

    @ApiProperty({
        description: 'Адрес',
        type: AddressDto,
    })
    @Type(() => AddressDto)
    @ValidateNested()
    address: AddressDto;
}
```

### DTO с массивом

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateVacancyDto {
    @ApiProperty({ description: 'Название вакансии', example: 'Senior Developer' })
    title: string;

    @ApiProperty({
        description: 'Требуемые навыки',
        type: [String],
        example: ['TypeScript', 'NestJS', 'PostgreSQL'],
    })
    skills: string[];
}
```

### DTO с enum

```typescript
import { ApiProperty } from '@nestjs/swagger';

export enum ApplicationStatus {
    NEW = 'NEW',
    VIEWED = 'VIEWED',
    INVITED = 'INVITED',
    REJECTED = 'REJECTED',
    ACCEPTED = 'ACCEPTED',
}

export class UpdateApplicationStatusDto {
    @ApiProperty({
        description: 'Статус отклика',
        enum: ApplicationStatus,
        example: ApplicationStatus.INVITED,
    })
    status: ApplicationStatus;
}
```

### DTO с опциональными полями

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class UpdateUserDto {
    @ApiProperty({ description: 'Email', example: 'user@example.com' })
    email: string;

    @ApiPropertyOptional({
        description: 'Имя пользователя',
        example: 'Иван',
    })
    firstName?: string;

    @ApiPropertyOptional({
        description: 'Фамилия пользователя',
        example: 'Иванов',
    })
    lastName?: string;
}
```

## Пагинация DTO

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';

export class PaginationDto {
    @ApiPropertyOptional({
        description: 'Номер страницы',
        example: 1,
        minimum: 1,
        default: 1,
    })
    @IsOptional()
    @Type(() => Number)
    @IsInt()
    @Min(1)
    page?: number = 1;

    @ApiPropertyOptional({
        description: 'Количество элементов на странице',
        example: 20,
        minimum: 1,
        maximum: 100,
        default: 20,
    })
    @IsOptional()
    @Type(() => Number)
    @IsInt()
    @Min(1)
    @Max(100)
    limit?: number = 20;
}
```

## Response DTO

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class UserDto {
    @ApiProperty({ description: 'ID пользователя', example: 'uuid-123' })
    id: string;

    @ApiProperty({ description: 'Email', example: 'user@example.com' })
    email: string;

    @ApiProperty({ description: 'Имя', example: 'Иван' })
    firstName: string;

    @ApiProperty({ description: 'Фамилия', example: 'Иванов' })
    lastName: string;

    @ApiProperty({
        description: 'Дата создания',
        example: '2024-01-15T10:00:00Z',
    })
    createdAt: string;
}

export class PaginatedUsersDto {
    @ApiProperty({
        description: 'Список пользователей',
        type: [UserDto],
    })
    items: UserDto[];

    @ApiProperty({ description: 'Общее количество', example: 100 })
    total: number;

    @ApiProperty({ description: 'Текущая страница', example: 1 })
    page: number;

    @ApiProperty({ description: 'Элементов на странице', example: 20 })
    limit: number;

    @ApiProperty({ description: 'Всего страниц', example: 5 })
    totalPages: number;
}
```

## Best Practices

1. **Всегда добавляйте @ApiProperty()** - для всех полей DTO
2. **Добавляйте примеры** - для лучшего понимания
3. **Используйте @ApiPropertyOptional()** - для опциональных полей
4. **Документируйте ограничения** - minLength, maxLength, pattern
5. **Используйте enum** - для ограниченного набора значений
6. **Документируйте вложенные объекты** - через type и @Type()
