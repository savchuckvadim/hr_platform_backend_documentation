# File Upload Service - Сервис загрузки файлов

## Обзор

**⚠️ DEPRECATED** - Используйте Storage Service вместо прямого использования S3 Service.

См. [Storage Module](./storage-module.md) для работы с файлами.

## Миграция

Вместо прямого использования S3 Service:

```typescript
// ❌ Старый способ
const result = await s3Service.uploadAvatar(file, userId);
```

Используйте Storage Service:

```typescript
// ✅ Новый способ
const result = await storageService.uploadFile(file, {
    path: FilePath.CANDIDATE_AVATAR,
    entityId: userId,
    accessLevel: S3AccessLevel.PUBLIC,
    validateType: true,
    validateSize: true,
});
```

## См. также

- [Storage Module](./storage-module.md) - Storage модуль с CRUD операциями
- [S3 Configuration](./s3-configuration.md) - настройка AWS S3
