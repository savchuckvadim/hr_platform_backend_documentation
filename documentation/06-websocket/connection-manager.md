# Connection Manager - Управление соединениями

## Обзор

Connection Manager — обязательная абстракция для управления WebSocket соединениями пользователей. Обеспечивает поддержку multi-device, multi-tab и правильную обработку реконнектов.

## Расположение

```
core/websocket/application/services/connection-manager.service.ts
```

## Проблема без Connection Manager

**❌ Плохо (прямое использование client.handshake.query.userId):**

```typescript
async handleConnection(client: Socket) {
  const userId = client.handshake.query.userId; // ❌ Небезопасно и ненадежно

  // Проблемы:
  // 1. Нет проверки аутентификации
  // 2. Нет поддержки multi-device
  // 3. Нет поддержки multi-tab
  // 4. Нет информации о deviceId
  // 5. Нет информации о roleContextId
}
```

**Проблемы:**
- Один пользователь может иметь 5 вкладок
- Несколько устройств (телефон, компьютер)
- Реконнект теряет информацию
- Нет масштабируемости

## Реализация

### Интерфейс Connection

```typescript
// domain/interfaces/connection.interface.ts
export interface Connection {
  userId: string;
  socketId: string;
  roleContextId: string;
  deviceId: string;
  deviceName?: string;
  userAgent?: string;
  ipAddress?: string;
  connectedAt: Date;
}
```

### ConnectionManagerService

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';
import { Connection } from '../../domain/interfaces/connection.interface';

@Injectable()
export class ConnectionManagerService {
  private readonly logger = new Logger(ConnectionManagerService.name);
  private readonly CONNECTION_TTL = 3600; // 1 час

  constructor(
    @InjectRedis() private readonly redis: Redis,
  ) {}

  /**
   * Добавление соединения
   * @param userId ID пользователя
   * @param socketId ID сокета
   * @param metadata Метаданные соединения
   */
  async addConnection(
    userId: string,
    socketId: string,
    metadata: {
      roleContextId: string;
      deviceId: string;
      deviceName?: string;
      userAgent?: string;
      ipAddress?: string;
    },
  ): Promise<void> {
    const connection: Connection = {
      userId,
      socketId,
      roleContextId: metadata.roleContextId,
      deviceId: metadata.deviceId,
      deviceName: metadata.deviceName,
      userAgent: metadata.userAgent,
      ipAddress: metadata.ipAddress,
      connectedAt: new Date(),
    };

    // Сохранение соединения в Redis
    const key = this.getConnectionKey(socketId);
    await this.redis.setex(
      key,
      this.CONNECTION_TTL,
      JSON.stringify(connection),
    );

    // Добавление socketId в список соединений пользователя
    const userConnectionsKey = this.getUserConnectionsKey(userId);
    await this.redis.sadd(userConnectionsKey, socketId);
    await this.redis.expire(userConnectionsKey, this.CONNECTION_TTL);

    // Добавление socketId в список соединений устройства
    const deviceConnectionsKey = this.getDeviceConnectionsKey(
      userId,
      metadata.deviceId,
    );
    await this.redis.sadd(deviceConnectionsKey, socketId);
    await this.redis.expire(deviceConnectionsKey, this.CONNECTION_TTL);

    this.logger.debug(
      `Connection added: user=${userId}, socket=${socketId}, device=${metadata.deviceId}`,
    );
  }

  /**
   * Удаление соединения
   * @param socketId ID сокета
   */
  async removeConnection(socketId: string): Promise<Connection | null> {
    const key = this.getConnectionKey(socketId);
    const connectionData = await this.redis.get(key);

    if (!connectionData) {
      return null;
    }

    const connection: Connection = JSON.parse(connectionData);

    // Удаление соединения
    await this.redis.del(key);

    // Удаление socketId из списка соединений пользователя
    const userConnectionsKey = this.getUserConnectionsKey(connection.userId);
    await this.redis.srem(userConnectionsKey, socketId);

    // Удаление socketId из списка соединений устройства
    const deviceConnectionsKey = this.getDeviceConnectionsKey(
      connection.userId,
      connection.deviceId,
    );
    await this.redis.srem(deviceConnectionsKey, socketId);

    // Если у пользователя больше нет соединений, удаляем ключ
    const remainingConnections = await this.redis.scard(userConnectionsKey);
    if (remainingConnections === 0) {
      await this.redis.del(userConnectionsKey);
    }

    this.logger.debug(
      `Connection removed: user=${connection.userId}, socket=${socketId}`,
    );

    return connection;
  }

