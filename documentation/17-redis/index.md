# Core Redis - Redis сервис и модуль

## Описание

Глобальный Redis сервис и модуль для кэширования, очередей, presence tracking и других операций с Redis. Обеспечивает единую точку доступа к Redis для всех модулей приложения.

**Ключевые особенности:**
- Глобальный модуль (`@Global()`) - доступен во всех модулях без явного импорта
- Единое подключение к Redis через ioredis
- Автоматическое управление жизненным циклом (подключение/отключение)
- Retry стратегия для автоматического переподключения
- Интеграция с ConfigService для централизованного управления конфигурацией
- Логирование событий подключения/ошибок/отключения

## Расположение

```
core/redis/
├── redis.service.ts    # RedisService - глобальный сервис
├── redis.module.ts     # RedisModule - глобальный модуль
├── redis.config.ts     # Конфигурация Redis
└── index.ts            # Экспорты модуля
```

## Настройка и подключение

### 1. Установка зависимостей

```bash
npm install ioredis
npm install @types/ioredis
```

### 2. Конфигурация

**Переменные окружения (`.env`):**
```env
# Вариант 1: Использование URL (приоритетный)
REDIS_URL=redis://localhost:6379

# Вариант 2: Использование host и port
REDIS_HOST=localhost
REDIS_PORT=6379

# Опциональные параметры
REDIS_CONNECT_TIMEOUT=10000
REDIS_MAX_RETRIES=3
```

**Приоритет конфигурации:**
- Если указан `REDIS_URL` - используется он
- Если `REDIS_URL` не указан - используются `REDIS_HOST` и `REDIS_PORT`
- Дефолтные значения: `host: 'redis'`, `port: 6379`

### 3. Реализация Redis Config

```typescript
// core/redis/redis.config.ts
import { ConfigService } from '@nestjs/config';

export function createRedisOptions(
    configService: ConfigService,
): {
    url: string | undefined;
    host: string | undefined;
    port: number;
    maxRetriesPerRequest: number;
    connectTimeout: number;
} {
    const url = configService.get<string>('REDIS_URL');
    const host = configService.get<string>('REDIS_HOST') ?? 'redis';
    const port = parseInt(
        configService.get<string>('REDIS_PORT') ?? '6379',
        10,
    );

    return {
        url,
        host,
        port,
        maxRetriesPerRequest: 3,
        connectTimeout: 10000,
    };
}
```

**Особенности:**
- Использует `ConfigService` для получения конфигурации
- Поддерживает как `REDIS_URL`, так и `REDIS_HOST`/`REDIS_PORT`
- Дефолтные значения для `host` и `port`
- Фиксированные значения для `maxRetriesPerRequest` и `connectTimeout`

### 4. Реализация RedisService

```typescript
// core/redis/redis.service.ts
import { Injectable, OnModuleDestroy, Logger } from '@nestjs/common';
import Redis from 'ioredis';
import { ConfigService } from '@nestjs/config';
import { createRedisOptions } from './redis.config';

@Injectable()
export class RedisService implements OnModuleDestroy {
    private readonly logger = new Logger(RedisService.name);
    private client: Redis;

    constructor(private readonly configService: ConfigService) {
        this.logger.log('Создание Redis клиента...');

        const {
            url: redisUrl,
            host,
            port,
            connectTimeout,
            maxRetriesPerRequest
        } = createRedisOptions(this.configService);

        this.logger.log(`Получены настройки Redis из конфига:`);
        this.logger.log(`REDIS_HOST: ${host}`);
        this.logger.log(`REDIS_PORT: ${port}`);

        if (redisUrl) {
            this.logger.log(`Используем REDIS_URL`);

            this.client = new Redis(redisUrl, {
                retryStrategy: times => {
                    const delay = Math.min(times * 50, 2000);
                    this.logger.log(`Повторная попытка через ${delay}ms`);
                    return delay;
                },
                maxRetriesPerRequest,
                connectTimeout,
            });
        } else {
            this.client = new Redis({
                host,
                port,
                retryStrategy: times => {
                    const delay = Math.min(times * 50, 2000);
                    this.logger.log(
                        `Повторная попытка подключения к Redis через ${delay}ms...`,
                    );
                    return delay;
                },
                maxRetriesPerRequest,
                connectTimeout,
            });
        }

        // Обработка событий подключения
        this.client.on('connect', () => {
            this.logger.log('Redis подключён ✅');
        });

        // Обработка ошибок
        this.client.on('error', err => {
            this.logger.error('Redis ошибка ❌: ' + err.message);
        });

        // Обработка отключения
        this.client.on('end', () => {
            this.logger.warn('Redis отключён 🛑');
        });
    }

    /**
     * Получить Redis клиент для прямого использования
     * @returns Redis клиент
     */
    getClient(): Redis {
        this.logger.debug('Redis клиент запрошен через getClient()');
        return this.client;
    }

    /**
     * Закрытие соединения при остановке модуля
     */
    async onModuleDestroy() {
        this.logger.warn('Redis клиент закрывается через onModuleDestroy...');
        await this.client.quit();
    }
}
```

