# Messages Module - Модуль сообщений

## Обзор

Модуль для отправки и получения сообщений в чатах на HR Platform. Поддержка текстовых сообщений, файлов, ответов на сообщения. Интеграция с WebSocket для real-time доставки.

## Структура модуля

```
messages/
├── api/
│   ├── controllers/
│   │   └── messages.controller.ts
│   └── dto/
│       ├── message.dto.ts
│       └── create-message.dto.ts
├── application/
│   └── services/
│       └── messages.service.ts
├── infrastructure/
│   ├── repositories/
│   │   ├── messages.repository.ts
│   │   └── messages.prisma.repository.ts
│   └── socket/
│       ├── messages.gateway.ts
│       └── socket-storage.service.ts
├── messages.module.ts
└── index.ts
```

## DTO

### CreateMessageDto

```typescript
// api/dto/create-message.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsString, IsEnum, IsOptional, IsUUID, IsNumber } from 'class-validator';
import { MessageType } from '@prisma/client';

export class CreateMessageDto {
    @ApiProperty({
        description: 'Chat ID',
        example: '123e4567-e89b-12d3-a456-426614174000'
    })
    @IsUUID('4')
    chatId: string;

    @ApiProperty({
        description: 'Content',
        example: 'Hello, how are you?',
        required: false
    })
    @IsOptional()
    @IsString()
    content?: string;

    @ApiProperty({
        description: 'Type',
        enum: MessageType,
        default: MessageType.TEXT,
        required: false
    })
    @IsEnum(MessageType)
    @IsOptional()
    type?: MessageType;

    @ApiProperty({
        description: 'File URL',
        example: 'https://example.com/file.jpg',
        required: false
    })
    @IsOptional()
    @IsString()
    fileUrl?: string;

    @ApiProperty({
        description: 'File Name',
        example: 'file.jpg',
        required: false
    })
    @IsOptional()
    @IsString()
    fileName?: string;

    @ApiProperty({
        description: 'File Size',
        example: 1000,
        required: false
    })
    @IsOptional()
    @IsNumber()
    fileSize?: number;

    @ApiProperty({
        description: 'Reply To ID',
        example: '123e4567-e89b-12d3-a456-426614174000',
        required: false
    })
    @IsOptional()
    @IsUUID('4')
    replyToId?: string;
}
```

### MessageDto

```typescript
// api/dto/message.dto.ts
import { Message, MessageType } from '@prisma/client';

export class MessageDto {
    id: string;
    chatId: string;
    senderId: string;
    content: string;
    type: MessageType;
    fileUrl?: string;
    fileName?: string;
    fileSize?: number;
    replyToId?: string;
    replyTo?: MessageDto;
    editedAt?: Date;
    deletedAt?: Date;
    createdAt: Date;
    updatedAt: Date;
    sender?: {
        id: string;
        name: string;
        email: string;
        avatar?: string;
    };
    readBy?: string[];

    constructor(message: Message & {
        sender?: any;
        replyTo?: Message;
        readBy?: string[];
    }) {
        this.id = message.id;
        this.chatId = message.chatId;
        this.senderId = message.senderId;
        this.content = message.content;
        this.type = message.type;
        this.fileUrl = message.fileUrl || undefined;
        this.fileName = message.fileName || undefined;
        this.fileSize = message.fileSize || undefined;
        this.replyToId = message.replyToId || undefined;
        this.replyTo = message.replyTo ? new MessageDto(message.replyTo) : undefined;
        this.editedAt = message.editedAt || undefined;
        this.deletedAt = message.deletedAt || undefined;
        this.createdAt = message.createdAt;
        this.updatedAt = message.updatedAt;
        this.sender = message.sender;
        this.readBy = message.readBy || [];
    }
}
```

## Service

### MessagesService