  /**
   * Получение соединения по socketId
   * @param socketId ID сокета
   */
  async getConnection(socketId: string): Promise<Connection | null> {
    const key = this.getConnectionKey(socketId);
    const connectionData = await this.redis.get(key);

    if (!connectionData) {
      return null;
    }

    return JSON.parse(connectionData);
  }

  /**
   * Получение всех соединений пользователя
   * @param userId ID пользователя
   */
  async getUserConnections(userId: string): Promise<Connection[]> {
    const userConnectionsKey = this.getUserConnectionsKey(userId);
    const socketIds = await this.redis.smembers(userConnectionsKey);

    if (socketIds.length === 0) {
      return [];
    }

    // Получение всех соединений через pipeline
    const pipeline = this.redis.pipeline();
    socketIds.forEach((socketId) => {
      pipeline.get(this.getConnectionKey(socketId));
    });

    const results = await pipeline.exec();
    const connections: Connection[] = [];

    results?.forEach((result) => {
      if (result[1] && typeof result[1] === 'string') {
        try {
          connections.push(JSON.parse(result[1]));
        } catch (e) {
          this.logger.warn(`Failed to parse connection: ${result[1]}`);
        }
      }
    });

    return connections.filter((c) => c !== null);
  }

  /**
   * Получение соединений устройства
   * @param userId ID пользователя
   * @param deviceId ID устройства
   */
  async getDeviceConnections(
    userId: string,
    deviceId: string,
  ): Promise<Connection[]> {
    const deviceConnectionsKey = this.getDeviceConnectionsKey(userId, deviceId);
    const socketIds = await this.redis.smembers(deviceConnectionsKey);

    if (socketIds.length === 0) {
      return [];
    }

    const pipeline = this.redis.pipeline();
    socketIds.forEach((socketId) => {
      pipeline.get(this.getConnectionKey(socketId));
    });

    const results = await pipeline.exec();
    const connections: Connection[] = [];

    results?.forEach((result) => {
      if (result[1] && typeof result[1] === 'string') {
        try {
          connections.push(JSON.parse(result[1]));
        } catch (e) {
          this.logger.warn(`Failed to parse connection: ${result[1]}`);
        }
      }
    });

    return connections.filter((c) => c !== null);
  }

  /**
   * Проверка наличия соединений у пользователя
   * @param userId ID пользователя
   */
  async hasConnections(userId: string): Promise<boolean> {
    const userConnectionsKey = this.getUserConnectionsKey(userId);
    const count = await this.redis.scard(userConnectionsKey);
    return count > 0;
  }

  /**
   * Получение количества соединений пользователя
   * @param userId ID пользователя
   */
  async getConnectionCount(userId: string): Promise<number> {
    const userConnectionsKey = this.getUserConnectionsKey(userId);
    return this.redis.scard(userConnectionsKey);
  }

  /**
   * Получение всех socketId пользователя
   * @param userId ID пользователя
   */
  async getUserSocketIds(userId: string): Promise<string[]> {
    const userConnectionsKey = this.getUserConnectionsKey(userId);
    return this.redis.smembers(userConnectionsKey);
  }

  /**
   * Отправка сообщения всем соединениям пользователя
   * @param userId ID пользователя
   * @param event Событие
   * @param data Данные
   */
  async emitToUser(
    userId: string,
    event: string,
    data: any,
    server: Server,
  ): Promise<void> {
    const socketIds = await this.getUserSocketIds(userId);

    socketIds.forEach((socketId) => {
      const socket = server.sockets.sockets.get(socketId);
      if (socket) {
        socket.emit(event, data);
      }
    });
  }

  /**
   * Отправка сообщения всем соединениям устройства
   * @param userId ID пользователя
   * @param deviceId ID устройства
   * @param event Событие
   * @param data Данные
   */
  async emitToDevice(
    userId: string,
    deviceId: string,
    event: string,
    data: any,
    server: Server,
  ): Promise<void> {
    const socketIds = await this.getDeviceConnections(userId, deviceId).then(
      (connections) => connections.map((c) => c.socketId),
    );

    socketIds.forEach((socketId) => {
      const socket = server.sockets.sockets.get(socketId);
      if (socket) {
        socket.emit(event, data);
      }
    });
  }

