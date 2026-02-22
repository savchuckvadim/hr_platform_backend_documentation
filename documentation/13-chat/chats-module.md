# Chats Module - Модуль управления чатами

## Обзор

Модуль для управления чатами на HR Platform. Создание чатов, управление участниками, получение списка чатов пользователя.

## Структура модуля

```
chats/
├── api/
│   ├── controllers/
│   │   └── chats.controller.ts
│   └── dto/
│       ├── chat.dto.ts
│       ├── create-chat.dto.ts
│       └── add-member.dto.ts
├── application/
│   └── services/
│       └── chats.service.ts
├── infrastructure/
│   └── repositories/
│       ├── chats.repository.ts
│       └── chats.prisma.repository.ts
├── chats.module.ts
└── index.ts
```

## DTO

### CreateChatDto

```typescript
// api/dto/create-chat.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEnum, IsOptional, IsString, IsArray, IsUUID } from 'class-validator';
import { ChatType } from '@prisma/client';

export class CreateChatDto {
    @ApiProperty({
        description: 'Chat type',
        enum: ChatType,
        example: ChatType.DIRECT
    })
    @IsEnum(ChatType)
    type: ChatType;

    @ApiProperty({
        description: 'Reply ID (for APPLICATION type chat - renamed from applicationId)',
        required: false
    })
    @IsOptional()
    @IsUUID()
    replyId?: string; // Renamed from applicationId

    @ApiProperty({
        description: 'Member IDs (excluding creator)',
        type: [String],
        example: ['user-id-1', 'user-id-2']
    })
    @IsArray()
    @IsUUID('4', { each: true })
    memberIds: string[];

    @ApiProperty({ description: 'Chat name (for GROUP type)', required: false })
    @IsOptional()
    @IsString()
    name?: string;

    @ApiProperty({ description: 'Chat description', required: false })
    @IsOptional()
    @IsString()
    description?: string;
}
```

### ChatDto

```typescript
// api/dto/chat.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { Chat, ChatMember, ChatType } from '@prisma/client';
import { ChatMemberWithUser } from '../types/chat-member-with-user.type';
import { MessageDto } from '@messages/dto/message.dto';

export class ChatDto {
    id: string;
    type: ChatType;
    name?: string;
    description?: string;
    avatar?: string;
    replyId?: string; // Renamed from applicationId
    createdBy: string;
    createdAt: Date;
    updatedAt: Date;
    members?: ChatMemberDto[];
    unreadCount?: number;
    lastMessage?: MessageDto;

    constructor(
        chat: Chat & {
            members?: ChatMemberWithUser[];
            unreadCount?: number;
            lastMessage?: any;
        }
    ) {
        this.id = chat.id;
        this.type = chat.type;
        this.name = chat.name || undefined;
        this.description = chat.description || undefined;
        this.avatar = chat.avatar || undefined;
        this.applicationId = (chat as any).applicationId || undefined;
        this.createdBy = chat.createdBy;
        this.createdAt = chat.createdAt;
        this.updatedAt = chat.updatedAt;
        this.members = chat.members?.map(m => new ChatMemberDto(m));
        this.unreadCount = chat.unreadCount;
        this.lastMessage = chat.lastMessage ? new MessageDto(chat.lastMessage) : undefined;
    }
}
```

## Service

### ChatsService

