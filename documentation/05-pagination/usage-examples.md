# Usage Examples - Примеры использования пагинации

## Обзор

Примеры использования модуля пагинации в различных модулях проекта. Демонстрирует offset-based и cursor-based пагинацию, работу с фильтрами, поиском и сортировкой.

## Пример 1: Вакансии (Offset-based)

**Модуль:** `vacancies`

**Сценарий:** Список вакансий с пагинацией, сортировкой и поиском.

### Controller

```typescript
import { Controller, Get, Query } from '@nestjs/common';
import { ApiOperation, ApiResponse } from '@nestjs/swagger';
import { Pagination } from '@core/pagination/decorators/pagination.decorator';
import { PaginationDto } from '@core/pagination/dto/pagination.dto';
import { PaginatedResultDto } from '@core/pagination/dto/paginated-result.dto';
import { PaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';
import { VacancyService } from '../application/services/vacancy.service';
import { VacancyDto } from '../dto/vacancy.dto';

@Controller('vacancies')
export class VacancyController {
  constructor(private readonly vacancyService: VacancyService) {}

  @Get()
  @ApiOperation({ summary: 'Get paginated list of vacancies' })
  @ApiResponse({
    status: 200,
    description: 'Paginated list of vacancies',
    type: PaginatedResultDto,
  })
  async findAll(
    @Pagination() pagination: PaginationDto,
    @Query('search') search?: string,
    @Query('companyId') companyId?: string,
  ): Promise<PaginatedResult<VacancyDto>> {
    return this.vacancyService.findAll({
      ...pagination,
      search,
      companyId,
    });
  }
}
```

### Service

```typescript
import { Injectable } from '@nestjs/common';
import { PaginationDto } from '@core/pagination/dto/pagination.dto';
import { PaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';
import { PaginationUtil } from '@core/pagination/utils/pagination.util';
import { VacancyRepository } from '../infrastructure/repositories/vacancy.repository';
import { VacancyDto } from '../dto/vacancy.dto';
import { VacancyMapper } from '../mappers/vacancy.mapper';

@Injectable()
export class VacancyService {
  constructor(
    private readonly vacancyRepository: VacancyRepository,
    private readonly vacancyMapper: VacancyMapper,
  ) {}

  async findAll(options: {
    page: number;
    limit: number;
    sortBy: string;
    sortOrder: 'asc' | 'desc';
    search?: string;
    companyId?: string;
  }): Promise<PaginatedResult<VacancyDto>> {
    // Валидация поля сортировки
    PaginationUtil.validateSortField(options.sortBy, [
      'createdAt',
      'title',
      'salary',
      'updatedAt',
    ]);

    // Построение where условия
    const where = {
      ...(options.search && {
        title: { contains: options.search, mode: 'insensitive' },
      }),
      ...(options.companyId && { companyId: options.companyId }),
    };

    // Вычисление skip
    const skip = PaginationUtil.getSkip(options.page, options.limit);

    // Параллельные запросы
    const [entities, total] = await Promise.all([
      this.vacancyRepository.findMany({
        where,
        skip,
        take: options.limit,
        orderBy: PaginationUtil.createPrismaOrderBy(
          options.sortBy,
          options.sortOrder,
        ),
      }),
      this.vacancyRepository.count({ where }),
    ]);

    // Маппинг в DTO
    const items = entities.map((entity) =>
      this.vacancyMapper.toDto(entity),
    );

    // Создание результата
    return PaginationUtil.createPaginatedResult(
      items,
      total,
      options.page,
      options.limit,
    );
  }
}
```

### Запрос

```
GET /vacancies?page=1&limit=10&sortBy=createdAt&sortOrder=desc&search=developer
```

### Ответ

```json
{
  "items": [
    {
      "id": "vacancy-1",
      "title": "Senior Developer",
      "companyId": "company-1",
      "salary": 150000,
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ],
  "total": 50,
  "page": 1,
  "limit": 10,
  "totalPages": 5
}
```

