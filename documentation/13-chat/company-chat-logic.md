# Company Chat Logic - Логика общения от компании

## Обзор

Логика определения, кто отправляет сообщения от компании: root HR (владелец) или нанятый HR сотрудник.

## Принципы

### Кто может отправлять сообщения от компании

1. **Root HR (HR_ADMIN)** - владелец компании, может отправлять сообщения от имени компании
2. **Нанятый HR** - HR сотрудник компании, может отправлять сообщения от имени компании

### Определение отправителя

При отправке сообщения в APPLICATION чате:
- `senderId` = конкретный User ID (HR, который отправил)
- В UI отображается: "От [Имя HR] (Компания)"

## Реализация

### Определение HR для чата

```typescript
// application/services/chats.service.ts

/**
 * Получение HR компании для чата по отклику
 */
private async getCompanyHrForChat(application: any): Promise<string | null> {
    const vacancy = application.vacancy;
    const companyId = vacancy.project.companyId;

    // Получаем всех HR компании
    const hrRoleContexts = await this.roleContextRepository.findByCompanyId(companyId);

    if (hrRoleContexts.length === 0) {
        return null;
    }

    // Приоритет: root HR (HR_ADMIN) > нанятый HR
    const rootHr = hrRoleContexts.find(rc => rc.employerRoleType === 'HR_ADMIN');
    if (rootHr) {
        return rootHr.userId;
    }

    // Если нет root HR, берем первого нанятого HR
    return hrRoleContexts[0].userId;
}
```

### Отображение отправителя

```typescript
// В MessageDto
export class MessageDto {
    // ...
    sender: {
        id: string;
        name: string;
        email: string;
        avatar?: string;
        // Для HR компаний
        company?: {
            id: string;
            name: string;
        };
        roleContext?: {
            type: RoleType;
            employerRoleType?: EmployerRoleType;
        };
    };
    // ...
}
```

### В Repository

```typescript
// infrastructure/repositories/messages.prisma.repository.ts

async create(data: CreateMessageData): Promise<MessageWithRelations> {
    const message = await this.prisma.message.create({
        data: {
            chatId: data.chatId,
            senderId: data.senderId,
            content: data.content,
            type: data.type || MessageType.TEXT,
            fileUrl: data.fileUrl,
            fileName: data.fileName,
            fileSize: data.fileSize,
            replyToId: data.replyToId,
        },
        include: {
            sender: {
                include: {
                    roleContexts: {
                        include: {
                            company: true,
                        },
                    },
                },
            },
            replyTo: {
                include: {
                    sender: true,
                },
            },
        },
    });

    // Обогащаем данными о компании, если отправитель - HR
    const roleContext = message.sender.roleContexts?.find(
        rc => rc.type === 'EMPLOYER' && rc.companyId,
    );

    return {
        ...message,
        sender: {
            ...message.sender,
            company: roleContext?.company ? {
                id: roleContext.company.id,
                name: roleContext.company.name,
            } : undefined,
            roleContext: roleContext ? {
                type: roleContext.type,
                employerRoleType: roleContext.employerRoleType,
            } : undefined,
        },
    };
}
```

## UI отображение

### Для кандидата

```
[Имя HR] (Компания: ООО Пирожок)
Привет! Мы рассмотрели ваше резюме...
```

### Для HR

```
[Имя HR] (Вы - HR_ADMIN)
Привет! Мы рассмотрели ваше резюме...
```

## Best Practices

1. **Храните senderId** - конкретный User ID HR
2. **Обогащайте данными** - о компании в запросах
3. **Показывайте роль** - HR_ADMIN vs HR в UI
4. **Проверяйте права** - HR может писать только в чаты своей компании
