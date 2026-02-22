# Response Filters, Interceptors & Middleware - Обработка запросов и ответов

## Описание

Глобальные interceptors, filters и middleware для обработки запросов, стандартизации ответов, сбора метрик и обработки ошибок. Обеспечивают единообразный формат ответов API и централизованную обработку исключений.

**Ключевые особенности:**
- Глобальные interceptors для стандартизации ответов и сбора метрик
- Глобальный exception filter для обработки всех ошибок
- Middleware для логирования запросов
- Интеграция с Prometheus для сбора метрик (см. [Логирование и метрики](./09-logging-metrics/index.md))
- Стандартизированный формат ответов через `ApiResponse<T>`
- Обработка валидационных ошибок
- Логирование и уведомления об ошибках

## Расположение

```
core/interceptors/
├── response.interceptor.ts      # Стандартизация ответов
├── metrics.interceptor.ts        # Сбор метрик (см. 09-logging-metrics)
└── auth-cookie.interceptor.ts    # Установка auth cookies

core/filters/
├── global-exception.filter.ts    # Глобальная обработка ошибок
└── index.ts

core/middleware/
└── logger.middleware.ts         # Логирование запросов
```

## Middleware

### Logger Middleware

Логирует все входящие HTTP запросы и ответы, отправляя информацию в Telegram для мониторинга.

**Реализация:**
```typescript
// core/middleware/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { TelegramService } from '@core/telegram/telegram.service';
import dayjs from 'dayjs';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
    constructor(private readonly telegram: TelegramService) {}

    async use(req: Request, res: Response, next: NextFunction) {
        const timeStart = dayjs();
        let message = `📥 Входящий запрос: ${req.method}\n🧭 URL: ${req.originalUrl}\n`;

        if (Object.keys(req.query).length > 0) {
            message += `🔎 Query: ${JSON.stringify(req.query, null, 2)}\n`;
        }

        if (req.body && Object.keys(req.body).length > 0) {
            message += `📦 Body: ${JSON.stringify(req.body, null, 2)}\n`;
        }

        await this.telegram.sendMessage(message);

        res.on('finish', async () => {
            const duration = dayjs().diff(timeStart, 'ms');
            await this.telegram.sendMessage(
                `✅ Ответ: ${res.statusCode} за ${duration}мс\n🧭 ${req.method} ${req.originalUrl}`,
            );
        });

        next();
    }
}
```

**Особенности:**
- Логирует метод, URL, query параметры и body запроса
- Отслеживает время выполнения запроса
- Отправляет уведомления в Telegram при получении запроса и завершении ответа
- Использует `dayjs` для измерения времени выполнения

**Регистрация:**
```typescript
// app.module.ts
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { LoggerMiddleware } from '@core/middleware/logger.middleware';
import { TelegramModule } from '@core/telegram/telegram.module';

@Module({
    imports: [TelegramModule],
})
export class AppModule implements NestModule {
    configure(consumer: MiddlewareConsumer) {
        consumer
            .apply(LoggerMiddleware)
            .forRoutes('*'); // Применяется ко всем маршрутам
    }
}
```

**Использование для конкретных маршрутов:**
```typescript
configure(consumer: MiddlewareConsumer) {
    consumer
        .apply(LoggerMiddleware)
        .forRoutes(
            { path: 'auth/*', method: RequestMethod.ALL },
            { path: 'users/*', method: RequestMethod.ALL },
        );
}
```

**См. также:** [Core Telegram Module](./20-core-telegram/index.md) - для отправки уведомлений

## Interceptors

### Response Interceptor

Стандартизирует формат всех успешных ответов API, оборачивая данные в структуру `ApiResponse<T>`.

**Реализация:**
```typescript
// core/interceptors/response.interceptor.ts
import {
    Injectable,
    NestInterceptor,
    ExecutionContext,
    CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { ApiResponse, EResultCode } from '../interfaces/response.interface';

@Injectable()
export class ResponseInterceptor<T>
    implements NestInterceptor<T, ApiResponse<T>> {
    intercept(
        context: ExecutionContext,
        next: CallHandler,
    ): Observable<ApiResponse<T>> {
        const req = context.switchToHttp().getRequest();

        // Пропускаем без обертки, если это /metrics
        if (req.url === '/api/metrics') {
            return next.handle();
        }

        return next.handle().pipe(
            map(data => {
                return {
                    resultCode: EResultCode.SUCCESS,
                    data: data,
                };
            }),
        );
    }
}
```

