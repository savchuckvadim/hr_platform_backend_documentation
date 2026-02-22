# File Validation - Валидация файлов

## Обзор

Валидация типов и размеров файлов перед загрузкой в S3. Валидация выполняется через утилиты в Storage модуле.

## File Validation Util

### Реализация

```typescript
// infrastructure/utils/file-validation.util.ts
import { Injectable } from '@nestjs/common';
import { FilePath } from '../../domain/enums/file-path.enum';

@Injectable()
export class FileValidationUtil {
    /**
     * Получить разрешенные MIME типы для пути
     */
    getAllowedTypes(path: FilePath): string[] {
        const typeMap: Record<FilePath, string[]> = {
            [FilePath.CANDIDATE_AVATAR]: ['image/jpeg', 'image/png', 'image/webp'],
            [FilePath.CANDIDATE_HERO]: ['image/jpeg', 'image/png', 'image/webp'],
            [FilePath.CANDIDATE_RESUME]: [
                'application/pdf',
                'application/msword',
                'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            ],
            [FilePath.CANDIDATE_DOCUMENT]: [
                'application/pdf',
                'application/msword',
                'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            ],
            [FilePath.COMPANY_AVATAR]: ['image/jpeg', 'image/png', 'image/webp'],
            [FilePath.COMPANY_HERO]: ['image/jpeg', 'image/png', 'image/webp'],
            [FilePath.COMPANY_LOGO]: ['image/jpeg', 'image/png', 'image/webp'],
            [FilePath.COMPANY_DOCUMENT]: [
                'application/pdf',
                'application/msword',
                'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            ],
            [FilePath.VACANCY_IMAGE]: ['image/jpeg', 'image/png', 'image/webp'],
            [FilePath.VACANCY_DOCUMENT]: [
                'application/pdf',
                'application/msword',
                'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            ],
            [FilePath.POST_IMAGE]: ['image/jpeg', 'image/png', 'image/webp'],
            [FilePath.POST_VIDEO]: ['video/mp4', 'video/webm', 'video/quicktime'],
            [FilePath.MESSAGE_FILE]: [
                'application/pdf',
                'application/msword',
                'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
            ],
            [FilePath.MESSAGE_IMAGE]: ['image/jpeg', 'image/png', 'image/webp'],
        };

        return typeMap[path] || [];
    }

    /**
     * Проверка типа файла
     */
    isValidType(file: Express.Multer.File, path: FilePath): boolean {
        const allowedTypes = this.getAllowedTypes(path);
        return allowedTypes.includes(file.mimetype);
    }
}
```

## File Size Util

### Реализация

```typescript
// infrastructure/utils/file-size.util.ts
import { Injectable } from '@nestjs/common';
import { FilePath } from '../../domain/enums/file-path.enum';

@Injectable()
export class FileSizeUtil {
    /**
     * Получить максимальный размер файла для пути (в байтах)
     */
    getMaxSize(path: FilePath): number {
        const sizeMap: Record<FilePath, number> = {
            [FilePath.CANDIDATE_AVATAR]: 5 * 1024 * 1024, // 5MB
            [FilePath.CANDIDATE_HERO]: 5 * 1024 * 1024, // 5MB
            [FilePath.CANDIDATE_RESUME]: 10 * 1024 * 1024, // 10MB
            [FilePath.CANDIDATE_DOCUMENT]: 10 * 1024 * 1024, // 10MB
            [FilePath.COMPANY_AVATAR]: 5 * 1024 * 1024, // 5MB
            [FilePath.COMPANY_HERO]: 5 * 1024 * 1024, // 5MB
            [FilePath.COMPANY_LOGO]: 5 * 1024 * 1024, // 5MB
            [FilePath.COMPANY_DOCUMENT]: 10 * 1024 * 1024, // 10MB
            [FilePath.VACANCY_IMAGE]: 5 * 1024 * 1024, // 5MB
            [FilePath.VACANCY_DOCUMENT]: 10 * 1024 * 1024, // 10MB
            [FilePath.POST_IMAGE]: 10 * 1024 * 1024, // 10MB
            [FilePath.POST_VIDEO]: 100 * 1024 * 1024, // 100MB
            [FilePath.MESSAGE_FILE]: 10 * 1024 * 1024, // 10MB
            [FilePath.MESSAGE_IMAGE]: 10 * 1024 * 1024, // 10MB
        };

        return sizeMap[path] || 10 * 1024 * 1024; // По умолчанию 10MB
    }

    /**
     * Проверка размера файла
     */
    isValidSize(file: Express.Multer.File, path: FilePath): boolean {
        const maxSize = this.getMaxSize(path);
        return file.size <= maxSize;
    }
}
```

## Использование в Storage Service

Валидация выполняется автоматически в `StorageService.uploadFile()`:

```typescript
async uploadFile(file: Express.Multer.File, options: UploadFileOptions): Promise<FileInfo> {
    // Валидация типа файла (если требуется)
    if (options.validateType) {
        const allowedTypes = this.fileValidationUtil.getAllowedTypes(options.path);
        if (!allowedTypes.includes(file.mimetype)) {
            throw new BadRequestException(
                `Invalid file type. Allowed types: ${allowedTypes.join(', ')}`
            );
        }
    }

    // Валидация размера файла (если требуется)
    if (options.validateSize) {
        const maxSize = this.fileSizeUtil.getMaxSize(options.path);
        if (file.size > maxSize) {
            const maxSizeMB = maxSize / (1024 * 1024);
            throw new BadRequestException(
                `File size exceeds ${maxSizeMB}MB limit.`
            );
        }
    }

    // ... загрузка файла
}
```

## Best Practices

1. **Валидируйте перед загрузкой** - для экономии ресурсов
2. **Используйте утилиты** - для централизованного управления
3. **Понятные сообщения об ошибках** - для пользователей
4. **Проверяйте MIME тип** - не только расширение
