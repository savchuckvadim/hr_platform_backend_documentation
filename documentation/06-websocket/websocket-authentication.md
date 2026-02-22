# WebSocket Authentication - JWT аутентификация

## Обзор

JWT аутентификация для WebSocket соединений. Обеспечивает безопасное подключение клиентов через проверку токенов.

## Расположение

```
core/websocket/infrastructure/guards/ws-jwt-auth.guard.ts
```

## Проблема без JWT

**❌ Плохо (небезопасно):**

```typescript
async handleConnection(client: Socket) {
  const userId = client.handshake.query.userId; // ❌ Небезопасно!

  // Проблемы:
  // 1. Любой может подделать userId
  // 2. Нет проверки токена
  // 3. Нет информации о роли
  // 4. Нет информации о deviceId
}
```

## Реализация

### WsJwtAuthGuard

```typescript
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
  Logger,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { WsException } from '@nestjs/websockets';
import { Socket } from 'socket.io';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class WsJwtAuthGuard implements CanActivate {
  private readonly logger = new Logger(WsJwtAuthGuard.name);

  constructor(
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const client: Socket = context.switchToWs().getClient();

    try {
      // Получение токена из handshake
      const token = this.extractToken(client);

      if (!token) {
        throw new UnauthorizedException('Token not provided');
      }

      // Верификация токена
      const payload = await this.jwtService.verifyAsync(token, {
        secret: this.configService.get<string>('JWT_SECRET'),
      });

      // Прикрепление пользователя к сокету
      client.data.user = {
        id: payload.sub, // userId
        roleContextId: payload.roleContextId,
        roleType: payload.roleType,
        companyId: payload.companyId,
        employerRoleType: payload.employerRoleType,
      };

      // Прикрепление deviceId (если есть)
      if (client.handshake.query.deviceId) {
        client.data.deviceId = client.handshake.query.deviceId;
      }

      return true;
    } catch (error) {
      this.logger.warn(`WebSocket authentication failed: ${error.message}`);
      throw new WsException('Unauthorized');
    }
  }

  private extractToken(client: Socket): string | null {
    // 1. Пробуем получить из auth.token (рекомендуется)
    if (client.handshake.auth?.token) {
      return client.handshake.auth.token;
    }

    // 2. Пробуем получить из query параметров (fallback)
    if (client.handshake.query?.token) {
      return client.handshake.query.token as string;
    }

    // 3. Пробуем получить из заголовков (fallback)
    const authHeader = client.handshake.headers.authorization;
    if (authHeader && authHeader.startsWith('Bearer ')) {
      return authHeader.substring(7);
    }

    return null;
  }
}
```

### Использование в Gateway

```typescript
@WebSocketGateway({
  namespace: '/',
})
export class AppWebSocketGateway implements OnGatewayConnection {
  constructor(
    private readonly wsJwtAuthGuard: WsJwtAuthGuard,
  ) {}

  async handleConnection(client: Socket) {
    try {
      // ✅ Проверка аутентификации
      const context = this.createContext(client);
      await this.wsJwtAuthGuard.canActivate(context);

      // После успешной аутентификации user доступен в client.data.user
      const user = client.data.user;

      this.logger.log(`User ${user.id} connected`);
    } catch (error) {
      this.logger.warn(`Connection rejected: ${error.message}`);
      client.disconnect();
    }
  }

  private createContext(client: Socket): ExecutionContext {
    return {
      switchToWs: () => ({
        getClient: () => client,
        getData: () => ({}),
      }),
    } as ExecutionContext;
  }
}
```

### Альтернатива: Middleware

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Socket } from 'socket.io';

@Injectable()
export class WsJwtMiddleware implements NestMiddleware {
  constructor(
    private readonly jwtService: JwtService,
  ) {}

  use(socket: Socket, next: (err?: Error) => void) {
    try {
      const token = this.extractToken(socket);

      if (!token) {
        return next(new Error('Token not provided'));
      }

      const payload = this.jwtService.verify(token);

      socket.data.user = {
        id: payload.sub,
        roleContextId: payload.roleContextId,
        roleType: payload.roleType,
      };

      next();
    } catch (error) {
      next(new Error('Unauthorized'));
    }
  }