**Формат ответа:**
```typescript
interface ApiResponse<T> {
    resultCode: EResultCode.SUCCESS;
    data: T;
}
```

**Пример ответа:**
```json
{
  "resultCode": 0,
  "data": {
    "id": "123",
    "email": "user@example.com"
  }
}
```

**Регистрация:**
```typescript
// app.module.ts
import { APP_INTERCEPTOR } from '@nestjs/core';
import { ResponseInterceptor } from '@core/interceptors/response.interceptor';

@Module({
    providers: [
        {
            provide: APP_INTERCEPTOR,
            useClass: ResponseInterceptor,
        },
    ],
})
export class AppModule {}
```

### Metrics Interceptor

Собирает метрики HTTP запросов для Prometheus. Детальное описание см. в разделе [Логирование и метрики](./09-logging-metrics/index.md).

**Реализация:**
```typescript
// core/interceptors/metrics.interceptor.ts
import {
    CallHandler,
    ExecutionContext,
    Injectable,
    NestInterceptor,
} from '@nestjs/common';
import { Counter } from 'prom-client';
import { InjectMetric } from '@willsoto/nestjs-prometheus';
import { tap } from 'rxjs/operators';

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
    constructor(
        @InjectMetric('http_requests_total')
        private readonly counter: Counter<string>,
    ) {}

    intercept(context: ExecutionContext, next: CallHandler) {
        const req = context.switchToHttp().getRequest();

        return next.handle().pipe(
            tap(() => {
                const status = context.switchToHttp().getResponse().statusCode;
                this.counter
                    .labels(req.method, req.url, status.toString())
                    .inc();
            }),
        );
    }
}
```

**Особенности:**
- Использует `@InjectMetric` для инъекции Prometheus метрик
- Собирает метрики: метод, URL, статус код
- Интегрирован с PrometheusModule (см. [09-logging-metrics](./09-logging-metrics/integration.md))

**Регистрация:**
```typescript
// app.module.ts
import { APP_INTERCEPTOR } from '@nestjs/core';
import { MetricsInterceptor } from '@core/interceptors/metrics.interceptor';

@Module({
    providers: [
        {
            provide: APP_INTERCEPTOR,
            useClass: MetricsInterceptor,
        },
    ],
})
export class AppModule {}
```

