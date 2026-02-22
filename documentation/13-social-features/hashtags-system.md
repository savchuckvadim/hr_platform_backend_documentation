# Hashtags System - Система хэштегов

## Обзор

Система хэштегов для категоризации и поиска постов на HR Platform.

## Модель данных

### PostHashtag

```prisma
model PostHashtag {
  id        String   @id @default(uuid())
  postId    String   @map("post_id")
  hashtag   String   @db.VarChar(100)
  createdAt DateTime @default(now())

  post Post @relation(fields: [postId], references: [id], onDelete: Cascade)

  @@unique([postId, hashtag])
  @@map("post_hashtags")
  @@index([postId])
  @@index([hashtag])
}
```

## Service методы

### PostService

```typescript
// application/services/post.service.ts

/**
 * Поиск постов по хэштегу
 */
async getPostsByHashtag(
    hashtag: string,
    currentUserId?: string,
    cursor?: string,
    limit?: number,
): Promise<PaginatedPostsDto> {
    const result = await this.postRepository.getByHashtag(
        hashtag,
        currentUserId,
        cursor,
        limit,
    );
    return {
        posts: result.posts.map(post => new PostDto(post)),
        nextCursor: result.nextCursor,
        hasNext: result.hasNext,
    };
}

/**
 * Получение популярных хэштегов
 */
async getPopularHashtags(limit: number = 10): Promise<Array<{
    hashtag: string;
    count: number;
}>> {
    return this.postRepository.getPopularHashtags(limit);
}
```

## Repository методы

```typescript
// infrastructure/repositories/post.repository.ts

abstract getByHashtag(
    hashtag: string,
    currentUserId?: string,
    cursor?: string,
    limit?: number,
): Promise<{ posts: FullPost[]; nextCursor?: string; hasNext: boolean }>;

abstract getPopularHashtags(limit: number): Promise<Array<{
    hashtag: string;
    count: number;
}>>;
```

## Controller endpoints

```typescript
// api/controllers/posts.controller.ts

@Get('hashtag/:hashtag')
@ApiOperation({ summary: 'Get posts by hashtag' })
@ApiQuery({ name: 'cursor', required: false, type: String })
@ApiQuery({ name: 'limit', required: false, type: Number })
async getPostsByHashtag(
    @Param('hashtag') hashtag: string,
    @Query('cursor') cursor?: string,
    @Query('limit') limit?: number,
    @CurrentUser() user?: { id: string },
): Promise<PaginatedPostsDto> {
    return this.postService.getPostsByHashtag(
        hashtag,
        user?.id,
        cursor,
        limit ? parseInt(limit.toString()) : undefined,
    );
}

@Get('hashtags/popular')
@ApiOperation({ summary: 'Get popular hashtags' })
async getPopularHashtags(
    @Query('limit') limit?: number,
): Promise<Array<{ hashtag: string; count: number }>> {
    return this.postService.getPopularHashtags(limit ? parseInt(limit.toString()) : 10);
}
```

## Best Practices

1. **Нормализуйте хэштеги** - lowercase, без спецсимволов
2. **Используйте индексы** - для быстрого поиска
3. **Кэшируйте популярные** - для производительности
4. **Ограничивайте количество** - на пост
