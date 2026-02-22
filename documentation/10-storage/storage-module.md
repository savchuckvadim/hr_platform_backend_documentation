# Storage Module - Модуль хранения файлов

## Обзор

Storage модуль предоставляет высокоуровневый API для работы с файлами. Использует S3 сервис для хранения и предоставляет CRUD операции с поддержкой различных путей через enums.

## Структура модуля

```
core/storage/
├── api/
│   ├── controllers/
│   │   └── storage.controller.ts
│   └── dto/
│       ├── upload-file.dto.ts
│       └── file.dto.ts
├── application/
│   └── services/
│       └── storage.service.ts
├── domain/
│   └── enums/
│       └── file-path.enum.ts
├── infrastructure/
│   └── utils/
│       ├── file-validation.util.ts
│       └── file-size.util.ts
├── storage.module.ts
└── index.ts
```

## File Path Enum

### Определение путей

```typescript
// domain/enums/file-path.enum.ts
export enum FilePath {
    // Candidate paths
    CANDIDATE_AVATAR = 'candidate/avatar',
    CANDIDATE_HERO = 'candidate/hero',
    CANDIDATE_RESUME = 'candidate/resume',
    CANDIDATE_DOCUMENT = 'candidate/document',

    // Company paths
    COMPANY_AVATAR = 'company/avatar',
    COMPANY_HERO = 'company/hero',
    COMPANY_LOGO = 'company/logo',
    COMPANY_DOCUMENT = 'company/document',

    // Vacancy paths
    VACANCY_IMAGE = 'vacancy/image',
    VACANCY_DOCUMENT = 'vacancy/document',

    // Post paths
    POST_IMAGE = 'post/image',
    POST_VIDEO = 'post/video',

    // Message paths
    MESSAGE_FILE = 'message/file',
    MESSAGE_IMAGE = 'message/image',
}
```

## Storage Service

### CRUD операции

```typescript
// application/services/storage.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { S3Service, S3AccessLevel } from '@core/s3';
import { FilePath } from '../domain/enums/file-path.enum';
import { FileValidationUtil } from '../infrastructure/utils/file-validation.util';
import { FileSizeUtil } from '../infrastructure/utils/file-size.util';
import { randomUUID } from 'crypto';

export interface UploadFileOptions {
    path: FilePath;
    accessLevel?: S3AccessLevel;
    entityId?: string; // ID сущности (userId, companyId и т.д.)
    validateType?: boolean;
    validateSize?: boolean;
}

export interface FileInfo {
    url: string;
    key: string;
    path: FilePath;
    size: number;
    mimeType: string;
    originalName: string;
}

@Injectable()
export class StorageService {
    constructor(
        private readonly s3Service: S3Service,
        private readonly fileValidationUtil: FileValidationUtil,
        private readonly fileSizeUtil: FileSizeUtil,
    ) {}

    /**
     * Загрузка файла
     */
    async uploadFile(
        file: Express.Multer.File,
        options: UploadFileOptions,
    ): Promise<FileInfo> {
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

        // Генерируем ключ файла
        const key = this.generateFileKey(file, options);

        // Загружаем в S3
        const result = await this.s3Service.uploadFile(
            file,
            key,
            options.accessLevel || S3AccessLevel.PRIVATE,
        );

        return {
            url: result.url,
            key: result.key,
            path: options.path,
            size: file.size,
            mimeType: file.mimetype,
            originalName: file.originalname,
        };
    }

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

    /**
     * Генерация ключа файла
     */
    private generateFileKey(
        file: Express.Multer.File,
        options: UploadFileOptions,
    ): string {
        const fileExtension = file.originalname.split('.').pop();
        const fileName = `${randomUUID()}.${fileExtension}`;

        // Если указан entityId, добавляем его в путь
        if (options.entityId) {
            return `${options.path}/${options.entityId}/${fileName}`;
        }

        return `${options.path}/${fileName}`;
    }
}
```

## File Validation Util

### Утилита валидации

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

### Утилита размера

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

## DTO

### UploadFileDto

