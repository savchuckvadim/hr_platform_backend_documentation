# Registration Flow - Потоки регистрации

## Обзор

Система регистрации поддерживает два типа пользователей: кандидаты и работодатели. Каждый тип имеет свой endpoint и DTO с разными полями.

## Регистрация кандидата

### Endpoint

```
POST /auth/register/candidate
```

### DTO

**Файл:** `api/dto/register-candidate.dto.ts`

```typescript
export class RegisterCandidateDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain uppercase, lowercase and number',
  })
  password: string;

  @IsString()
  @IsNotEmpty()
  firstName: string;

  @IsString()
  @IsOptional()
  lastName?: string;

  @IsString()
  @IsOptional()
  middleName?: string;
}
```

### Поток регистрации

**Шаг 1: Валидация данных**

```typescript
// AuthController
@Post('register/candidate')
@Public()
async registerCandidate(@Body() dto: RegisterCandidateDto) {
  return this.authService.registerCandidate(dto);
}
```

**Шаг 2: Проверка существования пользователя**

```typescript
// AuthService
const existingUser = await this.userRepository.findByEmail(dto.email);
if (existingUser) {
  throw new ConflictException('User with this email already exists');
}
```

**Шаг 3: Создание User**

```typescript
const hashedPassword = await bcrypt.hash(dto.password, 10);
const activationLink = randomUUID();

const user = await this.userRepository.create({
  email: dto.email,
  password: hashedPassword,
  isActivated: false,
  activationLink,
});
```

**Шаг 4: Получение UserRole и создание RoleContext**

```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';

// Получаем роль CANDIDATE из таблицы user_roles (используем enum)
const userRole = await this.userRoleRepository.findByName(UserRoleName.CANDIDATE);
if (!userRole) {
  throw new InternalServerErrorException('CANDIDATE role not found. Run seed to create default roles.');
}

// Создаем RoleContext
const roleContext = await this.roleContextRepository.create({
  userId: user.id,
  userRoleId: userRole.id,
  companyId: null,
  hrRoleId: null,
});
```

**Шаг 5: Создание CandidateProfile**

```typescript
const profile = await this.candidateProfileRepository.create({
  userId: user.id,
  firstName: dto.firstName,
  lastName: dto.lastName,
  middleName: dto.middleName,
});
```

**Шаг 6: Генерация токенов**

```typescript
const deviceId = this.generateDeviceId(request);
const { accessToken, refreshToken } = await this.generateTokens(
  user,
  roleContext,
  deviceId,
  request,
);
```

**Шаг 7: Сохранение refresh token**

```typescript
const hashedRefreshToken = await bcrypt.hash(refreshToken, 10);
await this.tokenRepository.create({
  userId: user.id,
  roleContextId: roleContext.id,
  refreshToken: hashedRefreshToken,
  deviceId,
  deviceName: this.extractDeviceName(request),
  userAgent: request.headers['user-agent'],
  ipAddress: request.ip,
  expiresAt: this.calculateRefreshTokenExpiry(),
});
```

**Шаг 8: Отправка email активации**

```typescript
// Через EventBus
// Загружаем userRole для события
const userRole = await roleContext.userRole;

this.eventBus.emit(AppEvent.USER_CREATED, {
  userId: user.id,
  email: user.email,
  userRoleName: userRole.name, // Значение из БД (для предустановленных совпадает с UserRoleName enum)
  activationLink: `${baseUrl}/auth/activate/${user.activationLink}`,
});
```

**Шаг 9: Возврат ответа**

```typescript
return {
  accessToken,
  refreshToken, // В production - только в HttpOnly cookie
  user: {
    id: user.id,
    email: user.email,
    userRoleName: roleContext.userRole.name,
  },
};
```

### Полная последовательность

```
1. POST /auth/register/candidate
   ↓
2. Валидация DTO
   ↓
3. Проверка существования email
   ↓
4. Создание User
   ↓
5. Создание RoleContext (CANDIDATE)
   ↓
6. Создание CandidateProfile
   ↓
7. Генерация токенов
   ↓
8. Сохранение refresh token в БД
   ↓
9. Отправка email активации (через EventBus)
   ↓
10. Возврат токенов и данных пользователя
```

## Регистрация работодателя

### Endpoint

