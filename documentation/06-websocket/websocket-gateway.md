# WebSocket Gateway - Единый Gateway

## Обзор

Единый WebSocket Gateway для всех real-time функций проекта. Gateway является тонким слоем, который только маршрутизирует события к соответствующим сервисам. Вся бизнес-логика находится в модулях (chat, presence).

## Расположение

```
core/websocket/api/gateway/websocket.gateway.ts
```

## Архитектурный подход

### ❌ Плохо (множество Gateway)

```typescript
// ❌ НЕ ДЕЛАТЬ ТАК
@WebSocketGateway() export class ChatGateway {}
@WebSocketGateway() export class PresenceGateway {}
@WebSocketGateway() export class CallsGateway {}
```

**Проблемы:**
- Дублирование кода
- Сложность управления соединениями
- Нет единой точки входа
- Сложно масштабировать

### ✅ Хорошо (один Gateway)

```typescript
// ✅ ПРАВИЛЬНО
@WebSocketGateway({
  namespace: '/',
  cors: {
    origin: process.env.FRONTEND_URL,
    credentials: true,
  },
})
export class AppWebSocketGateway {}
```

**Преимущества:**
- Единая точка входа
- Централизованное управление соединениями
- Простое масштабирование
- Легко поддерживать

## Реализация

### Базовая структура

```typescript
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayConnection,
  OnGatewayDisconnect,
  ConnectedSocket,
  MessageBody,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Injectable, UseGuards, Logger } from '@nestjs/common';
import { WsJwtAuthGuard } from '../infrastructure/guards/ws-jwt-auth.guard';
import { ConnectionManagerService } from '../../application/services/connection-manager.service';
import { ChatService } from '@chat/application/services/chat.service';
import { PresenceService } from '@presence/application/services/presence.service';

@WebSocketGateway({
  namespace: '/',
  cors: {
    origin: process.env.FRONTEND_URL,
    credentials: true,
  },
  transports: ['websocket', 'polling'],
})
@Injectable()
export class AppWebSocketGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private readonly logger = new Logger(AppWebSocketGateway.name);

  constructor(
    private readonly connectionManager: ConnectionManagerService,
    private readonly chatService: ChatService,
    private readonly presenceService: PresenceService,
  ) {}

  // Обработка подключения
  async handleConnection(client: Socket) {
    // JWT аутентификация происходит в guard
    // После успешной аутентификации user доступен в client.data.user
    const user = client.data.user;

    if (!user) {
      this.logger.warn(`Connection rejected: no user`);
      client.disconnect();
      return;
    }

    // Регистрация соединения
    await this.connectionManager.addConnection(user.id, client.id, {
      roleContextId: user.roleContextId,
      deviceId: client.handshake.query.deviceId as string,
    });

    // Присоединение к комнатам пользователя
    client.join(`user:${user.id}`);
    client.join(`role:${user.roleContextId}`);

    // Уведомление о подключении (presence)
    await this.presenceService.handleUserConnected(user.id);

    this.logger.log(`User ${user.id} connected (socket: ${client.id})`);
  }

  // Обработка отключения
  async handleDisconnect(client: Socket) {
    const user = client.data.user;

    if (!user) {
      return;
    }

    // Удаление соединения
    await this.connectionManager.removeConnection(client.id);

    // Уведомление об отключении (presence)
    const hasOtherConnections = await this.connectionManager.hasConnections(user.id);
    if (!hasOtherConnections) {
      await this.presenceService.handleUserDisconnected(user.id);
    }

    this.logger.log(`User ${user.id} disconnected (socket: ${client.id})`);
  }

  // ============================================
  // Chat Events
  // ============================================

  @UseGuards(WsJwtAuthGuard)
  @SubscribeMessage('chat:send')
  async handleSendMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() payload: { chatId: string; content: string },
  ) {
    const user = client.data.user;

    // ✅ Делегируем в сервис - Gateway не знает о бизнес-логике
    return this.chatService.handleSendMessage({
      userId: user.id,
      roleContextId: user.roleContextId,
      chatId: payload.chatId,
      content: payload.content,
      socketId: client.id,
    });
  }

  @UseGuards(WsJwtAuthGuard)
  @SubscribeMessage('chat:join')
  async handleJoinChat(
    @ConnectedSocket() client: Socket,
    @MessageBody() payload: { chatId: string },
  ) {
    const user = client.data.user;

    // Присоединение к комнате чата
    client.join(`chat:${payload.chatId}`);

    // Уведомление сервиса
    await this.chatService.handleUserJoinedChat({
      userId: user.id,
      chatId: payload.chatId,
      socketId: client.id,
    });

    return { success: true };
  }

  @UseGuards(WsJwtAuthGuard)
  @SubscribeMessage('chat:leave')
  async handleLeaveChat(
    @ConnectedSocket() client: Socket,
    @MessageBody() payload: { chatId: string },
  ) {
    const user = client.data.user;

    // Покидание комнаты чата
    client.leave(`chat:${payload.chatId}`);

    await this.chatService.handleUserLeftChat({
      userId: user.id,
      chatId: payload.chatId,
      socketId: client.id,
    });

    return { success: true };
  }

  @UseGuards(WsJwtAuthGuard)
  @SubscribeMessage('chat:typing')
  async handleTyping(
    @ConnectedSocket() client: Socket,
    @MessageBody() payload: { chatId: string; isTyping: boolean },
  ) {
    const user = client.data.user;

    // Отправка события typing в комнату чата (кроме отправителя)
    client.to(`chat:${payload.chatId}`).emit('chat:typing', {
      userId: user.id,
      isTyping: payload.isTyping,
    });

    return { success: true };
  }

  // ============================================
  // Presence Events
  // ============================================

  @UseGuards(WsJwtAuthGuard)
  @SubscribeMessage('presence:ping')
  async handlePresencePing(
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user;

    // ✅ Делегируем в сервис
    await this.presenceService.handlePing(user.id);

    return { success: true };
  }

}
```

