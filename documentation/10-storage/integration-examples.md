# Integration Examples - Примеры интеграции

## Обзор

Примеры интеграции Storage Service в различные модули HR Platform.

## Пример 1: Загрузка аватара кандидата

```typescript
// api/controllers/profile.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, UseGuards } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { StorageService } from '@core/storage/application/services/storage.service';
import { FilePath } from '@core/storage/domain/enums/file-path.enum';
import { S3AccessLevel } from '@core/s3';
import { ProfileService } from '@profile/application/services/profile.service';

@Controller('profile')
@UseGuards(JwtAuthGuard)
export class ProfileController {
    constructor(
        private readonly storageService: StorageService,
        private readonly profileService: ProfileService,
    ) {}

    @Post('avatar')
    @UseInterceptors(FileInterceptor('file'))
    async uploadAvatar(
        @UploadedFile() file: Express.Multer.File,
        @CurrentUser() user: { id: string },
    ) {
        const fileInfo = await this.storageService.uploadFile(file, {
            path: FilePath.CANDIDATE_AVATAR,
            entityId: user.id,
            accessLevel: S3AccessLevel.PUBLIC,
            validateType: true,
            validateSize: true,
        });

        await this.profileService.updateAvatar(user.id, fileInfo.url);
        return { url: fileInfo.url };
    }
}
```

## Пример 2: Загрузка аватара компании

```typescript
// api/controllers/company.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, UseGuards, Param } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { JwtAuthGuard, RolesGuard } from '@core/guards';
import { Roles } from '@core/decorators/roles.decorator';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { StorageService } from '@core/storage/application/services/storage.service';
import { FilePath } from '@core/storage/domain/enums/file-path.enum';
import { S3AccessLevel } from '@core/s3';
import { CompanyService } from '@company/application/services/company.service';

@Controller('companies')
@UseGuards(JwtAuthGuard, RolesGuard)
export class CompanyController {
    constructor(
        private readonly storageService: StorageService,
        private readonly companyService: CompanyService,
    ) {}

    @Post(':id/avatar')
    @Roles(RoleType.EMPLOYER)
    @UseInterceptors(FileInterceptor('file'))
    async uploadCompanyAvatar(
        @Param('id') companyId: string,
        @UploadedFile() file: Express.Multer.File,
        @CurrentUser() user: { id: string; companyId?: string },
    ) {
        // Проверяем права (только HR_ADMIN компании)
        if (user.companyId !== companyId) {
            throw new ForbiddenException('You can only update your own company avatar');
        }

        const fileInfo = await this.storageService.uploadFile(file, {
            path: FilePath.COMPANY_AVATAR,
            entityId: companyId,
            accessLevel: S3AccessLevel.PUBLIC,
            validateType: true,
            validateSize: true,
        });

        await this.companyService.updateAvatar(companyId, fileInfo.url);
        return { url: fileInfo.url };
    }
}
```

## Пример 3: Загрузка резюме

```typescript
// api/controllers/resume.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, UseGuards, Param } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { StorageService } from '@core/storage/application/services/storage.service';
import { FilePath } from '@core/storage/domain/enums/file-path.enum';
import { S3AccessLevel } from '@core/s3';
import { ResumeService } from '@resume/application/services/resume.service';

@Controller('resumes')
@UseGuards(JwtAuthGuard)
export class ResumeController {
    constructor(
        private readonly storageService: StorageService,
        private readonly resumeService: ResumeService,
    ) {}

    @Post(':id/document')
    @UseInterceptors(FileInterceptor('file'))
    async uploadResumeDocument(
        @Param('id') resumeId: string,
        @UploadedFile() file: Express.Multer.File,
        @CurrentUser() user: { id: string },
    ) {
        // Проверяем права
        const resume = await this.resumeService.findById(resumeId);
        if (resume.candidateId !== user.id) {
            throw new ForbiddenException('You can only upload documents to your own resume');
        }

        const fileInfo = await this.storageService.uploadFile(file, {
            path: FilePath.CANDIDATE_RESUME,
            entityId: resumeId,
            accessLevel: S3AccessLevel.PRIVATE,
            validateType: true,
            validateSize: true,
        });

        await this.resumeService.updateDocumentUrl(resumeId, fileInfo.url, fileInfo.key);
        return { url: fileInfo.url };
    }
}
```

## Пример 4: Загрузка изображения вакансии

```typescript
// api/controllers/vacancy.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, UseGuards, Param } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { JwtAuthGuard, RolesGuard } from '@core/guards';
import { Roles } from '@core/decorators/roles.decorator';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { StorageService } from '@core/storage/application/services/storage.service';
import { FilePath } from '@core/storage/domain/enums/file-path.enum';
import { S3AccessLevel } from '@core/s3';
import { VacancyService } from '@vacancy/application/services/vacancy.service';

@Controller('vacancies')
@UseGuards(JwtAuthGuard, RolesGuard)
export class VacancyController {
    constructor(
        private readonly storageService: StorageService,
        private readonly vacancyService: VacancyService,
    ) {}

    @Post(':id/image')
    @Roles(RoleType.EMPLOYER)
    @UseInterceptors(FileInterceptor('file'))
    async uploadVacancyImage(
        @Param('id') vacancyId: string,
        @UploadedFile() file: Express.Multer.File,
        @CurrentUser() user: { id: string; companyId?: string },
    ) {
        // Проверяем права (HR компании)
        const vacancy = await this.vacancyService.findById(vacancyId);
        if (vacancy.project.companyId !== user.companyId) {
            throw new ForbiddenException('You can only update vacancies of your company');
        }

        const fileInfo = await this.storageService.uploadFile(file, {
            path: FilePath.VACANCY_IMAGE,
            entityId: vacancyId,
            accessLevel: S3AccessLevel.PUBLIC,
            validateType: true,
            validateSize: true,
        });

        await this.vacancyService.updateImage(vacancyId, fileInfo.url);
        return { url: fileInfo.url };
    }
}
```

## Пример 5: Загрузка медиа для поста

```typescript
// api/controllers/post.controller.ts
import { Controller, Post, UseInterceptors, UploadedFile, UseGuards, Body } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { StorageService } from '@core/storage/application/services/storage.service';
import { FilePath } from '@core/storage/domain/enums/file-path.enum';
import { S3AccessLevel } from '@core/s3';

@Controller('posts')
@UseGuards(JwtAuthGuard)
export class PostController {
    constructor(private readonly storageService: StorageService) {}

    @Post('upload-media')
    @UseInterceptors(FileInterceptor('file'))
    async uploadPostMedia(
        @UploadedFile() file: Express.Multer.File,
        @Body('type') type: 'image' | 'video',
        @CurrentUser() user: { id: string },
    ) {
        const path = type === 'video' ? FilePath.POST_VIDEO : FilePath.POST_IMAGE;

        const fileInfo = await this.storageService.uploadFile(file, {
            path,
            entityId: user.id,
            accessLevel: S3AccessLevel.PUBLIC,
            validateType: true,
            validateSize: true,
        });

        return { url: fileInfo.url };
    }
}
```

## Best Practices

1. **Проверяйте права доступа** - перед загрузкой
2. **Валидируйте файлы** - тип и размер
3. **Обновляйте БД** - после успешной загрузки
4. **Удаляйте старые файлы** - при обновлении
5. **Храните ключи в БД** - для возможности удаления
