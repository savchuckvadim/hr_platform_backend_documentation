# Followers System - Система подписок

## Обзор

Система подписок на профили кандидатов и компаний на HR Platform.

## Модель данных

### Follow

```prisma
model Follow {
  id          String   @id @default(uuid())
  followerId  String   @map("follower_id")  // Кто подписывается
  followingId String   @map("following_id")  // На кого подписываются (Profile ID)
  profileType ProfileType @map("profile_type") // CANDIDATE | COMPANY
  createdAt   DateTime @default(now())

  follower User @relation("Follower", fields: [followerId], references: [id], onDelete: Cascade)
  following // Полиморфная связь (CandidateProfile или Company)

  @@unique([followerId, followingId, profileType])
  @@map("follows")
  @@index([followerId])
  @@index([followingId, profileType])
}
```

## Service методы

### FollowersService

```typescript
// application/services/followers.service.ts
import { Injectable, ForbiddenException } from '@nestjs/common';
import { FollowersRepository } from '../infrastructure/repositories/followers.repository';
import { AppEventBus } from '@core/events/event-bus.service';
import { AppEvent } from '@core/events/events.types';
import { ProfileType } from '@prisma/client';

@Injectable()
export class FollowersService {
    constructor(
        private readonly repository: FollowersRepository,
        private readonly eventBus: AppEventBus,
    ) {}

    /**
     * Подписка на профиль
     */
    async follow(
        currentUserId: string,
        followingId: string,
        profileType: ProfileType,
    ): Promise<void> {
        // Нельзя подписаться на себя
        if (currentUserId === followingId) {
            throw new ForbiddenException('You cannot follow yourself');
        }

        await this.repository.follow(currentUserId, followingId, profileType);

        // Публикуем событие
        this.eventBus.emit(AppEvent.PROFILE_FOLLOWED, {
            followerId: currentUserId,
            followingId,
            profileType,
        });
    }

    /**
     * Отписка от профиля
     */
    async unfollow(
        currentUserId: string,
        followingId: string,
        profileType: ProfileType,
    ): Promise<void> {
        await this.repository.unfollow(currentUserId, followingId, profileType);

        this.eventBus.emit(AppEvent.PROFILE_UNFOLLOWED, {
            followerId: currentUserId,
            followingId,
            profileType,
        });
    }

    /**
     * Получение подписчиков профиля
     */
    async getFollowers(
        profileId: string,
        profileType: ProfileType,
        currentUserId?: string,
    ): Promise<Array<{
        id: string;
        name: string;
        email: string;
        avatar?: string;
        followedAt: Date;
    }>> {
        return this.repository.getFollowers(profileId, profileType);
    }

    /**
     * Получение подписок профиля
     */
    async getFollowing(
        profileId: string,
        profileType: ProfileType,
        currentUserId?: string,
    ): Promise<Array<{
        id: string;
        name: string;
        email: string;
        avatar?: string;
        followedAt: Date;
    }>> {
        return this.repository.getFollowing(profileId, profileType);
    }

    /**
     * Проверка подписки
     */
    async isFollowing(
        followerId: string,
        followingId: string,
        profileType: ProfileType,
    ): Promise<boolean> {
        return this.repository.isFollowing(followerId, followingId, profileType);
    }
}
```

## Controller endpoints

```typescript
// api/controllers/followers.controller.ts
import { Controller, Post, Delete, Get, Param, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';
import { JwtAuthGuard } from '@core/guards/jwt-auth.guard';
import { CurrentUser } from '@core/decorators/current-user.decorator';
import { FollowersService } from '../application/services/followers.service';
import { ProfileType } from '@prisma/client';

@ApiTags('Followers')
@Controller('followers')
@UseGuards(JwtAuthGuard)
export class FollowersController {
    constructor(private readonly followersService: FollowersService) {}

    @Post(':profileId')
    @ApiOperation({ summary: 'Follow a profile' })
    async follow(
        @Param('profileId') profileId: string,
        @Query('profileType') profileType: ProfileType,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.followersService.follow(user.id, profileId, profileType);
    }

    @Delete(':profileId')
    @ApiOperation({ summary: 'Unfollow a profile' })
    async unfollow(
        @Param('profileId') profileId: string,
        @Query('profileType') profileType: ProfileType,
        @CurrentUser() user: { id: string },
    ): Promise<void> {
        return this.followersService.unfollow(user.id, profileId, profileType);
    }

    @Get(':profileId/followers')
    @ApiOperation({ summary: 'Get profile followers' })
    async getFollowers(
        @Param('profileId') profileId: string,
        @Query('profileType') profileType: ProfileType,
        @CurrentUser() user?: { id: string },
    ) {
        return this.followersService.getFollowers(profileId, profileType, user?.id);
    }

    @Get(':profileId/following')
    @ApiOperation({ summary: 'Get profile following' })
    async getFollowing(
        @Param('profileId') profileId: string,
        @Query('profileType') profileType: ProfileType,
        @CurrentUser() user?: { id: string },
    ) {
        return this.followersService.getFollowing(profileId, profileType, user?.id);
    }
}
```

## Best Practices

1. **Проверяйте дубликаты** - через unique constraint
2. **Публикуйте события** - для уведомлений
3. **Используйте индексы** - для быстрого поиска
4. **Ограничивайте подписки** - если нужно
