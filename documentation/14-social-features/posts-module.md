# Posts Module - Модуль постов

## Обзор

Модуль для создания и управления постами на HR Platform. Поддержка постов на стенах кандидатов и компаний, с указанием конкретного автора (HR для компаний).

## Структура модуля

```
posts/
├── api/
│   ├── controllers/
│   │   └── posts.controller.ts
│   └── dto/
│       └── post.dto.ts
├── application/
│   ├── services/
│   │   └── post.service.ts
│   └── events/
│       └── post.events.ts
├── domain/
│   └── entity/
│       └── post.entity.ts
├── infrastructure/
│   ├── repositories/
│   │   ├── post.repository.ts
│   │   └── post.prisma.repository.ts
│   └── listeners/
├── posts.module.ts
└── index.ts
```

## Модель данных

### Post

```prisma
model Post {
  id            String      @id @default(uuid())
  authorId      String      @map("author_id")      // User ID - кто создал пост
  profileId     String      @map("profile_id")     // User ID (для CANDIDATE) или Company ID (для COMPANY)
  profileType   ProfileType @map("profile_type")   // CANDIDATE | COMPANY
  text          String?     @db.Text
  image         String?     @db.VarChar(500)
  video         String?     @db.VarChar(500)
  link          String?     @db.VarChar(500)
  viewsCount    Int         @default(0) @map("views_count")
  originalPostId String?    @map("original_post_id") // Для репостов
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt
  deletedAt     DateTime?   @map("deleted_at")

  author        User        @relation("PostAuthor", fields: [authorId], references: [id], onDelete: Cascade)
  // Полиморфная связь: для CANDIDATE - User.id, для COMPANY - Company.id
  profileUser   User?       @relation("PostProfileUser", fields: [profileId], references: [id], onDelete: Cascade)
  company       Company?    @relation("PostProfileCompany", fields: [profileId], references: [id], onDelete: Cascade)
  likes         PostLike[]
  hashtags      PostHashtag[]
  reposts       Post[]      @relation("PostRepost")
  originalPost  Post?       @relation("PostRepost", fields: [originalPostId], references: [id], onDelete: SetNull)

  @@map("posts")
  @@index([authorId])
  @@index([profileId, profileType])
  @@index([createdAt])
  @@index([deletedAt])
  @@index([originalPostId])
}

enum ProfileType {
  CANDIDATE
  COMPANY
}
```

### PostLike

```prisma
model PostLike {
  id        String   @id @default(uuid())
  postId    String   @map("post_id")
  userId    String   @map("user_id")
  isLike    Boolean  @default(true) @map("is_like") // true = лайк, false = дизлайк
  createdAt DateTime @default(now())

  post Post @relation(fields: [postId], references: [id], onDelete: Cascade)
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([postId, userId])
  @@map("post_likes")
  @@index([postId])
  @@index([userId])
}
```

## DTO

### CreatePostDto

```typescript
// api/dto/post.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsString, IsOptional, MaxLength, IsEnum } from 'class-validator';
import { ProfileType } from '@prisma/client';

export class CreatePostDto {
    @ApiProperty({ description: 'Text content', example: 'My post text', required: false })
    @IsOptional()
    @IsString()
    @MaxLength(2000, { message: 'Text must not exceed 2000 characters' })
    text?: string;

    @ApiProperty({ description: 'Image URL', example: 'https://example.com/image.jpg', required: false })
    @IsOptional()
    @IsString()
    @MaxLength(500, { message: 'Image URL must not exceed 500 characters' })
    image?: string;

    @ApiProperty({ description: 'Video URL', example: 'https://example.com/video.mp4', required: false })
    @IsOptional()
    @IsString()
    @MaxLength(500, { message: 'Video URL must not exceed 500 characters' })
    video?: string;

    @ApiProperty({ description: 'Link URL', example: 'https://example.com', required: false })
    @IsOptional()
    @IsString()
    @MaxLength(500, { message: 'Link URL must not exceed 500 characters' })
    link?: string;

    @ApiProperty({
        description: 'Profile ID (User ID for CANDIDATE or Company ID for COMPANY)',
        example: 'user-id-or-company-id',
        required: false
    })
    @IsOptional()
    @IsString()
    profileId?: string;

    @ApiProperty({
        description: 'Profile Type (CANDIDATE or COMPANY). If not provided, uses current user profile',
        enum: ProfileType,
        required: false
    })
    @IsOptional()
    @IsEnum(ProfileType)
    profileType?: ProfileType;

    @ApiProperty({
        description: 'Hashtags',
        example: ['#hr', '#recruiting'],
        required: false
    })
    @IsOptional()
    @IsString({ each: true })
    hashtags?: string[];
}
```