**Особенности реализации:**
- Использует `ConfigService` для получения конфигурации
- Поддерживает подключение через `REDIS_URL` или `REDIS_HOST`/`REDIS_PORT`
- Retry стратегия: экспоненциальная задержка до 2000ms
- Логирование всех событий (connect, error, end)
- Автоматическое закрытие соединения при остановке приложения
- Метод `getClient()` возвращает экземпляр Redis клиента для прямого использования

**Retry стратегия:**
- При каждой неудачной попытке задержка увеличивается: `times * 50ms`
- Максимальная задержка: `2000ms`
- Пример: 1-я попытка - 50ms, 2-я - 100ms, 3-я - 150ms, ... до 2000ms

### 5. Реализация RedisModule

```typescript
// core/redis/redis.module.ts
import { Global, Module } from '@nestjs/common';
import { RedisService } from './redis.service';

@Global()
@Module({
    providers: [RedisService],
    exports: [RedisService],
})
export class RedisModule {}
```

**Особенности:**
- Декоратор `@Global()` делает модуль глобальным - доступен во всех модулях без явного импорта
- Экспортирует `RedisService` для использования в других модулях
- Должен быть импортирован в корневом модуле приложения (`AppModule`)

### 6. Регистрация в AppModule

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { RedisModule } from '@core/redis/redis.module';
import { ConfigModule } from '@nestjs/config';

@Module({
    imports: [
        ConfigModule.forRoot({
            isGlobal: true, // ConfigModule также должен быть глобальным
        }),
        RedisModule, // ✅ Импортируем глобальный RedisModule
        // ... другие модули
    ],
})
export class AppModule {}
```

**Важно:**
- `ConfigModule` должен быть импортирован **до** `RedisModule`, так как `RedisService` использует `ConfigService`
- `ConfigModule` должен быть глобальным (`isGlobal: true`) или явно импортирован в `RedisModule`

### 7. Экспорты модуля

```typescript
// core/redis/index.ts
export * from './redis.module';
export * from './redis.service';
export * from './redis.config';
```

## Использование в модулях

### В модулях (явный импорт опционален)

Хотя `RedisModule` является глобальным и не требует явного импорта, рекомендуется явно импортировать его для ясности:

```typescript
// modules/presence/presence.module.ts
import { Module } from '@nestjs/common';
import { RedisModule } from '@core/redis/redis.module';
import { PresenceService } from './application/services/presence.service';

@Module({
    imports: [RedisModule], // ✅ Явный импорт (опционально, но рекомендуется)
    providers: [PresenceService],
    exports: [PresenceService],
})
export class PresenceModule {}
```

### В сервисах

**Базовое использование:**
```typescript
// application/services/presence.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '@core/redis/redis.service';

@Injectable()
export class PresenceService {
    constructor(private readonly redis: RedisService) {}

    async markOnline(userId: string, ttl: number = 60): Promise<void> {
        const client = this.redis.getClient();
        await client.set(`presence:user:${userId}`, '1', 'EX', ttl);
    }

    async markOffline(userId: string): Promise<void> {
        const client = this.redis.getClient();
        await client.del(`presence:user:${userId}`);
    }

    async isOnline(userId: string): Promise<boolean> {
        const client = this.redis.getClient();
        const result = await client.exists(`presence:user:${userId}`);
        return result === 1;
    }
}
```

**Использование для кэширования:**
```typescript
// application/services/user.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '@core/redis/redis.service';

