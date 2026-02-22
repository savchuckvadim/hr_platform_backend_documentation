# Multi-node Support - Горизонтальное масштабирование

## Обзор

Поддержка горизонтального масштабирования WebSocket через Redis adapter для Socket.IO. Обеспечивает синхронизацию событий между несколькими инстансами приложения.

## Проблема без Redis Adapter

**❌ Проблема:**

При наличии нескольких инстансов приложения:
- События отправляются только на том инстансе, где подключен клиент
- Другие инстансы не получают события
- Presence статусы не синхронизируются
- Уведомления не доставляются, если пользователь подключен к другому инстансу

**Пример:**
```
Instance 1: User A подключен
Instance 2: User B отправляет сообщение User A

❌ User A не получит сообщение, т.к. оно отправлено на Instance 2,
   а User A подключен к Instance 1
```

## Решение: Redis Adapter

Redis adapter синхронизирует события между всеми инстансами через Redis pub/sub.

## Установка

```bash
npm install @socket.io/redis-adapter
npm install ioredis
```

## Реализация

### Redis IO Adapter

```typescript
// infrastructure/adapters/redis-io.adapter.ts
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class RedisIoAdapter extends IoAdapter {
  private readonly logger = new Logger(RedisIoAdapter.name);
  private adapterConstructor: ReturnType<typeof createAdapter>;

  constructor(
    private readonly configService: ConfigService,
  ) {
    super();
  }

  async connectToRedis(): Promise<void> {
    const redisUrl = this.configService.get<string>('REDIS_URL');

    const pubClient = createClient({ url: redisUrl });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);

    this.logger.log('Redis adapter connected');
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);

    if (this.adapterConstructor) {
      server.adapter(this.adapterConstructor);
      this.logger.log('Redis adapter attached to Socket.IO server');
    } else {
      this.logger.warn('Redis adapter not initialized, using default adapter');
    }

    return server;
  }
}
```

### Регистрация в модуле

```typescript
// websocket.module.ts
import { Module } from '@nestjs/common';
import { WebSocketModule } from '@nestjs/websockets';
import { RedisIoAdapter } from './infrastructure/adapters/redis-io.adapter';
import { AppWebSocketGateway } from './api/gateway/websocket.gateway';

@Module({
  imports: [WebSocketModule],
  providers: [
    AppWebSocketGateway,
    RedisIoAdapter,
  ],
})
export class WebSocketModule {
  constructor(private readonly redisAdapter: RedisIoAdapter) {}

  async onModuleInit() {
    // Инициализация Redis adapter при старте модуля
    await this.redisAdapter.connectToRedis();
  }
}
```

### Регистрация в main.ts

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { RedisIoAdapter } from './core/websocket/infrastructure/adapters/redis-io.adapter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Установка Redis adapter для WebSocket
  const redisAdapter = app.get(RedisIoAdapter);
  app.useWebSocketAdapter(redisAdapter);

  await app.listen(3000);
}
bootstrap();
```

## Как это работает

### Архитектура

```
┌─────────────┐         ┌─────────────┐
│  Instance 1 │         │  Instance 2 │
│             │         │             │
│  User A ────┼─────────┼─── User B   │
│  Socket     │         │   Socket    │
└──────┬──────┘         └──────┬──────┘
       │                        │
       │  Redis Pub/Sub         │
       └──────────┬─────────────┘
                  │
            ┌─────▼─────┐
            │   Redis   │
            │  Adapter  │
            └───────────┘
```

### Поток событий

1. **User A на Instance 1 отправляет сообщение**
   ```
   Instance 1 → Redis Pub → Все инстансы получают событие
   ```

2. **User B на Instance 2 получает сообщение**
   ```
   Redis Sub → Instance 2 → Socket B → User B
   ```

3. **Presence обновления синхронизируются**
   ```
   Instance 1: User A стал онлайн
   → Redis Pub → Instance 2 получает событие
   → Instance 2 отправляет presence:online всем своим клиентам
   ```

## Преимущества

1. **Горизонтальное масштабирование** — можно запускать несколько инстансов
2. **Синхронизация событий** — все инстансы получают все события
3. **Presence синхронизация** — статусы онлайн/офлайн синхронизируются
4. **Уведомления** — доставляются независимо от инстанса
5. **Load balancing** — можно распределять нагрузку между инстансами

## Конфигурация

### Environment Variables

```env
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=your-password
```

### Production настройки

```typescript
const pubClient = createClient({
  url: redisUrl,
  password: process.env.REDIS_PASSWORD,
  socket: {
    reconnectStrategy: (retries) => {
      if (retries > 10) {
        return new Error('Too many retries');
      }
      return Math.min(retries * 100, 3000);
    },
  },
});
```

## Мониторинг

### Метрики для отслеживания

- Количество подключений на инстанс
- Количество событий через Redis
- Задержка синхронизации между инстансами
- Ошибки подключения к Redis

### Логирование

```typescript
pubClient.on('error', (error) => {
  this.logger.error('Redis pub client error:', error);
});

subClient.on('error', (error) => {
  this.logger.error('Redis sub client error:', error);
});

pubClient.on('connect', () => {
  this.logger.log('Redis pub client connected');
});

subClient.on('connect', () => {
  this.logger.log('Redis sub client connected');
});
```

## Best Practices

1. **Всегда используйте Redis adapter в production** — для масштабируемости
2. **Мониторьте Redis** — следите за производительностью
3. **Настройте reconnection strategy** — для устойчивости
4. **Используйте отдельный Redis** — не смешивайте с другими данными
5. **Тестируйте с несколькими инстансами** — перед деплоем

## Альтернативы

### RabbitMQ Adapter

Если используется RabbitMQ, можно использовать RabbitMQ adapter:

```bash
npm install @socket.io/rabbitmq-adapter
```

### Kafka Adapter

Для больших масштабов можно использовать Kafka:

```bash
npm install @socket.io/kafka-adapter
```

## Troubleshooting

### Проблема: События не синхронизируются

**Решение:**
1. Проверьте подключение к Redis
2. Убедитесь, что adapter инициализирован до создания сервера
3. Проверьте логи Redis на ошибки

### Проблема: Медленная синхронизация

**Решение:**
1. Используйте Redis Cluster для больших нагрузок
2. Оптимизируйте размер payload событий
3. Используйте compression для больших сообщений
