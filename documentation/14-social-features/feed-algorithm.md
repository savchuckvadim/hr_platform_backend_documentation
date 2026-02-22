# Feed Algorithm - Алгоритм ленты

## Обзор

Алгоритм формирования персональной ленты активности на HR Platform.

## Принципы алгоритма

1. **Посты от подписок** - посты профилей, на которые подписан пользователь
2. **Сортировка по времени** - новые посты выше
3. **Cursor pagination** - для эффективной загрузки
4. **Фильтрация удаленных** - исключаем удаленные посты

## Service методы

### PostService

```typescript
// application/services/post.service.ts

/**
 * Получение ленты активности
 */
async getFeed(
    currentUserId: string,
    cursor?: string,
    limit: number = 20,
): Promise<PaginatedPostsDto> {
    // Получаем подписки пользователя
    const following = await this.followersService.getFollowing(currentUserId);
    const followingProfileIds = following.map(f => f.followingId);

    // Получаем посты от подписок
    const result = await this.postRepository.getFeed(
        currentUserId,
        followingProfileIds,
        cursor,
        limit,
    );

    return {
        posts: result.posts.map(post => new PostDto(post)),
        nextCursor: result.nextCursor,
        hasNext: result.hasNext,
    };
}
```

## Repository методы

```typescript
// infrastructure/repositories/post.repository.ts

abstract getFeed(
    currentUserId: string,
    followingProfileIds: string[],
    cursor?: string,
    limit?: number,
): Promise<{ posts: FullPost[]; nextCursor?: string; hasNext: boolean }>;
```

### Prisma реализация

```typescript
// infrastructure/repositories/post.prisma.repository.ts

async getFeed(
    currentUserId: string,
    followingProfileIds: string[],
    cursor?: string,
    limit: number = 20,
): Promise<{ posts: FullPost[]; nextCursor?: string; hasNext: boolean }> {
    const where: Prisma.PostWhereInput = {
        deletedAt: null,
        OR: [
            { profileId: { in: followingProfileIds } },
            { authorId: currentUserId }, // Свои посты тоже в ленте
        ],
    };

    if (cursor) {
        where.id = { lt: cursor };
    }

    const posts = await this.prisma.post.findMany({
        where,
        take: limit + 1,
        orderBy: { createdAt: 'desc' },
        include: {
            author: {
                include: {
                    profile: true,
                },
            },
            // ... другие includes
        },
    });

    const hasNext = posts.length > limit;
    const resultPosts = hasNext ? posts.slice(0, limit) : posts;

    return {
        posts: await Promise.all(resultPosts.map(post => this.enrichPost(post, currentUserId))),
        nextCursor: hasNext ? resultPosts[resultPosts.length - 1].id : undefined,
        hasNext,
    };
}
```

## Best Practices

1. **Используйте cursor pagination** - для больших объемов
2. **Кэшируйте подписки** - для производительности
3. **Фильтруйте удаленные** - для чистоты ленты
4. **Ограничивайте количество** - для производительности
