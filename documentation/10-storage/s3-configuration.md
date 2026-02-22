# S3 Configuration - Настройка AWS S3

## Обзор

Настройка AWS S3 клиента и конфигурации для HR Platform. S3 сервис работает только с уровнями доступа (PUBLIC, PRIVATE, APP), не знает о типах файлов.

## Установка зависимостей

```bash
npm install @aws-sdk/client-s3
```

## Environment Variables

```env
# AWS S3 Configuration
AWS_REGION=us-east-1
AWS_ACCESS_KEY=your-access-key
AWS_SECRET_KEY=your-secret-key
AWS_BUCKET_NAME=hr-platform-storage
```

## S3 Module

```typescript
// core/s3/s3.module.ts
import { Global, Module } from '@nestjs/common';
import { S3Service } from './s3.service';

@Global()
@Module({
    providers: [S3Service],
    exports: [S3Service],
})
export class S3Module {}
```

## S3 Service

### Уровни доступа

S3 сервис работает только с тремя уровнями доступа:

```typescript
// core/s3/enums/access-level.enum.ts
export enum S3AccessLevel {
    PUBLIC = 'PUBLIC',   // Публичный доступ
    PRIVATE = 'PRIVATE', // Приватный доступ
    APP = 'APP',         // Доступ только для приложения
}
```

### S3 Service реализация

```typescript
// core/s3/s3.service.ts
import { Injectable, BadRequestException, Logger } from '@nestjs/common';
import { S3Client, PutObjectCommand, DeleteObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { randomUUID } from 'crypto';
import { S3AccessLevel } from './enums/access-level.enum';

export interface UploadFileResult {
    url: string;
    key: string;
}

@Injectable()
export class S3Service {
    private readonly logger = new Logger(S3Service.name);
    private s3: S3Client | null = null;
    private bucket: string;
    private region: string;

    constructor() {
        const region = process.env.AWS_REGION;
        const accessKeyId = process.env.AWS_ACCESS_KEY;
        const secretAccessKey = process.env.AWS_SECRET_KEY;
        const bucket = process.env.AWS_BUCKET_NAME;

        if (!region || !accessKeyId || !secretAccessKey || !bucket) {
            this.logger.warn('⚠️ AWS S3 credentials not configured. File uploads will fail.');
            this.logger.warn('Required environment variables: AWS_REGION, AWS_ACCESS_KEY, AWS_SECRET_KEY, AWS_BUCKET_NAME');
            return;
        }

        this.bucket = bucket;
        this.region = region;

        try {
            this.s3 = new S3Client({
                region: region,
                credentials: {
                    accessKeyId: accessKeyId,
                    secretAccessKey: secretAccessKey,
                },
                followRegionRedirects: true,
            });
            this.logger.log('✅ S3 client initialized successfully');
        } catch (error) {
            this.logger.error('❌ Failed to initialize S3 client:', error);
        }
    }

    private validateS3Config(): void {
        if (!this.s3 || !this.bucket || !this.region) {
            throw new BadRequestException(
                'S3 storage is not configured. Please set AWS_REGION, AWS_ACCESS_KEY, AWS_SECRET_KEY, and AWS_BUCKET_NAME environment variables.'
            );
        }
    }

    /**
     * Загружает файл в S3
     * @param file - файл из multer
     * @param key - ключ файла в S3 (путь)
     * @param accessLevel - уровень доступа (PUBLIC, PRIVATE, APP)
     * @returns URL загруженного файла и ключ
     */
    async uploadFile(
        file: Express.Multer.File,
        key: string,
        accessLevel: S3AccessLevel = S3AccessLevel.PRIVATE,
    ): Promise<UploadFileResult> {
        this.validateS3Config();

        // Определяем ACL в зависимости от уровня доступа
        let acl: string | undefined;
        if (accessLevel === S3AccessLevel.PUBLIC) {
            acl = 'public-read';
        }

        const command = new PutObjectCommand({
            Bucket: this.bucket,
            Key: key,
            Body: file.buffer,
            ContentType: file.mimetype,
            ACL: acl,
        });

        await this.s3!.send(command);

        // Формируем URL
        const url = `https://${this.bucket}.s3.${this.region}.amazonaws.com/${key}`;

        this.logger.log(`File uploaded: ${key} (${accessLevel})`);

        return { url, key };
    }

    /**
     * Удаляет файл из S3 по ключу
     * @param key - ключ файла в S3
     */
    async deleteFile(key: string): Promise<void> {
        this.validateS3Config();

        try {
            const command = new DeleteObjectCommand({
                Bucket: this.bucket,
                Key: key,
            });

            await this.s3!.send(command);
            this.logger.log(`File deleted: ${key}`);
        } catch (error) {
            this.logger.error('Error deleting file from S3:', error);
            throw error;
        }
    }

    /**
     * Удаляет несколько файлов из S3
     * @param keys - массив ключей файлов
     */
    async deleteFiles(keys: string[]): Promise<void> {
        await Promise.allSettled(keys.map(key => this.deleteFile(key)));
    }

    /**
     * Генерирует presigned URL для временного доступа к файлу
     * @param key - ключ файла в S3
     * @param expiresIn - время жизни URL в секундах (по умолчанию 1 час)
     * @returns Presigned URL
     */
    async generatePresignedUrl(
        key: string,
        expiresIn: number = 3600,
    ): Promise<string> {
        this.validateS3Config();

        const command = new GetObjectCommand({
            Bucket: this.bucket,
            Key: key,
        });

        const url = await getSignedUrl(this.s3!, command, { expiresIn });
        return url;
    }

    /**
     * Получает публичный URL файла (для PUBLIC файлов)
     * @param key - ключ файла в S3
     * @returns Публичный URL
     */
    getPublicUrl(key: string): string {
        return `https://${this.bucket}.s3.${this.region}.amazonaws.com/${key}`;
    }
}
```

## Уровни доступа

### PUBLIC

Файлы с публичным доступом. Доступны по прямому URL без аутентификации.

```typescript
const result = await s3Service.uploadFile(file, 'public/avatar.jpg', S3AccessLevel.PUBLIC);
// URL: https://bucket.s3.region.amazonaws.com/public/avatar.jpg
```

### PRIVATE

Приватные файлы. Требуют presigned URL для доступа.

```typescript
const result = await s3Service.uploadFile(file, 'private/resume.pdf', S3AccessLevel.PRIVATE);
// Для доступа нужен presigned URL
const url = await s3Service.generatePresignedUrl(result.key, 3600);
```

### APP

Файлы доступные только для приложения. Используются для внутренних нужд.

```typescript
const result = await s3Service.uploadFile(file, 'app/temp/file.txt', S3AccessLevel.APP);
```

## Best Practices

1. **Используйте Global Module** - для доступности во всех модулях
2. **Проверяйте конфигурацию** - при инициализации
3. **Логируйте ошибки** - для отладки
4. **Используйте правильный уровень доступа** - для безопасности
5. **Храните ключи в БД** - для возможности удаления