### PostDto

```typescript
export class PostDto {
    id: string;
    authorId: string;
    profileId: string;
    profileType: ProfileType;
    text?: string;
    image?: string;
    video?: string;
    link?: string;
    viewsCount: number;
    likesCount: number;
    dislikesCount: number;
    repostsCount: number;
    userLike?: { isLike: boolean } | null;
    originalPost?: PostDto | null;
    author: {
        id: string;
        name: string;
        email: string;
        avatar?: string;
    };
    profile: {
        id: string;
        name: string;
        type: ProfileType;
    };
    hashtags: string[];
    createdAt: Date;
    updatedAt: Date;

    constructor(post: FullPost) {
        this.id = post.id;
        this.authorId = post.authorId;
        this.profileId = post.profileId;
        this.profileType = post.profileType;
        this.text = post.text || undefined;
        this.image = post.image || undefined;
        this.video = post.video || undefined;
        this.link = post.link || undefined;
        this.viewsCount = post.viewsCount;
        this.likesCount = post.likesCount;
        this.dislikesCount = post.dislikesCount;
        this.repostsCount = post.repostsCount;
        this.userLike = post.userLike || undefined;
        this.originalPost = post.originalPost ? new PostDto(post.originalPost) : undefined;
        this.author = post.author;
        this.profile = post.profile;
        this.hashtags = post.hashtags || [];
        this.createdAt = post.createdAt;
        this.updatedAt = post.updatedAt;
    }
}
```

## Service

### PostService

