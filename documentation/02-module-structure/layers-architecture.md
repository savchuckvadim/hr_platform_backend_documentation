# Архитектура модуля - Структура и компоненты

## Общее описание

Каждый модуль в проекте следует принципам луковой (onion) архитектуры и состоит из следующих слоев:

```
module-name/
├── api/                    # API слой (контроллеры, DTO)
│   ├── controllers/
│   └── dto/
├── application/            # Application слой (бизнес-логика)
│   ├── services/
│   ├── use-cases/
│   └── events/            # Локальные события модуля (опционально)
├── domain/                 # Domain слой (сущности)
│   └── entity/
├── infrastructure/         # Infrastructure слой (репозитории, внешние сервисы)
│   ├── repositories/
│   ├── processors/        # Обработчики очередей (опционально)
│   └── listeners/         # Listeners для глобальных событий (опционально)
├── __tests__/             # Тесты модуля
├── module-name.module.ts  # NestJS модуль
└── index.ts               # Экспорты модуля
```

## Обязательные компоненты

### 1. API слой (`api/`)

**Всегда присутствует** для модулей с REST API endpoints.

- **`controllers/`** - NestJS контроллеры для обработки HTTP запросов
- **`dto/`** - Data Transfer Objects для валидации входящих/исходящих данных

**Пример:**
```typescript
// api/controllers/applications.controller.ts
@Controller('applications')
export class ApplicationsController {
  @Post()
  async create(@Body() dto: CreateApplicationDto) { }
}
```

### 2. Application слой (`application/`)

**Всегда присутствует** - содержит бизнес-логику модуля.

- **`services/`** - Сервисы с бизнес-логикой
- **`use-cases/`** - Use Cases для координации операций (опционально, но рекомендуется)
- **`events/`** - Локальные события модуля (см. раздел про события)

**Пример:**
```typescript
// application/services/application.service.ts
@Injectable()
export class ApplicationService {
  async create(dto: CreateApplicationDto, candidateId: string) { }
}
```

### 3. Domain слой (`domain/`)

**Всегда присутствует** - содержит доменные сущности.

- **`entity/`** - Доменные сущности (чистая бизнес-логика, без зависимостей от БД)

**Пример:**
```typescript
// domain/entity/application.entity.ts
export class Application {
  constructor(
    public readonly id: string,
    public readonly candidateId: string,
    public readonly vacancyId: string,
    public readonly status: ApplicationStatus,
  ) {}
}
```

### 4. Infrastructure слой (`infrastructure/`)

**Всегда присутствует** - содержит реализацию доступа к данным и внешним сервисам.

- **`repositories/`** - Репозитории для работы с БД
- **`processors/`** - Обработчики очередей (см. раздел про очереди)
- **`listeners/`** - Listeners для глобальных событий (см. раздел про события)

#### Структура репозиториев

Репозитории следуют паттерну Repository с абстрактным классом и конкретной реализацией через Prisma.

**Абстрактный репозиторий:**
```typescript
// infrastructure/repositories/user.repository.ts
import { User } from "generated/prisma";
import { CreateUserDto } from "../../api/dto/user.dto";
import { UserWithFollowStatusType } from "../../application/types/user.type";

export abstract class UserRepository {
  abstract getAll(currentUserId: string): Promise<UserWithFollowStatusType[]>;
  abstract getByIds(ids: string[]): Promise<User[]>;
  abstract findByEmail(email: string): Promise<User>;
  abstract findById(id: string): Promise<UserWithFollowStatusType>;
  abstract create(user: CreateUserDto): Promise<User>;
  abstract activate(activationLink: string): Promise<User>;
  abstract update(user: User): Promise<User>;
  abstract delete(id: string): Promise<void>;
}
```