  // ============================================
  // Private helpers
  // ============================================

  private getConnectionKey(socketId: string): string {
    return `ws:connection:${socketId}`;
  }

  private getUserConnectionsKey(userId: string): string {
    return `ws:user:${userId}:connections`;
  }

  private getDeviceConnectionsKey(userId: string, deviceId: string): string {
    return `ws:user:${userId}:device:${deviceId}:connections`;
  }
}
```

## Redis структура

### Ключи

```
ws:connection:{socketId}              # Информация о соединении (TTL: 1 час)
ws:user:{userId}:connections         # Set socketId пользователя (TTL: 1 час)
ws:user:{userId}:device:{deviceId}:connections  # Set socketId устройства (TTL: 1 час)
```

### Пример данных

```json
// ws:connection:socket-123
{
  "userId": "user-456",
  "socketId": "socket-123",
  "roleContextId": "role-789",
  "deviceId": "device-abc",
  "deviceName": "Chrome on Windows",
  "userAgent": "Mozilla/5.0...",
  "ipAddress": "192.168.1.1",
  "connectedAt": "2024-01-15T10:00:00Z"
}

// ws:user:user-456:connections
Set: ["socket-123", "socket-124", "socket-125"]

// ws:user:user-456:device:device-abc:connections
Set: ["socket-123", "socket-124"]
```

## Использование в Gateway

```typescript
@WebSocketGateway()
export class AppWebSocketGateway {
  constructor(
    private readonly connectionManager: ConnectionManagerService,
  ) {}

  async handleConnection(client: Socket) {
    const user = client.data.user; // Из JWT guard

    // ✅ Регистрация соединения
    await this.connectionManager.addConnection(user.id, client.id, {
      roleContextId: user.roleContextId,
      deviceId: client.handshake.query.deviceId as string,
      deviceName: client.handshake.query.deviceName,
      userAgent: client.handshake.headers['user-agent'],
      ipAddress: client.handshake.address,
    });
  }

  async handleDisconnect(client: Socket) {
    // ✅ Удаление соединения
    const connection = await this.connectionManager.removeConnection(client.id);

    if (connection) {
      // Проверка наличия других соединений
      const hasOtherConnections = await this.connectionManager.hasConnections(
        connection.userId,
      );

      if (!hasOtherConnections) {
        // Пользователь полностью отключился
        await this.presenceService.handleUserDisconnected(connection.userId);
      }
    }
  }
}
```

## Использование в сервисах

### Отправка события конкретному пользователю

```typescript
// Отправка события всем соединениям пользователя
await this.connectionManager.emitToUser(
  userId,
  'event:name',
  eventData,
  this.gateway.server,
);
```

### Отправка события конкретному устройству

```typescript
// Отправка только на конкретное устройство
await this.connectionManager.emitToDevice(
  userId,
  deviceId,
  'event:name',
  eventData,
  this.gateway.server,
);
```

### Получение всех соединений пользователя

```typescript
// Получение информации о всех соединениях
const connections = await this.connectionManager.getUserConnections(userId);

console.log(`User has ${connections.length} active connections`);
connections.forEach((conn) => {
  console.log(`- Socket: ${conn.socketId}, Device: ${conn.deviceId}`);
});
```

## Преимущества

1. **Multi-device поддержка** — один пользователь может иметь несколько устройств
2. **Multi-tab поддержка** — один пользователь может иметь несколько вкладок
3. **Реконнект** — информация сохраняется в Redis
4. **Масштабируемость** — работает с несколькими инстансами через Redis
5. **Метаданные** — хранит информацию о deviceId, roleContextId и т.д.
6. **Таргетированная рассылка** — можно отправлять на конкретное устройство или все устройства

## Best Practices

1. **Всегда используйте ConnectionManager** для управления соединениями
2. **Не полагайтесь на client.handshake.query.userId** — используйте JWT
3. **Используйте TTL** для автоматической очистки старых соединений
4. **Логируйте важные события** подключения/отключения
5. **Обрабатывайте ошибки** при работе с Redis