```typescript
// application/services/messages.service.ts
import { Injectable, ForbiddenException, NotFoundException } from '@nestjs/common';
import { MessagesRepository } from '../infrastructure/repositories/messages.repository';
import { ChatsRepository } from '@chats/infrastructure/repositories/chats.repository';
import { CreateMessageDto, MessageDto } from '../api/dto';
import { MessagesGateway } from '../infrastructure/socket/messages.gateway';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';

@Injectable()
export class MessagesService {
    constructor(
        private readonly repository: MessagesRepository,
        private readonly chatsRepository: ChatsRepository,
        private readonly messagesGateway: MessagesGateway,
        private readonly eventBus: AppEventBus,
    ) {}

    /**
     * Создание сообщения
     */
    async createMessage(userId: string, createMessageDto: CreateMessageDto): Promise<MessageDto> {
        // Проверяем, что пользователь является участником чата
        const isMember = await this.chatsRepository.isMember(createMessageDto.chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        // Валидация: должен быть либо content, либо fileUrl
        if (!createMessageDto.content && !createMessageDto.fileUrl) {
            throw new ForbiddenException('Message must have either content or file');
        }

        const message = await this.repository.create({
            ...createMessageDto,
            senderId: userId,
        });

        const messageDto = new MessageDto(message);

        // Отправляем сообщение через WebSocket всем участникам чата
        await this.messagesGateway.broadcastMessage(messageDto, createMessageDto.chatId, userId);

        // Публикуем событие для уведомлений
        this.eventBus.emit(AppEvent.MESSAGE_CREATED, {
            messageId: message.id,
            chatId: createMessageDto.chatId,
            senderId: userId,
        });

        return messageDto;
    }

    /**
     * Получение сообщений чата
     */
    async getChatMessages(
        chatId: string,
        userId: string,
        limit?: number,
        offset?: number,
    ): Promise<MessageDto[]> {
        // Проверяем, что пользователь является участником чата
        const isMember = await this.chatsRepository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        const messages = await this.repository.findByChatId(chatId, limit, offset);
        return messages.map(msg => new MessageDto(msg));
    }

    /**
     * Получение сообщения по ID
     */
    async getMessageById(messageId: string, userId: string): Promise<MessageDto> {
        const message = await this.repository.findById(messageId);

        // Проверяем, что пользователь является участником чата
        const isMember = await this.chatsRepository.isMember(message.chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        return new MessageDto(message);
    }

    /**
     * Обновление сообщения
     */
    async updateMessage(messageId: string, userId: string, content: string): Promise<MessageDto> {
        const message = await this.repository.findById(messageId);

        // Только отправитель может редактировать сообщение
        if (message.senderId !== userId) {
            throw new ForbiddenException('You can only edit your own messages');
        }

        const updated = await this.repository.update(messageId, content);
        const messageDto = new MessageDto(updated);

        // Отправляем обновленное сообщение через WebSocket
        await this.messagesGateway.broadcastMessage(
            messageDto,
            message.chatId,
            userId,
        );

        return messageDto;
    }

    /**
     * Удаление сообщения
     */
    async deleteMessage(messageId: string, userId: string): Promise<MessageDto> {
        const message = await this.repository.findById(messageId);

        // Только отправитель может удалить сообщение
        if (message.senderId !== userId) {
            throw new ForbiddenException('You can only delete your own messages');
        }

        const deleted = await this.repository.delete(messageId);
        const messageDto = new MessageDto(deleted);

        // Отправляем удаленное сообщение через WebSocket
        await this.messagesGateway.broadcastMessage(
            messageDto,
            message.chatId,
            userId,
        );

        return messageDto;
    }

    /**
     * Отметить сообщение как прочитанное
     */
    async markAsRead(messageId: string, userId: string): Promise<void> {
        const message = await this.repository.findById(messageId);

        // Проверяем, что пользователь является участником чата
        const isMember = await this.chatsRepository.isMember(message.chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        await this.repository.markAsRead(messageId, userId);
    }

    /**
     * Отметить все сообщения чата как прочитанные
     */
    async markChatAsRead(chatId: string, userId: string): Promise<void> {
        // Проверяем, что пользователь является участником чата
        const isMember = await this.chatsRepository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        await this.repository.markChatMessagesAsRead(chatId, userId);
    }

    /**
     * Получение количества непрочитанных сообщений
     */
    async getUnreadCount(chatId: string, userId: string): Promise<number> {
        // Проверяем, что пользователь является участником чата
        const isMember = await this.chatsRepository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        return this.repository.getUnreadCount(chatId, userId);
    }
}
```

## Controller

### MessagesController