**Prisma реализация:**
```typescript
// infrastructure/repositories/user.prisma.repository.ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "@/core";
import { User } from "generated/prisma";
import { UserRepository } from "./user.repository";
import { CreateUserDto } from "../../api/dto/user.dto";
import { UserWithFollowStatusType } from "../../application/types/user.type";

@Injectable()
export class UserPrismaRepository implements UserRepository {
  constructor(private readonly prisma: PrismaService) {}

  async getAll(currentUserId: string): Promise<UserWithFollowStatusType[]> {
    const users = await this.prisma.user.findMany({
      where: {
        id: { not: currentUserId },
      },
      include: {
        followers: true,
        following: true,
        profile: true,
      },
    });

    return users.map((user) => ({
      ...user,
      avatarUrl: user.profile?.avatar || '',
      isFollowing: user.following.some(f => f.followerId === currentUserId),
      isFollower: user.followers.some(f => f.followingId === currentUserId),
      followersCount: user.followers.length,
      followingCount: user.following.length,
    }));
  }

  async findById(id: string): Promise<UserWithFollowStatusType> {
    const user = await this.prisma.user.findUnique({
      where: { id },
      include: {
        followers: true,
        following: true,
        profile: true,
      },
    });

    if (!user) {
      throw new NotFoundException('User not found');
    }

    return {
      ...user,
      avatarUrl: user.profile?.avatar || '',
      followersCount: user.followers.length,
      followingCount: user.following.length,
    };
  }

  async create(user: CreateUserDto): Promise<User> {
    // Реализация создания пользователя
    return await this.prisma.user.create({
      data: {
        ...user,
        password: await hash(user.password, 10),
        activationLink: randomUUID(),
      },
    });
  }

  // Остальные методы...
}
```

**Регистрация в модуле:**
```typescript
// user.module.ts
@Module({
  providers: [
    UserService,
    {
      provide: UserRepository,
      useClass: UserPrismaRepository, // ✅ Используем Prisma реализацию
    },
  ],
})
export class UserModule {}
```

**Принципы:**
- Абстрактный класс определяет контракт репозитория
- Prisma реализация содержит всю логику работы с БД
- Сервисы зависят от абстракции, а не от конкретной реализации
- Легко заменить Prisma на другую реализацию (например, для тестов)

### 5. Тесты (`__tests__/`)

**Всегда присутствует** - тесты для всех слоев модуля.

См. подробнее: [Тестирование модулей](./testing.md)

## Опциональные компоненты

### Очереди (Queues)

#### Когда использовать очереди?

Используйте очереди (BullMQ) когда:

1. **Долгие операции** - задачи, которые выполняются долго (отправка email, обработка файлов)
2. **Асинхронная обработка** - не нужно ждать результата сразу
3. **Retry логика** - нужны повторные попытки при ошибках
4. **Масштабируемость** - нужно распределить нагрузку между воркерами
5. **Очередь задач** - много задач одного типа, которые нужно обработать последовательно

**Примеры:**
- Отправка email
- Генерация отчетов
- Обработка изображений
- Интеграция с внешними API
- Массовая рассылка

#### Структура с очередями

Если модуль использует очереди, добавляются:

```
infrastructure/
├── processors/           # Обработчики задач из очереди
│   └── mail.processor.ts
```

**Важно:** Вся бизнес-логика должна быть в сервисах, processor только координирует.

См. подробнее: [Queue Processors](../../08-queue/queue-processors.md)

**Пример:**
```typescript
// infrastructure/processors/mail.processor.ts
@Processor(MAIL_QUEUE_NAME)
export class MailProcessor extends WorkerHost {
  constructor(
    private readonly mailService: MailService,
    private readonly eventBus: AppEventBus,
  ) {}

  async process(job: Job<SendEmailJobData>): Promise<void> {
    const { to, subject, html } = job.data;

    // ✅ Только вызов сервиса для отправки email
    await this.mailService.sendEmail({
      to,
      subject,
      html,
    });
  }

  // ✅ @OnWorkerEvent - инфраструктурные вещи + бизнес-события
  @OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
  async onCompleted(job: Job<SendEmailJobData>) {
    // ✅ Логирование, метрики, мониторинг
    this.logger.log(`Email job completed: ${job.id}`);
    this.metrics.increment('email.sent');

    // ✅ Эмитим бизнес-событие после успешного завершения задачи
    this.eventBus.emit(AppEvent.EMAIL_SENT, {
      emailType: job.data.emailType,
      to: job.data.to,
      subject: job.data.subject,
    });
  }
}
```

