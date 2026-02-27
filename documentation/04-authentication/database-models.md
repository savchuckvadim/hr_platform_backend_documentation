# Database Models - Модели базы данных

## Обзор

Модели базы данных для модуля аутентификации описывают структуру хранения пользователей, ролей, компаний и токенов.

## Модели

### User

Базовая сущность пользователя. Содержит только данные для аутентификации.

**Таблица:** `users`

**Поля:**
- `id` - UUID, primary key
- `email` - varchar(255), unique, not null - уникальный email для входа
- `password` - varchar, not null - хэшированный пароль (bcrypt/argon2)
- `isActivated` - boolean, default: false - активирован ли email
- `activationLink` - varchar(255), unique, nullable - ссылка для активации email
- `createdAt` - timestamp, default: now()
- `updatedAt` - timestamp, auto-update

**Индексы:**
- `email` - для быстрого поиска по email

**Принципы:**
- Один email может существовать только один раз в системе
- Email используется для входа (login)
- Пароль всегда хранится в хэшированном виде
- `isActivated` проверяется при login

**Связи:**
- One-to-many с `role_contexts` (один User может иметь несколько ролей)
- One-to-many с `tokens` (один User может иметь несколько активных сессий)
- One-to-one с `candidate_profiles` (опционально, если есть роль CANDIDATE)
- One-to-many с `resumes` (резюме кандидата - ссылается на User.id, не на CandidateProfile.id)
- One-to-many с `companies` (ownedCompanies - компании, где user = owner)

### UserRole (Enum)

Системные роли для аутентификации пользователей. Используется enum для типобезопасности.

**Enum:** `UserRole`

**Значения:**
- `CANDIDATE` - соискатель
- `EMPLOYER` - работодатель (владелец компании или HR сотрудник)
- `ADMIN` - администратор системы

**Принципы:**
- Используется enum для системных ролей
- Три предустановленные роли: CANDIDATE, EMPLOYER, ADMIN
- Enum обеспечивает типобезопасность на уровне БД и приложения
- Роли используются напрямую в RoleContext без промежуточной таблицы

### HrRole

Роли HR сотрудников внутри компании. Расширяемая таблица для добавления новых HR ролей.

**Таблица:** `hr_roles`

**Поля:**
- `id` - UUID, primary key
- `name` - varchar(50), unique, not null - название роли: `HR | HR_ADMIN`
- `slug` - varchar(50), unique, not null - slug роли
- `description` - text, nullable - описание роли
- `permissions` - json, nullable - JSON с правами доступа (для будущего расширения)
- `isActive` - boolean, default: true - активна ли роль
- `createdAt` - timestamp, default: now()
- `updatedAt` - timestamp, auto-update

**Индексы:**
- `name` - для поиска по названию
- `slug` - для поиска по slug
- `isActive` - для фильтрации активных ролей

**Стандартные роли:**
- `HR` - обычный HR сотрудник
- `HR_ADMIN` - HR администратор, может добавлять других HR в компанию

**Принципы:**
- Расширяемая таблица для HR ролей
- Позволяет добавлять новые HR роли без миграции enum
- `permissions` JSON для будущего расширения прав доступа
- Связана с `role_contexts` через `hr_role_id` (только для EMPLOYER ролей)

### RoleContext

Определяет роль пользователя в системе. Один User может иметь несколько RoleContext для разных ролей.

**Таблица:** `role_contexts`

**Поля:**
- `id` - UUID, primary key
- `userId` - varchar, not null, foreign key → users.id
- `userRole` - enum UserRole, not null - системная роль (CANDIDATE | EMPLOYER | ADMIN)
- `companyId` - varchar, nullable, foreign key → companies.id - nullable в БД, но **обязательно для EMPLOYER** (валидация на уровне приложения)
- `hrRoleId` - varchar, nullable, foreign key → hr_roles.id - роль HR внутри компании (HR | HR_ADMIN) - только для EMPLOYER
- `createdAt` - timestamp, default: now()

**Индексы:**
- `userId` - для поиска всех ролей пользователя
- `userRole` - для фильтрации по системной роли
- `companyId` - для поиска EMPLOYER сотрудников компании
- `hrRoleId` - для фильтрации по HR роли

**Типы ролей:**

1. **CANDIDATE** (userRole = 'CANDIDATE')
   - Соискатель
   - Создаётся при регистрации как кандидат
   - Связан с `candidate_profiles`
   - `hrRoleId` = null

2. **EMPLOYER** (userRole = 'EMPLOYER')
   - Работодатель (владелец компании или HR сотрудник)
   - Создаётся при регистрации в существующую компанию или при добавлении HR-ADMIN
   - Связан с `companies` через `companyId` (обязательно для EMPLOYER, валидация на уровне приложения)
   - Может быть несколько EMPLOYER в одной компании
   - Типы работодателя (hrRoleId → hr_roles):
     - **HR** - обычный HR сотрудник
     - **HR_ADMIN** - HR администратор, может добавлять других HR в компанию
   - Владелец компании регистрируется как EMPLOYER с `hrRoleId: HR_ADMIN`
   - **Важно:** При удалении Company все связанные RoleContext с EMPLOYER ролью удаляются (Cascade) - это предотвращает "висячие" роли

