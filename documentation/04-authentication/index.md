# Authentication Module - Модуль аутентификации

## Описание

Модуль аутентификации обеспечивает безопасную систему входа, регистрации и управления сессиями для HR Platform. Реализует multi-device поддержку с привязкой refresh токенов к устройствам и role context, позволяя одному пользователю иметь несколько активных сессий в разных ролях на разных устройствах.

## Задачи

### ✅ Концепция и архитектура
**Статус**: Завершено
**Файл**: [concept.md](./concept.md)
**Описание**: Основные принципы и архитектура аутентификации

### ✅ Database Models
**Статус**: Завершено
**Файл**: [database-models.md](./database-models.md)
**Описание**: Модели БД для аутентификации (User, RoleContext, Company, Token)

### ✅ Passport Strategies
**Статус**: Завершено
**Файл**: [passport-strategies.md](./passport-strategies.md)
**Описание**: JWT и Refresh стратегии Passport

### ✅ Guards и Decorators
**Статус**: Завершено
**Файл**: [guards-decorators.md](./guards-decorators.md)
**Описание**: Guards, decorators, enums для защиты маршрутов

### ✅ Registration Flow
**Статус**: Завершено
**Файл**: [registration-flow.md](./registration-flow.md)
**Описание**: Потоки регистрации кандидата и работодателя

### ✅ Login и Refresh Flow
**Статус**: Завершено
**Файл**: [login-refresh-flow.md](./login-refresh-flow.md)
**Описание**: Потоки входа, обновления токенов, logout

### ✅ Password Recovery Flow
**Статус**: Завершено
**Файл**: [password-recovery-flow.md](./password-recovery-flow.md)
**Описание**: Поток восстановления пароля через email

### ✅ Token Management
**Статус**: Завершено
**Файл**: [token-management.md](./token-management.md)
**Описание**: Управление токенами, cron очистка, sliding session

### ✅ HR Roles Guard
**Статус**: Завершено
**Файл**: [hr-roles-guard.md](./hr-roles-guard.md)
**Описание**: Проверка прав работодателей (HR vs HR_ADMIN)

### 📋 Future Extensions
**Статус**: Планируется
**Файл**: [future-extensions.md](./future-extensions.md)
**Описание**: Планируемые расширения (OAuth, Phone OTP, 2FA, WebAuthn)

## Ключевые концепции

- **Multi-device**: Один пользователь может иметь несколько активных сессий на разных устройствах
- **Multi-role**: Один email может быть зарегистрирован как кандидат и как работодатель
- **Role Context**: Каждая сессия привязана к конкретной роли (CANDIDATE, EMPLOYER, ADMIN)
- **Device-bound tokens**: Refresh token привязан к (userId + roleContextId + deviceId)
- **Sliding session**: Access token продлевается при любой активности
- **Stateful refresh**: Refresh токены хранятся в БД в хэшированном виде
- **Stateless access**: Access токены - JWT без хранения в БД

## Структура модуля

```
auth/
├── api/
│   ├── controllers/
│   │   └── auth.controller.ts
│   └── dto/
│       ├── register-candidate.dto.ts
│       ├── register-employer.dto.ts
│       ├── login.dto.ts
│       ├── refresh.dto.ts
│       ├── forgot-password.dto.ts
│       └── reset-password.dto.ts
├── application/
│   └── services/
│       └── auth.service.ts
├── domain/
│   ├── entities/
│   │   ├── role-context.entity.ts
│   │   └── token.entity.ts
│   └── interfaces/
│       └── jwt-payload.interface.ts
├── infrastructure/
│   ├── strategies/
│   │   ├── jwt.strategy.ts
│   │   └── refresh.strategy.ts
│   ├── guards/
│   │   ├── jwt-auth.guard.ts
│   │   ├── refresh-auth.guard.ts
│   │   └── roles.guard.ts
│   ├── decorators/
│   │   ├── current-user.decorator.ts
│   │   ├── current-role.decorator.ts
│   │   └── roles.decorator.ts
│   ├── interceptors/
│   │   └── token-refresh.interceptor.ts
│   ├── repositories/
│   │   └── password-reset-token.repository.ts
│   └── cron/
│       └── password-reset-cleanup.cron.ts
├── enums/
│   └── auth-status.enum.ts
├── __tests__/
├── auth.module.ts
└── index.ts
```

## Ссылки

- [Концепция аутентификации](./concept.md) - основные принципы и архитектура
- [Модели БД](./database-models.md) - структура базы данных
- [Passport стратегии](./passport-strategies.md) - JWT и Refresh стратегии
- [Guards и Decorators](./guards-decorators.md) - защита маршрутов
- [Employer Roles Guard](./hr-roles-guard.md) - проверка прав работодателей (HR vs HR_ADMIN)
- [Потоки регистрации](./registration-flow.md) - регистрация кандидата, работодателя и HR
- [Потоки входа и обновления](./login-refresh-flow.md) - login, refresh, logout
- [Восстановление пароля](./password-recovery-flow.md) - forgot-password, reset-password
- [Управление токенами](./token-management.md) - cron, sliding session
- [Планируемые расширения](./future-extensions.md) - OAuth, Phone OTP, 2FA, WebAuthn