**См. также:** [Логирование и метрики - Metrics Interceptor](./09-logging-metrics/integration.md#metrics-interceptor)

### Auth Cookie Interceptor

Устанавливает HTTP-only cookies для access и refresh токенов при аутентификации.

**Реализация:**
```typescript
// core/interceptors/auth-cookie.interceptor.ts
import {
    CallHandler,
    ExecutionContext,
    Injectable,
    NestInterceptor
} from '@nestjs/common';
import { Observable, tap } from 'rxjs';
import { Response } from 'express';
import { CookieService } from '@/core/cookie/cookie.service';

/**
 * Interceptor for setting auth cookie
 * for example: login, activate, refresh token, etc.
 */
@Injectable()
export class AuthCookieInterceptor implements NestInterceptor {
    constructor(private cookieService: CookieService) {}

    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        const ctx = context.switchToHttp();
        const res = ctx.getResponse<Response>();

        return next.handle().pipe(
            tap((data) => {
                if (data?.tokens?.accessToken) {
                    this.cookieService.setAccessToken(
                        res,
                        data.tokens.accessToken,
                    );
                }

                if (data?.tokens?.refreshToken) {
                    this.cookieService.setRefreshToken(
                        res,
                        data.tokens.refreshToken,
                    );
                }

                // Удаляем tokens из ответа (они уже в cookies)
                if (data?.tokens) {
                    delete data.tokens;
                }
            }),
        );
    }
}
```

**Особенности:**
- Использует `CookieService` для установки cookies
- Удаляет токены из тела ответа после установки cookies
- Применяется только к endpoints аутентификации (login, refresh, activate)

**Использование:**
```typescript
// auth.controller.ts
import { UseInterceptors } from '@nestjs/common';
import { AuthCookieInterceptor } from '@core/interceptors/auth-cookie.interceptor';

@Controller('auth')
export class AuthController {
    @Post('login')
    @UseInterceptors(AuthCookieInterceptor)
    async login(@Body() dto: LoginDto) {
        // Возвращает { tokens: { accessToken, refreshToken } }
        // Interceptor устанавливает cookies и удаляет tokens из ответа
        return this.authService.login(dto);
    }
}
```

**См. также:** [Authentication Module - Token Management](./04-authentication/token-management.md)

## Filters

### Global Exception Filter

Глобальный фильтр для обработки всех исключений приложения. Стандартизирует формат ошибок, логирует их и отправляет уведомления.

**Реализация:**
```typescript
// core/filters/global-exception.filter.ts
import {
    ExceptionFilter,
    Catch,
    ArgumentsHost,
    HttpException,
    HttpStatus,
    BadRequestException,
    Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';
import { TelegramService } from '@core/telegram/telegram.service';
import * as path from 'path';
import { ApiResponse, EResultCode } from '../interfaces/response.interface';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
    private readonly logger = new Logger(GlobalExceptionFilter.name);

    constructor(private readonly telegram: TelegramService) {}

    async catch(exception: unknown, host: ArgumentsHost) {
        const ctx = host.switchToHttp();
        const request = ctx.getRequest<Request>();
        const response = ctx.getResponse<Response>();

        const status =
            exception instanceof HttpException
                ? exception.getStatus()
                : HttpStatus.INTERNAL_SERVER_ERROR;

        const error =
            exception instanceof Error
                ? exception
                : new Error(JSON.stringify(exception));

        // Обработка валидационных ошибок
        if (
            exception instanceof BadRequestException &&
            typeof exception.getResponse === 'function'
        ) {
            return await this.handleValidationException(
                exception,
                request,
                response,
            );
        }

        // Разбор stack trace для детальной информации
        let file = '';
        let line = '';
        let func = '';
        let code = '';
        try {
            const stackLines = error.stack?.split('\n') || [];
            const target = stackLines.find(
                l => l.includes('/src/') || l.includes('src\\'),
            );
            if (target) {
                const match = target.match(/\((.*):(\d+):(\d+)\)/);
                if (match) {
                    const [, filepath, lineno] = match;
                    file = path.relative(process.cwd(), filepath);
                    line = lineno;
                }
            }

            func = stackLines[1]?.trim().split(' ')[1] || 'unknown';
            code = stackLines[1] || '';
        } catch (e) {
            console.warn('Stack trace parse failed', e);
        }

        // Сбор контекста запроса
        const ip =
            request.headers['x-forwarded-for'] || request.socket.remoteAddress;
        const userAgent = request.headers['user-agent'] || 'unknown';
        const referer = request.headers['referer'] || 'n/a';

        // Формирование детального сообщения об ошибке
        const message = `⚠️ Ошибка: ${error.name}\n\n📄 Файл: ${file}\n🔢 Строка: ${line}\n🔧 Функция: ${func}\n\n💥 Код: ${code}\n\n📬 Сообщение: ${error.message}\n\n📍 URL: ${request.method} ${request.url}\n🧭 User-Agent: ${userAgent}\n🌍 IP: ${ip}\n🔗 Referer: ${referer}`;

        // Отправка уведомления в Telegram
        await this.telegram.sendMessage(message);
        this.logger.error(message);

        // Стандартизированный ответ
        const responseBody: ApiResponse<null> = {
            resultCode: EResultCode.ERROR,
            message: error.message,
        };

        response.status(status).json(responseBody);
    }

    private async handleValidationException(
        exception: BadRequestException,
        request: Request,
        response: Response,
    ) {
        const res = exception.getResponse();
        const messageArray =
            typeof res === 'object' && res !== null && 'message' in res
                ? (res as any).message
                : [];

        const validationMessages = Array.isArray(messageArray)
            ? messageArray.join('\n- ')
            : String(messageArray);

        const fullMessage = `❌ Validation error:\n- ${validationMessages}\n\n📍 URL: ${request.method} ${request.url}`;
        this.logger.warn(fullMessage);
        await this.telegram.sendMessage(fullMessage);

        return response.status(400).json({
            resultCode: EResultCode.ERROR,
            message: 'Validation failed',
            errors: messageArray,
        });
    }
}
```

**Особенности:**
- Обрабатывает все исключения через `@Catch()`
- Специальная обработка валидационных ошибок (`BadRequestException`)
- Разбор stack trace для детальной информации
- Сбор контекста запроса (IP, User-Agent, Referer)
- Отправка уведомлений в Telegram
- Стандартизированный формат ошибок

**Формат ответа при ошибке:**
```json
{
  "resultCode": 1,
  "message": "Error message"
}
```

**Формат ответа при валидационной ошибке:**
```json
{
  "resultCode": 1,
  "message": "Validation failed",
  "errors": [
    "email must be an email",
    "password must be longer than or equal to 8 characters"
  ]
}
```

**Регистрация:**
```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { GlobalExceptionFilter } from './core/filters/global-exception.filter';
import { TelegramService } from '@core/telegram/telegram.service';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);

    // Получаем TelegramService из контейнера
    const telegramService = app.get(TelegramService);

    app.useGlobalFilters(new GlobalExceptionFilter(telegramService));

    await app.listen(3000);
}
bootstrap();
```

## Интерфейсы

### ApiResponse

Стандартизированный интерфейс для всех ответов API.

```typescript
// core/interfaces/response.interface.ts
export enum EResultCode {
    SUCCESS = 0,
    ERROR = 1,
}

export interface ApiResponse<T> {
    resultCode: EResultCode;
    data?: T;
    message?: string;
    errors?: string[];
}
```

**Успешный ответ:**
```typescript
{
    resultCode: EResultCode.SUCCESS,
    data: { /* данные */ }
}
```

**Ошибка:**
```typescript
{
    resultCode: EResultCode.ERROR,
    message: "Error message"
}
```

**Валидационная ошибка:**
```typescript
{
    resultCode: EResultCode.ERROR,
    message: "Validation failed",
    errors: ["field1 error", "field2 error"]
}
```

## Регистрация в AppModule

Полный пример регистрации всех interceptors, filters и middleware:

```typescript
// app.module.ts
import { Module, MiddlewareConsumer, NestModule } from '@nestjs/common';
import { APP_INTERCEPTOR, APP_FILTER } from '@nestjs/core';
import { ResponseInterceptor } from '@core/interceptors/response.interceptor';
import { MetricsInterceptor } from '@core/interceptors/metrics.interceptor';
import { GlobalExceptionFilter } from '@core/filters/global-exception.filter';
import { LoggerMiddleware } from '@core/middleware/logger.middleware';
import { TelegramModule } from '@core/telegram/telegram.module';

@Module({
    imports: [
        TelegramModule, // Для GlobalExceptionFilter и LoggerMiddleware (см. 20-core-telegram)
        // ... другие модули
    ],
    providers: [
        // Global Interceptors
        {
            provide: APP_INTERCEPTOR,
            useClass: ResponseInterceptor,
        },
        {
            provide: APP_INTERCEPTOR,
            useClass: MetricsInterceptor,
        },
        // Global Filters
        {
            provide: APP_FILTER,
            useClass: GlobalExceptionFilter,
        },
    ],
})
export class AppModule implements NestModule {
    configure(consumer: MiddlewareConsumer) {
        consumer
            .apply(LoggerMiddleware)
            .forRoutes('*'); // Применяется ко всем маршрутам
    }
}
```

**Порядок выполнения:**
1. **Request** → Middleware (LoggerMiddleware)
2. **Request** → Interceptors (до обработчика)
3. **Handler** → Обработка запроса
4. **Response** → Interceptors (после обработчика)
5. **Exception** → Filters (если произошла ошибка)

## Best Practices

### Middleware

1. **Используйте для логирования** - запросов и ответов
2. **Измеряйте время выполнения** - для мониторинга производительности
3. **Применяйте выборочно** - не ко всем маршрутам, если не нужно
4. **Обрабатывайте ошибки** - в middleware

### Interceptors

1. **Используйте глобальные interceptors** - через `APP_INTERCEPTOR`
   ```typescript
   {
       provide: APP_INTERCEPTOR,
       useClass: ResponseInterceptor,
   }
   ```

2. **Не блокируйте поток** - используйте RxJS операторы правильно
   ```typescript
   // ✅ Хорошо
   return next.handle().pipe(map(data => transform(data)));

   // ❌ Плохо
   const data = await next.handle().toPromise();
   return transform(data);
   ```

3. **Пропускайте специальные endpoints** - например, `/metrics`
   ```typescript
   if (req.url === '/api/metrics') {
       return next.handle();
   }
   ```

4. **Используйте типизацию** - для типобезопасности
   ```typescript
   implements NestInterceptor<T, ApiResponse<T>>
   ```

### Filters

1. **Обрабатывайте все исключения** - через `@Catch()`
   ```typescript
   @Catch()
   export class GlobalExceptionFilter implements ExceptionFilter {}
   ```

2. **Стандартизируйте формат ошибок** - для консистентности
   ```typescript
   {
       resultCode: EResultCode.ERROR,
       message: error.message,
   }
   ```

3. **Логируйте ошибки** - для отладки
   ```typescript
   this.logger.error(message, error.stack);
   ```

4. **Различайте типы ошибок** - валидация, авторизация, бизнес-логика
   ```typescript
   if (exception instanceof BadRequestException) {
       return this.handleValidationException(...);
   }
   ```

5. **Не раскрывайте внутренние детали** - в production
   ```typescript
   // В production не показывайте stack trace
   const message = process.env.NODE_ENV === 'production'
       ? 'Internal server error'
       : error.message;
   ```

6. **Собирайте контекст** - для лучшей диагностики
   ```typescript
   const ip = request.headers['x-forwarded-for'];
   const userAgent = request.headers['user-agent'];
   ```

## Связи с другими модулями

- **[Логирование и метрики](./09-logging-metrics/index.md)** - детальное описание MetricsInterceptor и интеграции с Prometheus
- **[Authentication Module](./04-authentication/index.md)** - использование AuthCookieInterceptor для установки токенов
- **[Swagger документация](./07-swagger/index.md)** - документирование формата ответов ApiResponse
- **[Core Telegram Module](./20-core-telegram/index.md)** - отправка уведомлений об ошибках и запросах (используется в GlobalExceptionFilter и LoggerMiddleware)

## Структура использования в проекте

```
core/interceptors/              # Глобальные interceptors
├── response.interceptor.ts      # Стандартизация ответов
├── metrics.interceptor.ts       # Сбор метрик (см. 09-logging-metrics)
└── auth-cookie.interceptor.ts  # Установка auth cookies

core/filters/                   # Глобальные filters
├── global-exception.filter.ts     # Обработка всех ошибок
└── index.ts

core/middleware/                # Middleware
└── logger.middleware.ts        # Логирование запросов

core/interfaces/                 # Общие интерфейсы
└── response.interface.ts       # ApiResponse, EResultCode

app.module.ts                   # Регистрация через APP_INTERCEPTOR/APP_FILTER
main.ts                         # Регистрация GlobalExceptionFilter
```

## Примечания

- Все interceptors и filters являются глобальными и применяются ко всем запросам
- Middleware применяется в порядке регистрации через `configure()`
- Порядок регистрации interceptors важен - они выполняются в порядке регистрации
- `ResponseInterceptor` должен быть зарегистрирован после `MetricsInterceptor`, чтобы метрики собирались до обертки ответа
- `GlobalExceptionFilter` должен быть зарегистрирован последним, чтобы обрабатывать все ошибки
- Для локального применения используйте `@UseInterceptors()`, `@UseFilters()` и `apply()` на уровне контроллера или метода