```typescript
// api/controllers/messages.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiQuery } from '@nestjs/swagger';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { MessagesService } from '../application/services/messages.service';
import { CreateMessageDto, MessageDto } from '../api/dto';

@ApiTags('Messages')
@Controller('messages')
@UseGuards(JwtAuthGuard)
export class MessagesController {
    constructor(private readonly messagesService: MessagesService) {}

    @Post()
    @ApiOperation({ summary: 'Create a new message' })
    @ApiResponse({ status: 201, description: 'Message created', type: MessageDto })
    async createMessage(
        @Body() dto: CreateMessageDto,
        @CurrentUser() user: { id: string },
    ): Promise<MessageDto> {
        return this.messagesService.createMessage(user.id, dto);
    }

    @Get('chat/:chatId')
    @ApiOperation({ summary: 'Get chat messages' })
    @ApiQuery({ name: 'limit', required: false, type: Number })
    @ApiQuery({ name: 'offset', required: false, type: Number })
    @ApiResponse({ status: 200, description: 'Chat messages', type: [MessageDto] })
    async getChatMessages(
        @Param('chatId') chatId: string,
        @Query('limit') limit?: number,
        @Query('offset') offset?: number,
        @CurrentUser() user?: { id: string },
    ): Promise<MessageDto[]> {
        return this.messagesService.getChatMessages(
            chatId,
            user!.id,
            limit ? parseInt(limit.toString()) : undefined,
            offset ? parseInt(offset.toString()) : undefined,
        );
    }

    @Get(':id')
    @ApiOperation({ summary: 'Get message by ID' })
    @ApiResponse({ status: 200, description: 'Message', type: MessageDto })
    async getMessageById(
        @Param('id') id: string,
        @CurrentUser() user: { id: string },
    ): Promise<MessageDto> {
        return this.messagesService.getMessageById(id, user.id);
    }

    @Put(':id')
    @ApiOperation({ summary: 'Update message' })
    @ApiResponse({ status: 200, description: 'Message updated', type: MessageDto })
    async updateMessage(
        @Param('id') id: string,
        @Body() body: { content: string },
        @CurrentUser() user: { id: string },
    ): Promise<MessageDto> {
        return this.messagesService.updateMessage(id, user.id, body.content);
    }

    @Delete(':id')
    @ApiOperation({ summary: 'Delete message' })
    @ApiResponse({ status: 200, description: 'Message deleted', type: MessageDto })
    async deleteMessage(
        @Param('id') id: string,
        @CurrentUser() user: { id: string },
    ): Promise<MessageDto> {
        return this.messagesService.deleteMessage(id, user.id);
    }

    @Post(':id/read')
    @ApiOperation({ summary: 'Mark message as read' })
    async markAsRead(
        @Param('id') id: string,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.messagesService.markAsRead(id, user.id);
    }

    @Post('chat/:chatId/read')
    @ApiOperation({ summary: 'Mark all chat messages as read' })
    async markChatAsRead(
        @Param('chatId') chatId: string,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.messagesService.markChatAsRead(chatId, user.id);
    }

    @Get('chat/:chatId/unread')
    @ApiOperation({ summary: 'Get unread messages count' })
    async getUnreadCount(
        @Param('chatId') chatId: string,
        @CurrentUser() user: { id: string },
    ): Promise<{ count: number }> {
        const count = await this.messagesService.getUnreadCount(chatId, user.id);
        return { count };
    }
}
```

## Repository

### MessagesRepository

```typescript
// infrastructure/repositories/messages.repository.ts
import { Message, MessageType } from '@prisma/client';

export type MessageWithRelations = Message & {
    sender?: {
        id: string;
        name: string;
        email: string;
        avatar?: string;
    };
    replyTo?: Message & {
        sender?: {
            id: string;
            name: string;
            email: string;
        };
    };
    readBy?: string[];
};

export abstract class MessagesRepository {
    abstract findById(id: string): Promise<MessageWithRelations>;
    abstract findByChatId(chatId: string, limit?: number, offset?: number): Promise<MessageWithRelations[]>;
    abstract create(data: {
        chatId: string;
        senderId: string;
        content: string;
        type?: MessageType;
        fileUrl?: string;
        fileName?: string;
        fileSize?: number;
        replyToId?: string;
    }): Promise<Message & { sender?: any }>;
    abstract update(id: string, content: string): Promise<Message>;
    abstract delete(id: string): Promise<Message>;
    abstract markAsRead(messageId: string, userId: string): Promise<void>;
    abstract markChatMessagesAsRead(chatId: string, userId: string): Promise<void>;
    abstract getUnreadCount(chatId: string, userId: string): Promise<number>;
}
```

## Best Practices

1. **Проверяйте права доступа** - перед операциями
2. **Валидируйте данные** - content или fileUrl обязательны
3. **Используйте soft delete** - для истории
4. **Публикуйте события** - для уведомлений
5. **Отслеживайте прочитанные** - через lastReadAt