```typescript
// application/services/chats.service.ts
import { Injectable, ForbiddenException, NotFoundException } from '@nestjs/common';
import { ChatsRepository } from '../infrastructure/repositories/chats.repository';
import { CreateChatDto, ChatDto, AddMemberDto } from '../api/dto';
import { ChatType } from '@prisma/client';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import { ApplicationRepository } from '@applications/infrastructure/repositories/application.repository';
import { RoleContextRepository } from '@auth/infrastructure/repositories/role-context.repository';

@Injectable()
export class ChatsService {
    constructor(
        private readonly repository: ChatsRepository,
        private readonly eventBus: AppEventBus,
        private readonly applicationRepository: ApplicationRepository,
        private readonly roleContextRepository: RoleContextRepository,
    ) {}

    /**
     * Создание чата
     */
    async createChat(userId: string, createChatDto: CreateChatDto): Promise<ChatDto> {
        // Добавляем текущего пользователя в список участников
        const memberIds = [...createChatDto.memberIds];
        if (!memberIds.includes(userId)) {
            memberIds.push(userId);
        }

        // Для приватного чата должно быть ровно 2 участника
        if (createChatDto.type === ChatType.DIRECT && memberIds.length !== 2) {
            throw new ForbiddenException('Direct chat must have exactly 2 members');
        }

        // Для APPLICATION чата проверяем права и связь с откликом
        if (createChatDto.type === ChatType.APPLICATION) {
            if (!createChatDto.applicationId) {
                throw new ForbiddenException('Application ID is required for APPLICATION type chat');
            }

            const reply = await this.replyRepository.findById(createChatDto.replyId);
            if (!application) {
                throw new NotFoundException('Application not found');
            }

            // Проверяем, что пользователь является участником отклика
            // (кандидат или HR компании)
            const isParticipant = await this.validateApplicationParticipant(
                userId,
                application,
            );
            if (!isParticipant) {
                throw new ForbiddenException('You are not a participant of this application');
            }

            // Для APPLICATION чата участники - кандидат и HR компании
            const participantIds = await this.getApplicationParticipants(application);
            memberIds.push(...participantIds.filter(id => !memberIds.includes(id)));
        }

        const chat = await this.repository.create({
            ...createChatDto,
            memberIds,
            createdBy: userId,
        });

        // Публикуем событие
        this.eventBus.emit(AppEvent.CHAT_CREATED, {
            chatId: chat.id,
            type: chat.type,
            memberIds: chat.members?.map(m => m.userId) || [],
        });

        return new ChatDto(chat);
    }

    /**
     * Получение чатов пользователя
     */
    async getUserChats(userId: string): Promise<ChatDto[]> {
        const chats = await this.repository.findByUserId(userId);
        return chats.map(chat => new ChatDto(chat));
    }

    /**
     * Получение чата по ID
     */
    async getChatById(chatId: string, userId: string): Promise<ChatDto> {
        const chat = await this.repository.findById(chatId, userId);
        if (!chat) {
            throw new NotFoundException('Chat not found');
        }

        // Проверяем, что пользователь является участником
        const isMember = await this.repository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        return new ChatDto(chat);
    }

    /**
     * Добавление участника в чат
     */
    async addMember(chatId: string, userId: string, addMemberDto: AddMemberDto): Promise<void> {
        // Проверяем, что пользователь является участником чата
        const isMember = await this.repository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        await this.repository.addMember(chatId, addMemberDto.userId, addMemberDto.role);

        this.eventBus.emit(AppEvent.CHAT_MEMBER_ADDED, {
            chatId,
            userId: addMemberDto.userId,
        });
    }

    /**
     * Удаление участника из чата
     */
    async removeMember(chatId: string, userId: string, memberId: string): Promise<void> {
        // Проверяем права доступа
        const isMember = await this.repository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        // Пользователь может удалить только себя, если он не владелец
        if (userId !== memberId) {
            // TODO: Проверить, является ли пользователь владельцем/админом
        }

        await this.repository.removeMember(chatId, memberId);

        this.eventBus.emit(AppEvent.CHAT_MEMBER_REMOVED, {
            chatId,
            userId: memberId,
        });
    }

    /**
     * Обновление чата
     */
    async updateChat(
        chatId: string,
        userId: string,
        data: Partial<{ name: string; description: string; avatar: string }>,
    ): Promise<ChatDto> {
        const isMember = await this.repository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        const chat = await this.repository.update(chatId, data);
        return new ChatDto(chat);
    }

    /**
     * Отметить чат как прочитанный
     */
    async markAsRead(chatId: string, userId: string): Promise<void> {
        const isMember = await this.repository.isMember(chatId, userId);
        if (!isMember) {
            throw new ForbiddenException('You are not a member of this chat');
        }

        await this.repository.updateLastRead(chatId, userId);
    }

    /**
     * Удаление чата
     */
    async deleteChat(chatId: string, userId: string): Promise<void> {
        const chat = await this.repository.findById(chatId, userId);

        // Только создатель может удалить чат
        if (chat.createdBy !== userId) {
            throw new ForbiddenException('Only chat creator can delete the chat');
        }

        await this.repository.delete(chatId);

        this.eventBus.emit(AppEvent.CHAT_DELETED, {
            chatId,
        });
    }

    /**
     * Проверка участника отклика
     */
    private async validateApplicationParticipant(
        userId: string,
        application: any,
    ): Promise<boolean> {
        // Кандидат
        if (reply.candidateId === userId) {
            return true;
        }

        // HR компании (root или нанятый)
        const roleContext = await this.roleContextRepository.findActiveByUserId(userId);
        if (roleContext?.type === 'EMPLOYER' && roleContext.companyId) {
            // Получаем компанию из вакансии
            const vacancy = application.vacancy;
            const companyId = vacancy.project.companyId;
            return roleContext.companyId === companyId;
        }

        return false;
    }

    /**
     * Получение участников отклика
     */
    private async getApplicationParticipants(application: any): Promise<string[]> {
        const participants: string[] = [];

        // Кандидат
        participants.push(reply.candidateId);

        // HR компании (root или нанятый)
        // Получаем компанию из вакансии
        const vacancy = application.vacancy;
        const companyId = vacancy.project.companyId;

        // Получаем всех HR компании
        const hrRoleContexts = await this.roleContextRepository.findByCompanyId(companyId);
        const hrUserIds = hrRoleContexts.map(rc => rc.userId);
        participants.push(...hrUserIds);

        return participants;
    }
}
```

