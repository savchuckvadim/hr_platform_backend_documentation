# Зависимости между слоями

## Общее описание

В луковой архитектуре важно соблюдать правила зависимостей между слоями. Это обеспечивает изоляцию бизнес-логики от инфраструктуры и делает код более тестируемым и поддерживаемым.

## Правила зависимостей

### Основной принцип

**Зависимости направлены внутрь** - от внешних слоев к внутренним:

```
API → Application → Domain ← Infrastructure
```

### Детальные правила

1. **Domain слой** - самый внутренний, не зависит ни от чего
   - Содержит только бизнес-сущности и интерфейсы
   - Не импортирует ничего из других слоев
   - Может содержать enum'ы, типы, интерфейсы репозиториев

2. **Application слой** - зависит только от Domain
   - Импортирует сущности из `domain/`
   - Импортирует интерфейсы репозиториев из `domain/`
   - НЕ импортирует ничего из `infrastructure/` или `api/`
   - Содержит бизнес-логику и use cases

3. **Infrastructure слой** - зависит от Domain и Application
   - Реализует интерфейсы репозиториев из `domain/`
   - Может использовать сервисы из `application/` (через DI)
   - Содержит конкретные реализации (Prisma, внешние API)

4. **API слой** - зависит от всех остальных
   - Импортирует DTO и контроллеры
   - Использует сервисы из `application/`
   - Использует репозитории через DI (интерфейсы из `domain/`)

## Примеры правильных зависимостей

### ✅ Правильно

```typescript
// domain/entity/user.entity.ts
export class User {
  id: string;
  email: string;
}

// domain/repository/user.repository.interface.ts
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
}

// application/services/user.service.ts
import { User } from '../../domain/entity/user.entity';
import { IUserRepository } from '../../domain/repository/user.repository.interface';

export class UserService {
  constructor(private userRepository: IUserRepository) {}
  
  async getUser(id: string): Promise<User> {
    return this.userRepository.findById(id);
  }
}

// infrastructure/repositories/user.repository.ts
import { User } from '../../domain/entity/user.entity';
import { IUserRepository } from '../../domain/repository/user.repository.interface';

export class UserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    // Prisma реализация
  }
}
```

### ❌ Неправильно

```typescript
// ❌ Application слой не должен импортировать из Infrastructure
import { UserRepository } from '../../infrastructure/repositories/user.repository';

// ❌ Domain слой не должен импортировать из других слоев
import { UserService } from '../../application/services/user.service';

// ❌ Infrastructure не должен напрямую использовать Application сервисы
// (только через DI в NestJS модуле)
```

## Dependency Injection в NestJS

В NestJS зависимости между слоями настраиваются через модули:

```typescript
// user.module.ts
@Module({
  controllers: [UserController], // API слой
  providers: [
    UserService, // Application слой
    {
      provide: 'IUserRepository', // Интерфейс из Domain
      useClass: UserRepository, // Реализация из Infrastructure
    },
  ],
})
export class UserModule {}
```

## Исключения

### События

- **Локальные события** (`application/events/`) - используются только внутри модуля
- **Глобальные события** (`core/events/`) - могут использоваться из любого слоя, но предпочтительно из `application/`

### Очереди

- Processors для очередей находятся в `infrastructure/processors/`
- Они могут использовать сервисы из `application/` через DI

## Ключевые концепции

- **Инверсия зависимостей** - слои зависят от абстракций (интерфейсов), а не от конкретных реализаций
- **Изоляция** - бизнес-логика не знает о деталях реализации (Prisma, HTTP, WebSocket)
- **Тестируемость** - легко мокировать зависимости через интерфейсы
- **Гибкость** - можно менять реализации без изменения бизнес-логики

## Ссылки

- [Архитектура слоев](./layers-architecture.md) - детальное описание структуры модуля
- [Тестирование модулей](./testing.md) - как тестировать модули с правильными зависимостями
