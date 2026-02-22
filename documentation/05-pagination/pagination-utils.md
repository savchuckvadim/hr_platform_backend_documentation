# Pagination Utils - Утилиты для пагинации

## Обзор

Утилиты для работы с пагинацией предоставляют вспомогательные функции для вычисления skip, totalPages, валидации параметров и создания пагинированных результатов.

## Расположение

```
core/pagination/utils/
└── pagination.util.ts
```

## PaginationUtil

**Расположение:** `core/pagination/utils/pagination.util.ts`

**Назначение:** Статический класс с утилитами для работы с пагинацией.

### Реализация

```typescript
import { PaginatedResult, CursorPaginatedResult } from '../interfaces/paginated-result.interface';

export class PaginationUtil {
  /**
   * Создаёт пагинированный результат (offset-based)
   * @param items Массив элементов
   * @param total Общее количество элементов
   * @param page Текущая страница
   * @param limit Количество элементов на странице
   * @returns PaginatedResult
   */
  static createPaginatedResult<T>(
    items: T[],
    total: number,
    page: number,
    limit: number,
  ): PaginatedResult<T> {
    return {
      items,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  /**
   * Создаёт cursor-based пагинированный результат
   * @param items Массив элементов
   * @param limit Количество элементов на странице
   * @param total Общее количество (опционально)
   * @returns CursorPaginatedResult
   */
  static createCursorPaginatedResult<T>(
    items: T[],
    limit: number,
    total?: number,
  ): CursorPaginatedResult<T> {
    const hasNext = items.length > limit;
    const actualItems = hasNext ? items.slice(0, limit) : items;
    const nextCursor = hasNext && actualItems.length > 0
      ? this.getCursorFromItem(actualItems[actualItems.length - 1])
      : undefined;

    return {
      items: actualItems,
      nextCursor,
      hasNext,
      total,
    };
  }

  /**
   * Вычисляет skip (offset) для Prisma/TypeORM запросов
   * @param page Номер страницы (начинается с 1)
   * @param limit Количество элементов на странице
   * @returns skip значение
   */
  static getSkip(page: number, limit: number): number {
    return (page - 1) * limit;
  }

  /**
   * Вычисляет totalPages
   * @param total Общее количество элементов
   * @param limit Количество элементов на странице
   * @returns Количество страниц
   */
  static getTotalPages(total: number, limit: number): number {
    return Math.ceil(total / limit);
  }

  /**
   * Валидирует параметры пагинации
   * @param page Номер страницы
   * @param limit Количество элементов
   * @param maxLimit Максимальный лимит
   * @throws BadRequestException если параметры невалидны
   */
  static validatePagination(
    page: number,
    limit: number,
    maxLimit: number = 100,
  ): void {
    if (page < 1) {
      throw new BadRequestException('Page must be greater than 0');
    }
    if (limit < 1) {
      throw new BadRequestException('Limit must be greater than 0');
    }
    if (limit > maxLimit) {
      throw new BadRequestException(`Limit cannot exceed ${maxLimit}`);
    }
  }

  /**
   * Нормализует параметры пагинации (устанавливает значения по умолчанию)
   * @param page Номер страницы
   * @param limit Количество элементов
   * @param defaultLimit Лимит по умолчанию
   * @param maxLimit Максимальный лимит
   * @returns Нормализованные параметры
   */
  static normalizePagination(
    page?: number,
    limit?: number,
    defaultLimit: number = 10,
    maxLimit: number = 100,
  ): { page: number; limit: number } {
    const normalizedPage = page && page > 0 ? page : 1;
    const normalizedLimit = limit && limit > 0
      ? Math.min(limit, maxLimit)
      : defaultLimit;

    return {
      page: normalizedPage,
      limit: normalizedLimit,
    };
  }

  /**
   * Извлекает cursor из элемента (обычно ID)
   * @param item Элемент
   * @returns Cursor (ID элемента)
   */
  static getCursorFromItem(item: any): string {
    if (!item) {
      throw new Error('Item is required to extract cursor');
    }

    // Предполагаем, что у элемента есть поле id
    if (item.id) {
      return String(item.id);
    }

    // Если нет id, пытаемся найти первое поле с типом string или number
    const keys = Object.keys(item);
    for (const key of keys) {
      const value = item[key];
      if (typeof value === 'string' || typeof value === 'number') {
        return String(value);
      }
    }

    throw new Error('Cannot extract cursor from item: no suitable field found');
  }

  /**
   * Валидирует разрешённые поля для сортировки
   * @param sortBy Поле для сортировки
   * @param allowedFields Массив разрешённых полей
   * @throws BadRequestException если поле не разрешено
   */
  static validateSortField(
    sortBy: string,
    allowedFields: string[],
  ): void {
    if (!allowedFields.includes(sortBy)) {
      throw new BadRequestException(
        `Invalid sort field: ${sortBy}. Allowed fields: ${allowedFields.join(', ')}`
      );
    }
  }

  /**
   * Создаёт объект сортировки для Prisma
   * @param sortBy Поле для сортировки
   * @param sortOrder Порядок сортировки
   * @returns Объект сортировки для Prisma
   */
  static createPrismaOrderBy(
    sortBy: string,
    sortOrder: 'asc' | 'desc',
  ): Record<string, 'asc' | 'desc'> {
    return {
      [sortBy]: sortOrder,
    };
  }

  /**
   * Создаёт объект сортировки для TypeORM
   * @param sortBy Поле для сортировки
   * @param sortOrder Порядок сортировки
   * @returns Объект сортировки для TypeORM
   */
  static createTypeOrmOrder(
    sortBy: string,
    sortOrder: 'asc' | 'desc',
  ): Record<string, 'ASC' | 'DESC'> {
    return {
      [sortBy]: sortOrder.toUpperCase() as 'ASC' | 'DESC',
    };
  }
}
```