## Controller

### ChatsController

```typescript
// api/controllers/chats.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { ChatsService } from '../application/services/chats.service';
import { CreateChatDto, ChatDto, AddMemberDto } from '../api/dto';

@ApiTags('Chats')
@Controller('chats')
@UseGuards(JwtAuthGuard)
export class ChatsController {
    constructor(private readonly chatsService: ChatsService) {}

    @Post()
    @ApiOperation({ summary: 'Create a new chat' })
    @ApiResponse({ status: 201, description: 'Chat created', type: ChatDto })
    async createChat(
        @Body() dto: CreateChatDto,
        @CurrentUser() user: { id: string },
    ): Promise<ChatDto> {
        return this.chatsService.createChat(user.id, dto);
    }

    @Get()
    @ApiOperation({ summary: 'Get user chats' })
    @ApiResponse({ status: 200, description: 'User chats', type: [ChatDto] })
    async getUserChats(@CurrentUser() user: { id: string }): Promise<ChatDto[]> {
        return this.chatsService.getUserChats(user.id);
    }

    @Get(':id')
    @ApiOperation({ summary: 'Get chat by ID' })
    @ApiResponse({ status: 200, description: 'Chat', type: ChatDto })
    async getChatById(
        @Param('id') id: string,
        @CurrentUser() user: { id: string },
    ): Promise<ChatDto> {
        return this.chatsService.getChatById(id, user.id);
    }

    @Post(':id/members')
    @ApiOperation({ summary: 'Add member to chat' })
    async addMember(
        @Param('id') chatId: string,
        @Body() dto: AddMemberDto,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.chatsService.addMember(chatId, user.id, dto);
    }

    @Delete(':id/members/:memberId')
    @ApiOperation({ summary: 'Remove member from chat' })
    async removeMember(
        @Param('id') chatId: string,
        @Param('memberId') memberId: string,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.chatsService.removeMember(chatId, user.id, memberId);
    }

    @Put(':id')
    @ApiOperation({ summary: 'Update chat' })
    async updateChat(
        @Param('id') chatId: string,
        @Body() data: Partial<{ name: string; description: string; avatar: string }>,
        @CurrentUser() user: { id: string },
    ): Promise<ChatDto> {
        return this.chatsService.updateChat(chatId, user.id, data);
    }

    @Post(':id/read')
    @ApiOperation({ summary: 'Mark chat as read' })
    async markAsRead(
        @Param('id') chatId: string,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.chatsService.markAsRead(chatId, user.id);
    }

    @Delete(':id')
    @ApiOperation({ summary: 'Delete chat' })
    async deleteChat(
        @Param('id') chatId: string,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.chatsService.deleteChat(chatId, user.id);
    }
}
```

## Repository

### ChatsRepository

```typescript
// infrastructure/repositories/chats.repository.ts
import { Chat, ChatMember, ChatType } from '@prisma/client';
import { ChatMemberWithUser } from '../types/chat-member-with-user.type';

export abstract class ChatsRepository {
    abstract findById(id: string, userId?: string): Promise<Chat & { members?: ChatMemberWithUser[] }>;
    abstract findByUserId(userId: string): Promise<(Chat & { members?: ChatMemberWithUser[] })[]>;
    abstract findPrivateChat(userId1: string, userId2: string): Promise<Chat | null>;
    abstract create(data: {
        type: ChatType;
        replyId?: string; // Renamed from applicationId
        createdBy: string;
        name?: string;
        description?: string;
        memberIds: string[];
    }): Promise<Chat & { members: ChatMemberWithUser[] }>;
    abstract addMember(chatId: string, userId: string, role?: string): Promise<ChatMemberWithUser>;
    abstract removeMember(chatId: string, userId: string): Promise<void>;
    abstract update(chatId: string, data: Partial<Chat>): Promise<Chat>;
    abstract delete(chatId: string): Promise<void>;
    abstract isMember(chatId: string, userId: string): Promise<boolean>;
    abstract updateLastRead(chatId: string, userId: string): Promise<void>;
}
```

## Best Practices

1. **Проверяйте права доступа** - перед операциями
2. **Валидируйте участников** - для APPLICATION чатов
3. **Публикуйте события** - для уведомлений
4. **Используйте soft delete** - для истории
5. **Отслеживайте lastReadAt** - для непрочитанных сообщений
