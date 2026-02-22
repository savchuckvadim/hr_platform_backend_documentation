# Integration - Интеграция в проект

## Обзор

Интеграция системы логирования и метрик в HR Platform через interceptors, middleware и сервисы.

## Установка зависимостей

```bash
npm install @willsoto/nestjs-prometheus prom-client
npm install --save-dev @types/node
```

## Metrics Module

### Структура

```
core/metrics/
├── metrics.module.ts
├── metrics.service.ts
└── consts/
    └── metrics.constants.ts
```

### Metrics Module

```typescript
// core/metrics/metrics.module.ts
import { Global, Module } from '@nestjs/common';
import {
    PrometheusModule,
    makeCounterProvider,
    makeHistogramProvider,
    makeGaugeProvider,
} from '@willsoto/nestjs-prometheus';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { MetricsInterceptor } from '../interceptors/metrics.interceptor';

@Global()
@Module({
    imports: [
        PrometheusModule.register({
            defaultMetrics: {
                enabled: true, // Стандартные метрики Node.js
            },
            path: '/api/metrics', // Endpoint для Prometheus
        }),
    ],
    providers: [
        MetricsInterceptor,
        {
            provide: APP_INTERCEPTOR,
            useClass: MetricsInterceptor,
        },
        // HTTP метрики
        makeCounterProvider({
            name: 'http_requests_total',
            help: 'Total number of HTTP requests',
            labelNames: ['method', 'url', 'status'],
        }),
        makeCounterProvider({
            name: 'http_requests_errors_total',
            help: 'Total number of failed HTTP requests',
            labelNames: ['method', 'url', 'status'],
        }),
        makeHistogramProvider({
            name: 'http_request_duration_seconds',
            help: 'Duration of HTTP requests in seconds',
            labelNames: ['method', 'url'],
            buckets: [0.05, 0.1, 0.3, 0.5, 1, 2, 5, 10],
        }),
        // Бизнес-метрики
        makeCounterProvider({
            name: 'users_registered_total',
            help: 'Total number of registered users',
            labelNames: ['role_type'],
        }),
        makeCounterProvider({
            name: 'applications_created_total',
            help: 'Total number of applications created',
            labelNames: ['status'],
        }),
        makeCounterProvider({
            name: 'emails_sent_total',
            help: 'Total number of emails sent',
            labelNames: ['email_type', 'status'],
        }),
        makeCounterProvider({
            name: 'queue_jobs_total',
            help: 'Total number of queue jobs',
            labelNames: ['queue_name', 'job_name', 'status'],
        }),
        makeHistogramProvider({
            name: 'queue_job_duration_seconds',
            help: 'Duration of queue jobs in seconds',
            labelNames: ['queue_name', 'job_name'],
            buckets: [0.1, 0.5, 1, 2, 5, 10, 30, 60],
        }),
        makeGaugeProvider({
            name: 'queue_jobs_active',
            help: 'Number of active queue jobs',
            labelNames: ['queue_name'],
        }),
    ],
    exports: [PrometheusModule],
})
export class MetricsModule {}
```

## Metrics Interceptor

```typescript
// core/interceptors/metrics.interceptor.ts
import {
    CallHandler,
    ExecutionContext,
    Injectable,
    NestInterceptor,
} from '@nestjs/common';
import { Counter, Histogram } from 'prom-client';
import { InjectMetric } from '@willsoto/nestjs-prometheus';
import { tap } from 'rxjs/operators';
import { Observable } from 'rxjs';

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
    constructor(
        @InjectMetric('http_requests_total')
        private readonly httpRequestsTotal: Counter<string>,
        @InjectMetric('http_requests_errors_total')
        private readonly httpRequestsErrorsTotal: Counter<string>,
        @InjectMetric('http_request_duration_seconds')
        private readonly httpRequestDuration: Histogram<string>,
    ) {}

    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        const req = context.switchToHttp().getRequest();
        const method = req.method;
        const url = req.url;
        const start = Date.now();

        return next.handle().pipe(
            tap({
                next: () => {
                    const duration = (Date.now() - start) / 1000;
                    const status = context.switchToHttp().getResponse().statusCode;

                    // ✅ Метрика количества запросов
                    this.httpRequestsTotal
                        .labels(method, url, status.toString())
                        .inc();

                    // ✅ Метрика длительности запросов
                    this.httpRequestDuration
                        .labels(method, url)
                        .observe(duration);

                    // ✅ Метрика ошибок (4xx, 5xx)
                    if (status >= 400) {
                        this.httpRequestsErrorsTotal
                            .labels(method, url, status.toString())
                            .inc();
                    }
                },
                error: (error) => {
                    const duration = (Date.now() - start) / 1000;
                    const status = error.status || 500;

                    // ✅ Метрика длительности даже при ошибке
                    this.httpRequestDuration
                        .labels(method, url)
                        .observe(duration);

                    // ✅ Метрика ошибок
                    this.httpRequestsErrorsTotal
                        .labels(method, url, status.toString())
                        .inc();
                },
            }),
        );
    }
}
```

