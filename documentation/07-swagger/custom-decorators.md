# Custom Decorators - Кастомные декораторы

## Обзор

Кастомные декораторы для упрощения документирования API endpoints и автоматизации повторяющихся паттернов.

## Расположение

```
core/decorators/api/
```

## Реализация

### ApiResponseDocumentation Decorator

```typescript
// core/decorators/api/api-response.documentation.decorator.ts
import { applyDecorators, Type } from '@nestjs/common';
import { ApiResponse, ApiResponseOptions } from '@nestjs/swagger';
import { ApiResponse as ApiResponseInterface, EResultCode } from '../../interfaces/response.interface';

/**
 * Декоратор для документирования стандартизированных ответов API
 * Автоматически добавляет успешный ответ и ошибки
 */
export function ApiResponseDocumentation<T>(
    options: {
        status?: number;
        description?: string;
        type?: Type<T>;
        isArray?: boolean;
        errors?: Array<{ status: number; description: string }>;
    } = {},
) {
    const {
        status = 200,
        description = 'Успешный ответ',
        type,
        isArray = false,
        errors = [
            { status: 400, description: 'Ошибка валидации' },
            { status: 401, description: 'Не авторизован' },
            { status: 403, description: 'Доступ запрещен' },
            { status: 500, description: 'Внутренняя ошибка сервера' },
        ],
    } = options;

    const decorators: Array<ClassDecorator | MethodDecorator | PropertyDecorator> = [];

    // Успешный ответ
    const successResponse: ApiResponseOptions = {
        status,
        description,
        type: type
            ? isArray
                ? undefined // Swagger не поддерживает массив напрямую в type
                : type
            : undefined,
        schema: type && isArray
            ? {
                  type: 'array',
                  items: { $ref: `#/components/schemas/${type.name}` },
              }
            : undefined,
    };

    decorators.push(ApiResponse(successResponse));

    // Ошибки
    errors.forEach((error) => {
        decorators.push(
            ApiResponse({
                status: error.status,
                description: error.description,
                schema: {
                    type: 'object',
                    properties: {
                        resultCode: {
                            type: 'string',
                            enum: [EResultCode.ERROR],
                            example: EResultCode.ERROR,
                        },
                        message: {
                            type: 'string',
                            example: error.description,
                        },
                    },
                },
            }),
        );
    });

    return applyDecorators(...decorators);
}
```

### Использование

```typescript
import { Controller, Get } from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';
import { ApiResponseDocumentation } from '@core/decorators/api/api-response.documentation.decorator';
import { UserDto } from './dto/user.dto';

@ApiTags('Users')
@Controller('users')
export class UsersController {
    @Get()
    @ApiOperation({ summary: 'Получить список пользователей' })
    @ApiResponseDocumentation({
        description: 'Список пользователей',
        type: UserDto,
        isArray: true,
        errors: [
            { status: 401, description: 'Не авторизован' },
            { status: 500, description: 'Внутренняя ошибка сервера' },
        ],
    })
    async getAllUsers() {
        // ...
    }
}
```

## Best Practices

1. **Используйте кастомные декораторы** - для упрощения документирования
2. **Документируйте все ошибки** - для полноты API документации
3. **Переиспользуйте декораторы** - для консистентности
4. **Добавляйте примеры** - для лучшего понимания API
