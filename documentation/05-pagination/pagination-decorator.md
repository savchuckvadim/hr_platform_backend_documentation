# Pagination Decorator - Декоратор для автоматической пагинации

## Обзор

Декоратор для автоматической обработки параметров пагинации в контроллерах. Упрощает работу с пагинацией, автоматически извлекая и валидируя параметры из query string.

## Расположение

```
core/pagination/decorators/
└── pagination.decorator.ts
```

## @Pagination() Decorator

**Расположение:** `core/pagination/decorators/pagination.decorator.ts`

**Назначение:** Декоратор для автоматического извлечения и валидации параметров пагинации из query string.

### Реализация

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { PaginationDto } from '../dto/pagination.dto';
import { CursorPaginationDto } from '../dto/cursor-pagination.dto';
import { PaginationUtil } from '../utils/pagination.util';

/**
 * Декоратор для извлечения параметров offset-based пагинации
 * Автоматически нормализует и валидирует параметры
 *
 * @param defaultLimit Лимит по умолчанию (по умолчанию 10)
 * @param maxLimit Максимальный лимит (по умолчанию 100)
 * @returns Нормализованные параметры пагинации
 *
 * @example
 * @Get()
 * async findAll(@Pagination() pagination: PaginationDto) {
 *   // pagination.page, pagination.limit уже нормализованы
 * }
 */
export const Pagination = createParamDecorator(
  (data: { defaultLimit?: number; maxLimit?: number } = {}, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const query = request.query;

    const defaultLimit = data.defaultLimit ?? 10;
    const maxLimit = data.maxLimit ?? 100;

    const page = query.page ? parseInt(query.page, 10) : undefined;
    const limit = query.limit ? parseInt(query.limit, 10) : undefined;
    const sortBy = query.sortBy || 'createdAt';
    const sortOrder = query.sortOrder || 'desc';

    const { page: normalizedPage, limit: normalizedLimit } =
      PaginationUtil.normalizePagination(page, limit, defaultLimit, maxLimit);

    return {
      page: normalizedPage,
      limit: normalizedLimit,
      sortBy,
      sortOrder,
    } as PaginationDto;
  },
);

/**
 * Декоратор для извлечения параметров cursor-based пагинации
 *
 * @param defaultLimit Лимит по умолчанию (по умолчанию 20)
 * @param maxLimit Максимальный лимит (по умолчанию 100)
 * @returns Нормализованные параметры cursor пагинации
 *
 * @example
 * @Get()
 * async findAll(@CursorPagination() pagination: CursorPaginationDto) {
 *   // pagination.cursor, pagination.limit уже нормализованы
 * }
 */
