# File Deletion - Удаление файлов

## Обзор

Удаление файлов из S3 через Storage Service при удалении сущностей или обновлении файлов.

## Использование

### Storage Service

```typescript
// application/services/storage.service.ts

/**
 * Удаление файла
 */
async deleteFile(key: string): Promise<void> {
    await this.s3Service.deleteFile(key);
}

/**
 * Удаление нескольких файлов
 */
async deleteFiles(keys: string[]): Promise<void> {
    await this.s3Service.deleteFiles(keys);
}
```

### Примеры использования

```typescript
// Удаление одного файла
await storageService.deleteFile('candidate/avatar/user-123/file.jpg');

// Удаление нескольких файлов
await storageService.deleteFiles([
    'candidate/avatar/user-123/file.jpg',
    'candidate/hero/user-123/file.jpg',
]);
```

## Использование в сервисах

### При удалении поста

```typescript
// application/services/post.service.ts
async deletePost(id: string, userId: string): Promise<void> {
    const post = await this.postRepository.findById(id, userId);

    // Удаляем медиа файлы
    const mediaKeys: string[] = [];
    if (post.image) {
        // Извлекаем ключ из URL
        const key = this.extractKeyFromUrl(post.image);
        mediaKeys.push(key);
    }
    if (post.video) {
        const key = this.extractKeyFromUrl(post.video);
        mediaKeys.push(key);
    }

    if (mediaKeys.length > 0) {
        await this.storageService.deleteFiles(mediaKeys);
    }

    // Удаляем пост из БД
    await this.postRepository.delete(id, userId);
}
```

### При обновлении аватара

```typescript
// application/services/profile.service.ts
async updateAvatar(userId: string, file: Express.Multer.File): Promise<string> {
    const profile = await this.profileRepository.findByUserId(userId);

    // Загружаем новый аватар
    const fileInfo = await this.storageService.uploadFile(file, {
        path: FilePath.CANDIDATE_AVATAR,
        entityId: userId,
        accessLevel: S3AccessLevel.PUBLIC,
    });

    // Удаляем старый аватар, если есть
    if (profile.avatar) {
        const oldKey = this.extractKeyFromUrl(profile.avatar);
        await this.storageService.deleteFile(oldKey).catch((err) => {
            this.logger.warn('Failed to delete old avatar', err);
        });
    }

    // Обновляем профиль
    await this.profileRepository.update(userId, { avatar: fileInfo.url });

    return fileInfo.url;
}
```

## Best Practices

1. **Удаляйте файлы при удалении сущностей** - для экономии места
2. **Удаляйте старые файлы при обновлении** - для предотвращения мусора
3. **Используйте Promise.allSettled** - для параллельного удаления
4. **Не блокируйте удаление сущности** - если удаление файла не удалось
5. **Логируйте ошибки** - для отладки
