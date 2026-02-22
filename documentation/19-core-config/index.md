# Core Config - Конфигурации

## Описание

Централизованные конфигурации для различных компонентов приложения: Swagger, CORS, логирование, почта.

## Расположение

```
core/config/
├── swagger/
│   └── swagger.config.ts
├── cors/
│   └── cors.config.ts
├── logs/
│   └── logger.ts
└── mail/
    └── mailer.config.ts
```

## Swagger Config

См. [Swagger Setup](../07-swagger/swagger-setup.md)

## CORS Config

Конфигурация CORS для разрешения запросов с определенных доменов.

```typescript
// core/config/cors/cors.config.ts
const domains = (process.env.CORS_ORIGIN ?? '')
    .split(',')
    .map((origin) => origin.trim());

export const cors = {
    origin: domains,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
    credentials: true,
    allowedHeaders: ['Content-Type', 'Authorization'],
};
```

**Использование:**

```typescript
// main.ts
import { cors } from './core/config/cors/cors.config';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.enableCors(cors);
    await app.listen(3000);
}
```

**Environment Variables:**

```env
CORS_ORIGIN=http://localhost:3000,https://hr-platform.com
```

## Logger Config

Конфигурация логирования.

```typescript
// core/config/logs/logger.ts
import { Logger } from '@nestjs/common';

export const createLogger = () => {
    return new Logger();
};
```

## Mail Config

Конфигурация почтового сервера.

```typescript
// core/config/mail/mailer.config.ts
import { MailerOptions } from '@nestjs-modules/mailer';

export const getMailerConfig = (): MailerOptions => {
    return {
        transport: {
            host: process.env.MAIL_HOST,
            port: parseInt(process.env.MAIL_PORT || '587'),
            secure: process.env.MAIL_SECURE === 'true',
            auth: {
                user: process.env.MAIL_USER,
                pass: process.env.MAIL_PASSWORD,
            },
        },
        defaults: {
            from: process.env.MAIL_FROM,
        },
    };
};
```

**Environment Variables:**

```env
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_SECURE=false
MAIL_USER=your-email@gmail.com
MAIL_PASSWORD=your-password
MAIL_FROM=noreply@hr-platform.com
```

## Best Practices

1. **Централизуйте конфигурации** - в core/config
2. **Используйте environment variables** - для секретов
3. **Валидируйте конфигурации** - при старте приложения
4. **Документируйте переменные** - в .env.example