```
POST /auth/register/employer
```

### DTO

**Файл:** `api/dto/register-employer.dto.ts`

```typescript
export class RegisterEmployerDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
  password: string;

  @IsString()
  @IsNotEmpty()
  companyName: string;

  @IsString()
  @IsOptional()
  @Matches(/^\d{10}|\d{12}$/, {
    message: 'INN must be 10 or 12 digits',
  })
  inn?: string;

  @IsString()
  @IsNotEmpty()
  companyTypeId: string; // ID из таблицы company_types
}
```

### Поток регистрации

**Шаг 1-3:** Аналогично регистрации кандидата (валидация, проверка, создание User)

**Шаг 4: Создание Company**

```typescript
const company = await this.companyRepository.create({
  name: dto.companyName,
  inn: dto.inn,
  companyTypeId: dto.companyTypeId, // FK к company_types
  ownerId: user.id,
});
```

**Шаг 5: Получение ролей и создание RoleContext (EMPLOYER с HR_ADMIN)**

```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

// Получаем роль EMPLOYER из таблицы user_roles (используем enum)
const userRole = await this.userRoleRepository.findByName(UserRoleName.EMPLOYER);
if (!userRole) {
  throw new InternalServerErrorException('EMPLOYER role not found. Run seed to create default roles.');
}

// Получаем роль HR_ADMIN из таблицы hr_roles (используем enum)
const hrRole = await this.hrRoleRepository.findByName(HrRoleName.HR_ADMIN);
if (!hrRole) {
  throw new InternalServerErrorException('HR_ADMIN role not found. Run seed to create default roles.');
}

// Владелец компании регистрируется как EMPLOYER с правами HR_ADMIN
const employerRoleContext = await this.roleContextRepository.create({
  userId: user.id,
  userRoleId: userRole.id, // FK к user_roles
  companyId: company.id, // Обязательно для EMPLOYER роли
  hrRoleId: hrRole.id, // FK к hr_roles - владелец = HR_ADMIN
});
```

**Шаг 6: EmployerProfile больше не создаётся**

Вся информация о компании хранится в таблице `companies`.
Владелец компании работает через EMPLOYER роль.

**Шаг 7: Генерация токенов**

Аналогично регистрации кандидата (генерация токенов, сохранение, email, ответ)

### Полная последовательность

```
1. POST /auth/register/employer
   ↓
2. Валидация DTO
   ↓
3. Проверка существования email
   ↓
4. Создание User
   ↓
5. Создание Company (с companyTypeId)
   ↓
6. Получение UserRole (EMPLOYER) и HrRole (HR_ADMIN) из таблиц
7. Создание RoleContext (userRoleId, companyId, hrRoleId: HR_ADMIN)
   - Владелец компании = EMPLOYER с правами HR_ADMIN
   - EmployerProfile не создаётся - вся информация в Company
   ↓
7. Генерация токенов (для EMPLOYER роли)
   ↓
9. Сохранение refresh token в БД
   ↓
10. Отправка email активации
   ↓
11. Возврат токенов и данных пользователя
```

## Регистрация HR в существующую компанию

### Endpoint

```
POST /auth/register/hr
```

**Требования:**
- Только HR_ADMIN может регистрировать новых HR
- Компания должна существовать
- HR всегда привязан к компании

### DTO

**Файл:** `api/dto/register-hr.dto.ts`

```typescript
export class RegisterHrDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
  password: string;

  @IsString()
  @IsNotEmpty()
  companyId: string; // ID существующей компании

  @IsEnum(HrRoleName)
  @IsNotEmpty()
  hrRoleName: HrRoleName; // HR | HR_ADMIN - используем enum для типобезопасности
}
```

### Поток регистрации

**Шаг 1: Проверка прав**

```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

// Проверяем что текущий пользователь - HR_ADMIN этой компании
const currentRole = request.user.roleContext;
const currentUserRole = request.user.userRole;
const currentHrRole = request.user.hrRole;

if (currentUserRole?.name !== UserRoleName.EMPLOYER || currentHrRole?.name !== HrRoleName.HR_ADMIN) {
  throw new ForbiddenException('Only HR_ADMIN can register new HR');
}

if (currentRole.companyId !== dto.companyId) {
  throw new ForbiddenException('Can only register HR in your own company');
}
```

