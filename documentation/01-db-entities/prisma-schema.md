# Prisma Schema - Схема базы данных HR Platform

## Обзор

Схема базы данных для HR Platform построена на принципах:
- Единая система аутентификации для кандидатов и работодателей
- Раздельные профили для каждого типа пользователей
- Гибкая система навыков на основе хэштегов
- Полный цикл взаимодействия: вакансии → отклики → общение
- Поддержка файлов, чатов

## Модели Prisma

### 1. User - Пользователь

Единая сущность для аутентификации. Содержит базовые данные и роль (кандидат или работодатель).

```prisma
model User {
  id             String    @id @default(uuid())
  email          String    @unique @db.VarChar(255)
  password       String
  // DEPRECATED: role больше не используется - роли через RoleContext
  isActivated    Boolean   @default(false)
  activationLink String?   @unique @db.VarChar(255)
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt
  createdBy      String?   @map("created_by")
  updatedBy      String?   @map("updated_by")

  // Relations
  tokens            Token[]
  roleContexts      RoleContext[]
  candidateProfile  CandidateProfile?
  // DEPRECATED: employerProfile больше не используется - информация в Company
  chats             ChatMember[]
  messages          Message[]
  files             File[]
  sentReplies       Reply[]            @relation("ReplyCandidate")
  ownedCompanies    Company[]          // Компании, где user = owner (EMPLOYER с HR_ADMIN)
  // Self-referencing for created_by/updated_by
  createdUsers      User[]             @relation("UserCreatedBy")
  updatedUsers      User[]             @relation("UserUpdatedBy")

  createdByUser     User?              @relation("UserCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser     User?              @relation("UserUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("users")
  @@index([email])
  @@index([createdBy])
  @@index([updatedBy])
}

// DEPRECATED: UserRole больше не используется
// Роли теперь определяются через RoleContext
// enum UserRole {
//   CANDIDATE
//   EMPLOYER  // Работодатель (владелец компании или HR сотрудник)
//   ADMIN
// }
```

**Принципы:**
- Email уникален и используется для входа
- `isActivated` - подтверждение email
- Роли определяются через RoleContext (multi-role поддержка)
- Один User может иметь несколько RoleContext (CANDIDATE, EMPLOYER, ADMIN)

### 2. Token - Токены аутентификации

Поддержка multi-device токенов. Каждое устройство имеет свой refresh token.

```prisma
model Token {
  id           String   @id @default(uuid())
  userId       String   @map("user_id")
  roleContextId String  @map("role_context_id") // Обязательно - токен привязан к роли
  refreshToken String   @unique @map("refresh_token") @db.VarChar(500)
  deviceId     String   @map("device_id") @db.VarChar(255) // NOT NULL - обязателен
  deviceName   String?  @map("device_name") @db.VarChar(255)
  userAgent    String?  @map("user_agent") @db.Text
  ipAddress    String?  @map("ip_address") @db.VarChar(45)
  expiresAt    DateTime @map("expires_at")
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  user        User        @relation(fields: [userId], references: [id], onDelete: Cascade)
  roleContext RoleContext @relation(fields: [roleContextId], references: [id], onDelete: Cascade)

  @@unique([userId, roleContextId, deviceId]) // Один токен на user+role+device
  @@map("tokens")
  @@index([userId])
  @@index([roleContextId])
  @@index([refreshToken])
  @@index([deviceId])
  @@index([expiresAt])
}
```

**Принципы:**
- Один refresh token = одно устройство + одна роль
- Токен привязан к RoleContext (userId + roleContextId + deviceId)
- `deviceId` обязателен (NOT NULL)
- `expiresAt` для автоматической очистки истекших токенов
- Cascade delete при удалении пользователя или RoleContext

### 3. CandidateProfile - Профиль кандидата

Опциональный профиль для кандидатов. Содержит основную информацию и контакты.