3. **ADMIN** (userRole = 'ADMIN')
   - Администратор системы
   - Создаётся вручную
   - Полный доступ ко всем ресурсам
   - `companyId` = null, `hrRoleId` = null

**Принципы:**
- Один User может иметь несколько RoleContext
- В рамках одной сессии активен только один RoleContext
- `userRole` - системная роль (enum UserRole)
- `companyId` - nullable в БД, но обязательно для EMPLOYER (валидация на уровне приложения)
- `hrRoleId` - роль HR внутри компании (FK к `hr_roles`) - только для EMPLOYER
- RoleContext определяет доступ к ресурсам через Guards
- При удалении User все RoleContext удаляются (cascade)
- При удалении Company все связанные RoleContext удаляются (cascade) - предотвращает "висячие" EMPLOYER роли
- При удалении HrRole RoleContext остается, но `hrRoleId` = null (set null)

**Связи:**
- Many-to-one с `users` (один User - много RoleContext)
- `userRole` - enum UserRole (системная роль)
- Many-to-one с `hr_roles` (HR роль - только для EMPLOYER)
- Many-to-one с `companies` (для EMPLOYER роли)
- One-to-many с `tokens` (один RoleContext - много активных сессий)

### Company

Компания работодателя. Создаётся при регистрации работодателя.

**Таблица:** `companies`

**Поля:**
- `id` - UUID, primary key
- `name` - varchar(255), not null - название компании
- `inn` - varchar(20), nullable - ИНН (налоговый идентификатор)
- `companyTypeId` - varchar, not null, foreign key → company_types.id - тип компании
- `ownerId` - varchar, not null, foreign key → users.id - владелец компании (EMPLOYER с HR_ADMIN)
- `createdAt` - timestamp, default: now()
- `updatedAt` - timestamp, auto-update

**Индексы:**
- `ownerId` - для поиска компаний владельца
- `inn` - для поиска по ИНН (если указан)
- `companyTypeId` - для фильтрации по типу компании

**Связь с CompanyType:**
- Типы компаний хранятся в отдельной таблице `company_types`
- Позволяет добавлять новые типы без миграции БД
- Типы: ORGANIZATION, IP, LAWYER, SELF_EMPLOYED, OTHER

**Принципы:**
- При создании компании владелец регистрируется как EMPLOYER с `hrRoleId: HR_ADMIN` (FK к таблице hr_roles, используется enum `HrRoleName.HR_ADMIN`)
- Владелец компании = EMPLOYER с правами HR_ADMIN
- Компания может иметь несколько EMPLOYER сотрудников
- HR_ADMIN может добавлять других HR в компанию
- При удалении владельца компания удаляется (cascade) или передаётся другому HR_ADMIN
- Связана с CompanyType через `company_type_id` (типы в отдельной таблице)

**Связи:**
- Many-to-one с `users` (ownerId - владелец компании)
- Many-to-one с `company_types` (companyTypeId - тип компании)
- One-to-many с `role_contexts` (EMPLOYER сотрудники компании)
- One-to-many с `projects` (проекты компании)

### Token

Хранилище refresh токенов. Каждая запись = одна сессия на одном устройстве с определённой ролью.

**Таблица:** `tokens`

**Поля:**
- `id` - UUID, primary key
- `userId` - varchar, not null, foreign key → users.id
- `roleContextId` - varchar, not null, foreign key → role_contexts.id - активный role context
- `refreshToken` - varchar(500), unique, not null - хэшированный refresh token
- `deviceId` - varchar(255), not null - уникальный идентификатор устройства
- `deviceName` - varchar(255), nullable - название устройства (опционально)
- `userAgent` - text, nullable - user agent браузера
- `ipAddress` - varchar(45), nullable - IP адрес
- `expiresAt` - timestamp, not null - срок истечения токена
- `createdAt` - timestamp, default: now()
- `updatedAt` - timestamp, auto-update

**Индексы:**
- `userId` - для поиска всех токенов пользователя
- `roleContextId` - для поиска токенов по роли
- `refreshToken` - для быстрого поиска токена (hash lookup)
- `deviceId` - для поиска токенов устройства
- `expiresAt` - для cron очистки истёкших токенов
- `(userId, roleContextId, deviceId)` - unique constraint

**Unique Constraint:**
```
(userId, roleContextId, deviceId) - уникально
```

Это означает:
- Один device может иметь только один активный refresh token
- При смене роли на устройстве - старый токен удаляется, создаётся новый
- Один пользователь может иметь несколько токенов на разных устройствах

**Принципы:**
- Refresh token хранится в хэшированном виде (bcrypt/argon2)
- При сравнении используется hash comparison
- Один device = один refresh token = один активный role context
- Logout одного устройства удаляет одну запись
- Logout всех устройств удаляет все записи пользователя
- Автоматическая очистка истёкших токенов через cron

**Связи:**
- Many-to-one с `users` (один User - много токенов)
- Many-to-one с `role_contexts` (один RoleContext - много токенов)