**Важно:**
- `process()` только обрабатывает очередь и вызывает сервис
- `@OnWorkerEvent('completed')` выполняет логирование, метрики и эмитит бизнес-события
- Дополнительная бизнес-логика (например, отправка в Telegram) через отдельные listeners на бизнес-события

См. подробнее: [Queue Processors](../../08-queue/queue-processors.md)

### События (Events)

#### Когда использовать события?

Используйте события когда:

1. **Разделение ответственности** - один модуль создает сущность, другой должен отреагировать
2. **Слабая связанность** - модули не должны знать друг о друге напрямую
3. **Множественные обработчики** - одно действие должно вызвать несколько реакций
4. **Асинхронная обработка** - обработка может быть асинхронной

**Примеры:**
- При создании пользователя → отправить welcome email
- При создании отклика → уведомить работодателя
- При изменении статуса → обновить аналитику

#### Локальные vs Глобальные события

##### Локальные события (внутри модуля)

**Когда использовать:**
- События используются только внутри модуля
- Нет необходимости, чтобы другие модули подписывались
- Внутренняя координация между компонентами модуля

**Где хранить:**
```
application/
└── events/
    └── resume-updated.event.ts  # Локальное событие модуля
```

**Пример:**
```typescript
// application/events/resume-updated.event.ts
export class ResumeUpdatedEvent {
  constructor(
    public readonly resumeId: string,
    public readonly candidateId: string,
  ) {}
}
```

**Использование:**
```typescript
// Внутри модуля через локальный EventEmitter
this.eventEmitter.emit('resume.updated', new ResumeUpdatedEvent(...));
```

##### Глобальные события (между модулями)

**Когда использовать:**
- События должны быть доступны другим модулям
- Другие модули должны подписаться на события
- Нужна централизованная система событий

**Где хранить:**
```
src/core/events/
└── events.types.ts  # Глобальные события в Core Events Module
```

**Пример:**
```typescript
// src/core/events/events.types.ts
export enum AppEvent {
  USER_CREATED = 'user.created',
  APPLICATION_CREATED = 'application.created',
  // ...
}

export interface EventPayloadMap {
  [AppEvent.USER_CREATED]: {
    userId: string;
    email: string;
  };
  // ...
}
```

**Где публикуются глобальные события:**

Глобальные события могут публиковаться из разных мест в зависимости от типа операции:

1. **Из сервиса или Use Case** (для легких операций):
```typescript
// application/services/user.service.ts
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly eventBus: AppEventBus,
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.create(dto);

    // ✅ Публикуем глобальное событие после успешного создания
    this.eventBus.emit(AppEvent.USER_CREATED, {
      userId: user.id,
      email: user.email,
      role: user.role,
    });

    return user;
  }
}
```

2. **Из Use Case** (рекомендуется для координации):
```typescript
// application/use-cases/create-application.use-case.ts
@Injectable()
export class CreateApplicationUseCase {
  constructor(
    private readonly applicationService: ApplicationService,
    private readonly eventBus: AppEventBus,
  ) {}

  async execute(dto: CreateApplicationDto, candidateId: string) {
    const application = await this.applicationService.create(dto, candidateId);

    // ✅ Публикуем глобальное событие
    this.eventBus.emit(AppEvent.APPLICATION_CREATED, {
      applicationId: application.id,
      candidateId,
      vacancyId: dto.vacancyId,
    });

    return application;
  }
}
```