```prisma
model CandidateProfile {
  id          String    @id @default(uuid())
  userId      String    @unique @map("user_id")
  firstName   String?   @map("first_name") @db.VarChar(100)
  lastName    String?   @map("last_name") @db.VarChar(100)
  middleName  String?   @map("middle_name") @db.VarChar(100)
  birthday    DateTime?
  phone       String?   @db.VarChar(20)
  contactEmail String?  @map("contact_email") @db.VarChar(255)
  contactPhone String?  @map("contact_phone") @db.VarChar(20)
  avatar      String?   @db.VarChar(500)
  bio         String?   @db.Text
  location    String?   @db.VarChar(255)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String?   @map("created_by")
  updatedBy   String?   @map("updated_by")

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  resumes      Resume[]
  createdByUser User?   @relation("CandidateProfileCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?   @relation("CandidateProfileUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("candidate_profiles")
  @@index([userId])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Связь 1:1 с User (только для role = CANDIDATE)
- Основные контакты в профиле (`phone`, `contactEmail`, `contactPhone`)
- В резюме могут быть переопределены контакты
- `birthday` только в профиле, не в резюме

### 4. UserRole - Системные роли

Системные роли для аутентификации пользователей. Расширяемая таблица для добавления новых ролей.

```prisma
model UserRole {
  id          String    @id @default(uuid())
  name        String    @unique @db.VarChar(50) // CANDIDATE | EMPLOYER | ADMIN
  slug        String    @unique @db.VarChar(50)
  description String?   @db.Text
  isActive    Boolean   @default(true) @map("is_active")
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String?   @map("created_by")
  updatedBy   String?   @map("updated_by")

  roleContexts RoleContext[]
  createdByUser User?   @relation("UserRoleCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?   @relation("UserRoleUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("user_roles")
  @@index([name])
  @@index([slug])
  @@index([isActive])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Расширяемая таблица для системных ролей
- Позволяет добавлять новые роли без миграции enum
- Предустановленные роли: CANDIDATE, EMPLOYER, ADMIN (обязательно создаются через seed)
- Для предустановленных ролей используется enum `UserRoleName` для типобезопасности
- Новые роли можно добавлять в БД, но для них enum не требуется

### 5. HrRole - Роли HR внутри компании

Роли HR сотрудников внутри компании. Расширяемая таблица для добавления новых HR ролей.

```prisma
model HrRole {
  id          String    @id @default(uuid())
  name        String    @unique @db.VarChar(50) // HR | HR_ADMIN
  slug        String    @unique @db.VarChar(50)
  description String?   @db.Text
  permissions Json?      // JSON с правами доступа (для будущего расширения)
  isActive    Boolean   @default(true) @map("is_active")
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String?   @map("created_by")
  updatedBy   String?   @map("updated_by")

  roleContexts RoleContext[]
  createdByUser User?   @relation("HrRoleCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?   @relation("HrRoleUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("hr_roles")
  @@index([name])
  @@index([slug])
  @@index([isActive])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Расширяемая таблица для HR ролей
- Позволяет добавлять новые HR роли без миграции enum
- Предустановленные роли: HR, HR_ADMIN (обязательно создаются через seed)
- Для предустановленных ролей используется enum `HrRoleName` для типобезопасности
- Новые HR роли можно добавлять в БД, но для них enum не требуется
- `permissions` JSON для будущего расширения прав доступа

### 6. RoleContext - Контекст роли пользователя

Определяет роль пользователя в системе. Один User может иметь несколько RoleContext.

```prisma
model RoleContext {
  id          String    @id @default(uuid())
  userId      String    @map("user_id")
  userRoleId  String    @map("user_role_id") // FK к user_roles (CANDIDATE | EMPLOYER | ADMIN)
  companyId   String?   @map("company_id") // Обязательно для EMPLOYER
  hrRoleId    String?    @map("hr_role_id") // FK к hr_roles (HR | HR_ADMIN) - только для EMPLOYER
  createdAt   DateTime  @default(now())
  createdBy   String?   @map("created_by")

  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  userRole    UserRole @relation(fields: [userRoleId], references: [id], onDelete: Restrict)
  company     Company? @relation(fields: [companyId], references: [id], onDelete: SetNull)
  hrRole      HrRole?  @relation(fields: [hrRoleId], references: [id], onDelete: SetNull)
  tokens      Token[]
  createdByUser User?  @relation("RoleContextCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)

  @@map("role_contexts")
  @@index([userId])
  @@index([userRoleId])
  @@index([companyId])
  @@index([hrRoleId])
  @@index([createdBy])
}
```

**Принципы:**
- Один User может иметь несколько RoleContext (multi-role)
- В рамках одной сессии активен только один RoleContext
- `userRoleId` - системная роль (CANDIDATE, EMPLOYER, ADMIN)
- `hrRoleId` - роль HR внутри компании (HR, HR_ADMIN) - только для EMPLOYER
- EMPLOYER роли всегда привязаны к Company через companyId
- Владелец компании = EMPLOYER с hrRoleId: HR_ADMIN

### 7. CompanyType - Тип компании

Типы компаний в отдельной таблице для масштабируемости.

```prisma
model CompanyType {
  id          String    @id @default(uuid())
  name        String    @unique @db.VarChar(100) // ORGANIZATION | IP | LAWYER | SELF_EMPLOYED | OTHER
  slug        String    @unique @db.VarChar(100)
  description String?   @db.Text
  isActive    Boolean   @default(true) @map("is_active")
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String?   @map("created_by")
  updatedBy   String?   @map("updated_by")

  companies   Company[]
  createdByUser User?   @relation("CompanyTypeCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?   @relation("CompanyTypeUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("company_types")
  @@index([name])
  @@index([slug])
  @@index([createdBy])
  @@index([updatedBy])
}
```

### 8. Company - Компания

Компания работодателя. Создаётся при регистрации владельца (EMPLOYER с HR_ADMIN).

```prisma
model Company {
  id            String        @id @default(uuid())
  name          String        @db.VarChar(255)
  inn           String?       @db.VarChar(20)
  companyTypeId String        @map("company_type_id")
  ownerId       String        @map("owner_id") // User (HR с HR_ADMIN)
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt
  createdBy     String?       @map("created_by")
  updatedBy     String?       @map("updated_by")

  companyType   CompanyType   @relation(fields: [companyTypeId], references: [id], onDelete: Restrict)
  owner         User          @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  roleContexts  RoleContext[] // HR сотрудники компании
  projects      Project[]
  experiences   Experience[]  // Опыт работы в этой компании
  createdByUser User?         @relation("CompanyCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?         @relation("CompanyUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("companies")
  @@index([ownerId])
  @@index([inn])
  @@index([companyTypeId])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- При создании компании владелец регистрируется как EMPLOYER с employerRoleType: HR_ADMIN
- Связана с CompanyType через companyTypeId
- Может иметь несколько HR сотрудников через RoleContext

### 9. Project - Проект компании

Проект компании для группировки вакансий.

```prisma
model Project {
  id          String    @id @default(uuid())
  companyId   String    @map("company_id")
  name        String    @db.VarChar(255)
  description String?   @db.Text
  isActive    Boolean   @default(true) @map("is_active")
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String?   @map("created_by")
  updatedBy   String?   @map("updated_by")

  company     Company   @relation(fields: [companyId], references: [id], onDelete: Cascade)
  vacancies   Vacancy[]
  createdByUser User?   @relation("ProjectCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?   @relation("ProjectUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("projects")
  @@index([companyId])
  @@index([isActive])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Проект принадлежит компании
- Вакансии привязаны к проектам, а не напрямую к компаниям
- Пример: ООО Пирожок → Шаурмечная (проект) → Программист (вакансия)

### DEPRECATED: EmployerProfile

**⚠️ DEPRECATED - больше не используется**

Вся информация о компании теперь хранится в таблице `companies`.
Владелец компании = User с RoleContext (EMPLOYER, HR_ADMIN).

Оставлено для обратной совместимости, но не должно использоваться в новой архитектуре.

### 10. Resume - Резюме кандидата

Резюме кандидата. Может быть несколько резюме у одного кандидата.

```prisma
model Resume {
  id           String    @id @default(uuid())
  candidateId  String    @map("candidate_id")
  title        String    @db.VarChar(255)
  position     String?   @db.VarChar(255)
  salaryMin    Int?      @map("salary_min")
  salaryMax    Int?      @map("salary_max")
  currency     String?   @default("RUB") @db.VarChar(3)
  contactEmail String?   @map("contact_email") @db.VarChar(255)
  contactPhone String?   @map("contact_phone") @db.VarChar(20)
  summary      String?   @db.Text
  isActive     Boolean   @default(true) @map("is_active")
  isPublic     Boolean   @default(false) @map("is_public")
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
  createdBy    String?   @map("created_by")
  updatedBy    String?   @map("updated_by")

  candidate      CandidateProfile @relation(fields: [candidateId], references: [id], onDelete: Cascade)
  skills         ResumeSkill[]
  experiences    Experience[]
  educations     Education[]
  replies        Reply[]
  resumeVersions ResumeVersion[]
  createdByUser  User?           @relation("ResumeCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser   User?           @relation("ResumeUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("resumes")
  @@index([candidateId])
  @@index([isActive])
  @@index([isPublic])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Один кандидат может иметь несколько резюме
- `contactEmail` и `contactPhone` опционально переопределяют контакты из профиля
- `isActive` - активно ли резюме
- `isPublic` - публично ли резюме для просмотра работодателями
- Версионирование заложено через `ResumeVersion` (на будущее)

### 11. ResumeVersion - Версия резюме (на будущее)

Версионирование резюме для отслеживания изменений. На MVP не реализуется, но структура заложена.

```prisma
model ResumeVersion {
  id        String   @id @default(uuid())
  resumeId  String   @map("resume_id")
  version   Int      @default(1)
  data      Json     // JSON snapshot резюме на момент версии
  createdAt DateTime @default(now())
  createdBy String?  @map("created_by")

  resume        Resume @relation(fields: [resumeId], references: [id], onDelete: Cascade)
  createdByUser User?  @relation("ResumeVersionCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)

  @@unique([resumeId, version])
  @@map("resume_versions")
  @@index([resumeId])
  @@index([createdBy])
}
```

**Принципы:**
- На MVP не используется, но структура заложена
- `data` содержит JSON snapshot резюме
- Можно использовать для отслеживания изменений и в Application

### 12. Experience - Опыт работы

Опыт работы в резюме. Отдельная таблица для фильтрации и индексации.

```prisma
model Experience {
  id          String    @id @default(uuid())
  resumeId    String    @map("resume_id")
  company     String    @db.VarChar(255)
  companyId   String?   @map("company_id") // FK to companies - связь с компанией в системе (опционально)
  position    String    @db.VarChar(255)
  description String?   @db.Text
  startDate   DateTime  @map("start_date")
  endDate     DateTime? @map("end_date")
  isCurrent   Boolean   @default(false) @map("is_current")
  location    String?   @db.VarChar(255)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String?   @map("created_by")
  updatedBy   String?   @map("updated_by")

  resume  Resume   @relation(fields: [resumeId], references: [id], onDelete: Cascade)
  companyRelation Company? @relation(fields: [companyId], references: [id], onDelete: SetNull)
  createdByUser   User?    @relation("ExperienceCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser   User?    @relation("ExperienceUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("experiences")
  @@index([resumeId])
  @@index([company])
  @@index([companyId])
  @@index([position])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Отдельная таблица для удобной фильтрации и поиска
- `isCurrent` - текущее место работы
- Индексы для быстрого поиска по компании и должности

### 13. Education - Образование

Образование в резюме. Отдельная таблица для фильтрации.

```prisma
model Education {
  id          String    @id @default(uuid())
  resumeId    String    @map("resume_id")
  institution String    @db.VarChar(255)
  degree      String?   @db.VarChar(255)
  field       String?   @db.VarChar(255)
  startDate   DateTime? @map("start_date")
  endDate     DateTime? @map("end_date")
  description String?   @db.Text
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  createdBy   String?   @map("created_by")
  updatedBy   String?   @map("updated_by")

  resume       Resume @relation(fields: [resumeId], references: [id], onDelete: Cascade)
  createdByUser User? @relation("EducationCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User? @relation("EducationUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("educations")
  @@index([resumeId])
  @@index([institution])
  @@index([degree])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Отдельная таблица для фильтрации по образованию
- Индексы для быстрого поиска

### 14. Vacancy - Вакансия

Вакансия от работодателя. Привязана к проекту компании.

```prisma
model Vacancy {
  id              String    @id @default(uuid())
  projectId       String?   @map("project_id") // Необязательно - вакансия может существовать без проекта
  title           String    @db.VarChar(255)
  description     String    @db.Text
  position        String?   @db.VarChar(255)
  salaryMin       Int?      @map("salary_min")
  salaryMax       Int?      @map("salary_max")
  currency        String?   @default("RUB") @db.VarChar(3)
  location        String?   @db.VarChar(255)
  workType        WorkType  @default(FULL_TIME) @map("work_type")
  employmentType  EmploymentType @default(PERMANENT) @map("employment_type")
  isActive        Boolean   @default(true) @map("is_active")
  isPublic        Boolean   @default(true) @map("is_public")
  expiresAt       DateTime? @map("expires_at")
  viewsCount      Int       @default(0) @map("views_count")
  repliesCount    Int       @default(0) @map("replies_count") // Renamed from applications_count
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  createdBy       String?   @map("created_by")
  updatedBy       String?   @map("updated_by")

  project     Project?  @relation(fields: [projectId], references: [id], onDelete: SetNull)
  skills      VacancySkill[]
  replies     Reply[]
  createdByUser User?   @relation("VacancyCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?   @relation("VacancyUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("vacancies")
  @@index([projectId])
  @@index([isActive])
  @@index([isPublic])
  @@index([workType])
  @@index([employmentType])
  @@index([expiresAt])
  @@index([createdBy])
  @@index([updatedBy])
}

enum WorkType {
  FULL_TIME
  PART_TIME
  CONTRACT
  INTERNSHIP
  REMOTE
  HYBRID
}

enum EmploymentType {
  PERMANENT
  TEMPORARY
  CONTRACT
  INTERNSHIP
}
```

**Принципы:**
- Один работодатель может иметь множество вакансий
- `isActive` - активно ли размещение
- `isPublic` - видно ли всем или только по ссылке
- `expiresAt` - срок действия вакансии
- Счетчики для статистики

### 15. Reply - Отклик (бывший Application)

Отклик кандидата на вакансию. Центральная сущность для взаимодействия. Переименован из Application в Reply.

```prisma
model Reply {
  id              String        @id @default(uuid())
  candidateId     String        @map("candidate_id")
  vacancyId       String        @map("vacancy_id")
  resumeId        String?       @map("resume_id")
  resumeVersionId String?       @map("resume_version_id")
  status          ReplyStatus   @default(NEW)
  coverLetter     String?       @map("cover_letter") @db.Text
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
  createdBy       String?       @map("created_by")
  updatedBy       String?       @map("updated_by")

  candidate      User          @relation("ReplyCandidate", fields: [candidateId], references: [id], onDelete: Cascade)
  // Связь с компанией через vacancy.project.company
  vacancy        Vacancy       @relation(fields: [vacancyId], references: [id], onDelete: Cascade)
  resume         Resume?       @relation(fields: [resumeId], references: [id], onDelete: SetNull)
  resumeVersion  ResumeVersion? @relation(fields: [resumeVersionId], references: [id], onDelete: SetNull)
  chat           Chat?
  createdByUser  User?         @relation("ReplyCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser  User?         @relation("ReplyUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@unique([candidateId, vacancyId])
  @@map("replies")
  @@index([candidateId])
  @@index([vacancyId])
  @@index([status])
  @@index([createdAt])
  @@index([createdBy])
  @@index([updatedBy])
}

enum ReplyStatus {
  NEW           // Новый отклик
  VIEWED        // Просмотрен работодателем
  INVITED       // Приглашен на собеседование
  REJECTED      // Отклонен
  ACCEPTED      // Принят
  WITHDRAWN     // Отозван кандидатом
}
```

**Принципы:**
- Один кандидат может откликнуться на вакансию только один раз (unique constraint)
- `resumeId` - какое резюме использовано (может быть null, если отклик без резюме)
- `resumeVersionId` - версия резюме на момент отклика (для истории)
- `status` - статус отклика с различными этапами
- `coverLetter` - сопроводительное письмо
- Связь с Chat для общения по отклику
- В будущем: лимит 10 откликов/день (логика в сервисе)

### 16. Skill - Навык/Хэштег

Универсальная система навыков для резюме и вакансий.

```prisma
model Skill {
  id          String   @id @default(uuid())
  name        String   @unique @db.VarChar(100)
  slug        String   @unique @db.VarChar(100)
  category    String?  @db.VarChar(50)
  description String?  @db.Text
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  createdBy   String?  @map("created_by")
  updatedBy   String?  @map("updated_by")

  resumeSkills ResumeSkill[]
  vacancySkills VacancySkill[]
  createdByUser User?  @relation("SkillCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?  @relation("SkillUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("skills")
  @@index([name])
  @@index([slug])
  @@index([category])
  @@index([createdBy])
  @@index([updatedBy])
}
```

**Принципы:**
- Централизованная таблица навыков
- `slug` для URL-friendly идентификации
- `category` для группировки (например: "programming", "design", "languages")
- Связь many-to-many с резюме и вакансиями

### 17. ResumeSkill - Навыки в резюме

Связь резюме и навыков (many-to-many).

```prisma
model ResumeSkill {
  id        String   @id @default(uuid())
  resumeId  String   @map("resume_id")
  skillId   String   @map("skill_id")
  level     SkillLevel? @default(BASIC)
  createdAt DateTime @default(now())
  createdBy String?  @map("created_by")

  resume       Resume @relation(fields: [resumeId], references: [id], onDelete: Cascade)
  skill        Skill  @relation(fields: [skillId], references: [id], onDelete: Cascade)
  createdByUser User? @relation("ResumeSkillCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)

  @@unique([resumeId, skillId])
  @@map("resume_skills")
  @@index([resumeId])
  @@index([skillId])
  @@index([createdBy])
}

enum SkillLevel {
  BASIC
  INTERMEDIATE
  ADVANCED
  EXPERT
}
```

**Принципы:**
- Many-to-many связь
- `level` - уровень владения навыком
- Unique constraint для предотвращения дубликатов

### 18. VacancySkill - Требования в вакансии

Связь вакансии и требуемых навыков (many-to-many).

```prisma
model VacancySkill {
  id        String   @id @default(uuid())
  vacancyId String   @map("vacancy_id")
  skillId   String   @map("skill_id")
  isRequired Boolean @default(false) @map("is_required")
  level     SkillLevel?
  createdAt DateTime @default(now())
  createdBy String?  @map("created_by")

  vacancy       Vacancy @relation(fields: [vacancyId], references: [id], onDelete: Cascade)
  skill         Skill   @relation(fields: [skillId], references: [id], onDelete: Cascade)
  createdByUser User?   @relation("VacancySkillCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)

  @@unique([vacancyId, skillId])
  @@map("vacancy_skills")
  @@index([vacancyId])
  @@index([skillId])
  @@index([createdBy])
}
```

**Принципы:**
- Many-to-many связь
- `isRequired` - обязателен ли навык
- `level` - требуемый уровень

### 19. Chat - Чат

Чат для общения между кандидатом и работодателем.

```prisma
model Chat {
  id            String    @id @default(uuid())
  replyId       String?   @unique @map("reply_id") // Renamed from applicationId
  type          ChatType  @default(DIRECT)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  createdBy     String?   @map("created_by")
  updatedBy     String?   @map("updated_by")

  reply     Reply?       @relation(fields: [replyId], references: [id], onDelete: Cascade)
  members   ChatMember[]
  messages  Message[]
  createdByUser User?    @relation("ChatCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?    @relation("ChatUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("chats")
  @@index([replyId])
  @@index([type])
  @@index([createdBy])
  @@index([updatedBy])
}

enum ChatType {
  DIRECT      // Прямой чат между двумя пользователями
  APPLICATION // Чат по отклику
  GROUP       // Групповой чат (на будущее)
}
```

**Принципы:**
- Чат может быть привязан к Reply (бывший Application)
- Или прямой чат между пользователями
- `type` для различных типов чатов

### 20. ChatMember - Участники чата

Участники чата (many-to-many).

```prisma
model ChatMember {
  id        String   @id @default(uuid())
  chatId    String   @map("chat_id")
  userId    String   @map("user_id")
  role      ChatRole @default(MEMBER)
  joinedAt  DateTime @default(now())
  leftAt    DateTime?
  lastReadAt DateTime? @map("last_read_at")

  chat Chat @relation(fields: [chatId], references: [id], onDelete: Cascade)
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([chatId, userId])
  @@map("chat_members")
  @@index([chatId])
  @@index([userId])
}

enum ChatRole {
  MEMBER
  ADMIN
  OWNER
}
```

**Принципы:**
- Many-to-many связь
- `lastReadAt` для отслеживания прочитанных сообщений
- `leftAt` для истории участников

### 21. Message - Сообщение

Сообщение в чате.

```prisma
model Message {
  id        String      @id @default(uuid())
  chatId    String      @map("chat_id")
  userId    String      @map("user_id")
  content   String      @db.Text
  type      MessageType @default(TEXT)
  fileId    String?     @map("file_id")
  replyToId String?     @map("reply_to_id")
  isEdited  Boolean     @default(false) @map("is_edited")
  isDeleted Boolean     @default(false) @map("is_deleted")
  createdAt DateTime    @default(now())
  updatedAt DateTime    @updatedAt
  createdBy String?     @map("created_by")
  updatedBy  String?     @map("updated_by")

  chat         Chat     @relation(fields: [chatId], references: [id], onDelete: Cascade)
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  file         File?    @relation(fields: [fileId], references: [id], onDelete: SetNull)
  replyTo      Message? @relation("MessageReply", fields: [replyToId], references: [id], onDelete: SetNull)
  replies      Message[] @relation("MessageReply")
  createdByUser User?    @relation("MessageCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)
  updatedByUser User?    @relation("MessageUpdatedBy", fields: [updatedBy], references: [id], onDelete: SetNull)

  @@map("messages")
  @@index([chatId])
  @@index([userId])
  @@index([createdAt])
  @@index([createdBy])
  @@index([updatedBy])
}

enum MessageType {
  TEXT
  IMAGE
  FILE
  SYSTEM
}
```

**Принципы:**
- Сообщения в чате
- Поддержка файлов и ответов
- Мягкое удаление через `isDeleted`
- `isEdited` для отслеживания редактирования

### 22. File - Файл

Загруженные файлы (резюме, документы, изображения).

```prisma
model File {
  id          String    @id @default(uuid())
  userId      String    @map("user_id")
  name        String    @db.VarChar(255)
  originalName String   @map("original_name") @db.VarChar(255)
  mimeType    String    @map("mime_type") @db.VarChar(100)
  size        Int
  url         String    @db.VarChar(500)
  s3Key       String    @map("s3_key") @db.VarChar(500)
  type        FileType
  createdAt   DateTime  @default(now())
  createdBy   String?   @map("created_by")

  user         User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  messages     Message[]
  createdByUser User?    @relation("FileCreatedBy", fields: [createdBy], references: [id], onDelete: SetNull)

  @@map("files")
  @@index([userId])
  @@index([type])
  @@index([createdAt])
  @@index([createdBy])
}

enum FileType {
  RESUME      // Резюме
  DOCUMENT    // Документ
  AVATAR      // Аватар
  IMAGE       // Изображение
  VACANCY_IMAGE // Изображение вакансии
}
```

**Принципы:**
- Хранение метаданных файлов
- `s3Key` для связи с S3
- `url` для доступа к файлу
- Различные типы файлов


## Индексы и оптимизация

### Критичные индексы

```prisma
// User
@@index([email])
@@index([role])

// Reply (formerly Application)
@@index([candidateId])
@@index([vacancyId])
@@index([status])
@@index([createdAt])

// Vacancy
@@index([employerId])
@@index([isActive])
@@index([isPublic])

// Resume
@@index([candidateId])
@@index([isActive])

// Message
@@index([chatId])
@@index([createdAt])

```

## Миграции

### Создание миграции

```bash
cd apps/api
npx prisma migrate dev --name init_hr_platform
```

### Применение миграций

```bash
# Development
npx prisma migrate dev

# Production
npx prisma migrate deploy
```

## Примечания по реализации

1. **Версионирование резюме**: Структура заложена, но на MVP не используется. Можно добавить `resumeVersionId` в Reply для истории.

2. **Лимит откликов**: Логика 10 откликов/день реализуется в сервисе, не на уровне БД.

3. **Контакты**: Основные контакты в профиле, опциональные переопределения в резюме.

4. **JSON для резюме**: Для фронта можно собирать полный JSON резюме из связанных таблиц (Experience, Education, Skills).

5. **Мягкое удаление**: Для некоторых сущностей (Message) используется `isDeleted` вместо физического удаления.

6. **Cascade delete**: Настроено для автоматической очистки связанных данных при удалении родительских сущностей.
