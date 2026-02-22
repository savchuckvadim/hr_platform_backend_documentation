# Event Routing - Маршрутизация событий

## Обзор

Маршрутизация WebSocket событий к соответствующим сервисам. Gateway является тонким слоем, который только маршрутизирует события, не содержа бизнес-логику.

## Принцип

**Gateway → Service → EventBus → Listener → WebSocket.emit()**

Gateway не должен напрямую отправлять события через `server.emit()`. Вместо этого события публикуются через EventBus, а listeners отправляют их через WebSocket.

## Архитектура

```
Client
  ↓
Gateway (маршрутизация)
  ↓
Service (бизнес-логика)
  ↓
EventBus.emit() (публикация события)
  ↓
Listener (подписка на событие)
  ↓
Gateway.server.emit() (отправка через WebSocket)
  ↓
Client
```

## Реализация

### Gateway (маршрутизация)

```typescript
@WebSocketGateway()
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

    // ✅ Делегируем в сервис - Gateway не знает о логике
    return this.chatService.handleSendMessage({
      userId: user.id,
      roleContextId: user.roleContextId,
      chatId: payload.chatId,
      content: payload.content,
      socketId: client.id,
    });
  }
}
```

### Service (бизнес-логика + EventBus)

```typescript
@Injectable()
export class ChatService {
  constructor(
    private readonly messageRepository: MessageRepository,
    private readonly eventBus: AppEventBus,
  ) {}

  async handleSendMessage(data: SendMessageData): Promise<Message> {
    // 1. Бизнес-логика
    const message = await this.messageRepository.create({
      chatId: data.chatId,
      senderId: data.userId,
      content: data.content,
    });

    // 2. Публикация события через EventBus
    this.eventBus.emit(AppEvent.MESSAGE_CREATED, {
      messageId: message.id,
      chatId: data.chatId,
      senderId: data.userId,
      content: message.content,
      createdAt: message.createdAt,
    });

    return message;
  }
}
```

### Listener (отправка через WebSocket)

```typescript
// infrastructure/listeners/chat-websocket.listener.ts
@Injectable()
export class ChatWebSocketListener {
  constructor(
    @InjectWebSocketGateway() private readonly gateway: AppWebSocketGateway,
  ) {}

  @OnAppEvent(AppEvent.MESSAGE_CREATED)
  async handleMessageCreated(payload: MessageCreatedPayload) {
    // ✅ Отправка через WebSocket в комнату чата
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

## Типы событий

### Chat Events

| Событие | Gateway → Service | Service → EventBus | Listener → WebSocket |
|---------|------------------|-------------------|---------------------|
| `chat:send` | `ChatService.handleSendMessage()` | `MESSAGE_CREATED` | `chat:message` |
| `chat:join` | `ChatService.handleUserJoinedChat()` | `USER_JOINED_CHAT` | `chat:user-joined` |
| `chat:leave` | `ChatService.handleUserLeftChat()` | `USER_LEFT_CHAT` | `chat:user-left` |
| `chat:typing` | `ChatService.handleTyping()` | - | `chat:typing` (direct) |

### Presence Events

| Событие | Gateway → Service | Service → EventBus | Listener → WebSocket |
|---------|------------------|-------------------|---------------------|
| `presence:ping` | `PresenceService.handlePing()` | - | - (internal) |
| - | `PresenceService.handleUserConnected()` | `USER_ONLINE` | `presence:online` |
| - | `PresenceService.handleUserDisconnected()` | `USER_OFFLINE` | `presence:offline` |


## Примеры

### Отправка сообщения

**1. Клиент отправляет событие:**
```typescript
socket.emit('chat:send', {
  chatId: 'chat-123',
  content: 'Привет!',
});
```

**2. Gateway маршрутизирует:**
```typescript
@SubscribeMessage('chat:send')
async handleSendMessage(client: Socket, payload: SendMessageDto) {
  return this.chatService.handleSendMessage({
    userId: client.data.user.id,
    chatId: payload.chatId,
    content: payload.content,
  });
}
```

**3. Service обрабатывает и публикует событие:**
```typescript
async handleSendMessage(data: SendMessageData) {
  const message = await this.createMessage(data);

  // Публикация события
  this.eventBus.emit(AppEvent.MESSAGE_CREATED, {
    messageId: message.id,
    chatId: data.chatId,
    senderId: data.userId,
    content: message.content,
  });

  return message;
}
```

**4. Listener отправляет через WebSocket:**
```typescript
@OnAppEvent(AppEvent.MESSAGE_CREATED)
async handleMessageCreated(payload: MessageCreatedPayload) {
  // Отправка в комнату чата
  this.gateway.server.to(`chat:${payload.chatId}`).emit('chat:message', {
    id: payload.messageId,
    content: payload.content,
    senderId: payload.senderId,
  });
}
```

**5. Клиент получает событие:**
```typescript
socket.on('chat:message', (message) => {
  console.log('New message:', message);
});
```

### Presence обновления

**1. User подключается:**
```typescript
// В Gateway
async handleConnection(client: Socket) {
  const user = client.data.user;
  await this.presenceService.handleUserConnected(user.id);
}
```

**2. PresenceService публикует событие:**
```typescript
async handleUserConnected(userId: string) {
  const becameOnline = await this.markOnline(userId);

  if (becameOnline) {
    // Публикация события
    this.eventBus.emit(AppEvent.USER_ONLINE, {
      userId,
      timestamp: new Date(),
    });
  }
}
```

**3. Listener отправляет через WebSocket:**
```typescript
@OnAppEvent(AppEvent.USER_ONLINE)
async handleUserOnline(payload: UserOnlinePayload) {
  // Отправка только заинтересованным пользователям
  const interestedUsers = await this.getInterestedUsers(payload.userId);

  interestedUsers.forEach(userId => {
    this.gateway.server.to(`user:${userId}`).emit('presence:online', {
      userId: payload.userId,
    });
  });
}
```

## Best Practices

1. **Gateway только маршрутизирует** — не содержит бизнес-логику
2. **Service публикует события** — через EventBus
3. **Listener отправляет через WebSocket** — не напрямую из Service
4. **Таргетированная рассылка** — только в нужные rooms
5. **Не broadcast всем** — только заинтересованным пользователям