**Пример данных:**
```
id    userId  roleContextId  deviceId  refreshToken  expiresAt
1     U1      RC1 (CANDIDATE)  PHONE    hash(...)    2024-01-15
2     U1      RC2 (EMPLOYER)   PC      hash(...)    2024-01-15
3     U1      RC1 (CANDIDATE)  PC      hash(...)    2024-01-14
```

В этом примере:
- Пользователь U1 имеет 2 роли (CANDIDATE и EMPLOYER)
- На телефоне активна роль CANDIDATE
- На компьютере активна роль EMPLOYER
- Ранее на компьютере была роль CANDIDATE, но после смены роли старый токен остался (или был удалён при logout)

## Схема связей

```
User (1) ──< (N) RoleContext
User (1) ──< (N) Token
User (1) ──< (1) CandidateProfile (optional)
User (1) ──< (N) Resume (candidateId ссылается на User.id)
User (1) ──< (N) Company (ownedCompanies - где user = owner)

Company (1) ──< (N) RoleContext (EMPLOYER roles)
Company (1) ──< (1) User (ownerId)
Company (1) ──< (1) CompanyType (companyTypeId)
Company (1) ──< (N) Project (проекты компании)

RoleContext (1) ──< (N) Token
```

## Миграции

### Создание таблиц

```sql
-- Users
CREATE TABLE users (
  id VARCHAR PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR NOT NULL,
  is_activated BOOLEAN DEFAULT FALSE,
  activation_link VARCHAR(255) UNIQUE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- Role Contexts
CREATE TABLE role_contexts (
  id VARCHAR PRIMARY KEY,
  user_id VARCHAR NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type VARCHAR NOT NULL, -- CANDIDATE | EMPLOYER | ADMIN (глобальный уровень)
  company_id VARCHAR REFERENCES companies(id) ON DELETE CASCADE, -- Cascade для предотвращения "висячих" EMPLOYER ролей
  employer_role_type VARCHAR, -- HR | HR_ADMIN (только для EMPLOYER type, масштабируемо)
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_role_contexts_user_id ON role_contexts(user_id);
CREATE INDEX idx_role_contexts_type ON role_contexts(type);
CREATE INDEX idx_role_contexts_company_id ON role_contexts(company_id);
CREATE INDEX idx_role_contexts_employer_role_type ON role_contexts(employer_role_type);

-- Company Types
CREATE TABLE company_types (
  id VARCHAR PRIMARY KEY,
  name VARCHAR(100) UNIQUE NOT NULL, -- ORGANIZATION | IP | LAWYER | SELF_EMPLOYED | OTHER
  slug VARCHAR(100) UNIQUE NOT NULL,
  description TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_company_types_name ON company_types(name);
CREATE INDEX idx_company_types_slug ON company_types(slug);
CREATE INDEX idx_company_types_is_active ON company_types(is_active);

-- Companies
CREATE TABLE companies (
  id VARCHAR PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  inn VARCHAR(20),
  company_type_id VARCHAR NOT NULL REFERENCES company_types(id) ON DELETE RESTRICT,
  owner_id VARCHAR NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_companies_owner_id ON companies(owner_id);
CREATE INDEX idx_companies_inn ON companies(inn);
CREATE INDEX idx_companies_company_type_id ON companies(company_type_id);

-- Tokens
CREATE TABLE tokens (
  id VARCHAR PRIMARY KEY,
  user_id VARCHAR NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role_context_id VARCHAR NOT NULL REFERENCES role_contexts(id) ON DELETE CASCADE,
  refresh_token VARCHAR(500) UNIQUE NOT NULL,
  device_id VARCHAR(255) NOT NULL,
  device_name VARCHAR(255),
  user_agent TEXT,
  ip_address VARCHAR(45),
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_tokens_user_id ON tokens(user_id);
CREATE INDEX idx_tokens_role_context_id ON tokens(role_context_id);
CREATE INDEX idx_tokens_refresh_token ON tokens(refresh_token);
CREATE INDEX idx_tokens_device_id ON tokens(device_id);
CREATE INDEX idx_tokens_expires_at ON tokens(expires_at);

CREATE UNIQUE INDEX idx_tokens_user_role_device ON tokens(user_id, role_context_id, device_id);
```

## Примечания

1. **Cascade Delete:**
   - При удалении User удаляются все RoleContext и Token
   - При удалении RoleContext удаляются все связанные Token
   - При удалении Company все связанные RoleContext с EMPLOYER ролью удаляются (Cascade) - это предотвращает "висячие" роли работодателей

2. **Unique Constraints:**
   - `users.email` - уникальный email
   - `tokens.refresh_token` - уникальный refresh token
   - `tokens(userId, roleContextId, deviceId)` - один токен на device+role

3. **Индексы:**
   - Все foreign keys индексированы для быстрого поиска
   - `tokens.expires_at` индексирован для cron очистки
   - `tokens.refresh_token` индексирован для быстрого lookup

4. **Безопасность:**
   - Пароли хранятся в хэшированном виде
   - Refresh токены хранятся в хэшированном виде
   - Никогда не хранить plain text токены
