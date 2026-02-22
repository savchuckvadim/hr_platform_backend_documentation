# Redis Storage - Хранение статусов в Redis

## Обзор

Хранение статусов присутствия в Redis с использованием TTL (Time To Live) для автоматического определения оффлайн статуса. Использует keyspace events для получения уведомлений об истечении ключей.

## Структура ключей

### Формат ключа

```
presence:user:{userId}
```

**Пример:**
```
presence:user:user-123
```

### Значение

Просто маркер наличия:
```
"1"
```

### TTL

60 секунд (настраивается)

## Redis команды

### SET с NX (markOnline)

```redis
SET presence:user:{userId} 1 EX 60 NX
```

**Параметры:**
- `EX 60` - TTL 60 секунд
- `NX` - только если ключа нет (не перезаписывает существующий)

**Возвращает:**
- `OK` - если ключ создан (пользователь стал онлайн)
- `null` - если ключ уже существует

**Использование:**
```typescript
const result = await redis.set(
  `presence:user:${userId}`,
  '1',
  'EX',
  60,
  'NX'
);
// result === 'OK' если пользователь стал онлайн
```

### SET без NX (refresh)

```redis
SET presence:user:{userId} 1 EX 60
```

**Параметры:**
- `EX 60` - TTL 60 секунд
- Без `NX` - всегда обновляет ключ

**Использование:**
```typescript
await redis.set(
  `presence:user:${userId}`,
  '1',
  'EX',
  60
);
// Всегда обновляет TTL
```

### DEL (markOffline)

```redis
DEL presence:user:{userId}
```

**Возвращает:**
- `1` - если ключ был удален (пользователь был online)
- `0` - если ключа не было

**Использование:**
```typescript
const result = await redis.del(`presence:user:${userId}`);
// result === 1 если пользователь был online
```

### EXISTS (isOnline)

```redis
EXISTS presence:user:{userId}
```

**Возвращает:**
- `1` - если ключ существует (пользователь онлайн)
- `0` - если ключа нет

**Использование:**
```typescript
const exists = await redis.exists(`presence:user:${userId}`);
// exists === 1 если пользователь онлайн
```

### Pipeline (getOnlineUsers)

```redis
EXISTS presence:user:{userId1}
EXISTS presence:user:{userId2}
EXISTS presence:user:{userId3}
```

**Использование:**
```typescript
const pipeline = redis.pipeline();
userIds.forEach(userId => {
  pipeline.exists(`presence:user:${userId}`);
});
const results = await pipeline.exec();

const onlineUsers = new Set<string>();
results?.forEach((result, index) => {
  if (result[1] === 1) {
    onlineUsers.add(userIds[index]);
  }
});
```

### SCAN (getAllOnlineUsers)

```redis
SCAN 0 MATCH presence:user:* COUNT 100
```

**Использование:**
```typescript
const onlineUsers: string[] = [];
let cursor = '0';

do {
  const [nextCursor, keys] = await redis.scan(
    cursor,
    'MATCH',
    'presence:user:*',
    'COUNT',
    100
  );

  keys.forEach(key => {
    const userId = key.replace('presence:user:', '');
    onlineUsers.push(userId);
  });

  cursor = nextCursor;
} while (cursor !== '0');
```

## Keyspace Events

### Настройка Redis

```typescript
await redis.config('SET', 'notify-keyspace-events', 'Ex');
```

**Параметры:**
- `E` - включить keyspace events
- `x` - включить expired events

### Подписка на события

```typescript
const expiredSubscriber = redis.duplicate();
await expiredSubscriber.psubscribe('__keyevent@0__:expired');

expiredSubscriber.on('pmessage', async (pattern, channel, key) => {
  if (key.startsWith('presence:user:')) {
    const userId = key.replace('presence:user:', '');
    await this.handlePresenceExpired(userId);
  }
});
```

**Канал:**
```
__keyevent@0__:expired
```

**Формат:**
- `@0` - база данных 0
- `expired` - события истечения ключей

## PresenceRepository

### Реализация

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