## Response Interceptor

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
    implements NestInterceptor<T, ApiResponse<T>>
{
    intercept(
        context: ExecutionContext,
        next: CallHandler,
    ): Observable<ApiResponse<T>> {
        const req = context.switchToHttp().getRequest();

        // ✅ Пропускаем без обертки, если это /api/metrics
        if (req.url === '/api/metrics') {
            return next.handle();
        }

        return next.handle().pipe(
            map((data) => {
                return {
                    resultCode: EResultCode.SUCCESS,
                    data: data,
                };
            }),
        );
    }
}
```

## Logger Middleware

```typescript
// core/middleware/logger.middleware.ts
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import * as dayjs from 'dayjs';
import { TelegramService } from '@telegram/application/services/telegram.service';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
    private readonly logger = new Logger(LoggerMiddleware.name);

    constructor(
        private readonly telegram: TelegramService,
    ) {}

    async use(req: Request, res: Response, next: NextFunction) {
        const timeStart = dayjs();

        // ✅ Структурированное логирование входящего запроса
        const logData = {
            timestamp: new Date().toISOString(),
            level: 'info',
            message: 'Incoming request',
            method: req.method,
            url: req.originalUrl,
            query: Object.keys(req.query).length > 0 ? req.query : undefined,
            body: req.body && Object.keys(req.body).length > 0 ? req.body : undefined,
            ip: req.ip,
            userAgent: req.get('user-agent'),
        };

        this.logger.log(JSON.stringify(logData));

        // ✅ Отправка в Telegram (опционально, для критических запросов)
        if (this.shouldLogToTelegram(req)) {
            let message = `📥 Входящий запрос: ${req.method}\n🧭 URL: ${req.originalUrl}\n`;

            if (Object.keys(req.query).length > 0) {
                message += `🔎 Query: ${JSON.stringify(req.query, null, 2)}\n`;
            }

            if (req.body && Object.keys(req.body).length > 0) {
                message += `📦 Body: ${JSON.stringify(req.body, null, 2)}\n`;
            }

            await this.telegram.sendMessage(message).catch((err) => {
                this.logger.error('Failed to send Telegram notification', err);
            });
        }

        res.on('finish', async () => {
            const duration = dayjs().diff(timeStart, 'ms');

            // ✅ Структурированное логирование ответа
            const responseLogData = {
                timestamp: new Date().toISOString(),
                level: res.statusCode >= 400 ? 'error' : 'info',
                message: 'Request completed',
                method: req.method,
                url: req.originalUrl,
                statusCode: res.statusCode,
                duration: `${duration}ms`,
            };

            this.logger.log(JSON.stringify(responseLogData));

            // ✅ Отправка в Telegram (опционально)
            if (this.shouldLogToTelegram(req)) {
                await this.telegram
                    .sendMessage(
                        `✅ Ответ: ${res.statusCode} за ${duration}мс\n🧭 ${req.method} ${req.originalUrl}`,
                    )
                    .catch((err) => {
                        this.logger.error('Failed to send Telegram notification', err);
                    });
            }
        });

        next();
    }

    private shouldLogToTelegram(req: Request): boolean {
        // ✅ Логируем в Telegram только критичные запросы
        const criticalPaths = ['/auth/login', '/auth/register', '/auth/forgot-password'];
        return criticalPaths.some((path) => req.originalUrl.includes(path));
    }
}
```

## Регистрация в App Module

```typescript
// app.module.ts
import { Module, MiddlewareConsumer, NestModule } from '@nestjs/common';
import { APP_INTERCEPTOR, APP_FILTER } from '@nestjs/core';
import { MetricsModule } from './core/metrics/metrics.module';
import { MetricsInterceptor } from './core/interceptors/metrics.interceptor';
import { ResponseInterceptor } from './core/interceptors/response.interceptor';
import { LoggerMiddleware } from './core/middleware/logger.middleware';
import { GlobalExceptionFilter } from './core/filters/global-exception.filter';

@Module({
    imports: [
        MetricsModule,
        // ... другие модули
    ],
    providers: [
        {
            provide: APP_INTERCEPTOR,
            useClass: MetricsInterceptor,
        },
        {
            provide: APP_INTERCEPTOR,
            useClass: ResponseInterceptor,
        },
        {
            provide: APP_FILTER,
            useClass: GlobalExceptionFilter,
        },
    ],
})
export class AppModule implements NestModule {
    configure(consumer: MiddlewareConsumer) {
        consumer.apply(LoggerMiddleware).forRoutes('*');
    }
}
```

## Endpoint для метрик

```typescript
// api/controllers/metrics.controller.ts
import { Controller, Get } from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';
import { Public } from '../decorators/public.decorator';
import { Registry, collectDefaultMetrics } from 'prom-client';

@ApiTags('Metrics')
@Controller('metrics')
export class MetricsController {
    private readonly registry: Registry;

    constructor() {
        this.registry = new Registry();
        collectDefaultMetrics({ register: this.registry });
    }

    @Get()
    @Public()
    @ApiOperation({ summary: 'Prometheus metrics endpoint' })
    async getMetrics(): Promise<string> {
        return this.registry.metrics();
    }
}
```

## Best Practices

1. **Глобальные interceptors** - через APP_INTERCEPTOR
2. **Структурированное логирование** - JSON формат
3. **Не логируйте чувствительные данные** - пароли, токены
4. **Условное логирование в Telegram** - только критичные запросы
5. **Обработка ошибок** - в middleware и interceptors