## Пример 2: Отклики (Cursor-based)

**Модуль:** `applications`

**Сценарий:** Список откликов с cursor-based пагинацией для real-time обновлений.

### Controller

```typescript
import { Controller, Get, Query } from '@nestjs/common';
import { ApiOperation, ApiResponse } from '@nestjs/swagger';
import { CursorPagination } from '@core/pagination/decorators/pagination.decorator';
import { CursorPaginationDto } from '@core/pagination/dto/cursor-pagination.dto';
import { CursorPaginatedResultDto } from '@core/pagination/dto/cursor-paginated-result.dto';
import { CursorPaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';
import { ApplicationService } from '../application/services/application.service';
import { ApplicationDto } from '../dto/application.dto';

@Controller('applications')
export class ApplicationController {
  constructor(private readonly applicationService: ApplicationService) {}

  @Get()
  @ApiOperation({ summary: 'Get cursor-paginated list of applications' })
  @ApiResponse({
    status: 200,
    description: 'Cursor-paginated list of applications',
    type: CursorPaginatedResultDto,
  })
  async findAll(
    @CursorPagination() pagination: CursorPaginationDto,
    @Query('vacancyId') vacancyId?: string,
    @Query('status') status?: string,
  ): Promise<CursorPaginatedResult<ApplicationDto>> {
    return this.applicationService.findAll({
      ...pagination,
      vacancyId,
      status,
    });
  }
}
```

### Service

```typescript
import { Injectable } from '@nestjs/common';
import { CursorPaginationDto } from '@core/pagination/dto/cursor-pagination.dto';
import { CursorPaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';
import { PaginationUtil } from '@core/pagination/utils/pagination.util';
import { ApplicationRepository } from '../infrastructure/repositories/application.repository';
import { ApplicationDto } from '../dto/application.dto';
import { ApplicationMapper } from '../mappers/application.mapper';

@Injectable()
export class ApplicationService {
  constructor(
    private readonly applicationRepository: ApplicationRepository,
    private readonly applicationMapper: ApplicationMapper,
  ) {}

  async findAll(options: {
    cursor?: string;
    limit: number;
    vacancyId?: string;
    status?: string;
  }): Promise<CursorPaginatedResult<ApplicationDto>> {
    // Построение where условия
    const where = {
      ...(options.vacancyId && { vacancyId: options.vacancyId }),
      ...(options.status && { status: options.status }),
    };

    // Запрашиваем limit + 1 для определения hasNext
    const entities = await this.applicationRepository.findMany({
      where,
      take: options.limit + 1,
      cursor: options.cursor ? { id: options.cursor } : undefined,
      orderBy: { createdAt: 'desc' },
    });

    // Маппинг в DTO
    const items = entities.map((entity) =>
      this.applicationMapper.toDto(entity),
    );

    // Создание результата
    return PaginationUtil.createCursorPaginatedResult(
      items,
      options.limit,
    );
  }
}
```

### Запрос

```
GET /applications?cursor=app-id-123&limit=20&vacancyId=vacancy-1&status=PENDING
```

### Ответ

```json
{
  "items": [
    {
      "id": "app-id-124",
      "vacancyId": "vacancy-1",
      "candidateId": "candidate-1",
      "status": "PENDING",
      "createdAt": "2024-01-15T10:00:00Z"
    }
  ],
  "nextCursor": "app-id-144",
  "hasNext": true
}
```

## Пример 3: Резюме с расширенными фильтрами

**Модуль:** `resumes`

**Сценарий:** Поиск резюме с множественными фильтрами и сортировкой.

### DTO

