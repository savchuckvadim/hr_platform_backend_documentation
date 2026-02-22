# Presence Service - Сервис присутствия

## Обзор

Сервис для управления статусами присутствия пользователей. Использует Redis с TTL для автоматического определения оффлайн статуса и публикует события через EventBus.

## Расположение

```
presence/application/services/presence.service.ts
```

## Реализация

### PresenceService

```typescript
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import { PresenceRepository } from '../../infrastructure/repositories/presence.repository';

@Injectable()
export class PresenceService implements OnModuleInit {
  private readonly logger = new Logger(PresenceService.name);
  private readonly TTL = 60; // 60 секунд
  private expiredSubscriber: Redis;

  constructor(
    @InjectRedis() private readonly redis: Redis,
    private readonly presenceRepository: PresenceRepository,
    private readonly eventBus: AppEventBus,
  ) {}

  async onModuleInit() {
    // Настройка Redis для отправки событий истечения ключей
    await this.redis.config('SET', 'notify-keyspace-events', 'Ex');

    // Подписка на события истечения
    this.expiredSubscriber = this.redis.duplicate();
    await this.expiredSubscriber.psubscribe('__keyevent@0__:expired');

    this.expiredSubscriber.on('pmessage', async (pattern, channel, key) => {
      if (key.startsWith('presence:user:')) {
        const userId = key.replace('presence:user:', '');
        await this.handlePresenceExpired(userId);
      }
    });

    this.logger.log('Presence service initialized');
  }

  /**
   * Помечает пользователя как онлайн
   * @param userId ID пользователя
   * @returns true если пользователь стал онлайн (был offline)
   */
  async markOnline(userId: string): Promise<boolean> {
    // Используем SET key value EX TTL NX
    // NX - только если ключа нет (не перезаписывает существующий)
    const result = await this.presenceRepository.setOnline(userId, this.TTL);

    if (result) {
      this.logger.debug(`User ${userId} became online`);

      // ✅ Публикация события через EventBus
      this.eventBus.emit(AppEvent.USER_ONLINE, {
        userId,
        timestamp: new Date(),
      });
    }

    return result;
  }

  /**
   * Продлевает TTL для пользователя (обновляет время онлайн)
   * @param userId ID пользователя
   */
  async refresh(userId: string): Promise<void> {
    // Используем SET key value EX TTL (без NX)
    // Всегда обновляет ключ, даже если он существует
    await this.presenceRepository.refreshOnline(userId, this.TTL);
  }

  /**
   * Помечает пользователя как оффлайн
   * @param userId ID пользователя
   * @returns true если пользователь был online
   */
  async markOffline(userId: string): Promise<boolean> {
    const wasOnline = await this.presenceRepository.deleteOnline(userId);

    if (wasOnline) {
      this.logger.debug(`User ${userId} became offline`);

      // ✅ Публикация события через EventBus
      this.eventBus.emit(AppEvent.USER_OFFLINE, {
        userId,
        timestamp: new Date(),
      });
    }

    return wasOnline;
  }

  /**
   * Проверяет, онлайн ли пользователь
   * @param userId ID пользователя
   */
  async isOnline(userId: string): Promise<boolean> {
    return this.presenceRepository.isOnline(userId);
  }

  /**
   * Получает список онлайн пользователей из массива userIds
   * @param userIds Массив ID пользователей
   * @returns Set с ID онлайн пользователей
   */
  async getOnlineUsers(userIds: string[]): Promise<Set<string>> {
    return this.presenceRepository.getOnlineUsers(userIds);
  }

  /**
   * Получает список всех онлайн пользователей
   * @returns Массив ID онлайн пользователей
   */
  async getAllOnlineUsers(): Promise<string[]> {
    return this.presenceRepository.getAllOnlineUsers();
  }

  /**
   * Обработка подключения пользователя
   * @param userId ID пользователя
   */
  async handleUserConnected(userId: string): Promise<void> {
    const becameOnline = await this.markOnline(userId);

    if (becameOnline) {
      this.logger.log(`🟢 User ${userId} became online`);
    }
  }

  /**
   * Обработка отключения пользователя
   * @param userId ID пользователя
   */
  async handleUserDisconnected(userId: string): Promise<void> {
    const wasOnline = await this.markOffline(userId);

    if (wasOnline) {
      this.logger.log(`🔴 User ${userId} became offline`);
    }
  }

  /**
   * Обработка ping от клиента
   * @param userId ID пользователя
   */
  async handlePing(userId: string): Promise<void> {
    // Проверяем, был ли пользователь онлайн до ping
    const wasOnline = await this.isOnline(userId);

    // Обновляем TTL
    await this.refresh(userId);

    // Если ключ был истек, но пользователь все еще подключен, помечаем как онлайн
    if (!wasOnline) {
      await this.markOnline(userId);
    }
  }

  /**
   * Обработка истечения presence ключа
   * @param userId ID пользователя
   */
  private async handlePresenceExpired(userId: string): Promise<void> {
    this.logger.debug(`Presence expired for user: ${userId}`);

    // ✅ Публикация события через EventBus
    this.eventBus.emit(AppEvent.USER_OFFLINE, {
      userId,
      timestamp: new Date(),
      reason: 'expired',
    });
  }
}
```

