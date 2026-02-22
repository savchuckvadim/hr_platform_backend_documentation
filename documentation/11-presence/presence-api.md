# Presence API - REST API endpoints

## Обзор

REST API endpoints для получения статусов присутствия пользователей. Используется для получения статусов без WebSocket соединения.

## Расположение

```
presence/api/controllers/presence.controller.ts
```

## Endpoints

### GET /presence/:userId

Получение статуса конкретного пользователя.

**Параметры:**
- `userId` (path) - ID пользователя

**Ответ:**
```typescript
{
  userId: string;
  isOnline: boolean;
  lastSeen?: string; // ISO 8601 (если оффлайн)
}
```

**Пример:**
```typescript
GET /presence/user-123

Response 200:
{
  "userId": "user-123",
  "isOnline": true
}
```

### POST /presence/bulk

Получение статусов нескольких пользователей.

**Тело запроса:**
```typescript
{
  userIds: string[];
}
```

**Ответ:**
```typescript
{
  online: string[];  // Массив ID онлайн пользователей
  offline: string[]; // Массив ID оффлайн пользователей
}
```

**Пример:**
```typescript
POST /presence/bulk
{
  "userIds": ["user-1", "user-2", "user-3"]
}

Response 200:
{
  "online": ["user-1", "user-3"],
  "offline": ["user-2"]
}
```

## Реализация

### Controller

```typescript
import { Controller, Get, Post, Body, Param, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { JwtAuthGuard } from '@auth/infrastructure/guards/jwt-auth.guard';
import { PresenceService } from '../../application/services/presence.service';
import { BulkPresenceDto } from '../dto/bulk-presence.dto';

@ApiTags('Presence')
@Controller('presence')
@UseGuards(JwtAuthGuard)
export class PresenceController {
  constructor(
    private readonly presenceService: PresenceService,
  ) {}

  @Get(':userId')
  @ApiOperation({ summary: 'Get user presence status' })
  @ApiResponse({
    status: 200,
    description: 'User presence status',
    schema: {
      type: 'object',
      properties: {
        userId: { type: 'string' },
        isOnline: { type: 'boolean' },
        lastSeen: { type: 'string', format: 'date-time' },
      },
    },
  })
  async getUserPresence(@Param('userId') userId: string) {
    const isOnline = await this.presenceService.isOnline(userId);

    return {
      userId,
      isOnline,
      ...(isOnline ? {} : { lastSeen: null }), // TODO: реализовать lastSeen
    };
  }

  @Post('bulk')
  @ApiOperation({ summary: 'Get presence status for multiple users' })
  @ApiResponse({
    status: 200,
    description: 'Bulk presence status',
    schema: {
      type: 'object',
      properties: {
        online: { type: 'array', items: { type: 'string' } },
        offline: { type: 'array', items: { type: 'string' } },
      },
    },
  })
  async getBulkPresence(@Body() dto: BulkPresenceDto) {
    const onlineUsers = await this.presenceService.getOnlineUsers(dto.userIds);

    const online = Array.from(onlineUsers);
    const offline = dto.userIds.filter(id => !onlineUsers.has(id));

    return {
      online,
      offline,
    };
  }
}
```

### DTO

```typescript
// api/dto/bulk-presence.dto.ts
import { IsArray, IsString, ArrayMinSize, ArrayMaxSize } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class BulkPresenceDto {
  @ApiProperty({
    description: 'Array of user IDs',
    example: ['user-1', 'user-2', 'user-3'],
    minItems: 1,
    maxItems: 100,
  })
  @IsArray()
  @ArrayMinSize(1)
  @ArrayMaxSize(100)
  @IsString({ each: true })
  userIds: string[];
}
```

## Использование

### Получение статуса одного пользователя

```typescript
// Frontend
const response = await fetch('/api/presence/user-123', {
  headers: {
    'Authorization': `Bearer ${token}`,
  },
});

const data = await response.json();
// { userId: 'user-123', isOnline: true }
```

### Получение статусов нескольких пользователей

```typescript
// Frontend
const response = await fetch('/api/presence/bulk', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    userIds: ['user-1', 'user-2', 'user-3'],
  }),
});

const data = await response.json();
// { online: ['user-1', 'user-3'], offline: ['user-2'] }
```

## Best Practices

1. **Используйте bulk endpoint** - для получения статусов нескольких пользователей
2. **Ограничивайте количество** - максимум 100 пользователей за запрос
3. **Кэшируйте результаты** - на клиенте для снижения нагрузки
4. **Используйте WebSocket** - для real-time обновлений, а не polling
