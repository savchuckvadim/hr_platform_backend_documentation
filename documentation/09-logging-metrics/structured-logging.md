# Structured Logging - Структурированное логирование

## Обзор

Структурированное логирование в JSON формате для HR Platform.

## Реализация

### Custom Logger Service

```typescript
// core/logger/logger.service.ts
import { Injectable, LoggerService } from '@nestjs/common';
import * as dayjs from 'dayjs';

export interface LogContext {
    userId?: string;
    requestId?: string;
    trace?: string;
    method?: string;
    url?: string;
    [key: string]: any;
}

@Injectable()
export class StructuredLogger implements LoggerService {
    private context?: string;

    setContext(context: string) {
        this.context = context;
    }

    log(message: string, context?: LogContext) {
        this.writeLog('info', message, context);
    }

    error(message: string, trace?: string, context?: LogContext) {
        this.writeLog('error', message, { ...context, trace });
    }

    warn(message: string, context?: LogContext) {
        this.writeLog('warn', message, context);
    }

    debug(message: string, context?: LogContext) {
        this.writeLog('debug', message, context);
    }

    verbose(message: string, context?: LogContext) {
        this.writeLog('verbose', message, context);
    }

    private writeLog(level: string, message: string, context?: LogContext) {
        const logEntry = {
            timestamp: new Date().toISOString(),
            level,
            message,
            context: this.context,
            ...context,
        };

        // ✅ Вывод в JSON формате
        console.log(JSON.stringify(logEntry));
    }
}
```

### Использование в сервисах

```typescript
// application/services/auth.service.ts
import { Injectable } from '@nestjs/common';
import { StructuredLogger } from '@core/logger/logger.service';

@Injectable()
export class AuthService {
    constructor(
        private readonly logger: StructuredLogger,
    ) {
        this.logger.setContext('AuthService');
    }

    async login(email: string, password: string) {
        this.logger.log('Login attempt', {
            email,
            // Не логируем пароль!
        });

        try {
            const user = await this.userRepository.findByEmail(email);

            this.logger.log('User found', {
                userId: user.id,
            });

            // ... логика входа

            this.logger.log('Login successful', {
                userId: user.id,
            });

            return { accessToken, refreshToken };
        } catch (error) {
            this.logger.error('Login failed', error.stack, {
                email,
            });
            throw error;
        }
    }
}
```

## Уровни логирования

- **debug** - детальная информация для отладки
- **info** - информационные сообщения
- **warn** - предупреждения
- **error** - ошибки

## Контекстное логирование

### Request ID

```typescript
// core/interceptors/request-id.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestIdInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
        const request = context.switchToHttp().getRequest();
        request.id = uuidv4(); // Добавляем request ID

        return next.handle();
    }
}
```

### User ID

```typescript
// В guards или interceptors
const user = request.user;
if (user) {
    this.logger.log('Action performed', {
        userId: user.id,
        requestId: request.id,
    });
}
```

## Best Practices

1. **Используйте JSON формат** - для структурированного логирования
2. **Добавляйте контекст** - userId, requestId, trace
3. **Не логируйте чувствительные данные** - пароли, токены, персональные данные
4. **Используйте уровни логирования** - debug, info, warn, error
5. **Логируйте важные события** - регистрация, вход, создание откликов