```typescript
import { PaginationDto } from '@core/pagination/dto/pagination.dto';
import { IsOptional, IsString, IsArray, IsNumber, Min, Max } from 'class-validator';
import { ApiPropertyOptional } from '@nestjs/swagger';

export class ResumeQueryDto extends PaginationDto {
  @ApiPropertyOptional({ description: 'Search by title or description' })
  @IsOptional()
  @IsString()
  search?: string;

  @ApiPropertyOptional({ description: 'Filter by skills', type: [String] })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  skills?: string[];

  @ApiPropertyOptional({ description: 'Minimum salary' })
  @IsOptional()
  @IsNumber()
  @Min(0)
  minSalary?: number;

  @ApiPropertyOptional({ description: 'Maximum salary' })
  @IsOptional()
  @IsNumber()
  @Min(0)
  maxSalary?: number;

  @ApiPropertyOptional({ description: 'Filter by candidate ID' })
  @IsOptional()
  @IsString()
  candidateId?: string;
}
```

### Controller

```typescript
import { Controller, Get, Query } from '@nestjs/common';
import { ApiOperation, ApiResponse } from '@nestjs/swagger';
import { PaginatedResultDto } from '@core/pagination/dto/paginated-result.dto';
import { PaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';
import { ResumeService } from '../application/services/resume.service';
import { ResumeQueryDto } from '../dto/resume-query.dto';
import { ResumeDto } from '../dto/resume.dto';

@Controller('resumes')
export class ResumeController {
  constructor(private readonly resumeService: ResumeService) {}

  @Get()
  @ApiOperation({ summary: 'Search resumes with filters' })
  @ApiResponse({
    status: 200,
    description: 'Paginated list of resumes',
    type: PaginatedResultDto,
  })
  async search(
    @Query() query: ResumeQueryDto,
  ): Promise<PaginatedResult<ResumeDto>> {
    return this.resumeService.search(query);
  }
}
```

### Service

```typescript
import { Injectable } from '@nestjs/common';
import { PaginationUtil } from '@core/pagination/utils/pagination.util';
import { PaginatedResult } from '@core/pagination/interfaces/paginated-result.interface';
import { ResumeRepository } from '../infrastructure/repositories/resume.repository';
import { ResumeQueryDto } from '../dto/resume-query.dto';
import { ResumeDto } from '../dto/resume.dto';
import { ResumeMapper } from '../mappers/resume.mapper';

@Injectable()
export class ResumeService {
  constructor(
    private readonly resumeRepository: ResumeRepository,
    private readonly resumeMapper: ResumeMapper,
  ) {}

  async search(query: ResumeQueryDto): Promise<PaginatedResult<ResumeDto>> {
    // Валидация сортировки
    PaginationUtil.validateSortField(query.sortBy, [
      'createdAt',
      'updatedAt',
      'title',
      'salary',
    ]);

    // Нормализация пагинации
    const { page, limit } = PaginationUtil.normalizePagination(
      query.page,
      query.limit,
      10,
      50, // Максимум 50 для сложных запросов
    );

    // Построение where условия
    const where = {
      ...(query.search && {
        OR: [
          { title: { contains: query.search, mode: 'insensitive' } },
          { description: { contains: query.search, mode: 'insensitive' } },
        ],
      }),
      ...(query.skills && {
        skills: {
          some: {
            skill: {
              name: { in: query.skills },
            },
          },
        },
      }),
      ...(query.minSalary && { salary: { gte: query.minSalary } }),
      ...(query.maxSalary && { salary: { lte: query.maxSalary } }),
      ...(query.candidateId && { candidateId: query.candidateId }),
    };

    // Запросы
    const skip = PaginationUtil.getSkip(page, limit);
    const [entities, total] = await Promise.all([
      this.resumeRepository.findMany({
        where,
        skip,
        take: limit,
        orderBy: PaginationUtil.createPrismaOrderBy(
          query.sortBy,
          query.sortOrder,
        ),
        include: {
          skills: {
            include: {
              skill: true,
            },
          },
          candidate: {
            include: {
              profile: true,
            },
          },
        },
      }),
      this.resumeRepository.count({ where }),
    ]);

    // Маппинг
    const items = entities.map((entity) =>
      this.resumeMapper.toDto(entity),
    );

    return PaginationUtil.createPaginatedResult(items, total, page, limit);
  }
}
```