## Методы

### 1. createPaginatedResult

Создаёт пагинированный результат для offset-based пагинации.

**Параметры:**
- `items: T[]` - массив элементов
- `total: number` - общее количество элементов
- `page: number` - текущая страница
- `limit: number` - количество элементов на странице

**Возвращает:** `PaginatedResult<T>`

**Пример:**

```typescript
const result = PaginationUtil.createPaginatedResult(
  vacancies,
  100,
  1,
  10
);
// {
//   items: vacancies,
//   total: 100,
//   page: 1,
//   limit: 10,
//   totalPages: 10
// }
```

### 2. createCursorPaginatedResult

Создаёт cursor-based пагинированный результат.

**Параметры:**
- `items: T[]` - массив элементов (может быть на 1 больше limit для определения hasNext)
- `limit: number` - количество элементов на странице
- `total?: number` - общее количество (опционально)

**Возвращает:** `CursorPaginatedResult<T>`

**Пример:**

```typescript
// Запрашиваем limit + 1 для определения hasNext
const items = await repository.findMany({
  where: { ... },
  take: limit + 1,
  cursor: cursor ? { id: cursor } : undefined,
});

const result = PaginationUtil.createCursorPaginatedResult(
  items,
  limit
);
// {
//   items: items.slice(0, limit),
//   nextCursor: items[limit - 1]?.id,
//   hasNext: items.length > limit
// }
```

### 3. getSkip

Вычисляет skip (offset) для запросов к БД.

**Параметры:**
- `page: number` - номер страницы (начинается с 1)
- `limit: number` - количество элементов

**Возвращает:** `number`

**Пример:**

```typescript
const skip = PaginationUtil.getSkip(2, 10); // 10
// Для страницы 2 с лимитом 10 нужно пропустить 10 элементов
```

### 4. getTotalPages

Вычисляет общее количество страниц.

**Параметры:**
- `total: number` - общее количество элементов
- `limit: number` - количество элементов на странице

**Возвращает:** `number`

**Пример:**

```typescript
const totalPages = PaginationUtil.getTotalPages(100, 10); // 10
const totalPages = PaginationUtil.getTotalPages(95, 10); // 10 (округление вверх)
```

### 5. validatePagination

Валидирует параметры пагинации.

**Параметры:**
- `page: number` - номер страницы
- `limit: number` - количество элементов
- `maxLimit: number` - максимальный лимит (по умолчанию 100)

**Выбрасывает:** `BadRequestException` если параметры невалидны

**Пример:**

```typescript
PaginationUtil.validatePagination(1, 10); // OK
PaginationUtil.validatePagination(0, 10); // BadRequestException
PaginationUtil.validatePagination(1, 150); // BadRequestException (если maxLimit = 100)
```

### 6. normalizePagination

Нормализует параметры пагинации (устанавливает значения по умолчанию и ограничивает максимум).

**Параметры:**
- `page?: number` - номер страницы
- `limit?: number` - количество элементов
- `defaultLimit: number` - лимит по умолчанию (по умолчанию 10)
- `maxLimit: number` - максимальный лимит (по умолчанию 100)

**Возвращает:** `{ page: number; limit: number }`

**Пример:**

```typescript
const { page, limit } = PaginationUtil.normalizePagination(undefined, undefined);
// { page: 1, limit: 10 }

const { page, limit } = PaginationUtil.normalizePagination(2, 50);
// { page: 2, limit: 50 }

const { page, limit } = PaginationUtil.normalizePagination(1, 150);
// { page: 1, limit: 100 } (ограничено maxLimit)
```

### 7. getCursorFromItem

Извлекает cursor (обычно ID) из элемента.

**Параметры:**
- `item: any` - элемент

**Возвращает:** `string` - cursor (ID элемента)

**Выбрасывает:** `Error` если не удаётся извлечь cursor

**Пример:**

```typescript
const item = { id: '123', name: 'Vacancy' };
const cursor = PaginationUtil.getCursorFromItem(item); // '123'
```