3. **Из Worker (после обработки очереди)**:
```typescript
// infrastructure/processors/mail.processor.ts
@OnWorkerEvent('completed')
async onCompleted(job: Job<SendEmailJobData>) {
  // Логирование, метрики
  this.logger.log(`Email job completed: ${job.id}`);

  // ✅ Публикуем глобальное событие после успешной отправки
  this.eventBus.emit(AppEvent.EMAIL_SENT, {
    emailType: job.data.emailType,
    to: job.data.to,
    subject: job.data.subject,
  });
}
```

**Подписка в другом модуле:**
```typescript
// В любом модуле через listener
@OnAppEvent(AppEvent.USER_CREATED)
handleUserCreated(payload: EventPayloadMap[AppEvent.USER_CREATED]) {
  // Обработка события
}
```

**Правило:**
- Если событие нужно только внутри модуля → **локальное событие** в `application/events/`
- Если на событие должны подписаться другие модули → **глобальное событие** в `core/events/`

См. подробнее: [Core Events Module](../../15-events/index.md)

#### Структура с событиями

Если модуль использует события:

**Локальные события:**
```
application/
└── events/
    └── resume-updated.event.ts  # Локальное событие модуля
```

**Глобальные события:**
- События добавляются в `src/core/events/events.types.ts`
- Модуль использует `AppEventBus` из `EventModule`

**Listeners для бизнес-событий:**
```
infrastructure/
└── listeners/
    └── application-email.listener.ts  # Обработка глобальных событий (отправка email)
```

**Пример модуля с событиями:**
```typescript
// application.module.ts
@Module({
  imports: [EventModule], // ✅ Для глобальных событий
  providers: [
    ApplicationService,
    CreateApplicationUseCase,
    ApplicationEmailListener, // ✅ Listener для бизнес-событий (отправка email)
  ],
})
export class ApplicationModule {}
```

## Примеры структур модулей

### Простой модуль (без очередей и событий)

```
user/
├── api/
│   ├── controllers/
│   └── dto/
├── application/
│   └── services/
├── domain/
│   └── entity/
├── infrastructure/
│   └── repositories/
├── __tests__/
├── user.module.ts
└── index.ts
```

### Модуль с очередями

```
mail/
├── api/
├── application/
│   └── services/
│       └── mail.service.ts
├── domain/
├── infrastructure/
│   ├── repositories/
│   ├── processors/        # ✅ Worker - обработка очереди
│   │   └── mail.processor.ts
│   └── listeners/         # ✅ Listeners для бизнес-событий
│       └── email-telegram.listener.ts  # Обработка EMAIL_SENT события (Telegram)
├── __tests__/
├── mail.module.ts
└── index.ts
```

**Поток:**
1. Listener на глобальное событие (например, `USER_CREATED`) отправляет задачу в очередь
2. `MailProcessor.process()` обрабатывает задачу из очереди, вызывает `mailService.sendEmail()`
3. После успешного завершения задачи срабатывает `@OnWorkerEvent('completed')`
4. В `@OnWorkerEvent('completed')` выполняется:
   - Логирование и метрики (инфраструктурные вещи)
   - Эмит бизнес-события `EMAIL_SENT`
5. Listener на `EMAIL_SENT` обрабатывает событие (например, отправка в Telegram)

**Важно:** Listeners могут выполнять логику сразу или отправлять в очередь для тяжелых операций (когда нужен retry, долгое выполнение).

### Модуль с локальными событиями

```
resume/
├── api/
├── application/
│   ├── services/
│   ├── use-cases/
│   └── events/            # ✅ Локальные события
│       └── resume-updated.event.ts
├── domain/
├── infrastructure/
├── __tests__/
├── resume.module.ts
└── index.ts
```

### Модуль с глобальными событиями