## Методы

### markOnline

Помечает пользователя как онлайн. Использует `SET key value EX TTL NX` для атомарной операции.

**Параметры:**
- `userId: string` - ID пользователя

**Возвращает:** `Promise<boolean>` - true если пользователь стал онлайн (был offline)

**Пример:**
```typescript
const becameOnline = await presenceService.markOnline(userId);
if (becameOnline) {
  // Пользователь только что стал онлайн
  // Событие USER_ONLINE уже опубликовано
}
```

### refresh

Продлевает TTL для пользователя. Используется при получении ping от клиента.

**Параметры:**
- `userId: string` - ID пользователя

**Пример:**
```typescript
// Каждые 25 секунд от клиента
await presenceService.refresh(userId);
```

### markOffline

Помечает пользователя как оффлайн. Удаляет ключ из Redis.

**Параметры:**
- `userId: string` - ID пользователя

**Возвращает:** `Promise<boolean>` - true если пользователь был online

**Пример:**
```typescript
const wasOnline = await presenceService.markOffline(userId);
if (wasOnline) {
  // Пользователь только что стал оффлайн
  // Событие USER_OFFLINE уже опубликовано
}
```

### isOnline

Проверяет, онлайн ли пользователь.

**Параметры:**
- `userId: string` - ID пользователя

**Возвращает:** `Promise<boolean>`

**Пример:**
```typescript
const isOnline = await presenceService.isOnline(userId);
```

### getOnlineUsers

Получает список онлайн пользователей из массива userIds. Использует Redis pipeline для эффективности.

**Параметры:**
- `userIds: string[]` - Массив ID пользователей

**Возвращает:** `Promise<Set<string>>` - Set с ID онлайн пользователей

**Пример:**
```typescript
const onlineUsers = await presenceService.getOnlineUsers(['user1', 'user2', 'user3']);
// Set { 'user1', 'user3' }
```

### getAllOnlineUsers

Получает список всех онлайн пользователей. Использует `SCAN` для безопасности (не `KEYS`).

**Возвращает:** `Promise<string[]>` - Массив ID онлайн пользователей

**Пример:**
```typescript
const allOnlineUsers = await presenceService.getAllOnlineUsers();
// ['user1', 'user2', 'user3', ...]
```

## Интеграция с WebSocket

### Обработка подключения

```typescript
// В WebSocket Gateway
async handleConnection(client: Socket) {
  const user = client.data.user;

  // ✅ Уведомление PresenceService
  await this.presenceService.handleUserConnected(user.id);
}
```

### Обработка отключения

```typescript
// В WebSocket Gateway
async handleDisconnect(client: Socket) {
  const user = client.data.user;

  // Проверка наличия других соединений
  const hasOtherConnections = await this.connectionManager.hasConnections(user.id);

  if (!hasOtherConnections) {
    // ✅ Пользователь полностью отключился
    await this.presenceService.handleUserDisconnected(user.id);
  }
}
```

### Обработка ping

