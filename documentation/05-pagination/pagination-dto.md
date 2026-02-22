# Pagination DTO - DTO для пагинации

## Обзор

DTO (Data Transfer Objects) для пагинации обеспечивают единый формат запросов и ответов во всех модулях проекта. Поддерживают offset-based и cursor-based пагинацию с валидацией и Swagger документацией.

## Структура

```
core/pagination/dto/
├── pagination.dto.ts           # Offset-based запросы (page/limit)
├── cursor-pagination.dto.ts    # Cursor-based запросы (cursor/limit)
└── paginated-result.dto.ts     # Единый формат ответа
```

## 1. PaginationDto - Offset-based запросы

**Расположение:** `core/pagination/dto/pagination.dto.ts`

**Назначение:** DTO для offset-based пагинации (page/limit) с поддержкой сортировки.

**Реализация:**

```typescript
import { Type } from 'class-transformer';
import { IsEnum, IsInt, IsOptional, IsString, Max, Min } from 'class-validator';
import { ApiPropertyOptional } from '@nestjs/swagger';

export enum SortOrder {
  ASC = 'asc',
  DESC = 'desc',
}

export class PaginationDto {
  @ApiPropertyOptional({
    default: 1,
    minimum: 1,
    description: 'Page number (starts from 1)',
    example: 1
  })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @ApiPropertyOptional({
    default: 10,
    minimum: 1,
    maximum: 100,
    description: 'Number of items per page',
    example: 10
  })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @ApiPropertyOptional({
    default: 'createdAt',
    description: 'Field to sort by',
    example: 'createdAt'
  })
  @IsOptional()
  @IsString()
  sortBy?: string = 'createdAt';

  @ApiPropertyOptional({
    enum: SortOrder,
    default: SortOrder.DESC,
    description: 'Sort order',
    example: SortOrder.DESC
  })
  @IsOptional()
  @IsEnum(SortOrder)
  sortOrder?: SortOrder = SortOrder.DESC;
}
```

**Параметры:**

| Параметр | Тип | Обязательный | По умолчанию | Описание |
|----------|-----|--------------|--------------|----------|
| `page` | `number` | Нет | `1` | Номер страницы (начинается с 1) |
| `limit` | `number` | Нет | `10` | Количество элементов на странице (макс. 100) |
| `sortBy` | `string` | Нет | `'createdAt'` | Поле для сортировки |
| `sortOrder` | `SortOrder` | Нет | `DESC` | Порядок сортировки (ASC/DESC) |

**Валидация:**
- `page`: минимум 1, целое число
- `limit`: минимум 1, максимум 100, целое число
- `sortBy`: строка
- `sortOrder`: enum (ASC/DESC)

**Пример использования:**

```typescript
@Get()
async findAll(@Query() pagination: PaginationDto) {
  // pagination.page = 1 (если не указан)
  // pagination.limit = 10 (если не указан)
  // pagination.sortBy = 'createdAt' (если не указан)
  // pagination.sortOrder = SortOrder.DESC (если не указан)
}
```

## 2. CursorPaginationDto - Cursor-based запросы

**Расположение:** `core/pagination/dto/cursor-pagination.dto.ts`

**Назначение:** DTO для cursor-based пагинации (cursor/limit) для больших наборов данных.

**Реализация:**

```typescript
import { Type } from 'class-transformer';
import { IsInt, IsOptional, IsString, Max, Min } from 'class-validator';
import { ApiPropertyOptional } from '@nestjs/swagger';

export class CursorPaginationDto {
  @ApiPropertyOptional({
    description: 'Cursor for pagination (ID of last item from previous page)',
    example: 'item-id-123'
  })
  @IsOptional()
  @IsString()
  cursor?: string;

  @ApiPropertyOptional({
    default: 20,
    minimum: 1,
    maximum: 100,
    description: 'Number of items to return',
    example: 20
  })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;
}
```

**Параметры:**

| Параметр | Тип | Обязательный | По умолчанию | Описание |
|----------|-----|--------------|--------------|----------|
| `cursor` | `string` | Нет | - | ID последнего элемента предыдущей страницы |
| `limit` | `number` | Нет | `20` | Количество элементов (макс. 100) |

**Валидация:**
- `cursor`: строка (ID элемента)
- `limit`: минимум 1, максимум 100, целое число

**Пример использования:**

```typescript
@Get()
async findAll(@Query() pagination: CursorPaginationDto) {
  // pagination.cursor = undefined (если не указан - первая страница)
  // pagination.limit = 20 (если не указан)
}
```

## 3. PaginatedResultDto - Единый формат ответа

**Расположение:** `core/pagination/dto/paginated-result.dto.ts`

**Назначение:** Generic DTO для пагинированных ответов. Используется для Swagger документации.

**Реализация:**

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

/**
 * Generic DTO для пагинированного результата (offset-based)
 * Используется для Swagger документации
 */