### Запрос

```
GET /resumes?page=1&limit=20&sortBy=salary&sortOrder=desc&search=developer&skills=javascript&skills=typescript&minSalary=100000&maxSalary=200000
```

## Пример 4: Список пользователей (HR модуль)

**Модуль:** `users`

**Сценарий:** Список пользователей с фильтрацией по ролям.

### Controller

```typescript
import { Controller, Get, Query, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '@auth/infrastructure/guards/jwt-auth.guard';
import { RolesGuard } from '@auth/infrastructure/guards/roles.guard';
import { Roles } from '@auth/infrastructure/decorators/roles.decorator';
import { CurrentUser } from '@auth/infrastructure/decorators/current-user.decorator';
import { Pagination } from '@core/pagination/decorators/pagination.decorator';
import { PaginationDto } from '@core/pagination/dto/pagination.dto';
import { UserService } from '../application/services/user.service';

@Controller('users')
@UseGuards(JwtAuthGuard, RolesGuard)
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  @Roles('ADMIN')
  async findAll(
    @CurrentUser() currentUser: User,
    @Pagination({ defaultLimit: 20, maxLimit: 100 }) pagination: PaginationDto,
    @Query('role') role?: string,
  ) {
    return this.userService.findAll({
      ...pagination,
      role,
      excludeUserId: currentUser.id, // Исключаем текущего пользователя
    });
  }
}
```

## Пример 5: Чат сообщения (Cursor-based для real-time)

**Модуль:** `chat`

**Сценарий:** История сообщений в чате с cursor-based пагинацией для бесконечной прокрутки.

### Controller

```typescript
import { Controller, Get, Param, Query } from '@nestjs/common';
import { CursorPagination } from '@core/pagination/decorators/pagination.decorator';
import { CursorPaginationDto } from '@core/pagination/dto/cursor-pagination.dto';
import { MessageService } from '../application/services/message.service';

@Controller('chats/:chatId/messages')
export class MessageController {
  constructor(private readonly messageService: MessageService) {}

  @Get()
  async getMessages(
    @Param('chatId') chatId: string,
    @CursorPagination({ defaultLimit: 50, maxLimit: 100 }) pagination: CursorPaginationDto,
  ) {
    return this.messageService.findByChatId(chatId, pagination);
  }
}
```

### Service

```typescript
@Injectable()
export class MessageService {
  async findByChatId(
    chatId: string,
    pagination: CursorPaginationDto,
  ): Promise<CursorPaginatedResult<MessageDto>> {
    // Запрашиваем limit + 1 для определения hasNext
    const messages = await this.messageRepository.findMany({
      where: { chatId },
      take: pagination.limit + 1,
      cursor: pagination.cursor ? { id: pagination.cursor } : undefined,
      orderBy: { createdAt: 'desc' }, // Новые сообщения первыми
    });

    // Реверсируем для правильного порядка (старые → новые)
    const reversed = messages.reverse();

    return PaginationUtil.createCursorPaginatedResult(reversed, pagination.limit);
  }
}
```

## Best Practices

1. **Выбор типа пагинации:**
   - Offset-based: когда нужна информация о количестве страниц, простые списки
   - Cursor-based: для больших наборов данных, real-time обновлений, бесконечной прокрутки

2. **Валидация сортировки:**
   ```typescript
   PaginationUtil.validateSortField(sortBy, allowedFields);
   ```

3. **Ограничение лимитов:**
   ```typescript
   @Pagination({ maxLimit: 50 }) // Для тяжёлых запросов
   ```

4. **Параллельные запросы:**
   ```typescript
   const [items, total] = await Promise.all([...]);
   ```

5. **Маппинг в DTO:**
   ```typescript
   const items = entities.map(entity => mapper.toDto(entity));
   ```

6. **Swagger документация:**
   ```typescript
   @ApiResponse({ type: PaginatedResultDto })
   ```