  private extractToken(socket: Socket): string | null {
    return (
      socket.handshake.auth?.token ||
      socket.handshake.query?.token ||
      null
    );
  }
}
```

## Клиентская часть

### Подключение с токеном

**JavaScript/TypeScript:**

```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000', {
  auth: {
    token: 'your-jwt-token', // ✅ Рекомендуется
  },
  query: {
    deviceId: 'device-123', // Опционально
    deviceName: 'Chrome on Windows', // Опционально
  },
  transports: ['websocket', 'polling'],
});

socket.on('connect', () => {
  console.log('Connected!');
});

socket.on('connect_error', (error) => {
  console.error('Connection error:', error.message);
});
```

**React:**

```typescript
import { useEffect } from 'react';
import { io } from 'socket.io-client';

function useWebSocket(token: string) {
  useEffect(() => {
    const socket = io(process.env.REACT_APP_WS_URL, {
      auth: { token },
      query: {
        deviceId: localStorage.getItem('deviceId') || generateDeviceId(),
      },
    });

    socket.on('connect', () => {
      console.log('WebSocket connected');
    });

    return () => {
      socket.disconnect();
    };
  }, [token]);
}
```

## Структура JWT payload

JWT должен содержать следующую информацию:

```typescript
interface JwtPayload {
  sub: string;              // userId
  roleContextId: string;     // ID активного role context
  roleType: 'CANDIDATE' | 'EMPLOYER' | 'ADMIN';
  companyId?: string | null; // ID компании (для EMPLOYER)
  employerRoleType?: 'HR' | 'HR_ADMIN' | null; // Тип роли работодателя
  iat: number;              // issued at
  exp: number;              // expiration
}
```

После верификации токена эта информация доступна в `client.data.user`:

```typescript
// В Gateway или Guard
const user = client.data.user;
// {
//   id: 'user-123',
//   roleContextId: 'role-456',
//   roleType: 'EMPLOYER',
//   companyId: 'company-789',
//   employerRoleType: 'HR_ADMIN'
// }
```

## Защита событий

Все события должны быть защищены через guard:

```typescript
@UseGuards(WsJwtAuthGuard)
@SubscribeMessage('chat:send')
async handleSendMessage(
  @ConnectedSocket() client: Socket,
  @MessageBody() payload: SendMessageDto,
) {
  // ✅ client.data.user доступен после успешной аутентификации
  const user = client.data.user;

  return this.chatService.handleSendMessage({
    userId: user.id,
    roleContextId: user.roleContextId,
    ...payload,
  });
}
```

## Обработка ошибок

### Отклонение соединения

```typescript
async handleConnection(client: Socket) {
  try {
    await this.wsJwtAuthGuard.canActivate(context);
  } catch (error) {
    this.logger.warn(`Connection rejected: ${error.message}`);
    client.emit('error', { message: 'Authentication failed' });
    client.disconnect();
  }
}
```

### Отправка ошибки клиенту

```typescript
@UseGuards(WsJwtAuthGuard)
@SubscribeMessage('chat:send')
async handleSendMessage(
  @ConnectedSocket() client: Socket,
  @MessageBody() payload: SendMessageDto,
) {
  try {
    return await this.chatService.handleSendMessage({...});
  } catch (error) {
    client.emit('error', {
      message: error.message,
      code: error.code,
    });
    throw new WsException(error.message);
  }
}
```

## Best Practices

1. **Всегда используйте JWT для аутентификации** — не полагайтесь на query параметры
2. **Используйте auth.token** — более безопасно, чем query параметры
3. **Проверяйте токен при подключении** — не откладывайте проверку
4. **Обрабатывайте ошибки** — отправляйте понятные сообщения клиенту
5. **Логируйте неудачные попытки** — для мониторинга безопасности
6. **Используйте refresh token** — для обновления access token при истечении

## Обновление токена

Если токен истек, клиент должен обновить его:

```typescript
socket.on('disconnect', (reason) => {
  if (reason === 'io server disconnect') {
    // Сервер отключил из-за истечения токена
    // Обновляем токен и переподключаемся
    refreshAccessToken().then((newToken) => {
      socket.auth.token = newToken;
      socket.connect();
    });
  }
});
```

## Безопасность

1. **HTTPS/WSS** — всегда используйте зашифрованное соединение
2. **Валидация токена** — проверяйте подпись и срок действия
3. **Rate limiting** — ограничивайте количество попыток подключения
4. **Логирование** — логируйте все неудачные попытки аутентификации
5. **Мониторинг** — отслеживайте подозрительную активность