```typescript
// application/services/post.service.ts
import { Injectable, ForbiddenException, NotFoundException } from '@nestjs/common';
import { PostRepository } from '../infrastructure/repositories/post.repository';
import { CreatePostDto, UpdatePostDto, PostDto, PaginatedPostsDto } from '../api/dto/post.dto';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import { S3Service } from '@core/s3';
import { RoleContextRepository } from '@auth/infrastructure/repositories/role-context.repository';
import { CompanyRepository } from '@company/infrastructure/repositories/company.repository';
@Injectable()
export class PostService {
    constructor(
        private readonly postRepository: PostRepository,
        private readonly eventBus: AppEventBus,
        private readonly s3Service: S3Service,
        private readonly roleContextRepository: RoleContextRepository,
        private readonly companyRepository: CompanyRepository,
    ) {}

    /**
     * Создание поста
     * @param authorId - ID автора (User ID)
     * @param data - данные поста
     */
    async createPost(authorId: string, data: CreatePostDto): Promise<PostDto> {
        // Определяем профиль для поста
        let profileId: string;
        let profileType: ProfileType;

        if (data.profileId && data.profileType) {
            // Пост на указанной стене
            profileId = data.profileId;
            profileType = data.profileType;

            // Проверяем права доступа
            await this.validatePostAccess(authorId, profileId, profileType);
        } else {
            // Пост на своей стене - определяем профиль автора
            const roleContext = await this.roleContextRepository.findActiveByUserId(authorId);

            if (roleContext.userRole === 'CANDIDATE') {
                // Для кандидата используем userId напрямую
                profileId = authorId;
                profileType = ProfileType.CANDIDATE;
            } else if (roleContext.type === 'EMPLOYER') {
                // Для компаний - проверяем, что это HR компании
                if (!roleContext.companyId) {
                    throw new ForbiddenException('You must be an HR of a company to post');
                }
                profileId = roleContext.companyId;
                profileType = ProfileType.COMPANY;
            } else {
                throw new ForbiddenException('Invalid role for posting');
            }
        }

        // Создаем пост
        const post = await this.postRepository.create({
            authorId,
            profileId,
            profileType,
            text: data.text,
            image: data.image,
            video: data.video,
            link: data.link,
            hashtags: data.hashtags || [],
        });

        // Публикуем событие
        this.eventBus.emit(AppEvent.POST_CREATED, {
            postId: post.id,
            authorId: post.authorId,
            profileId: post.profileId,
            profileType: post.profileType,
        });

        return new PostDto(post);
    }

    /**
     * Проверка прав доступа для поста
     */
    private async validatePostAccess(
        authorId: string,
        profileId: string,
        profileType: ProfileType,
    ): Promise<void> {
        if (profileType === ProfileType.CANDIDATE) {
            // Для кандидата - можно постить только на своей стене (userId)
            if (authorId !== profileId) {
                throw new ForbiddenException('You can only post on your own profile');
            }
        } else if (profileType === ProfileType.COMPANY) {
            // Для компании - проверяем, что автор является HR этой компании
            const roleContext = await this.roleContextRepository.findByUserIdAndCompanyId(
                authorId,
                profileId,
            );

            if (!roleContext || roleContext.type !== 'EMPLOYER') {
                throw new ForbiddenException('You must be an HR of this company to post');
            }

            // Проверяем, что компания существует
            const company = await this.companyRepository.findById(profileId);
            if (!company) {
                throw new NotFoundException('Company not found');
            }
        }
    }

    /**
     * Получение поста по ID
     */
    async getPostById(id: string, currentUserId?: string): Promise<PostDto> {
        const post = await this.postRepository.findById(id, currentUserId);
        if (!post) {
            throw new NotFoundException('Post not found');
        }
        return new PostDto(post);
    }

    /**
     * Обновление поста
     */
    async updatePost(id: string, userId: string, data: UpdatePostDto): Promise<PostDto> {
        // Проверка прав выполняется в Repository
        const post = await this.postRepository.update(id, userId, data);

        this.eventBus.emit(AppEvent.POST_UPDATED, {
            postId: post.id,
            authorId: post.authorId,
        });

        return new PostDto(post);
    }

    /**
     * Удаление поста
     */
    async deletePost(id: string, userId: string): Promise<void> {
        // Получаем пост перед удалением для удаления медиа
        const post = await this.postRepository.findById(id, userId);

        // Удаляем медиа файлы из S3
        const mediaUrls: string[] = [];
        if (post.image) mediaUrls.push(post.image);
        if (post.video) mediaUrls.push(post.video);

        if (mediaUrls.length > 0) {
            await this.s3Service.deleteFiles(mediaUrls);
        }

        // Удаляем пост из БД
        await this.postRepository.delete(id, userId);

        this.eventBus.emit(AppEvent.POST_DELETED, {
            postId: id,
            authorId: userId,
        });
    }

    /**
     * Получение ленты активности
     */
    async getFeed(currentUserId: string, cursor?: string, limit?: number): Promise<PaginatedPostsDto> {
        const result = await this.postRepository.getFeed(currentUserId, cursor, limit);
        return {
            posts: result.posts.map(post => new PostDto(post)),
            nextCursor: result.nextCursor,
            hasNext: result.hasNext,
        };
    }

    /**
     * Получение постов профиля
     */
    async getPostsByProfile(
        profileId: string,
        profileType: ProfileType,
        currentUserId?: string,
        cursor?: string,
        limit?: number,
    ): Promise<PaginatedPostsDto> {
        const result = await this.postRepository.getByProfile(
            profileId,
            profileType,
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
}
```

## Controller

### PostsController

