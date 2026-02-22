# Social Features Module - Модуль социальных функций

## Описание

Опциональный модуль для социальной активности на HR Platform. Кандидаты и компании могут вести активность как в соцсети: создавать посты, ставить лайки, делать репосты, просматривать контент, использовать хэштеги для категоризации.

## Архитектурные принципы

### Ключевые принципы

1. **Два типа профилей** - candidate-profile и company-profile
2. **Автор поста** - конкретный HR (root HR или нанятый HR) для компаний
3. **Стена** - посты на стене профиля (кандидата или компании)
4. **Лента** - персональная лента активности
5. **Хэштеги** - для категоризации и поиска
6. **Event-driven** - интеграция с EventBus

## Структура модуля

### Posts Module

```
posts/
├── api/
│   ├── controllers/
│   │   └── posts.controller.ts
│   └── dto/
│       └── post.dto.ts
├── application/
│   ├── services/
│   │   └── post.service.ts
│   └── events/
│       └── post.events.ts
├── domain/
│   └── entity/
│       └── post.entity.ts
├── infrastructure/
│   ├── repositories/
│   │   ├── post.repository.ts
│   │   └── post.prisma.repository.ts
│   └── listeners/
├── posts.module.ts
└── index.ts
```

### Followers Module

```
followers/
├── api/
│   ├── controllers/
│   │   └── followers.controller.ts
│   └── dto/
│       └── follow.dto.ts
├── application/
│   └── services/
│       └── followers.service.ts
├── infrastructure/
│   └── repositories/
│       ├── followers.repository.ts
│       └── followers.prisma.repository.ts
├── followers.module.ts
└── index.ts
```

## Задачи

### ✅ Posts Module
**Статус**: Завершено
**Файл**: [posts-module.md](./posts-module.md)
**Описание**: Модуль для создания и управления постами

### ✅ Likes System
**Статус**: Завершено
**Файл**: [likes-system.md](./likes-system.md)
**Описание**: Система лайков для постов

### ✅ Reposts System
**Статус**: Завершено
**Файл**: [reposts-system.md](./reposts-system.md)
**Описание**: Система репостов постов

### ✅ Views Tracking
**Статус**: Завершено
**Файл**: [views-tracking.md](./views-tracking.md)
**Описание**: Отслеживание просмотров постов

### ✅ Hashtags System
**Статус**: Завершено
**Файл**: [hashtags-system.md](./hashtags-system.md)
**Описание**: Система хэштегов для постов

### ✅ Feed Algorithm
**Статус**: Завершено
**Файл**: [feed-algorithm.md](./feed-algorithm.md)
**Описание**: Алгоритм ленты активности

### ✅ Followers System
**Статус**: Завершено
**Файл**: [followers-system.md](./followers-system.md)
**Описание**: Система подписок на профили

## Ключевые концепции

- **Candidate Profile** - профиль кандидата, может создавать посты
- **Company Profile** - профиль компании, посты создают HR (root или нанятый)
- **Post Author** - конкретный пользователь (кандидат или HR)
- **Post Wall** - стена профиля (кандидата или компании)
- **Создание постов** - с текстом, изображениями, видео, хэштегами
- **Система лайков** - с возможностью отмены
- **Репосты** - постов других пользователей
- **Отслеживание просмотров** - для аналитики
- **Хэштеги** - для категоризации и поиска
- **Лента активности** - с алгоритмом сортировки
- **Интеграция с уведомлениями** - через EventBus

## Модель данных

### Post

```prisma
model Post {
  id            String   @id @default(uuid())
  authorId      String   @map("author_id")      // Кто создал пост (User ID)
  profileId     String   @map("profile_id")     // На чьей стене (CandidateProfile или Company ID)
  profileType   ProfileType @map("profile_type") // CANDIDATE | COMPANY
  text          String?  @db.Text
  image         String?  @db.VarChar(500)
  video         String?  @db.VarChar(500)
  viewsCount    Int      @default(0) @map("views_count")
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  deletedAt     DateTime? @map("deleted_at")

  author        User     @relation("PostAuthor", fields: [authorId], references: [id])
  profile       // Полиморфная связь (CandidateProfile или Company)
  likes         PostLike[]
  hashtags      PostHashtag[]
  reposts       Post[]   @relation("PostRepost")
  originalPost  Post?    @relation("PostRepost", fields: [originalPostId], references: [id])
  originalPostId String? @map("original_post_id")

  @@map("posts")
  @@index([authorId])
  @@index([profileId, profileType])
  @@index([createdAt])
  @@index([deletedAt])
}

enum ProfileType {
  CANDIDATE
  COMPANY
}
```

## Ссылки

- [Posts Module](./posts-module.md) - создание и управление постами
- [Likes System](./likes-system.md) - система лайков
- [Reposts System](./reposts-system.md) - система репостов
- [Views Tracking](./views-tracking.md) - отслеживание просмотров
- [Hashtags System](./hashtags-system.md) - система хэштегов
- [Feed Algorithm](./feed-algorithm.md) - алгоритм ленты
- [Followers System](./followers-system.md) - система подписок