export const CursorPagination = createParamDecorator(
  (data: { defaultLimit?: number; maxLimit?: number } = {}, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const query = request.query;

    const defaultLimit = data.defaultLimit ?? 20;
    const maxLimit = data.maxLimit ?? 100;

    const cursor = query.cursor;
    const limit = query.limit ? parseInt(query.limit, 10) : undefined;

    const normalizedLimit = limit && limit > 0
      ? Math.min(limit, maxLimit)
      : defaultLimit;

    return {
      cursor,
      limit: normalizedLimit,
    } as CursorPaginationDto;
  },
);
```

## Использование

### Offset-based пагинация

**Базовое использование:**

```typescript
import { Controller, Get } from '@nestjs/common';
import { Pagination } from '@core/pagination/decorators/pagination.decorator';
import { PaginationDto } from '@core/pagination/dto/pagination.dto';
import { PaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';

@Controller('vacancies')
export class VacancyController {
  @Get()
  async findAll(
    @Pagination() pagination: PaginationDto
  ): Promise<PaginatedResult<VacancyDto>> {
    return this.vacancyService.findAll(pagination);
  }
}
```

**С кастомными лимитами:**

```typescript
@Get()
async findAll(
  @Pagination({ defaultLimit: 20, maxLimit: 50 }) pagination: PaginationDto
): Promise<PaginatedResult<VacancyDto>> {
  return this.vacancyService.findAll(pagination);
}
```

**С расширенными параметрами:**

```typescript
import { Query } from '@nestjs/common';

@Get()
async search(
  @Pagination() pagination: PaginationDto,
  @Query('search') search?: string,
  @Query('companyId') companyId?: string,
): Promise<PaginatedResult<VacancyDto>> {
  return this.vacancyService.search({
    ...pagination,
    search,
    companyId,
  });
}
```

### Cursor-based пагинация

```typescript
import { Controller, Get } from '@nestjs/common';
import { CursorPagination } from '@core/pagination/decorators/pagination.decorator';
import { CursorPaginationDto } from '@core/pagination/dto/cursor-pagination.dto';
import { CursorPaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';

@Controller('applications')
export class ApplicationController {
  @Get()
  async findAll(
    @CursorPagination() pagination: CursorPaginationDto
  ): Promise<CursorPaginatedResult<ApplicationDto>> {
    return this.applicationService.findAll(pagination);
  }
}
```

**С кастомными лимитами:**

```typescript
@Get()
async findAll(
  @CursorPagination({ defaultLimit: 30, maxLimit: 100 }) pagination: CursorPaginationDto
): Promise<CursorPaginatedResult<ApplicationDto>> {
  return this.applicationService.findAll(pagination);
}
```

## Комбинирование с другими декораторами

### С @Query() для расширенных параметров

```typescript
import { Query } from '@nestjs/common';
import { IsOptional, IsString } from 'class-validator';

class VacancyQueryDto extends PaginationDto {
  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsString()
  companyId?: string;
}

@Get()
async search(
  @Query() query: VacancyQueryDto
): Promise<PaginatedResult<VacancyDto>> {
  // query уже содержит page, limit, sortBy, sortOrder из PaginationDto
  // плюс search и companyId
  return this.vacancyService.search(query);
}
```

### С @CurrentUser()

```typescript
import { UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '@auth/infrastructure/guards/jwt-auth.guard';
import { CurrentUser } from '@auth/infrastructure/decorators/current-user.decorator';

@Get()
@UseGuards(JwtAuthGuard)
async findMyVacancies(
  @CurrentUser() user: User,
  @Pagination() pagination: PaginationDto
): Promise<PaginatedResult<VacancyDto>> {
  return this.vacancyService.findByUserId(user.id, pagination);
}
```

## Swagger документация

Декоратор автоматически работает с Swagger через DTO:

```typescript
import { ApiOperation, ApiResponse } from '@nestjs/swagger';
import { PaginatedResultDto } from '@core/pagination/dto/paginated-result.dto';

@Get()
@ApiOperation({ summary: 'Get paginated list of vacancies' })
@ApiResponse({
  status: 200,
  description: 'Paginated list of vacancies',
  type: PaginatedResultDto,
})
async findAll(
  @Pagination() pagination: PaginationDto
): Promise<PaginatedResult<VacancyDto>> {
  return this.vacancyService.findAll(pagination);
}
```

## Примеры запросов

### Offset-based

```
GET /vacancies?page=1&limit=10&sortBy=createdAt&sortOrder=desc
GET /vacancies?page=2&limit=20
GET /vacancies (используются значения по умолчанию: page=1, limit=10)
```

### Cursor-based

```
GET /applications?cursor=app-id-123&limit=20
GET /applications?limit=30
GET /applications (используются значения по умолчанию: limit=20)
```

## Преимущества использования декоратора

1. **Автоматическая нормализация:**
   - Устанавливает значения по умолчанию
   - Ограничивает максимальные значения
   - Преобразует строки в числа

2. **Единообразие:**
   - Все контроллеры используют одинаковую логику
   - Легко поддерживать и изменять

3. **Валидация:**
   - Автоматическая валидация параметров
   - Защита от некорректных значений

4. **Чистый код:**
   - Меньше boilerplate кода в контроллерах
   - Фокус на бизнес-логике

## Best Practices

1. **Используйте декоратор для простых случаев:**
   ```typescript
   @Pagination() pagination: PaginationDto
   ```

2. **Расширяйте DTO для сложных случаев:**
   ```typescript
   class VacancyQueryDto extends PaginationDto {
     search?: string;
     companyId?: string;
   }
   ```

3. **Устанавливайте разумные лимиты:**
   ```typescript
   @Pagination({ defaultLimit: 20, maxLimit: 50 })
   ```

4. **Всегда документируйте в Swagger:**
   ```typescript
   @ApiResponse({ type: PaginatedResultDto })
   ```

5. **Используйте cursor-based для больших наборов данных:**
   ```typescript
   @CursorPagination() pagination: CursorPaginationDto
   ```