```typescript
// api/dto/upload-file.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEnum, IsOptional, IsString } from 'class-validator';
import { FilePath } from '../../domain/enums/file-path.enum';
import { S3AccessLevel } from '@core/s3';

export class UploadFileDto {
    @ApiProperty({
        description: 'File path',
        enum: FilePath,
        example: FilePath.CANDIDATE_AVATAR,
    })
    @IsEnum(FilePath)
    path: FilePath;

    @ApiProperty({
        description: 'Access level',
        enum: S3AccessLevel,
        default: S3AccessLevel.PRIVATE,
        required: false,
    })
    @IsOptional()
    @IsEnum(S3AccessLevel)
    accessLevel?: S3AccessLevel;

    @ApiProperty({
        description: 'Entity ID (userId, companyId, etc.)',
        required: false,
    })
    @IsOptional()
    @IsString()
    entityId?: string;

    @ApiProperty({
        description: 'Validate file type',
        default: true,
        required: false,
    })
    @IsOptional()
    validateType?: boolean;

    @ApiProperty({
        description: 'Validate file size',
        default: true,
        required: false,
    })
    @IsOptional()
    validateSize?: boolean;
}
```

## Controller

### StorageController

```typescript
// api/controllers/storage.controller.ts
import { Controller, Post, Get, Delete, Body, Param, UseInterceptors, UploadedFile, UseGuards } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { ApiTags, ApiOperation, ApiConsumes, ApiBody } from '@nestjs/swagger';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { StorageService } from '../application/services/storage.service';
import { UploadFileDto } from '../api/dto/upload-file.dto';

@ApiTags('Storage')
@Controller('storage')
@UseGuards(JwtAuthGuard)
export class StorageController {
    constructor(private readonly storageService: StorageService) {}

    @Post('upload')
    @UseInterceptors(FileInterceptor('file'))
    @ApiOperation({ summary: 'Upload a file' })
    @ApiConsumes('multipart/form-data')
    @ApiBody({ type: UploadFileDto })
    async uploadFile(
        @UploadedFile() file: Express.Multer.File,
        @Body() dto: UploadFileDto,
    ) {
        return this.storageService.uploadFile(file, {
            path: dto.path,
            accessLevel: dto.accessLevel,
            entityId: dto.entityId,
            validateType: dto.validateType ?? true,
            validateSize: dto.validateSize ?? true,
        });
    }

    @Get('file/:key')
    @ApiOperation({ summary: 'Get file URL (presigned for private files)' })
    async getFile(
        @Param('key') key: string,
        @Body('expiresIn') expiresIn?: number,
    ) {
        const url = await this.storageService.getFile(key, expiresIn);
        return { url };
    }

    @Delete('file/:key')
    @ApiOperation({ summary: 'Delete a file' })
    async deleteFile(@Param('key') key: string) {
        await this.storageService.deleteFile(key);
        return { success: true };
    }
}
```

## Использование

### Примеры

```typescript
// Загрузка аватара кандидата
const fileInfo = await storageService.uploadFile(file, {
    path: FilePath.CANDIDATE_AVATAR,
    entityId: userId,
    accessLevel: S3AccessLevel.PUBLIC,
    validateType: true,
    validateSize: true,
});

// Загрузка резюме
const resumeInfo = await storageService.uploadFile(file, {
    path: FilePath.CANDIDATE_RESUME,
    entityId: resumeId,
    accessLevel: S3AccessLevel.PRIVATE,
    validateType: true,
    validateSize: true,
});

// Получение файла (presigned URL для PRIVATE)
const url = await storageService.getFile(resumeInfo.key, 3600);

// Удаление файла
await storageService.deleteFile(resumeInfo.key);
```

## Best Practices

1. **Используйте enums путей** - для централизованного управления
2. **Валидируйте файлы** - тип и размер перед загрузкой
3. **Используйте правильный уровень доступа** - для безопасности
4. **Храните ключи в БД** - для возможности удаления
5. **Не плодите методы** - один метод для всех типов файлов
