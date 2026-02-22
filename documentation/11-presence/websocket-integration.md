# WebSocket Integration - Интеграция с WebSocket

## Обзор

Интеграция модуля Presence с WebSocket для real-time обновлений статусов онлайн/оффлайн. События публикуются через EventBus и отправляются через WebSocket listeners.

## Архитектура

```
PresenceService
  ↓
EventBus.emit(USER_ONLINE / USER_OFFLINE)
  ↓
PresenceWebSocketListener
  ↓
Gateway.server.emit(presence:online / presence:offline)
  ↓
Client
```

## События

### Клиент → Сервер

| Событие | Описание |
|---------|----------|
| `presence:ping` | Heartbeat для поддержания онлайн статуса (каждые 25 секунд) |

### Сервер → Клиент

| Событие | Описание |
|---------|----------|
| `presence:online` | Пользователь стал онлайн |
| `presence:offline` | Пользователь стал оффлайн |
| `presence:bulk-online` | Список всех онлайн пользователей (при подключении) |

## Реализация

### Gateway обработка

```typescript
// В AppWebSocketGateway
@WebSocketGateway()
export class AppWebSocketGateway {
  constructor(
    private readonly presenceService: PresenceService,
  ) {}

  async handleConnection(client: Socket) {
    const user = client.data.user;

    // ✅ Уведомление о подключении
    await this.presenceService.handleUserConnected(user.id);

    // Отправка списка всех онлайн пользователей новому клиенту
    const allOnlineUsers = await this.presenceService.getAllOnlineUsers();
    client.emit('presence:bulk-online', {
      users: allOnlineUsers,
    });
  }

  async handleDisconnect(client: Socket) {
    const user = client.data.user;

    // Проверка наличия других соединений
    const hasOtherConnections = await this.connectionManager.hasConnections(user.id);

    if (!hasOtherConnections) {
      // ✅ Пользователь полностью отключился
      await this.presenceService.handleUserDisconnected(user.id);
    }
  }

  @UseGuards(WsJwtAuthGuard)
  @SubscribeMessage('presence:ping')
  async handlePresencePing(@ConnectedSocket() client: Socket) {
    const user = client.data.user;

    // ✅ Обновление TTL
    await this.presenceService.handlePing(user.id);

    return { success: true };
  }
}
```

### WebSocket Listener

```typescript
// infrastructure/listeners/presence-websocket.listener.ts
import { Injectable } from '@nestjs/common';
import { OnAppEvent } from '@core/events/event-decorators';
import { AppEvent } from '@core/events/events.types';
import { AppWebSocketGateway } from '@core/websocket/api/gateway/websocket.gateway';
import { Logger } from '@nestjs/common';

@Injectable()
export class PresenceWebSocketListener {
  private readonly logger = new Logger(PresenceWebSocketListener.name);

  constructor(
    private readonly gateway: AppWebSocketGateway,
  ) {}

  @OnAppEvent(AppEvent.USER_ONLINE)
  async handleUserOnline(payload: { userId: string; timestamp: Date }) {
    this.logger.debug(`User ${payload.userId} became online`);

    // ✅ Получение списка заинтересованных пользователей
    const interestedUsers = await this.getInterestedUsers(payload.userId);

    // ✅ Таргетированная рассылка (не broadcast всем)
    interestedUsers.forEach(userId => {
      this.gateway.server.to(`user:${userId}`).emit('presence:online', {
        userId: payload.userId,
        timestamp: payload.timestamp,
      });
    });
  }

  @OnAppEvent(AppEvent.USER_OFFLINE)
  async handleUserOffline(payload: {
    userId: string;
    timestamp: Date;
    reason?: 'expired' | 'disconnected';
  }) {
    this.logger.debug(`User ${payload.userId} became offline (${payload.reason})`);

    const interestedUsers = await this.getInterestedUsers(payload.userId);

    interestedUsers.forEach(userId => {
      this.gateway.server.to(`user:${userId}`).emit('presence:offline', {
        userId: payload.userId,
        timestamp: payload.timestamp,
        reason: payload.reason,
      });
    });
  }

  /**
   * Получение списка заинтересованных пользователей
   * Пользователи, которые должны получить уведомление об изменении статуса
   */
  private async getInterestedUsers(userId: string): Promise<string[]> {
    // TODO: Реализовать логику получения заинтересованных пользователей
    // Например:
    // - Участники общих чатов
    // - Друзья/подписчики
    // - HR и кандидаты в общих вакансиях/откликах

    // Временная реализация: возвращаем всех онлайн пользователей
    // В production нужно реализовать правильную логику
    return [];
  }
}
```

## Клиентская часть

### Подключение и ping

```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000', {
  auth: { token: 'your-jwt-token' },
});

// Подключение
socket.on('connect', () => {
  console.log('Connected');

  // Получение списка онлайн пользователей
  socket.on('presence:bulk-online', (data) => {
    console.log('Online users:', data.users);
  });

  // Heartbeat каждые 25 секунд
  setInterval(() => {
    socket.emit('presence:ping');
  }, 25000);
});

// Получение событий онлайн/оффлайн
socket.on('presence:online', (data) => {
  console.log(`User ${data.userId} became online`);
  // Обновление UI
});

socket.on('presence:offline', (data) => {
  console.log(`User ${data.userId} became offline`);
  // Обновление UI
});
```

### React Hook

```typescript
import { useEffect, useState } from 'react';
import { useSocket } from './useSocket';

export function usePresence() {
  const socket = useSocket();
  const [onlineUsers, setOnlineUsers] = useState<Set<string>>(new Set());

  useEffect(() => {
    if (!socket) return;

    // Получение списка онлайн пользователей при подключении
    socket.on('presence:bulk-online', (data: { users: string[] }) => {
      setOnlineUsers(new Set(data.users));
    });

    // Обновление при изменении статуса
    socket.on('presence:online', (data: { userId: string }) => {
      setOnlineUsers(prev => new Set([...prev, data.userId]));
    });

    socket.on('presence:offline', (data: { userId: string }) => {
      setOnlineUsers(prev => {
        const next = new Set(prev);
        next.delete(data.userId);
        return next;
      });
    });

    // Heartbeat
    const pingInterval = setInterval(() => {
      socket.emit('presence:ping');
    }, 25000);

    return () => {
      socket.off('presence:bulk-online');
      socket.off('presence:online');
      socket.off('presence:offline');
      clearInterval(pingInterval);
    };
  }, [socket]);

  return {
    onlineUsers,
    isOnline: (userId: string) => onlineUsers.has(userId),
  };
}
```

## Структура данных

### presence:online

```typescript
{
  userId: string;
  timestamp: string; // ISO 8601
}
```

### presence:offline

```typescript
{
  userId: string;
  timestamp: string; // ISO 8601
  reason?: 'expired' | 'disconnected';
}
```

### presence:bulk-online

```typescript
{
  users: string[]; // Массив ID онлайн пользователей
}
```

## Best Practices

1. **Heartbeat каждые 25 секунд** - баланс между точностью и нагрузкой
2. **Таргетированная рассылка** - только заинтересованным пользователям
3. **Не broadcast всем** - только в нужные rooms
4. **Обработка реконнекта** - отправка bulk-online при переподключении
5. **Очистка при отключении** - удаление из списка онлайн пользователей
