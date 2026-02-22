# Chat Module - Модуль чата

## Описание

Модуль для общения между кандидатами и компаниями на HR Platform. Поддержка чатов по откликам, прямых чатов между пользователями, групповых чатов. Интеграция с WebSocket для real-time общения.

## Архитектурные принципы

### Ключевые принципы

1. **Типы чатов** - DIRECT (прямой), APPLICATION (по отклику), GROUP (групповой)
2. **Автор сообщения** - конкретный пользователь (кандидат или HR)
3. **От компании** - сообщения от root HR или нанятого HR
4. **WebSocket интеграция** - real-time доставка сообщений
5. **EventBus интеграция** - публикация событий для уведомлений
6. **Connection Manager** - управление соединениями через единый WebSocket Gateway

## Структура модуля

### Chats Module

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
├── domain/
│   └── entity/
│       └── chat.entity.ts
├── infrastructure/
│   ├── repositories/
│   │   ├── chats.repository.ts
│   │   └── chats.prisma.repository.ts
│   └── listeners/
├── chats.module.ts
└── index.ts
```

### Messages Module

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
├── domain/
│   └── entity/
│       └── message.entity.ts
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

## Задачи

### ✅ Chats Module
**Статус**: Завершено
**Файл**: [chats-module.md](./chats-module.md)
**Описание**: Модуль для управления чатами

### ✅ Messages Module
**Статус**: Завершено
**Файл**: [messages-module.md](./messages-module.md)
**Описание**: Модуль для отправки и получения сообщений

### ✅ WebSocket Integration
**Статус**: Завершено
**Файл**: [websocket-integration.md](./websocket-integration.md)
**Описание**: Интеграция с WebSocket для real-time общения

### ✅ Chat Types
**Статус**: Завершено
**Файл**: [chat-types.md](./chat-types.md)
**Описание**: Типы чатов и их использование

### ✅ Company Chat Logic
**Статус**: Завершено
**Файл**: [company-chat-logic.md](./company-chat-logic.md)
**Описание**: Логика общения от компании (root HR vs нанятый HR)

## Ключевые концепции

- **Chat Types** - DIRECT, APPLICATION, GROUP
- **Chat Members** - участники чата с ролями
- **Messages** - текстовые сообщения, файлы, ответы
- **Real-time** - доставка через WebSocket
- **Read Status** - отслеживание прочитанных сообщений
- **Company Chat** - от root HR или нанятого HR
- **Application Chat** - автоматическое создание при отклике

## Модель данных

### Chat

```prisma
model Chat {
  id            String    @id @default(uuid())
  applicationId String?   @unique @map("application_id")
  type          ChatType  @default(DIRECT)
  name          String?   @db.VarChar(255)
  description   String?   @db.Text
  avatar        String?   @db.VarChar(500)
  createdBy     String    @map("created_by")
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  application Application? @relation(fields: [applicationId], references: [id], onDelete: Cascade)
  members     ChatMember[]
  messages    Message[]

  @@map("chats")
  @@index([applicationId])
  @@index([type])
  @@index([createdBy])
}

enum ChatType {
  DIRECT      // Прямой чат между двумя пользователями
  APPLICATION // Чат по отклику (кандидат ↔ компания)
  GROUP       // Групповой чат (на будущее)
}
```

### ChatMember

```prisma
model ChatMember {
  id        String   @id @default(uuid())
  chatId    String   @map("chat_id")
  userId    String   @map("user_id")
  role      ChatMemberRole @default(MEMBER)
  joinedAt  DateTime @default(now()) @map("joined_at")
  leftAt    DateTime? @map("left_at")
  lastReadAt DateTime? @map("last_read_at")

  chat Chat @relation(fields: [chatId], references: [id], onDelete: Cascade)
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([chatId, userId])
  @@map("chat_members")
  @@index([chatId])
  @@index([userId])
}

enum ChatMemberRole {
  MEMBER
  ADMIN
  OWNER
}
```

### Message

```prisma
model Message {
  id        String      @id @default(uuid())
  chatId    String      @map("chat_id")
  senderId  String      @map("sender_id")  // User ID - кто отправил
  content   String      @db.Text
  type      MessageType @default(TEXT)
  fileUrl   String?     @map("file_url") @db.VarChar(500)
  fileName  String?     @map("file_name") @db.VarChar(255)
  fileSize  Int?        @map("file_size")
  replyToId String?     @map("reply_to_id")
  editedAt  DateTime?   @map("edited_at")
  deletedAt DateTime?   @map("deleted_at")
  createdAt DateTime    @default(now())
  updatedAt DateTime    @updatedAt

  chat    Chat     @relation(fields: [chatId], references: [id], onDelete: Cascade)
  sender  User     @relation(fields: [senderId], references: [id], onDelete: Cascade)
  replyTo Message? @relation("MessageReply", fields: [replyToId], references: [id], onDelete: SetNull)
  replies Message[] @relation("MessageReply")

  @@map("messages")
  @@index([chatId])
  @@index([senderId])
  @@index([createdAt])
  @@index([replyToId])
}

enum MessageType {
  TEXT
  IMAGE
  FILE
  SYSTEM
}
```

## Ссылки

- [Chats Module](./chats-module.md) - управление чатами
- [Messages Module](./messages-module.md) - отправка и получение сообщений
- [WebSocket Integration](./websocket-integration.md) - real-time общение
- [Chat Types](./chat-types.md) - типы чатов
- [Company Chat Logic](./company-chat-logic.md) - логика общения от компании