export class PaginatedResultDto<T = any> {
  @ApiProperty({
    type: [Object],
    description: 'Array of items',
    isArray: true
  })
  items: T[];

  @ApiProperty({
    example: 100,
    description: 'Total number of items'
  })
  total: number;

  @ApiProperty({
    example: 1,
    description: 'Current page number'
  })
  page: number;

  @ApiProperty({
    example: 10,
    description: 'Items per page'
  })
  limit: number;

  @ApiProperty({
    example: 10,
    description: 'Total number of pages'
  })
  totalPages: number;
}
```

**Поля:**

| Поле | Тип | Описание |
|------|-----|----------|
| `items` | `T[]` | Массив элементов |
| `total` | `number` | Общее количество элементов |
| `page` | `number` | Текущая страница |
| `limit` | `number` | Количество элементов на странице |
| `totalPages` | `number` | Общее количество страниц |

**Пример использования в Swagger:**

```typescript
@Get()
@ApiResponse({
  status: 200,
  description: 'Paginated list of vacancies',
  type: PaginatedResultDto,
  isArray: false
})
async findAll(@Query() pagination: PaginationDto): Promise<PaginatedResult<VacancyDto>> {
  // ...
}
```

## 4. CursorPaginatedResultDto - Cursor-based ответ

**Расположение:** `core/pagination/dto/cursor-paginated-result.dto.ts`

**Назначение:** DTO для cursor-based пагинированных ответов.

**Реализация:**

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

/**
 * Generic DTO для cursor-based пагинированного результата
 * Используется для Swagger документации
 */
export class CursorPaginatedResultDto<T = any> {
  @ApiProperty({
    type: [Object],
    description: 'Array of items',
    isArray: true
  })
  items: T[];

  @ApiPropertyOptional({
    description: 'Cursor for next page (ID of last item)',
    example: 'item-id-123'
  })
  nextCursor?: string;

  @ApiProperty({
    example: true,
    description: 'Whether there are more items'
  })
  hasNext: boolean;

  @ApiPropertyOptional({
    example: 100,
    description: 'Total number of items (optional, may be expensive to calculate)'
  })
  total?: number;
}
```

**Поля:**

| Поле | Тип | Описание |
|------|-----|----------|
| `items` | `T[]` | Массив элементов |
| `nextCursor` | `string?` | Cursor для следующей страницы (ID последнего элемента) |
| `hasNext` | `boolean` | Есть ли ещё элементы |
| `total` | `number?` | Общее количество (опционально, может быть дорого вычислять) |

**Пример использования:**

```typescript
@Get()
@ApiResponse({
  status: 200,
  description: 'Cursor-paginated list of applications',
  type: CursorPaginatedResultDto
})
async findAll(
  @Query() pagination: CursorPaginationDto
): Promise<CursorPaginatedResult<ApplicationDto>> {
  // ...
}
```

## 5. Интерфейсы

**Расположение:** `core/pagination/interfaces/paginated-result.interface.ts`

**Назначение:** TypeScript интерфейсы для типизации.

**Реализация:**

```typescript
/**
 * Интерфейс для offset-based пагинированного результата
 */
export interface PaginatedResult<T> {
  items: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

/**
 * Интерфейс для cursor-based пагинированного результата
 */
export interface CursorPaginatedResult<T> {
  items: T[];
  nextCursor?: string;
  hasNext: boolean;
  total?: number;
}
```

## Расширение DTO в модулях

Модули могут расширять базовые DTO для добавления специфичных параметров:

```typescript
// vacancies/dto/vacancy-query.dto.ts
import { PaginationDto } from '@core/pagination/dto/pagination.dto';
import { IsOptional, IsString, IsArray } from 'class-validator';

export class VacancyQueryDto extends PaginationDto {
  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  skills?: string[];

  @IsOptional()
  @IsString()
  companyId?: string;
}
```

## Best Practices

1. **Использование Generic типов:**
   ```typescript
   Promise<PaginatedResult<VacancyDto>>
   ```

2. **Валидация разрешённых полей для сортировки:**
   ```typescript
   const allowedSortFields = ['createdAt', 'title', 'salary'];
   if (!allowedSortFields.includes(pagination.sortBy)) {
     throw new BadRequestException(`Invalid sort field: ${pagination.sortBy}`);
   }
   ```

3. **Ограничение максимального лимита:**
   - По умолчанию: 100
   - Для тяжёлых запросов: 50 или меньше

4. **Swagger документация:**
   - Всегда используйте `@ApiResponse` с типом DTO
   - Указывайте примеры значений

5. **Обработка пустых результатов:**
   ```typescript
   if (items.length === 0) {
     return {
       items: [],
       total: 0,
       page: pagination.page,
       limit: pagination.limit,
       totalPages: 0
     };
   }
   ```
