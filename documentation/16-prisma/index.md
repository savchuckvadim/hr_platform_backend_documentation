# Core Prisma - Prisma сервис и модуль

## Описание

Глобальный Prisma сервис и модуль для работы с базой данных PostgreSQL. Обеспечивает единую точку доступа к БД для всех модулей приложения.

**Ключевые особенности:**
- Глобальный модуль (`@Global()`) - доступен во всех модулях без явного импорта
- Единое подключение к БД через PrismaClient
- Автоматическое управление жизненным циклом (подключение/отключение)
- Использование PrismaPg adapter для Prisma 7+
- Интеграция с ConfigService для централизованного управления конфигурацией

## Расположение

```
core/prisma/
├── prisma.service.ts    # PrismaService - глобальный сервис
├── prisma.module.ts     # PrismaModule - глобальный модуль
└── index.ts             # Экспорты модуля
```

## Настройка и подключение

### 1. Установка зависимостей

```bash
npm install @prisma/client
npm install @prisma/adapter-pg
npm install pg
```

### 2. Конфигурация

**Переменные окружения (`.env`):**
```env
DATABASE_URL="postgresql://user:password@localhost:5432/dbname?schema=public"
```

**Важно:** `DATABASE_URL` должен быть определён в конфигурации приложения. Если переменная отсутствует, PrismaService выбросит ошибку при инициализации.

### 3. Реализация PrismaService

```typescript
// core/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PrismaClient } from '@prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor(configService: ConfigService) {
    // Получаем DATABASE_URL из ConfigService (централизованное управление конфигурацией)
    const databaseUrl = configService.get<string>('DATABASE_URL');

    if (!databaseUrl) {
      throw new Error(
        'DATABASE_URL is not defined. Please check your .env file in apps/api/.env',
      );
    }

    // В Prisma 7+ необходимо использовать adapter для подключения
    // Для PostgreSQL используем PrismaPg adapter
    const adapter = new PrismaPg({
      connectionString: databaseUrl,
    });

    super({ adapter });
  }

  async onModuleInit(): Promise<void> {
    // Подключение к БД при инициализации модуля
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    // Отключение от БД при остановке модуля
    await this.$disconnect();
  }
}
```

**Особенности реализации:**
- Использует `ConfigService` для получения `DATABASE_URL` из централизованной конфигурации
- В Prisma 7+ требуется использовать adapter (`PrismaPg` для PostgreSQL)
- Автоматически подключается при старте приложения (`onModuleInit`)
- Автоматически отключается при остановке приложения (`onModuleDestroy`)
- Выбрасывает ошибку, если `DATABASE_URL` не определён

### 4. Реализация PrismaModule

```typescript
// core/prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

**Особенности:**
- Декоратор `@Global()` делает модуль глобальным - доступен во всех модулях без явного импорта
- Экспортирует `PrismaService` для использования в других модулях
- Должен быть импортирован в корневом модуле приложения (`AppModule`)

### 5. Регистрация в AppModule

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { PrismaModule } from '@core/prisma/prisma.module';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // ConfigModule также должен быть глобальным
    }),
    PrismaModule, // ✅ Импортируем глобальный PrismaModule
    // ... другие модули
  ],
})
export class AppModule {}
```

**Важно:**
- `ConfigModule` должен быть импортирован **до** `PrismaModule`, так как `PrismaService` использует `ConfigService`
- `ConfigModule` должен быть глобальным (`isGlobal: true`) или явно импортирован в `PrismaModule`

## Использование в модулях

### В модулях сущностей

Хотя `PrismaModule` является глобальным и не требует явного импорта, рекомендуется явно импортировать его для ясности:

```typescript
// modules/users/user.module.ts
import { Module } from '@nestjs/common';
import { PrismaModule } from '@core/prisma/prisma.module';
import { UserService } from './application/services/user.service';
import { UserRepository } from './infrastructure/repositories/user.repository';
import { UserPrismaRepository } from './infrastructure/repositories/user.prisma.repository';

@Module({
  imports: [PrismaModule], // ✅ Явный импорт (опционально, но рекомендуется)
  providers: [
    UserService,
    {
      provide: UserRepository, // Абстрактный репозиторий
      useClass: UserPrismaRepository, // Prisma реализация
    },
  ],
  exports: [UserService],
})
export class UserModule {}
```

### В репозиториях (EntityPrismaRepository)

Согласно структуре модулей, все репозитории следуют паттерну Repository с абстрактным классом и конкретной реализацией через Prisma:

**Абстрактный репозиторий:**
```typescript
// infrastructure/repositories/user.repository.ts
import { User } from '@prisma/client';
import { CreateUserDto } from '../../api/dto/create-user.dto';

export abstract class UserRepository {
  abstract findById(id: string): Promise<User | null>;
  abstract findByEmail(email: string): Promise<User | null>;
  abstract create(data: CreateUserDto): Promise<User>;
  abstract update(id: string, data: Partial<User>): Promise<User>;
  abstract delete(id: string): Promise<void>;
}
```