@Injectable()
export class UserService {
    constructor(private readonly redis: RedisService) {}

    async getUserFromCache(userId: string): Promise<any | null> {
        const client = this.redis.getClient();
        const cached = await client.get(`user:${userId}`);

        if (cached) {
            return JSON.parse(cached);
        }

        return null;
    }

    async setUserCache(userId: string, userData: any, ttl: number = 3600): Promise<void> {
        const client = this.redis.getClient();
        await client.setex(
            `user:${userId}`,
            ttl,
            JSON.stringify(userData),
        );
    }

    async invalidateUserCache(userId: string): Promise<void> {
        const client = this.redis.getClient();
        await client.del(`user:${userId}`);
    }
}
```

**Использование для очередей (BullMQ):**
```typescript
// infrastructure/processors/email.processor.ts
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { RedisService } from '@core/redis/redis.service';

@Processor('email', {
    connection: {
        // Используем Redis клиент из RedisService
        // BullMQ автоматически использует Redis для очередей
    },
})
export class EmailProcessor extends WorkerHost {
    constructor(private readonly redis: RedisService) {
        super();
    }

    async process(job: Job) {
        // Обработка задачи
    }
}
```

**Использование для WebSocket adapter (Socket.IO):**
```typescript
// core/websocket/websocket.module.ts
import { Module } from '@nestjs/common';
import { RedisService } from '@core/redis/redis.service';
import { createAdapter } from '@socket.io/redis-adapter';

@Module({
    imports: [RedisModule],
    providers: [
        {
            provide: 'REDIS_ADAPTER',
            useFactory: (redis: RedisService) => {
                const pubClient = redis.getClient();
                const subClient = pubClient.duplicate();
                return createAdapter(pubClient, subClient);
            },
            inject: [RedisService],
        },
    ],
})
export class WebSocketModule {}
```

**Использование для OTP хранения:**
```typescript
// application/services/phone-auth.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '@core/redis/redis.service';

@Injectable()
export class PhoneAuthService {
    constructor(private readonly redis: RedisService) {}

    async storeOtp(phone: string, code: string, ttl: number = 300): Promise<void> {
        const client = this.redis.getClient();
        await client.setex(`otp:${phone}`, ttl, code);
    }

    async verifyOtp(phone: string, code: string): Promise<boolean> {
        const client = this.redis.getClient();
        const storedCode = await client.get(`otp:${phone}`);

        if (!storedCode) {
            return false;
        }

        if (storedCode === code) {
            await client.del(`otp:${phone}`);
            return true;
        }

        return false;
    }
}
```

## Основные операции Redis

### String операции

```typescript
const client = this.redis.getClient();

// Установка значения
await client.set('key', 'value');

// Установка с TTL (в секундах)
await client.setex('key', 60, 'value');

// Получение значения
const value = await client.get('key');

// Удаление ключа
await client.del('key');

// Проверка существования
const exists = await client.exists('key');

// Установка TTL для существующего ключа
await client.expire('key', 60);
```

### Hash операции

```typescript
const client = this.redis.getClient();

// Установка поля в hash
await client.hset('user:1', 'name', 'John');
await client.hset('user:1', 'email', 'john@example.com');

// Получение всех полей hash
const user = await client.hgetall('user:1');

// Получение конкретного поля
const name = await client.hget('user:1', 'name');

// Удаление поля
await client.hdel('user:1', 'email');
```

### List операции

```typescript
const client = this.redis.getClient();

// Добавление в начало списка
await client.lpush('list:key', 'item1', 'item2');

// Добавление в конец списка
await client.rpush('list:key', 'item3');

// Получение элементов списка
const items = await client.lrange('list:key', 0, -1);

// Удаление элемента
await client.lrem('list:key', 1, 'item1');
```

### Set операции

```typescript
const client = this.redis.getClient();

// Добавление в set
await client.sadd('set:key', 'member1', 'member2');

// Проверка членства
const isMember = await client.sismember('set:key', 'member1');

// Получение всех членов
const members = await client.smembers('set:key');

