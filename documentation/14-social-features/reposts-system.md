# Reposts System - Система репостов

## Обзор

Система репостов постов других пользователей на HR Platform.

## Модель данных

Посты используют поле `originalPostId` для связи с оригинальным постом:

```prisma
model Post {
  // ...
  originalPostId String? @map("original_post_id")
  originalPost  Post?    @relation("PostRepost", fields: [originalPostId], references: [id], onDelete: SetNull)
  reposts       Post[]   @relation("PostRepost")
  // ...
}
```

## Service методы

### PostService

```typescript
// application/services/post.service.ts

/**
 * Репост поста
 */
async repost(
    postId: string,
    userId: string,
    data?: { text?: string },
): Promise<PostDto> {
    // Получаем оригинальный пост
    const originalPost = await this.postRepository.findById(postId);
    if (!originalPost) {
        throw new NotFoundException('Post not found');
    }

    // Определяем профиль автора репоста
    const roleContext = await this.roleContextRepository.findActiveByUserId(userId);

    let profileId: string;
    let profileType: ProfileType;

    if (roleContext.userRole === 'CANDIDATE') {
        // Для кандидата используем userId напрямую
        profileId = userId;
        profileType = ProfileType.CANDIDATE;
    } else if (roleContext.type === 'EMPLOYER') {
        if (!roleContext.companyId) {
            throw new ForbiddenException('You must be an HR of a company to repost');
        }
        profileId = roleContext.companyId;
        profileType = ProfileType.COMPANY;
    } else {
        throw new ForbiddenException('Invalid role for reposting');
    }

    // Создаем репост
    const repost = await this.postRepository.create({
        authorId: userId,
        profileId,
        profileType,
        text: data?.text,
        originalPostId: postId,
    });

    // Публикуем событие
    this.eventBus.emit(AppEvent.POST_REPOSTED, {
        postId: repost.id,
        originalPostId: postId,
        authorId: userId,
    });

    return new PostDto(repost);
}

/**
 * Получение пользователей, которые сделали репост
 */
async getRepostUsers(postId: string): Promise<Array<{
    id: string;
    name: string;
    email: string;
    avatar?: string;
    repostedAt: Date;
}>> {
    return this.postRepository.getRepostUsers(postId);
}
```

## Repository методы

```typescript
// infrastructure/repositories/post.repository.ts

abstract repost(
    postId: string,
    userId: string,
    data?: { text?: string },
): Promise<FullPost>;

abstract getRepostUsers(postId: string): Promise<Array<{
    id: string;
    name: string;
    email: string;
    avatar?: string;
    repostedAt: Date;
}>>;
```

## Controller endpoints

```typescript
// api/controllers/posts.controller.ts

@Post(':id/repost')
@ApiOperation({ summary: 'Repost a post' })
async repost(
    @Param('id') postId: string,
    @Body() body?: { text?: string },
    @CurrentUser() user?: { id: string },
): Promise<PostDto> {
    return this.postService.repost(postId, user!.id, body);
}

@Get(':id/reposts')
@ApiOperation({ summary: 'Get users who reposted' })
async getRepostUsers(
    @Param('id') postId: string,
): Promise<Array<{ id: string; name: string; email: string; avatar?: string; repostedAt: Date }>> {
    return this.postService.getRepostUsers(postId);
}
```

## Best Practices

1. **Сохраняйте связь с оригиналом** - через originalPostId
2. **Позволяйте добавить текст** - к репосту
3. **Публикуйте события** - для уведомлений
4. **Подсчитывайте репосты** - для статистики