@Injectable()
export class PresenceRepository {
  constructor(
    @InjectRedis() private readonly redis: Redis,
  ) {}

  private getKey(userId: string): string {
    return `presence:user:${userId}`;
  }

  /**
   * Помечает пользователя как онлайн (только если был offline)
   */
  async setOnline(userId: string, ttl: number): Promise<boolean> {
    const key = this.getKey(userId);
    const result = await this.redis.set(key, '1', 'EX', ttl, 'NX');
    return result === 'OK';
  }

  /**
   * Обновляет TTL для пользователя
   */
  async refreshOnline(userId: string, ttl: number): Promise<void> {
    const key = this.getKey(userId);
    await this.redis.set(key, '1', 'EX', ttl);
  }

  /**
   * Помечает пользователя как оффлайн
   */
  async deleteOnline(userId: string): Promise<boolean> {
    const key = this.getKey(userId);
    const result = await this.redis.del(key);
    return result === 1;
  }

  /**
   * Проверяет, онлайн ли пользователь
   */
  async isOnline(userId: string): Promise<boolean> {
    const key = this.getKey(userId);
    const result = await this.redis.exists(key);
    return result === 1;
  }

  /**
   * Получает список онлайн пользователей из массива userIds
   */
  async getOnlineUsers(userIds: string[]): Promise<Set<string>> {
    if (userIds.length === 0) {
      return new Set();
    }

    const pipeline = this.redis.pipeline();
    userIds.forEach(userId => {
      pipeline.exists(this.getKey(userId));
    });

    const results = await pipeline.exec();
    const onlineUsers = new Set<string>();

    results?.forEach((result, index) => {
      if (result[1] === 1) {
        onlineUsers.add(userIds[index]);
      }
    });

    return onlineUsers;
  }

  /**
   * Получает список всех онлайн пользователей
   */
  async getAllOnlineUsers(): Promise<string[]> {
    const onlineUsers: string[] = [];
    let cursor = '0';

    do {
      const [nextCursor, keys] = await this.redis.scan(
        cursor,
        'MATCH',
        'presence:user:*',
        'COUNT',
        100,
      );

      keys.forEach(key => {
        const userId = key.replace('presence:user:', '');
        onlineUsers.push(userId);
      });

      cursor = nextCursor;
    } while (cursor !== '0');

    return onlineUsers;
  }
}
```

## Производительность

### Оптимизации

1. **Pipeline для массовых проверок:**
   ```typescript
   const pipeline = redis.pipeline();
   userIds.forEach(id => pipeline.exists(this.key(id)));
   const result = await pipeline.exec();
   ```

2. **SCAN вместо KEYS:**
   ```typescript
   // ❌ Плохо (блокирует Redis)
   const keys = await redis.keys('presence:user:*');

   // ✅ Хорошо (не блокирует)
   const keys = await this.scanKeys('presence:user:*');
   ```

3. **Отдельное подключение для подписки:**
   ```typescript
   const expiredSubscriber = redis.duplicate();
   // Избегает блокировки основного подключения
   ```

### Ограничения

1. **TTL 60 секунд** - баланс между точностью и нагрузкой
2. **SCAN может пропустить ключи** - если они создаются/удаляются во время сканирования
3. **Keyspace events требуют настройки Redis** - нужно включить `notify-keyspace-events Ex`

## Мониторинг

### Метрики

- Количество онлайн пользователей
- Частота ping событий
- Количество expired событий
- Время отклика Redis операций

### Логирование

```typescript
this.logger.log(`🟢 User ${userId} became online`);
this.logger.log(`🔴 Presence expired for user: ${userId}`);
this.logger.debug(`Presence TTL refreshed for user: ${userId}`);
```

## Best Practices

1. **Используйте SCAN вместо KEYS** - для безопасности
2. **Pipeline для массовых операций** - для производительности
3. **Отдельное подключение для подписки** - для избежания блокировок
4. **Настройте keyspace events** - для получения уведомлений об истечении
5. **Мониторьте производительность** - следите за временем отклика Redis