// Удаление из set
await client.srem('set:key', 'member1');
```

### Sorted Set операции

```typescript
const client = this.redis.getClient();

// Добавление с score
await client.zadd('leaderboard', 100, 'user1', 200, 'user2');

// Получение по рангу
const topUsers = await client.zrange('leaderboard', 0, 9, 'WITHSCORES');

// Получение по score
const users = await client.zrangebyscore('leaderboard', 100, 200);
```

### Pub/Sub операции

```typescript
const client = this.redis.getClient();

// Подписка на канал
const subscriber = this.redis.getClient().duplicate();
await subscriber.subscribe('channel:name');

subscriber.on('message', (channel, message) => {
    console.log(`Received message: ${message} from channel: ${channel}`);
});

// Публикация сообщения
await client.publish('channel:name', 'Hello, World!');
```

## Преимущества

1. **Глобальный модуль** - доступен во всех модулях без явного импорта
2. **Единое подключение** - один экземпляр Redis клиента для всего приложения
3. **Автоматическое управление** - подключение/отключение при старте/остановке
4. **Retry стратегия** - автоматическое переподключение при сбоях
5. **Централизованная конфигурация** - использование ConfigService
6. **Логирование** - отслеживание всех событий подключения/ошибок
7. **Гибкость** - поддержка как URL, так и host/port конфигурации

## Best Practices

1. **Используйте через сервисы** - не напрямую в контроллерах
   ```typescript
   // ✅ Хорошо
   @Injectable()
   export class CacheService {
       constructor(private readonly redis: RedisService) {}
   }

   // ❌ Плохо
   @Controller()
   export class SomeController {
       constructor(private readonly redis: RedisService) {} // Не рекомендуется
   }
   ```

2. **Обрабатывайте ошибки** - при работе с Redis
   ```typescript
   try {
       await client.set('key', 'value');
   } catch (error) {
       this.logger.error('Redis error', error);
       throw new InternalServerErrorException('Cache error');
   }
   ```

3. **Используйте TTL** - для автоматической очистки данных
   ```typescript
   // ✅ Хорошо - данные автоматически удалятся через 1 час
   await client.setex('key', 3600, 'value');

   // ❌ Плохо - данные будут храниться бесконечно
   await client.set('key', 'value');
   ```

4. **Используйте префиксы для ключей** - для организации данных
   ```typescript
   // ✅ Хорошо
   await client.set('user:123:profile', '...');
   await client.set('user:123:settings', '...');

   // ❌ Плохо
   await client.set('123', '...'); // Непонятно, что это за данные
   ```

5. **Используйте транзакции** - для атомарных операций
   ```typescript
   const multi = client.multi();
   multi.set('key1', 'value1');
   multi.set('key2', 'value2');
   await multi.exec();
   ```

6. **Мониторьте использование памяти** - Redis хранит данные в памяти
   ```typescript
   // Проверка использования памяти
   const info = await client.info('memory');
   ```

7. **Используйте pipeline** - для множественных операций
   ```typescript
   const pipeline = client.pipeline();
   pipeline.set('key1', 'value1');
   pipeline.set('key2', 'value2');
   pipeline.set('key3', 'value3');
   await pipeline.exec();
   ```

## Структура использования в проекте

```
core/redis/                    # Глобальный Redis модуль
├── redis.service.ts
├── redis.module.ts
├── redis.config.ts
└── index.ts

modules/presence/              # Модуль присутствия
├── application/
│   └── services/
│       └── presence.service.ts    # Использует RedisService
└── presence.module.ts

modules/queue/                 # Модуль очередей (BullMQ)
├── infrastructure/
│   └── processors/
│       └── email.processor.ts    # Использует Redis для очередей
└── queue.module.ts

core/websocket/                # WebSocket модуль
├── websocket.gateway.ts          # Использует Redis adapter
└── websocket.module.ts
```

## Примечания

- `RedisService` является глобальным и доступен во всех модулях
- Все операции с Redis должны выполняться через `RedisService.getClient()`
- Retry стратегия автоматически переподключается при сбоях
- Логирование всех событий помогает отслеживать состояние подключения
- Используйте TTL для всех временных данных (OTP, кэш, presence)
- Для production рекомендуется использовать Redis Cluster или Sentinel для высокой доступности
