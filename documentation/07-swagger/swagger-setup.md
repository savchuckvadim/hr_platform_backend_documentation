# Swagger Setup - Настройка Swagger

## Обзор

Настройка Swagger/OpenAPI для автоматической генерации API документации в NestJS приложении.

## Расположение

```
core/config/swagger/swagger.config.ts
```

## Реализация

### Swagger Config

```typescript
import { INestApplication } from '@nestjs/common';
import {
    DocumentBuilder,
    SwaggerDocumentOptions,
    SwaggerModule,
} from '@nestjs/swagger';

export const getSwaggerConfig = (app: INestApplication) => {
    const config = new DocumentBuilder()
        .setTitle('HR Platform API')
        .setDescription('API для платформы поиска работы')
        .setVersion('1.0')
        .addTag('auth', 'Аутентификация и авторизация')
        .addTag('users', 'Управление пользователями')
        .addTag('candidates', 'Кандидаты')
        .addTag('companies', 'Компании')
        .addTag('vacancies', 'Вакансии')
        .addTag('resumes', 'Резюме')
        .addTag('applications', 'Отклики')
        .addTag('chat', 'Чат и сообщения')
        .addTag('presence', 'Присутствие пользователей')
        .addBearerAuth(
            {
                type: 'http',
                scheme: 'bearer',
                bearerFormat: 'JWT',
                name: 'JWT',
                description: 'Enter JWT token',
                in: 'header',
            },
            'JWT-auth', // This name here is important for matching up with @ApiBearerAuth() in your controller!
        )
        .build();

    const options: SwaggerDocumentOptions = {
        operationIdFactory: (controllerKey: string, methodKey: string) => {
            const cleanController = controllerKey.replace(/Controller$/i, '');
            return `${cleanController}_${methodKey}`;
        },
    };

    const documentFactory = () =>
        SwaggerModule.createDocument(app, config, options);

    SwaggerModule.setup('docs/api', app, documentFactory);
};
```

### Регистрация в main.ts

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { getSwaggerConfig } from './core/config/swagger/swagger.config';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);

    // Настройка Swagger
    getSwaggerConfig(app);

    await app.listen(3000);
}
bootstrap();
```

## Доступ к документации

После запуска сервера документация доступна по адресу:

- **Swagger UI**: `http://localhost:3000/docs/api`
- **Production**: `https://api.hr-platform.com/docs/api`

### JSON Schema

OpenAPI JSON схема доступна по адресу:
- `http://localhost:3000/docs/api-json`

Этот endpoint используется для кодогенерации клиентского кода (например, через Orval).

## Конфигурация

### Environment Variables

```env
# Swagger настройки (опционально)
SWAGGER_ENABLED=true
SWAGGER_PATH=docs/api
```

### Условная активация

```typescript
export const getSwaggerConfig = (app: INestApplication) => {
    // Включаем Swagger только в development
    if (process.env.NODE_ENV !== 'production') {
        const config = new DocumentBuilder()
            .setTitle('HR Platform API')
            // ...
            .build();

        SwaggerModule.setup('docs/api', app, documentFactory);
    }
};
```

## Best Practices

1. **Используйте теги** - для группировки endpoints по функциональности
2. **Добавляйте Bearer Auth** - для документирования JWT аутентификации
3. **Настраивайте operationId** - для генерации клиентского кода
4. **Отключайте в production** - для безопасности
5. **Версионируйте API** - через setVersion()