**Prisma реализация (EntityPrismaRepository):**
```typescript
// infrastructure/repositories/user.prisma.repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@core/prisma/prisma.service';
import { User } from '@prisma/client';
import { UserRepository } from './user.repository';
import { CreateUserDto } from '../../api/dto/create-user.dto';

@Injectable()
export class UserPrismaRepository implements UserRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id },
    });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { email },
    });
  }

  async create(data: CreateUserDto): Promise<User> {
    return this.prisma.user.create({
      data: {
        email: data.email,
        password: data.password, // Должен быть хэширован в сервисе
        // ... другие поля
      },
    });
  }

  async update(id: string, data: Partial<User>): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data,
    });
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({
      where: { id },
    });
  }
}
```

**Регистрация в модуле:**
```typescript
// user.module.ts
@Module({
  imports: [PrismaModule],
  providers: [
    UserService,
    {
      provide: UserRepository, // Абстрактный класс
      useClass: UserPrismaRepository, // Конкретная реализация
    },
  ],
})
export class UserModule {}
```

**Использование в сервисе:**
```typescript
// application/services/user.service.ts
import { Injectable } from '@nestjs/common';
import { UserRepository } from '../infrastructure/repositories/user.repository';

@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async findById(id: string) {
    return this.userRepository.findById(id);
  }
}
```

### Работа с отношениями (includes)

```typescript
// user.prisma.repository.ts
async findByIdWithProfile(id: string) {
  return this.prisma.user.findUnique({
    where: { id },
    include: {
      profile: true,
      roleContexts: {
        include: {
          userRole: true,
          hrRole: true,
          company: true,
        },
      },
    },
  });
}
```

### Транзакции

```typescript
// user.prisma.repository.ts
async createWithProfile(userData: CreateUserDto, profileData: CreateProfileDto) {
  return this.prisma.$transaction(async (tx) => {
    const user = await tx.user.create({
      data: userData,
    });

    const profile = await tx.candidateProfile.create({
      data: {
        ...profileData,
        userId: user.id,
      },
    });

    return { user, profile };
  });
}
```

## Преимущества

1. **Глобальный модуль** - доступен во всех модулях без явного импорта
2. **Единое подключение** - один экземпляр PrismaClient для всего приложения
3. **Автоматическое управление** - подключение/отключение при старте/остановке
4. **Типобезопасность** - полная типизация через Prisma Client
5. **Централизованная конфигурация** - использование ConfigService
6. **Поддержка Prisma 7+** - использование adapter для подключения

## Best Practices

1. **Используйте через репозитории** - не напрямую в сервисах
   - Сервисы должны работать с абстрактными репозиториями
   - Конкретная реализация через Prisma изолирована в `EntityPrismaRepository`

2. **Закрывайте соединение** - при остановке приложения
   - `onModuleDestroy` автоматически вызывает `$disconnect()`

3. **Обрабатывайте ошибки** - при работе с БД
   ```typescript
   try {
     return await this.prisma.user.findUnique({ where: { id } });
   } catch (error) {
     throw new InternalServerErrorException('Database error', error);
   }
   ```

4. **Используйте транзакции** - для сложных операций
   ```typescript
   await this.prisma.$transaction([
     this.prisma.user.create({ data: userData }),
     this.prisma.profile.create({ data: profileData }),
   ]);
   ```

5. **Используйте типизированные запросы** - Prisma предоставляет полную типизацию
   ```typescript
   const user: User = await this.prisma.user.findUnique({ where: { id } });
   ```

6. **Избегайте N+1 проблем** - используйте `include` для загрузки связанных данных
   ```typescript
   // ✅ Хорошо
   await this.prisma.user.findMany({
     include: { profile: true },
   });

   // ❌ Плохо
   const users = await this.prisma.user.findMany();
   for (const user of users) {
     user.profile = await this.prisma.profile.findUnique({ where: { userId: user.id } });
   }
   ```

## Структура использования в проекте

```
core/prisma/                    # Глобальный Prisma модуль
├── prisma.service.ts
├── prisma.module.ts
└── index.ts

modules/users/                  # Модуль пользователей
├── infrastructure/
│   └── repositories/
│       ├── user.repository.ts           # Абстрактный репозиторий
│       └── user.prisma.repository.ts    # Prisma реализация (использует PrismaService)
└── user.module.ts              # Регистрирует UserPrismaRepository

modules/applications/           # Модуль откликов
├── infrastructure/
│   └── repositories/
│       ├── application.repository.ts
│       └── application.prisma.repository.ts # Prisma реализация
└── application.module.ts
```

## Примечания

- `PrismaService` является глобальным и доступен во всех модулях
- Все репозитории должны следовать паттерну Repository с абстрактным классом и Prisma реализацией
- Сервисы работают только с абстрактными репозиториями, не зная о Prisma
- Это обеспечивает возможность замены реализации (например, на TypeORM) без изменения сервисов