## Принципы реализации

### 1. Gateway ничего не знает о логике

**❌ Плохо:**

```typescript
@SubscribeMessage('chat:send')
async handleSendMessage(client: Socket, payload) {
  // ❌ Бизнес-логика в Gateway
  const message = await this.messageRepository.create({
    chatId: payload.chatId,
    content: payload.content,
    senderId: client.data.user.id,
  });

  // ❌ Прямая отправка через server.emit
  this.server.to(`chat:${payload.chatId}`).emit('chat:message', message);

  // ❌ Проверка доступа в Gateway
  const hasAccess = await this.checkChatAccess(client.data.user.id, payload.chatId);
  if (!hasAccess) {
    throw new ForbiddenException();
  }
}
```

**✅ Хорошо:**

```typescript
@SubscribeMessage('chat:send')
async handleSendMessage(
  @ConnectedSocket() client: Socket,
  @MessageBody() payload: { chatId: string; content: string },
) {
  const user = client.data.user;

  // ✅ Делегируем в сервис - вся логика там
  return this.chatService.handleSendMessage({
    userId: user.id,
    roleContextId: user.roleContextId,
    chatId: payload.chatId,
    content: payload.content,
    socketId: client.id,
  });
}
```

### 2. EventBus интеграция

Gateway не должен напрямую дергать бизнес-логику. Правильный поток:

```
Client → Gateway → Service → EventBus.emit() → Listener → WebSocket.emit()
```

**Пример:**

```typescript
// В ChatService
async handleSendMessage(data: SendMessageData) {
  // Бизнес-логика
  const message = await this.messageRepository.create({...});

  // ✅ Публикуем событие через EventBus
  this.eventBus.emit(AppEvent.MESSAGE_CREATED, {
    messageId: message.id,
    chatId: data.chatId,
    senderId: data.userId,
    content: message.content,
  });

  return message;
}

// В ChatWebSocketListener (infrastructure/listeners/)
@OnAppEvent(AppEvent.MESSAGE_CREATED)
async handleMessageCreated(payload: MessageCreatedPayload) {
  // ✅ Отправка через WebSocket
  this.gateway.server.to(`chat:${payload.chatId}`).emit('chat:message', {
    id: payload.messageId,
    content: payload.content,
    senderId: payload.senderId,
  });
}
```

### 3. Rooms Management

Используйте комнаты для таргетированной рассылки:

