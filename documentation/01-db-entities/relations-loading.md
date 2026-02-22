# Получение сущностей со связями

## Общее описание

При работе с Prisma важно правильно загружать связанные сущности (relations). Неправильная загрузка может привести к:
- N+1 проблемам (множественные запросы к БД)
- Отсутствию данных (null вместо связанных сущностей)
- Избыточным данным (загрузка ненужных связей)

## Способы загрузки связей

### 1. Include (всегда загружает связи)

Используется когда нужно загрузить все записи связи:

```typescript
// Загрузка пользователя с профилем кандидата
const user = await prisma.user.findUnique({
  where: { id: userId },
  include: {
    candidateProfile: true,
  },
});

// user.candidateProfile будет загружен
```

### 2. Select (выборочная загрузка полей)

Используется для загрузки только нужных полей:

```typescript
const user = await prisma.user.findUnique({
  where: { id: userId },
  select: {
    id: true,
    email: true,
    candidateProfile: {
      select: {
        id: true,
        firstName: true,
        lastName: true,
      },
    },
  },
});
```

### 3. Вложенные Include

Для загрузки связей связей:

```typescript
const user = await prisma.user.findUnique({
  where: { id: userId },
  include: {
    candidateProfile: {
      include: {
        resumes: {
          include: {
            hashtags: true,
          },
        },
      },
    },
  },
});
```

## Паттерны использования

### В Repository (Infrastructure слой)

Репозитории должны предоставлять методы для разных сценариев загрузки:

```typescript
// infrastructure/repositories/user.repository.ts
export class UserRepository {
  async findById(id: string): Promise<User | null> {
    return prisma.user.findUnique({
      where: { id },
    });
  }
  
  async findByIdWithProfile(id: string): Promise<User | null> {
    return prisma.user.findUnique({
      where: { id },
      include: {
        candidateProfile: true,
      },
    });
  }
  
  async findByIdWithAllRelations(id: string): Promise<User | null> {
    return prisma.user.findUnique({
      where: { id },
      include: {
        candidateProfile: {
          include: {
            resumes: true,
          },
        },
        tokens: true,
        roleContexts: true,
      },
    });
  }
}
```

### В Service (Application слой)

Сервисы используют репозитории и определяют, какие связи нужны:

```typescript
// application/services/user.service.ts
export class UserService {
  constructor(private userRepository: IUserRepository) {}
  
  // Простой случай - без связей
  async getUser(id: string): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(id);
    return this.toDto(user);
  }
  
  // С профилем
  async getUserWithProfile(id: string): Promise<UserWithProfileDto> {
    const user = await this.userRepository.findByIdWithProfile(id);
    return this.toDtoWithProfile(user);
  }
}
```

## Оптимизация запросов

### Избегание N+1 проблемы

❌ **Плохо** - N+1 запросов:

```typescript
const users = await prisma.user.findMany();
for (const user of users) {
  const profile = await prisma.candidateProfile.findUnique({
    where: { userId: user.id },
  });
}
```

✅ **Хорошо** - один запрос:

```typescript
const users = await prisma.user.findMany({
  include: {
    candidateProfile: true,
  },
});
```

### Условная загрузка связей

Иногда нужно загружать связи только при необходимости:

```typescript
async findById(id: string, includeProfile = false): Promise<User | null> {
  const query: any = { where: { id } };
  
  if (includeProfile) {
    query.include = { candidateProfile: true };
  }
  
  return prisma.user.findUnique(query);
}
```

## Работа с массивами связей

### One-to-Many отношения

```typescript
// User имеет много Tokens
const user = await prisma.user.findUnique({
  where: { id: userId },
  include: {
    tokens: {
      where: {
        isActive: true, // Фильтрация
      },
      orderBy: {
        createdAt: 'desc', // Сортировка
      },
      take: 10, // Лимит
    },
  },
});
```

### Many-to-Many отношения

```typescript
// Resume имеет много Hashtags (через промежуточную таблицу)
const resume = await prisma.resume.findUnique({
  where: { id: resumeId },
  include: {
    hashtags: {
      include: {
        hashtag: true, // Загрузка связанного Hashtag
      },
    },
  },
});
```

## Типизация результатов

### С типами Prisma

Используйте утилитные типы Prisma для типизации:

```typescript
import { Prisma } from '@prisma/client';

type UserWithProfile = Prisma.UserGetPayload<{
  include: {
    candidateProfile: true;
  };
}>;

async findByIdWithProfile(id: string): Promise<UserWithProfile | null> {
  return prisma.user.findUnique({
    where: { id },
    include: {
      candidateProfile: true,
    },
  });
}
```

### Кастомные типы

Для более сложных случаев:

```typescript
type UserWithAllRelations = Prisma.UserGetPayload<{
  include: {
    candidateProfile: {
      include: {
        resumes: {
          include: {
            hashtags: {
              include: {
                hashtag: true;
              };
            };
          };
        };
      };
    };
    tokens: true;
    roleContexts: true;
  };
}>;
```

## Best Practices

1. **Создавайте отдельные методы репозитория** для разных сценариев загрузки
2. **Используйте include только когда нужно** - не загружайте лишние данные
3. **Фильтруйте и сортируйте связи** прямо в запросе
4. **Типизируйте результаты** через Prisma утилитные типы
5. **Избегайте глубокой вложенности** - максимум 2-3 уровня include
6. **Документируйте методы репозитория** - какие связи они загружают

## Примеры

### Полный пример репозитория

```typescript
// infrastructure/repositories/user.repository.ts
import { Prisma } from '@prisma/client';
import { prisma } from '../../../core/prisma/prisma.service';

export type UserWithProfile = Prisma.UserGetPayload<{
  include: {
    candidateProfile: true;
  };
}>;

export class UserRepository {
  async findById(id: string): Promise<User | null> {
    return prisma.user.findUnique({
      where: { id },
    });
  }
  
  async findByIdWithProfile(id: string): Promise<UserWithProfile | null> {
    return prisma.user.findUnique({
      where: { id },
      include: {
        candidateProfile: {
          include: {
            resumes: {
              include: {
                hashtags: {
                  include: {
                    hashtag: true,
                  },
                },
              },
            },
          },
        },
      },
    });
  }
  
  async findManyWithActiveTokens(): Promise<User[]> {
    return prisma.user.findMany({
      include: {
        tokens: {
          where: {
            isActive: true,
          },
        },
      },
    });
  }
}
```

## Ссылки

- [Prisma Schema](./prisma-schema.md) - схема базы данных и связи
- [Entity/DTO маппинг](./entity-dto-mapping.md) - преобразование Entity в DTO
- [Core Prisma](../16-prisma/index.md) - Prisma сервис и модуль
