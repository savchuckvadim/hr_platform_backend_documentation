# File Download Service - Сервис получения файлов

## Обзор

Получение файлов из S3 через Storage Service. Поддержка presigned URLs для приватных файлов.

## Использование

### Storage Service

```typescript
// application/services/storage.service.ts

/**
 * Получение файла (presigned URL для PRIVATE/APP)
 */
async getFile(key: string, expiresIn?: number): Promise<string> {
    // Для PUBLIC файлов возвращаем прямой URL
    if (key.startsWith('public/')) {
        return this.s3Service.getPublicUrl(key);
    }

    // Для PRIVATE/APP файлов генерируем presigned URL
    return this.s3Service.generatePresignedUrl(key, expiresIn || 3600);
}
```

### Примеры использования

```typescript
// Получение публичного файла
const publicUrl = await storageService.getFile('public/candidate/avatar/user-123/file.jpg');
// Возвращает: https://bucket.s3.region.amazonaws.com/public/candidate/avatar/user-123/file.jpg

// Получение приватного файла (presigned URL)
const privateUrl = await storageService.getFile('private/candidate/resume/resume-456/file.pdf', 3600);
// Возвращает presigned URL, действительный 1 час
```

## Best Practices

1. **Используйте presigned URLs** - для приватных файлов
2. **Ограничивайте время жизни** - для безопасности
3. **Проверяйте права доступа** - перед генерацией URL
4. **Храните ключи в БД** - для возможности удаления