```
application/
├── api/
├── application/
│   ├── services/
│   └── use-cases/
│       └── create-application.use-case.ts  # ✅ Публикует глобальное событие
├── domain/
├── infrastructure/
│   └── listeners/         # ✅ Listeners для бизнес-событий
│       └── application-email.listener.ts  # Обработка APPLICATION_CREATED события (отправка email)
├── __tests__/
├── application.module.ts
└── index.ts

# Событие определено в:
src/core/events/events.types.ts  # ✅ Глобальное событие APPLICATION_CREATED
```

### Полный модуль (очереди + события)

```
mail/
├── api/
│   ├── controllers/
│   └── dto/
├── application/
│   └── services/
│       └── mail.service.ts
├── domain/
├── infrastructure/
│   ├── processors/        # ✅ Worker - обработка очереди
│   │   └── mail.processor.ts
│   └── listeners/         # ✅ Listeners для бизнес-событий
│       └── email-telegram.listener.ts  # Обработка EMAIL_SENT события (Telegram)
├── __tests__/
├── mail.module.ts
└── index.ts

# Глобальное событие определено в:
src/core/events/events.types.ts  # ✅ EMAIL_SENT
```

**Поток:**
1. Listener на `USER_CREATED` отправляет задачу в очередь отправки email (тяжелая операция)
2. Worker обрабатывает задачу из очереди, вызывает `mailService.sendEmail()`
3. После успешной отправки Worker эмитит `EMAIL_SENT`
4. Listener на `EMAIL_SENT` обрабатывает событие (например, отправка в Telegram)

**Важно:** Listeners могут выполнять логику сразу (для быстрых операций) или отправлять в очередь (для тяжелых операций - когда нужен retry, долгое выполнение).

### Listeners - выполнение сразу или очередь?

**Выполнение сразу** - для быстрых операций:
- Создание уведомления в БД
- Обновление статуса
- Простые вычисления

**Отправка в очередь** - для тяжелых операций:
- Отправка email
- Генерация отчетов
- Обработка файлов
- Интеграция с внешними API

**Пример:**
```typescript
// Listener выполняет сразу
@OnAppEvent(AppEvent.APPLICATION_CREATED)
async handleApplicationCreated(payload) {
  // ✅ Быстрая операция - выполняется сразу
  // Пример: отправка email через очередь
  await this.mailQueue.add(MAIL_QUEUE_JOB_NAMES.SEND_EMAIL, {
    to: [payload.employerEmail],
    subject: 'New Application',
    html: await this.renderApplicationEmail(payload),
    emailType: EmailType.APPLICATION_NEW,
  });
}

// Listener отправляет в очередь
@OnAppEvent(AppEvent.USER_CREATED)
async handleUserCreated(payload) {
  // ✅ Тяжелая операция - отправка в очередь
  await this.queueDispatcher.dispatch(
    QueueNames.MAIL,
    JobNames.SEND_EMAIL,
    { to: [payload.email], subject: 'Welcome!' },
  );
}
```

## Принятие решений

### Нужна ли очередь?

```
┌─────────────────────────────────────┐
│ Задача долгая (>1 сек)?             │
│ Или нужен retry?                    │
│ Или асинхронная обработка?          │
└──────────────┬──────────────────────┘
               │
        ┌──────▼──────┐
        │   ДА        │ → Используй очередь
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   НЕТ       │ → Синхронная обработка
        └─────────────┘
```

### Локальное или глобальное событие?

```
┌─────────────────────────────────────┐
│ Другие модули должны подписаться?   │
└──────────────┬──────────────────────┘
               │
        ┌──────▼──────┐
        │   ДА        │ → Глобальное событие
        │             │   (core/events/)
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   НЕТ       │ → Локальное событие
        │             │   (application/events/)
        └─────────────┘
```

## Ссылки на документацию

- [Core Events Module](../../15-events/index.md) - централизованная система событий
- [Queue Processors](../../08-queue/queue-processors.md) - обработчики очередей
- [Тестирование модулей](./testing.md) - принципы тестирования
- [Создание нового модуля](./module-template.md) - шаблон модуля
