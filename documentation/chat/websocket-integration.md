# WebSocket Integration - Интеграция с WebSocket

## Обзор

Интеграция модуля сообщений с единым WebSocket Gateway для real-time доставки сообщений на HR Platform. Сообщения используют единый WebSocket Gateway через event routing.

## Архитектурный принцип

**НЕ создавать отдельный Gateway для messages!**

Используется единый WebSocket Gateway из `core/websocket`, который маршрутизирует события к соответствующим сервисам.

## Архитектура

```
Client
  ↓
Единый WebSocket Gateway (core/websocket)
  ↓
ChatService.handleSendMessage()
  ↓
EventBus.emit(MESSAGE_CREATED)
  ↓
ChatWebSocketListener
  ↓
Gateway.server.emit() (отправка через WebSocket)
  ↓
Client
```

## Интеграция с единым Gateway

### Gateway маршрутизация

Единый Gateway маршрутизирует события чата:

```typescript
// core/websocket/api/gateway/websocket.gateway.ts
@WebSocketGateway({
  namespace: '/',
  cors: {
    origin: process.env.FRONTEND_URL,
    credentials: true,
  },
})
export class AppWebSocketGateway {
  constructor(
    private readonly chatService: ChatService,
  ) {}

  @UseGuards(WsJwtAuthGuard)
  @SubscribeMessage('chat:send')
  async handleSendMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() payload: SendMessageDto,
  ) {
    const user = client.data.user;

    // ✅ Делегируем в ChatService - Gateway не знает о логике
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

    await this.chatService.handleUserJoinedChat({
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
}
```

## ChatService обработка

### Service методы

```typescript
// application/services/messages.service.ts
import { Injectable } from '@nestjs/common';
import { MessagesRepository } from '../infrastructure/repositories/messages.repository';
import { ChatsRepository } from '@chats/infrastructure/repositories/chats.repository';
import { CreateMessageDto, MessageDto } from '../api/dto';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';

@Injectable()
export class MessagesService {
    constructor(
        private readonly repository: MessagesRepository,
        private readonly chatsRepository: ChatsRepository,
        private readonly eventBus: AppEventBus,
    ) {}

    /**
     * Обработка отправки сообщения через WebSocket
     */
    async handleSendMessage(data: {
        userId: string;
        roleContextId: string;
        chatId: string;
        content: string;
        socketId: string;
    }): Promise<MessageDto> {
        // Проверяем, что пользователь является участником чата
        const isMember = await this.chatsRepository.isMember(data.chatId, data.userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        // Создаем сообщение
        const message = await this.repository.create({
            chatId: data.chatId,
            senderId: data.userId,
            content: data.content,
        });

        const messageDto = new MessageDto(message);

        // ✅ Публикуем событие через EventBus
        this.eventBus.emit(AppEvent.MESSAGE_CREATED, {
            messageId: message.id,
            chatId: data.chatId,
            senderId: data.userId,
            content: message.content,
            createdAt: message.createdAt,
        });

        return messageDto;
    }

    /**
     * Обработка присоединения к чату
     */
    async handleUserJoinedChat(data: {
        userId: string;
        chatId: string;
        socketId: string;
    }): Promise<void> {
        // Обновляем lastReadAt
        await this.chatsRepository.updateLastRead(data.chatId, data.userId);
    }

    /**
     * Обработка покидания чата
     */
    async handleUserLeftChat(data: {
        userId: string;
        chatId: string;
        socketId: string;
    }): Promise<void> {
        // Логика при покидании чата (если нужна)
    }
}
```

## WebSocket Listener

### Отправка через WebSocket

```typescript
// infrastructure/listeners/chat-websocket.listener.ts
import { Injectable } from '@nestjs/common';
import { OnAppEvent } from '@core/events/event-decorators';
import { AppEvent } from '@core/events/events.types';
import { AppWebSocketGateway } from '@core/websocket/api/gateway/websocket.gateway';

@Injectable()
export class ChatWebSocketListener {
    constructor(
        private readonly gateway: AppWebSocketGateway,
    ) {}

    @OnAppEvent(AppEvent.MESSAGE_CREATED)
    async handleMessageCreated(payload: MessageCreatedPayload) {
        // ✅ Отправка через единый WebSocket Gateway в комнату чата
        this.gateway.server.to(`chat:${payload.chatId}`).emit('chat:message', {
            id: payload.messageId,
            chatId: payload.chatId,
            senderId: payload.senderId,
            content: payload.content,
            createdAt: payload.createdAt,
        });
    }
}
```

## Connection Manager

### Управление соединениями

Connection Manager из `core/websocket` управляет соединениями:

```typescript
// При подключении в Gateway
async handleConnection(client: Socket) {
    const user = client.data.user;

    // Регистрация соединения
    await this.connectionManager.addConnection(user.id, client.id, {
        roleContextId: user.roleContextId,
        deviceId: client.handshake.query.deviceId as string,
    });

    // Присоединение к комнатам пользователя
    client.join(`user:${user.id}`);
    client.join(`role:${user.roleContextId}`);

    // Присоединение ко всем чатам пользователя
    const chats = await this.chatsRepository.findByUserId(user.id);
    chats.forEach(chat => {
        client.join(`chat:${chat.id}`);
    });
}
```

## События WebSocket

### Chat Events

| Событие | Направление | Описание |
|---------|------------|----------|
| `chat:send` | Client → Server | Отправка сообщения |
| `chat:join` | Client → Server | Присоединение к чату |
| `chat:leave` | Client → Server | Покидание чата |
| `chat:typing` | Client → Server | Индикатор набора текста |
| `chat:message` | Server → Client | Новое сообщение |
| `chat:typing` | Server → Client | Пользователь печатает |

## Best Practices

1. **Используйте единый Gateway** - не создавайте отдельные gateway
2. **Делегируйте в сервисы** - Gateway только маршрутизирует
3. **Публикуйте через EventBus** - для связи между модулями
4. **Используйте listeners** - для отправки через WebSocket
5. **Используйте rooms** - для таргетированной рассылки

## См. также

- [WebSocket Module](../06-websocket/index.md) - единый WebSocket Gateway
- [Event Routing](../06-websocket/event-routing.md) - маршрутизация событий