```typescript
// api/controllers/posts.controller.ts
import { Controller, Get, Post, Put, Delete, Body, Param, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiQuery } from '@nestjs/swagger';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { PostService } from '../application/services/post.service';
import { CreatePostDto, UpdatePostDto, PostDto, PaginatedPostsDto } from '../dto/post.dto';
import { ProfileType } from '@prisma/client';

@ApiTags('Posts')
@Controller('posts')
@UseGuards(JwtAuthGuard)
export class PostsController {
    constructor(private readonly postService: PostService) {}

    @Post()
    @ApiOperation({ summary: 'Create a new post' })
    @ApiResponse({ status: 201, description: 'Post created', type: PostDto })
    async createPost(
        @Body() dto: CreatePostDto,
        @CurrentUser() user: { id: string },
    ): Promise<PostDto> {
        return this.postService.createPost(user.id, dto);
    }

    @Get('feed')
    @ApiOperation({ summary: 'Get feed posts' })
    @ApiQuery({ name: 'cursor', required: false, type: String })
    @ApiQuery({ name: 'limit', required: false, type: Number })
    @ApiResponse({ status: 200, description: 'Feed posts', type: PaginatedPostsDto })
    async getFeed(
        @Query('cursor') cursor?: string,
        @Query('limit') limit?: number,
        @CurrentUser() user?: { id: string },
    ): Promise<PaginatedPostsDto> {
        return this.postService.getFeed(user!.id, cursor, limit ? parseInt(limit.toString()) : undefined);
    }

    @Get('profile/:profileId')
    @ApiOperation({ summary: 'Get posts by profile' })
    @ApiQuery({ name: 'profileType', enum: ProfileType, required: true })
    @ApiQuery({ name: 'cursor', required: false, type: String })
    @ApiQuery({ name: 'limit', required: false, type: Number })
    @ApiResponse({ status: 200, description: 'Profile posts', type: PaginatedPostsDto })
    async getPostsByProfile(
        @Param('profileId') profileId: string,
        @Query('profileType') profileType: ProfileType,
        @Query('cursor') cursor?: string,
        @Query('limit') limit?: number,
        @CurrentUser() user?: { id: string },
    ): Promise<PaginatedPostsDto> {
        return this.postService.getPostsByProfile(
            profileId,
            profileType,
            user?.id,
            cursor,
            limit ? parseInt(limit.toString()) : undefined,
        );
    }

    @Get(':id')
    @ApiOperation({ summary: 'Get post by ID' })
    @ApiResponse({ status: 200, description: 'Post', type: PostDto })
    async getPostById(
        @Param('id') id: string,
        @CurrentUser() user?: { id: string },
    ): Promise<PostDto> {
        return this.postService.getPostById(id, user?.id);
    }

    @Put(':id')
    @ApiOperation({ summary: 'Update post' })
    @ApiResponse({ status: 200, description: 'Post updated', type: PostDto })
    async updatePost(
        @Param('id') id: string,
        @Body() dto: UpdatePostDto,
        @CurrentUser() user: { id: string },
    ): Promise<PostDto> {
        return this.postService.updatePost(id, user.id, dto);
    }

    @Delete(':id')
    @ApiOperation({ summary: 'Delete post' })
    @ApiResponse({ status: 200, description: 'Post deleted' })
    async deletePost(
        @Param('id') id: string,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.postService.deletePost(id, user.id);
    }
}
```

## Repository

### PostRepository

```typescript
// infrastructure/repositories/post.repository.ts
import { CreatePostDto, UpdatePostDto } from '../../api/dto/post.dto';
import { FullPost } from '../../domain/types/post.type';
import { ProfileType } from '@prisma/client';

export abstract class PostRepository {
    abstract create(data: {
        authorId: string;
        profileId: string;
        profileType: ProfileType;
        text?: string;
        image?: string;
        video?: string;
        link?: string;
        hashtags?: string[];
    }): Promise<FullPost>;

    abstract findById(id: string, currentUserId?: string): Promise<FullPost | null>;

    abstract update(id: string, userId: string, data: UpdatePostDto): Promise<FullPost>;

    abstract delete(id: string, userId: string): Promise<void>;

    abstract getFeed(
        currentUserId: string,
        cursor?: string,
        limit?: number,
    ): Promise<{ posts: FullPost[]; nextCursor?: string; hasNext: boolean }>;

    abstract getByProfile(
        profileId: string,
        profileType: ProfileType,
        currentUserId?: string,
        cursor?: string,
        limit?: number,
    ): Promise<{ posts: FullPost[]; nextCursor?: string; hasNext: boolean }>;
}
```

## Особенности для HR Platform

### Кто может постить от компании

1. **Root HR (HR_ADMIN)** - владелец компании, может постить от имени компании
2. **Нанятый HR** - HR сотрудник компании, может постить от имени компании

### Проверка прав

```typescript
// В PostService.validatePostAccess
if (profileType === ProfileType.COMPANY) {
    // Проверяем, что автор является HR этой компании
    const roleContext = await this.roleContextRepository.findByUserIdAndCompanyId(
        authorId,
        profileId,
    );

    if (!roleContext || roleContext.type !== 'EMPLOYER') {
        throw new ForbiddenException('You must be an HR of this company to post');
    }
}
```

## Best Practices

1. **Проверяйте права доступа** - перед созданием поста
2. **Удаляйте медиа из S3** - при удалении поста
3. **Используйте cursor pagination** - для ленты
4. **Публикуйте события** - для уведомлений
5. **Валидируйте данные** - перед сохранением