**Шаг 2: Проверка существования компании**

```typescript
const company = await this.companyRepository.findById(dto.companyId);
if (!company) {
  throw new NotFoundException('Company not found');
}
```

**Шаг 3: Проверка существования пользователя**

```typescript
const existingUser = await this.userRepository.findByEmail(dto.email);
if (existingUser) {
  throw new ConflictException('User with this email already exists');
}
```

**Шаг 4: Создание User**

```typescript
const hashedPassword = await bcrypt.hash(dto.password, 10);
const activationLink = randomUUID();

const user = await this.userRepository.create({
  email: dto.email,
  password: hashedPassword,
  isActivated: false,
  activationLink,
});
```

**Шаг 5: Получение ролей и создание RoleContext (EMPLOYER)**

```typescript
import { UserRoleName } from '@auth/enums/user-role-name.enum';
import { HrRoleName } from '@auth/enums/hr-role-name.enum';

// Получаем роль EMPLOYER из таблицы user_roles (используем enum)
const userRole = await this.userRoleRepository.findByName(UserRoleName.EMPLOYER);
if (!userRole) {
  throw new InternalServerErrorException('EMPLOYER role not found. Run seed to create default roles.');
}

// Получаем HR роль из таблицы hr_roles (используем enum из DTO)
const hrRole = await this.hrRoleRepository.findByName(dto.hrRoleName);
if (!hrRole) {
  throw new BadRequestException(`HR role '${dto.hrRoleName}' not found. Run seed to create default roles.`);
}

const employerRoleContext = await this.roleContextRepository.create({
  userId: user.id,
  userRoleId: userRole.id, // FK к user_roles
  companyId: dto.companyId, // Обязательно для EMPLOYER
  hrRoleId: hrRole.id, // FK к hr_roles - HR или HR_ADMIN
});
```

**Шаг 6-9:** Аналогично регистрации кандидата (генерация токенов, сохранение, email, ответ)

### Полная последовательность

```
1. POST /auth/register/hr (требует аутентификации HR_ADMIN)
   ↓
2. Проверка прав (HR_ADMIN, та же компания)
   ↓
3. Проверка существования компании
   ↓
4. Проверка существования email
   ↓
5. Создание User
   ↓
6. Получение UserRole (EMPLOYER) и HrRole (HR или HR_ADMIN) из таблиц
7. Создание RoleContext (userRoleId, companyId, hrRoleId)
   ↓
7. Генерация токенов
   ↓
8. Сохранение refresh token в БД
   ↓
9. Отправка email активации
   ↓
10. Возврат токенов и данных пользователя
```

## Генерация токенов

### Access Token

```typescript
private async generateAccessToken(
  user: User,
  roleContext: RoleContext,
): Promise<string> {
  // Загружаем userRole и hrRole
  const userRole = await roleContext.userRole;
  const hrRole = roleContext.hrRoleId ? await roleContext.hrRole : null;

  const payload: JwtPayload = {
    sub: user.id,
    roleContextId: roleContext.id,
    userRoleName: userRole.name, // Название системной роли из user_roles
    companyId: roleContext.companyId, // null для CANDIDATE, обязателен для EMPLOYER
    hrRoleName: hrRole?.name || null, // Название HR роли из hr_roles (только для EMPLOYER)
  };

  return this.jwtService.sign(payload, {
    expiresIn: this.configService.get<string>('JWT_EXPIRES_IN', '15m'),
  });
}
```

### Refresh Token

```typescript
private async generateRefreshToken(): Promise<string> {
  return randomBytes(32).toString('hex');
}
```

### Сохранение refresh token

```typescript
private async saveRefreshToken(
  userId: string,
  roleContextId: string,
  refreshToken: string,
  deviceId: string,
  request: Request,
): Promise<void> {
  // Проверка unique constraint
  const existingToken = await this.tokenRepository.findByUserRoleDevice(
    userId,
    roleContextId,
    deviceId,
  );

  if (existingToken) {
    // Удаляем старый токен (смена роли на устройстве)
    await this.tokenRepository.delete(existingToken.id);
  }

  const hashedToken = await bcrypt.hash(refreshToken, 10);
  const expiresAt = new Date();
  expiresAt.setDate(expiresAt.getDate() + 7); // 7 дней

  await this.tokenRepository.create({
    userId,
    roleContextId,
    refreshToken: hashedToken,
    deviceId,
    deviceName: this.extractDeviceName(request),
    userAgent: request.headers['user-agent'],
    ipAddress: request.ip,
    expiresAt,
  });
}
```

