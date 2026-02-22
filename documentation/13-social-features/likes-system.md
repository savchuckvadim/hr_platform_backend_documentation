# Likes System - Система лайков

## Обзор

Система лайков и дизлайков для постов на HR Platform.

## Модель данных

### PostLike

```prisma
model PostLike {
  id        String   @id @default(uuid())
  postId    String   @map("post_id")
  userId    String   @map("user_id")
  isLike    Boolean  @default(true) @map("is_like") // true = лайк, false = дизлайк
  createdAt DateTime @default(now())

  post Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([postId, userId])
  @@map("post_likes")
  @@index([postId])
  @@index([userId])
}
```

## Service методы

### PostService

```typescript
// application/services/post.service.ts

/**
 * Лайк поста
 */
async likePost(postId: string, userId: string): Promise<void> {
    await this.postRepository.likePost(postId, userId, true);

    this.eventBus.emit(AppEvent.POST_LIKED, {
        postId,
        userId,
        isLike: true,
    });
}

/**
 * Дизлайк поста
 */
async dislikePost(postId: string, userId: string): Promise<void> {
    await this.postRepository.likePost(postId, userId, false);

    this.eventBus.emit(AppEvent.POST_LIKED, {
        postId,
        userId,
        isLike: false,
    });
}

/**
 * Убрать лайк/дизлайк
 */
async unlikePost(postId: string, userId: string): Promise<void> {
    await this.postRepository.unlikePost(postId, userId);

    this.eventBus.emit(AppEvent.POST_UNLIKED, {
        postId,
        userId,
    });
}
```

## Repository методы

```typescript
// infrastructure/repositories/post.repository.ts

abstract likePost(postId: string, userId: string, isLike: boolean): Promise<void>;

abstract unlikePost(postId: string, userId: string): Promise<void>;
```

### Prisma реализация

```typescript
// infrastructure/repositories/post.prisma.repository.ts

async likePost(postId: string, userId: string, isLike: boolean): Promise<void> {
    await this.prisma.postLike.upsert({
        where: {
            postId_userId: {
                postId,
                userId,
            },
        },
        create: {
            postId,
            userId,
            isLike,
        },
        update: {
            isLike,
        },
    });
}

async unlikePost(postId: string, userId: string): Promise<void> {
    await this.prisma.postLike.delete({
        where: {
            postId_userId: {
                postId,
                userId,
            },
        },
    });
}
```

## Controller endpoints

```typescript
// api/controllers/posts.controller.ts

@Post(':id/like')
@ApiOperation({ summary: 'Like a post' })
async likePost(
    @Param('id') postId: string,
    @CurrentUser() user: { id: string },
): Promise<void> {
    return this.postService.likePost(postId, user.id);
}

@Post(':id/dislike')
@ApiOperation({ summary: 'Dislike a post' })
async dislikePost(
    @Param('id') postId: string,
    @CurrentUser() user: { id: string },
): Promise<void> {
    return this.postService.dislikePost(postId, user.id);
}

@Delete(':id/like')
@ApiOperation({ summary: 'Unlike a post' })
async unlikePost(
    @Param('id') postId: string,
    @CurrentUser() user: { id: string },
): Promise<void> {
    return this.postService.unlikePost(postId, user.id);
}
```

## Best Practices

1. **Используйте upsert** - для переключения лайк/дизлайк
2. **Публикуйте события** - для уведомлений
3. **Подсчитывайте в запросах** - для производительности
4. **Уникальный constraint** - один лайк на пользователя и пост
