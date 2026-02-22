# Response Documentation - Документирование ответов

## Обзор

Документирование всех возможных ответов API (успешных и ошибок) через Swagger декораторы.

## Стандартизированный формат ответа

Все ответы API следуют стандартизированному формату:

```typescript
interface ApiResponse<T> {
    resultCode: 'SUCCESS' | 'ERROR';
    data?: T;
    message?: string;
    errors?: ValidationError[];
}
```

## Документирование успешных ответов

### Простой ответ

```typescript
import { ApiResponse } from '@nestjs/swagger';
import { UserDto } from './dto/user.dto';

@Get(':id')
@ApiResponse({
    status: 200,
    description: 'Пользователь найден',
    type: UserDto,
})
async getUser(@Param('id') id: string) {
    // ...
}
```

### Массив данных

```typescript
@Get()
@ApiResponse({
    status: 200,
    description: 'Список пользователей',
    type: UserDto,
    isArray: true,
})
async getAllUsers() {
    // ...
}
```

### Пагинированный ответ

```typescript
@Get()
@ApiResponse({
    status: 200,
    description: 'Пагинированный список пользователей',
    schema: {
        type: 'object',
        properties: {
            resultCode: { type: 'string', example: 'SUCCESS' },
            data: {
                type: 'object',
                properties: {
                    items: {
                        type: 'array',
                        items: { $ref: '#/components/schemas/UserDto' },
                    },
                    total: { type: 'number', example: 100 },
                    page: { type: 'number', example: 1 },
                    limit: { type: 'number', example: 20 },
                    totalPages: { type: 'number', example: 5 },
                },
            },
        },
    },
})
async getUsers(@Query() pagination: PaginationDto) {
    // ...
}
```

## Документирование ошибок

### Стандартные ошибки

```typescript
import { ApiResponse } from '@nestjs/swagger';

@Get(':id')
@ApiResponse({ status: 200, description: 'Успешный ответ', type: UserDto })
@ApiResponse({ status: 400, description: 'Ошибка валидации' })
@ApiResponse({ status: 401, description: 'Не авторизован' })
@ApiResponse({ status: 403, description: 'Доступ запрещен' })
@ApiResponse({ status: 404, description: 'Пользователь не найден' })
@ApiResponse({ status: 500, description: 'Внутренняя ошибка сервера' })
async getUser(@Param('id') id: string) {
    // ...
}
```

### Детальное описание ошибок

```typescript
@ApiResponse({
    status: 400,
    description: 'Ошибка валидации',
    schema: {
        type: 'object',
        properties: {
            resultCode: {
                type: 'string',
                enum: ['ERROR'],
                example: 'ERROR',
            },
            message: {
                type: 'string',
                example: 'Validation failed',
            },
            errors: {
                type: 'array',
                items: {
                    type: 'object',
                    properties: {
                        field: { type: 'string', example: 'email' },
                        message: { type: 'string', example: 'Email must be a valid email' },
                    },
                },
            },
        },
    },
})
```

## Использование кастомного декоратора

```typescript
import { ApiResponseDocumentation } from '@core/decorators/api/api-response.documentation.decorator';

@Get(':id')
@ApiOperation({ summary: 'Получить пользователя по ID' })
@ApiResponseDocumentation({
    description: 'Пользователь найден',
    type: UserDto,
    errors: [
        { status: 401, description: 'Не авторизован' },
        { status: 404, description: 'Пользователь не найден' },
    ],
})
async getUser(@Param('id') id: string) {
    // ...
}
```

## Best Practices

1. **Документируйте все возможные ответы** - успешные и ошибки
2. **Используйте стандартизированный формат** - для консистентности
3. **Добавляйте примеры** - для лучшего понимания
4. **Используйте кастомные декораторы** - для упрощения
5. **Группируйте ошибки** - по типам (валидация, авторизация, бизнес-логика)