## Генерация deviceId

### Метод 1: Из заголовков

```typescript
private generateDeviceId(request: Request): string {
  // Комбинация user-agent и IP
  const fingerprint = `${request.headers['user-agent']}-${request.ip}`;
  return createHash('sha256').update(fingerprint).digest('hex');
}
```

### Метод 2: Из клиента

```typescript
// Клиент отправляет deviceId в заголовке или body
// Если не передан - генерируем на сервере
private generateDeviceId(request: Request): string {
  const clientDeviceId = request.headers['x-device-id'] || request.body?.deviceId;

  if (clientDeviceId) {
    return clientDeviceId;
  }

  // Fallback: генерация на сервере
  return randomUUID();
}
```

## Обработка ошибок

### Типичные ошибки

1. **ConflictException (409)**
   - Email уже существует
   - `User with this email already exists`

2. **BadRequestException (400)**
   - Невалидные данные DTO
   - Ошибки валидации полей

3. **InternalServerError (500)**
   - Ошибка создания в БД
   - Ошибка отправки email

## Активация email

### Endpoint

```
GET /auth/activate/:link
```

### Поток активации

```typescript
@Get('activate/:link')
@Public()
async activate(@Param('link') link: string) {
  const user = await this.userRepository.findByActivationLink(link);

  if (!user) {
    throw new NotFoundException('Invalid activation link');
  }

  if (user.isActivated) {
    throw new BadRequestException('User already activated');
  }

  await this.userRepository.update(user.id, {
    isActivated: true,
    activationLink: null, // Очищаем ссылку
  });

  return { message: 'Email activated successfully' };
}
```

## События

### USER_CREATED

После успешной регистрации эмитится событие:

```typescript
this.eventBus.emit(AppEvent.USER_CREATED, {
  userId: user.id,
  email: user.email,
  userRoleName: roleContext.userRole.name,
  activationLink: `${baseUrl}/auth/activate/${user.activationLink}`,
});
```

**Listeners:**
- Email модуль - отправка письма активации
- Analytics модуль - регистрация события
- Notification модуль - welcome уведомление

## Тестирование

### Unit тесты

```typescript
describe('AuthService - registerCandidate', () => {
  it('should create user, role context and profile', async () => {
    const dto = {
      email: 'test@example.com',
      password: 'Password123',
      firstName: 'John',
    };

    const result = await authService.registerCandidate(dto);

    expect(result).toHaveProperty('accessToken');
    expect(result).toHaveProperty('refreshToken');
    expect(result.user.email).toBe(dto.email);

    // Проверка создания в БД
    const user = await userRepository.findByEmail(dto.email);
    expect(user).toBeDefined();
    expect(user.isActivated).toBe(false);

    const roleContext = await roleContextRepository.findByUserId(user.id);
    expect(roleContext.userRole.name).toBe(UserRoleName.CANDIDATE);
  });

  it('should throw ConflictException if email exists', async () => {
    await userRepository.create({ email: 'test@example.com', ... });

    await expect(
      authService.registerCandidate({ email: 'test@example.com', ... }),
    ).rejects.toThrow(ConflictException);
  });
});
```

## Заключение

Потоки регистрации обеспечивают:

✅ Раздельную регистрацию для кандидатов, работодателей и HR
✅ Создание всех необходимых сущностей (User, RoleContext, Profile, Company)
✅ Автоматическое создание root EMPLOYER (HR_ADMIN) для работодателей
✅ Регистрацию HR в существующую компанию (только HR_ADMIN)
✅ Поддержку типов компаний (ORGANIZATION, IP, LAWYER, SELF_EMPLOYED, OTHER)
✅ Поддержку типов HR ролей (HR, HR_ADMIN)
✅ Генерацию и сохранение токенов с companyId и hrRoleName
✅ Отправку email активации
✅ Валидацию данных
✅ Обработку ошибок

Все потоки следуют единой архитектуре и интегрированы с системой событий.