### 8. validateSortField

Валидирует разрешённые поля для сортировки.

**Параметры:**
- `sortBy: string` - поле для сортировки
- `allowedFields: string[]` - массив разрешённых полей

**Выбрасывает:** `BadRequestException` если поле не разрешено

**Пример:**

```typescript
const allowedFields = ['createdAt', 'title', 'salary'];
PaginationUtil.validateSortField('createdAt', allowedFields); // OK
PaginationUtil.validateSortField('invalidField', allowedFields); // BadRequestException
```

### 9. createPrismaOrderBy

Создаёт объект сортировки для Prisma.

**Параметры:**
- `sortBy: string` - поле для сортировки
- `sortOrder: 'asc' | 'desc'` - порядок сортировки

**Возвращает:** `Record<string, 'asc' | 'desc'>`

**Пример:**

```typescript
const orderBy = PaginationUtil.createPrismaOrderBy('createdAt', 'desc');
// { createdAt: 'desc' }

await prisma.vacancy.findMany({
  orderBy,
  // ...
});
```

### 10. createTypeOrmOrder

Создаёт объект сортировки для TypeORM.

**Параметры:**
- `sortBy: string` - поле для сортировки
- `sortOrder: 'asc' | 'desc'` - порядок сортировки

**Возвращает:** `Record<string, 'ASC' | 'DESC'>`

**Пример:**

```typescript
const order = PaginationUtil.createTypeOrmOrder('createdAt', 'desc');
// { createdAt: 'DESC' }

await repository.find({
  order,
  // ...
});
```

## Примеры использования

### Offset-based пагинация (Prisma)

```typescript
async findAll(pagination: PaginationDto): Promise<PaginatedResult<VacancyDto>> {
  const { page, limit, sortBy, sortOrder } = pagination;

  // Валидация
  PaginationUtil.validateSortField(sortBy, ['createdAt', 'title', 'salary']);

  // Вычисление skip
  const skip = PaginationUtil.getSkip(page, limit);

  // Запрос к БД
  const [items, total] = await Promise.all([
    this.prisma.vacancy.findMany({
      skip,
      take: limit,
      orderBy: PaginationUtil.createPrismaOrderBy(sortBy, sortOrder),
    }),
    this.prisma.vacancy.count(),
  ]);

  // Создание результата
  return PaginationUtil.createPaginatedResult(items, total, page, limit);
}
```

### Cursor-based пагинация (Prisma)

```typescript
async findAll(
  pagination: CursorPaginationDto
): Promise<CursorPaginatedResult<ApplicationDto>> {
  const { cursor, limit } = pagination;

  // Запрашиваем limit + 1 для определения hasNext
  const items = await this.prisma.application.findMany({
    take: limit + 1,
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: { createdAt: 'desc' },
  });

  // Создание результата
  return PaginationUtil.createCursorPaginatedResult(items, limit);
}
```

### С расширенными фильтрами

```typescript
async search(
  query: VacancyQueryDto
): Promise<PaginatedResult<VacancyDto>> {
  const { page, limit, sortBy, sortOrder, search, skills } = query;

  // Нормализация
  const { page: normalizedPage, limit: normalizedLimit } =
    PaginationUtil.normalizePagination(page, limit, 10, 50);

  // Валидация сортировки
  PaginationUtil.validateSortField(sortBy, ['createdAt', 'title', 'salary']);

  // Построение where
  const where = {
    ...(search && { title: { contains: search, mode: 'insensitive' } }),
    ...(skills && { skills: { some: { name: { in: skills } } } }),
  };

  // Запрос
  const skip = PaginationUtil.getSkip(normalizedPage, normalizedLimit);
  const [items, total] = await Promise.all([
    this.prisma.vacancy.findMany({
      where,
      skip,
      take: normalizedLimit,
      orderBy: PaginationUtil.createPrismaOrderBy(sortBy, sortOrder),
    }),
    this.prisma.vacancy.count({ where }),
  ]);

  return PaginationUtil.createPaginatedResult(
    items,
    total,
    normalizedPage,
    normalizedLimit
  );
}
```

## Best Practices

1. **Всегда валидируйте sortBy:**
   ```typescript
   PaginationUtil.validateSortField(sortBy, allowedFields);
   ```

2. **Используйте normalizePagination для установки значений по умолчанию:**
   ```typescript
   const { page, limit } = PaginationUtil.normalizePagination(page, limit);
   ```

3. **Для cursor-based пагинации запрашивайте limit + 1:**
   ```typescript
   const items = await repository.findMany({ take: limit + 1 });
   ```

4. **Используйте Promise.all для параллельных запросов:**
   ```typescript
   const [items, total] = await Promise.all([...]);
   ```

5. **Ограничивайте maxLimit для тяжёлых запросов:**
   ```typescript
   PaginationUtil.normalizePagination(page, limit, 10, 50); // max 50
   ```