```typescript
// В WebSocket Gateway
@SubscribeMessage('presence:ping')
async handlePresencePing(@ConnectedSocket() client: Socket) {
  const user = client.data.user;

  // ✅ Обновление TTL
  await this.presenceService.handlePing(user.id);
}
```

## EventBus интеграция

### События

```typescript
// core/events/events.types.ts
export enum AppEvent {
  USER_ONLINE = 'user.online',
  USER_OFFLINE = 'user.offline',
}

export interface EventPayloadMap {
  [AppEvent.USER_ONLINE]: {
    userId: string;
    timestamp: Date;
  };
  [AppEvent.USER_OFFLINE]: {
    userId: string;
    timestamp: Date;
    reason?: 'expired' | 'disconnected';
  };
}
```

### Listeners

```typescript
// infrastructure/listeners/presence-websocket.listener.ts
@Injectable()
export class PresenceWebSocketListener {
  constructor(
    @InjectWebSocketGateway() private readonly gateway: AppWebSocketGateway,
  ) {}

  @OnAppEvent(AppEvent.USER_ONLINE)
  async handleUserOnline(payload: UserOnlinePayload) {
    // ✅ Отправка только заинтересованным пользователям
    const interestedUsers = await this.getInterestedUsers(payload.userId);

    interestedUsers.forEach(userId => {
      this.gateway.server.to(`user:${userId}`).emit('presence:online', {
        userId: payload.userId,
        timestamp: payload.timestamp,
      });
    });
  }

  @OnAppEvent(AppEvent.USER_OFFLINE)
  async handleUserOffline(payload: UserOfflinePayload) {
    const interestedUsers = await this.getInterestedUsers(payload.userId);

    interestedUsers.forEach(userId => {
      this.gateway.server.to(`user:${userId}`).emit('presence:offline', {
        userId: payload.userId,
        timestamp: payload.timestamp,
        reason: payload.reason,
      });
    });
  }

  private async getInterestedUsers(userId: string): Promise<string[]> {
    // Получение списка заинтересованных пользователей
    // Например, участники общих чатов, друзья и т.д.
    // Это зависит от бизнес-логики приложения
    return []; // TODO: реализовать логику
  }
}
```

## Поток работы

### 1. Подключение пользователя

```
Клиент подключается к WebSocket
→ Gateway.handleConnection()
→ PresenceService.handleUserConnected()
→ PresenceService.markOnline()
→ Redis: SET presence:user:{userId} 1 EX 60 NX
→ EventBus.emit(USER_ONLINE)
→ Listener отправляет presence:online через WebSocket
```

### 2. Поддержание онлайн статуса

```
Клиент отправляет presence:ping каждые 25 секунд
→ Gateway.handlePresencePing()
→ PresenceService.handlePing()
→ PresenceService.refresh()
→ Redis: SET presence:user:{userId} 1 EX 60
→ TTL обновляется (60 секунд)
```

### 3. Истечение TTL

```
Redis ключ истекает (60 секунд без ping)
→ Redis отправляет событие expired через keyspace events
→ PresenceService.handlePresenceExpired()
→ EventBus.emit(USER_OFFLINE, { reason: 'expired' })
→ Listener отправляет presence:offline через WebSocket
```

### 4. Отключение пользователя

```
Клиент отключается от WebSocket
→ Gateway.handleDisconnect()
→ ConnectionManager проверяет наличие других соединений
→ Если нет других соединений:
  → PresenceService.handleUserDisconnected()
  → PresenceService.markOffline()
  → Redis: DEL presence:user:{userId}
  → EventBus.emit(USER_OFFLINE, { reason: 'disconnected' })
  → Listener отправляет presence:offline через WebSocket
```

## Best Practices

1. **Используйте TTL механизм** — автоматическое определение оффлайн
2. **Heartbeat каждые 25 секунд** — баланс между точностью и нагрузкой
3. **TTL 60 секунд** — баланс между точностью и производительностью
4. **Используйте SCAN вместо KEYS** — для безопасности
5. **Публикуйте события через EventBus** — не напрямую через WebSocket
6. **Таргетированная рассылка** — только заинтересованным пользователям
