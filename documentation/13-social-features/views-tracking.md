# Views Tracking - Отслеживание просмотров

## Обзор

Отслеживание просмотров постов для аналитики на HR Platform.

## Модель данных

Просмотры хранятся в счетчике `viewsCount` в таблице `Post`:

```prisma
model Post {
  // ...
  viewsCount Int @default(0) @map("views_count")
  // ...
}
```

## Service методы

### PostService

```typescript
// application/services/post.service.ts

/**
 * Увеличение счетчика просмотров
 */
async incrementViews(postId: string): Promise<void> {
    await this.postRepository.incrementViews(postId);
}
```

## Repository методы

```typescript
// infrastructure/repositories/post.repository.ts

abstract incrementViews(postId: string): Promise<void>;
```

### Prisma реализация

```typescript
// infrastructure/repositories/post.prisma.repository.ts

async incrementViews(postId: string): Promise<void> {
    await this.prisma.post.update({
        where: { id: postId },
        data: {
            viewsCount: {
                increment: 1,
            },
        },
    });
}
```

## Controller endpoints

```typescript
// api/controllers/posts.controller.ts

@Post(':id/view')
@ApiOperation({ summary: 'Increment post views' })
@Public() // Просмотры могут быть публичными
async incrementViews(
    @Param('id') postId: string,
): Promise<void> {
    return this.postService.incrementViews(postId);
}
```

## Best Practices

1. **Используйте атомарные операции** - increment для счетчика
2. **Публичные просмотры** - не требуют аутентификации
3. **Оптимизируйте запросы** - используйте индексы
4. **Учитывайте дубликаты** - можно добавить таблицу просмотров для детальной аналитики