```typescript
// Присоединение к комнате
client.join(`chat:${chatId}`);
client.join(`user:${userId}`);
client.join(`role:${roleContextId}`);

// Отправка в комнату (не broadcast всем)
this.server.to(`chat:${chatId}`).emit('chat:message', message);

// Отправка конкретному пользователю (все его сокеты)
```

**❌ Плохо (broadcast всем):**

```typescript
// ❌ Отправка всем подключенным клиентам
this.server.emit('presence:online', { userId });
```

**✅ Хорошо (таргетированная рассылка):**

```typescript
// ✅ Отправка только заинтересованным пользователям
// Например, только участникам чата или друзьям
const interestedUsers = await this.getInterestedUsers(userId);
interestedUsers.forEach(userId => {
  this.server.to(`user:${userId}`).emit('presence:online', { userId });
});
```

## Структура событий

### Chat Events

| Событие | Направление | Описание |
|---------|------------|----------|
| `chat:send` | Client → Server | Отправка сообщения |
| `chat:join` | Client → Server | Присоединение к чату |
| `chat:leave` | Client → Server | Покидание чата |
| `chat:typing` | Client → Server | Индикатор набора текста |
| `chat:message` | Server → Client | Новое сообщение |
| `chat:typing` | Server → Client | Пользователь печатает |

### Presence Events

| Событие | Направление | Описание |
|---------|------------|----------|
| `presence:ping` | Client → Server | Heartbeat для поддержания онлайн |
| `presence:online` | Server → Client | Пользователь стал онлайн |
| `presence:offline` | Server → Client | Пользователь стал оффлайн |
| `presence:bulk-online` | Server → Client | Список всех онлайн пользователей |


## Безопасность

### JWT Authentication

Все события защищены через `@UseGuards(WsJwtAuthGuard)`:

```typescript
@UseGuards(WsJwtAuthGuard)
@SubscribeMessage('chat:send')
async handleSendMessage(...) {
  // client.data.user доступен после успешной аутентификации
}
```

### Валидация данных

Используйте DTO для валидации:

```typescript
import { IsString, IsNotEmpty } from 'class-validator';

class SendMessageDto {
  @IsString()
  @IsNotEmpty()
  chatId: string;

  @IsString()
  @IsNotEmpty()
  content: string;
}

@SubscribeMessage('chat:send')
async handleSendMessage(
  @ConnectedSocket() client: Socket,
  @MessageBody() payload: SendMessageDto, // ✅ Валидация через class-validator
) {
  // ...
}
```

## Best Practices

1. **Всегда используйте guards для защиты событий**
2. **Делегируйте логику в сервисы, не делайте её в Gateway**
3. **Используйте EventBus для связи между модулями**
4. **Используйте комнаты для таргетированной рассылки**
5. **Не делайте broadcast всем без причины**
6. **Логируйте важные события**
7. **Обрабатывайте ошибки и отправляйте их клиенту**

## Примеры использования

### Отправка сообщения

**Клиент:**
```typescript
socket.emit('chat:send', {
  chatId: 'chat-123',
  content: 'Привет!',
});
```

**Сервер (Gateway):**
```typescript
@UseGuards(WsJwtAuthGuard)
@SubscribeMessage('chat:send')
async handleSendMessage(
  @ConnectedSocket() client: Socket,
  @MessageBody() payload: SendMessageDto,
) {
  return this.chatService.handleSendMessage({
    userId: client.data.user.id,
    chatId: payload.chatId,
    content: payload.content,
    socketId: client.id,
  });
}
```

**Сервер (ChatService):**
```typescript
async handleSendMessage(data: SendMessageData) {
  // Бизнес-логика
  const message = await this.createMessage(data);

  // Публикация события
  this.eventBus.emit(AppEvent.MESSAGE_CREATED, {
    messageId: message.id,
    chatId: data.chatId,
    senderId: data.userId,
  });

  return message;
}
```

**Сервер (Listener):**
```typescript
@OnAppEvent(AppEvent.MESSAGE_CREATED)
async handleMessageCreated(payload: MessageCreatedPayload) {
  // Отправка через WebSocket
  this.gateway.server.to(`chat:${payload.chatId}`).emit('chat:message', {
    id: payload.messageId,
    content: payload.content,
  });
}
```

**Клиент (получение):**
```typescript
socket.on('chat:message', (message) => {
  console.log('New message:', message);
});
```
