# Pagination Module - Переиспользуемый модуль пагинации

## Описание

Переиспользуемый модуль пагинации для всех модулей проекта. Поддерживает offset-based (page/limit) и cursor-based пагинацию с единым форматом ответа, валидацией параметров и автоматической генерацией Swagger документации.

## Расположение

```
core/
└── pagination/
    ├── dto/
    │   ├── pagination.dto.ts           # DTO для запросов (offset-based)
    │   ├── cursor-pagination.dto.ts    # DTO для cursor-based запросов
    │   └── paginated-result.dto.ts     # DTO для ответов
    ├── interfaces/
    │   ├── paginated-result.interface.ts
    │   └── pagination-options.interface.ts
    ├── utils/
    │   └── pagination.util.ts          # Утилиты для работы с пагинацией
    ├── decorators/
    │   └── pagination.decorator.ts     # Декоратор для автоматической пагинации
    ├── pagination.module.ts
    └── index.ts
```

## Ключевые концепции

### 1. Offset-based пагинация
- Используется для простых случаев и когда нужна информация о количестве страниц
- Параметры: `page`, `limit`
- Возвращает: `items`, `total`, `page`, `limit`, `totalPages`

### 2. Cursor-based пагинация
- Используется для больших наборов данных и real-time обновлений
- Параметры: `cursor`, `limit`
- Возвращает: `items`, `nextCursor`, `hasNext`, `total` (опционально)

### 3. Сортировка
- Поддержка сортировки по любому полю
- Параметры: `sortBy`, `sortOrder` (ASC/DESC)
- Валидация разрешённых полей для сортировки

### 4. Валидация и ограничения
- Минимальные/максимальные значения для `page` и `limit`
- Максимальный лимит для защиты от перегрузки (по умолчанию 100)
- Валидация через class-validator

## Задачи

### ✅ Pagination DTO
**Статус**: Завершено
**Файл**: [pagination-dto.md](./pagination-dto.md)
**Описание**: Базовые DTO для пагинации (query и response)

### ✅ Pagination Utils
**Статус**: Завершено
**Файл**: [pagination-utils.md](./pagination-utils.md)
**Описание**: Утилиты для работы с пагинацией (skip, totalPages, etc.)

### ✅ Pagination Decorator
**Статус**: Завершено
**Файл**: [pagination-decorator.md](./pagination-decorator.md)
**Описание**: Декоратор для автоматической пагинации в контроллерах

### ✅ Usage Examples
**Статус**: Завершено
**Файл**: [usage-examples.md](./usage-examples.md)
**Описание**: Примеры использования в разных модулях

## Ссылки

- [Pagination DTO](./pagination-dto.md) - DTO для запросов и ответов
- [Pagination Utils](./pagination-utils.md) - Утилиты для работы с пагинацией
- [Pagination Decorator](./pagination-decorator.md) - Декоратор для автоматической пагинации
- [Usage Examples](./usage-examples.md) - Примеры использования в модулях
