# Future Extensions - Планируемые расширения Auth Module

## Обзор

Этот документ описывает планируемые расширения функционала аутентификации, которые будут реализованы в будущем. Текущая архитектура уже поддерживает multi-device и multi-role систему, что позволяет легко интегрировать новые методы аутентификации.

## Планируемые расширения

### 1. OAuth 2.0 Авторизация

Поддержка авторизации через сторонние провайдеры без использования пароля.

#### Поддерживаемые провайдеры

1. **Google OAuth 2.0**
   - Стандартная OAuth 2.0 авторизация
   - Получение email и базовой информации профиля

2. **Yandex OAuth 2.0**
   - OAuth 2.0 авторизация через Yandex ID
   - Получение email и информации профиля

3. **Telegram Login Widget**
   - Авторизация через Telegram
   - Проверка подписи данных от Telegram

#### Архитектурный подход

**Принцип:**
- OAuth не создаёт отдельную сессию
- OAuth — это способ первичной идентификации пользователя
- После успешной OAuth авторизации система работает так же, как при обычном login
- Создаётся или находится User, создаётся RoleContext, генерируются токены (access + refresh)
- Refresh token привязывается к устройству (deviceId) и роли (roleContextId)

**Flow:**
```
OAuth Provider → Проверка/Создание User → Создание RoleContext → Генерация токенов → Стандартная сессия
```

#### Изменения в БД

**Новая таблица: `oauth_accounts`**

```prisma
model OAuthAccount {
  id            String    @id @default(uuid())
  userId        String    @map("user_id")
  provider      OAuthProvider
  providerUserId String  @map("provider_user_id")
  email         String?
  accessToken   String?  @map("access_token") @db.Text
  refreshToken  String?  @map("refresh_token") @db.Text
  expiresAt     DateTime? @map("expires_at")
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerUserId])
  @@map("oauth_accounts")
  @@index([userId])
  @@index([provider])
  @@index([email])
}

enum OAuthProvider {
  GOOGLE
  YANDEX
  TELEGRAM
}
```

**Принципы:**
- Один User может иметь несколько OAuthAccount (разные провайдеры)
- `providerUserId` — уникальный ID пользователя у провайдера
- `UNIQUE(provider, providerUserId)` — один провайдер = один аккаунт
- `accessToken` и `refreshToken` опциональны (зависит от провайдера)
- При удалении User все OAuthAccount удаляются (cascade)

#### Структура модуля

```
auth/
 ├ strategies/
 │   ├ google.strategy.ts      # Google OAuth 2.0
 │   ├ yandex.strategy.ts      # Yandex OAuth 2.0
 │   ├ telegram.strategy.ts    # Telegram Login Widget
 │
 ├ services/
 │   ├ oauth.service.ts        # Бизнес-логика OAuth
 │
 ├ dto/
 │   ├ oauth-callback.dto.ts
 │   ├ telegram-login.dto.ts
```

#### Google OAuth 2.0

**Библиотеки:**
```bash
npm install passport-google-oauth20
npm install @types/passport-google-oauth20
```

**Стратегия:**
```typescript
// infrastructure/strategies/google.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy, VerifyCallback } from 'passport-google-oauth20';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(
    private readonly oauthService: OAuthService,
    private readonly configService: ConfigService,
  ) {
    super({
      clientID: configService.get<string>('GOOGLE_CLIENT_ID'),
      clientSecret: configService.get<string>('GOOGLE_CLIENT_SECRET'),
      callbackURL: configService.get<string>('GOOGLE_CALLBACK_URL'),
      scope: ['email', 'profile'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ): Promise<any> {
    const { id, emails, displayName } = profile;
    const email = emails[0]?.value;

    const user = await this.oauthService.findOrCreateUser({
      provider: 'GOOGLE',
      providerUserId: id,
      email,
      name: displayName,
      accessToken,
      refreshToken,
    });

    return user;
  }
}
```

**Flow:**
1. Клиент редиректит на `/auth/google`
2. GoogleStrategy обрабатывает запрос
3. Google возвращает профиль пользователя
4. `validate()` вызывается с данными профиля
5. `OAuthService.findOrCreateUser()`:
   - Ищет OAuthAccount по `provider + providerUserId`
   - Если не найден:
     - Ищет User по email
     - Если User не найден — создаёт нового
     - Создаёт OAuthAccount
   - Если найден — обновляет токены (если нужно)
6. Возвращается User
7. AuthController создаёт RoleContext (если нужно)
8. Генерируются access + refresh токены
9. Refresh токен сохраняется с deviceId и roleContextId
10. Возвращаются токены клиенту

**Endpoints:**
```
GET  /auth/google          # Инициация OAuth
GET  /auth/google/callback # Callback от Google
```

#### Yandex OAuth 2.0

**Библиотеки:**
```bash
npm install passport-oauth2
npm install @types/passport-oauth2
```

**Стратегия:**
```typescript
// infrastructure/strategies/yandex.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-oauth2';

@Injectable()
export class YandexStrategy extends PassportStrategy(Strategy, 'yandex') {
  constructor(
    private readonly oauthService: OAuthService,
    private readonly configService: ConfigService,
  ) {
    super({
      authorizationURL: 'https://oauth.yandex.ru/authorize',
      tokenURL: 'https://oauth.yandex.ru/token',
      clientID: configService.get<string>('YANDEX_CLIENT_ID'),
      clientSecret: configService.get<string>('YANDEX_CLIENT_SECRET'),
      callbackURL: configService.get<string>('YANDEX_CALLBACK_URL'),
      scope: ['login:email', 'login:info'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ): Promise<any> {
    // Получение информации о пользователе через Yandex API
    const userInfo = await this.getUserInfo(accessToken);

    const user = await this.oauthService.findOrCreateUser({
      provider: 'YANDEX',
      providerUserId: userInfo.id,
      email: userInfo.default_email,
      name: userInfo.real_name || userInfo.login,
      accessToken,
      refreshToken,
    });

    return user;
  }

  private async getUserInfo(accessToken: string) {
    const response = await fetch('https://login.yandex.ru/info', {
      headers: { Authorization: `OAuth ${accessToken}` },
    });
    return response.json();
  }
}
```

**Flow:**
Аналогично Google OAuth.

**Endpoints:**
```
GET  /auth/yandex          # Инициация OAuth
GET  /auth/yandex/callback # Callback от Yandex
```

#### Telegram Login Widget

**Особенности:**
- Telegram не использует стандартный OAuth 2.0 flow
- Клиент получает данные через Telegram Widget на фронтенде
- Backend проверяет подпись данных

**Библиотеки:**
```bash
# Используем встроенный crypto для проверки подписи
# или
npm install @telegram-auth/server
```

**Стратегия:**
```typescript
// infrastructure/strategies/telegram.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-custom';
import { OAuthService } from '../../application/services/oauth.service';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class TelegramStrategy extends PassportStrategy(Strategy, 'telegram') {
  constructor(
    private readonly oauthService: OAuthService,
    private readonly configService: ConfigService,
  ) {
    super();
  }

  async validate(req: Request): Promise<any> {
    const { id, first_name, last_name, username, photo_url, auth_date, hash } = req.body;

    // Проверка подписи
    const botToken = this.configService.get<string>('TELEGRAM_BOT_TOKEN');
    const isValid = this.verifyTelegramAuth(req.body, botToken);

    if (!isValid) {
      throw new UnauthorizedException('Invalid Telegram authentication data');
    }

    // Проверка времени (auth_date не старше 24 часов)
    const authDate = new Date(auth_date * 1000);
    const now = new Date();
    if (now.getTime() - authDate.getTime() > 24 * 60 * 60 * 1000) {
      throw new UnauthorizedException('Telegram authentication data expired');
    }

    const user = await this.oauthService.findOrCreateUser({
      provider: 'TELEGRAM',
      providerUserId: id.toString(),
      email: null, // Telegram не предоставляет email
      name: `${first_name} ${last_name || ''}`.trim() || username,
      accessToken: null,
      refreshToken: null,
    });

    return user;
  }

  private verifyTelegramAuth(data: any, botToken: string): boolean {
    const { hash, ...userData } = data;
    const dataCheckString = Object.keys(userData)
      .sort()
      .map(key => `${key}=${userData[key]}`)
      .join('\n');

    const secretKey = crypto
      .createHash('sha256')
      .update(botToken)
      .digest();

    const calculatedHash = crypto
      .createHmac('sha256', secretKey)
      .update(dataCheckString)
      .digest('hex');

    return calculatedHash === hash;
  }
}
```

**DTO:**
```typescript
// api/dto/telegram-login.dto.ts
export class TelegramLoginDto {
  @IsNumber()
  id: number;

  @IsString()
  first_name: string;

  @IsString()
  @IsOptional()
  last_name?: string;

  @IsString()
  @IsOptional()
  username?: string;

  @IsString()
  @IsOptional()
  photo_url?: string;

  @IsNumber()
  auth_date: number;

  @IsString()
  hash: string;
}
```

**Flow:**
1. Клиент получает данные через Telegram Widget на фронтенде
2. Клиент отправляет POST `/auth/telegram` с данными
3. TelegramStrategy проверяет подпись и timestamp
4. `OAuthService.findOrCreateUser()`:
   - Ищет OAuthAccount по `provider + providerUserId`
   - Если не найден — создаёт User и OAuthAccount
5. Создаётся RoleContext (если нужно)
6. Генерируются токены
7. Возвращаются токены

**Endpoints:**
```
POST /auth/telegram        # Авторизация через Telegram
```

#### OAuth Service

```typescript
// application/services/oauth.service.ts
@Injectable()
export class OAuthService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly oauthAccountRepository: OAuthAccountRepository,
    private readonly roleContextRepository: RoleContextRepository,
  ) {}

  async findOrCreateUser(data: {
    provider: OAuthProvider;
    providerUserId: string;
    email: string | null;
    name: string;
    accessToken?: string;
    refreshToken?: string;
  }): Promise<User> {
    // 1. Ищем существующий OAuthAccount
    let oauthAccount = await this.oauthAccountRepository.findByProvider(
      data.provider,
      data.providerUserId,
    );

    if (oauthAccount) {
      // Обновляем токены если нужно
      if (data.accessToken) {
        await this.oauthAccountRepository.update(oauthAccount.id, {
          accessToken: data.accessToken,
          refreshToken: data.refreshToken,
        });
      }
      return this.userRepository.findById(oauthAccount.userId);
    }

    // 2. Если OAuthAccount не найден, ищем User по email
    let user: User | null = null;
    if (data.email) {
      user = await this.userRepository.findByEmail(data.email);
    }

    // 3. Если User не найден, создаём нового
    if (!user) {
      user = await this.userRepository.create({
        email: data.email || `telegram_${data.providerUserId}@temp.local`, // Временный email для Telegram
        password: null, // OAuth users не имеют пароля
        isActivated: true, // OAuth аккаунты считаются активированными
      });
    }

    // 4. Создаём OAuthAccount
    await this.oauthAccountRepository.create({
      userId: user.id,
      provider: data.provider,
      providerUserId: data.providerUserId,
      email: data.email,
      accessToken: data.accessToken,
      refreshToken: data.refreshToken,
    });

    return user;
  }
}
```

#### Security Considerations для OAuth

1. **State параметр**
   - Генерация уникального `state` для каждого OAuth запроса
   - Сохранение в сессии или Redis
   - Проверка при callback для защиты от CSRF

2. **Replay attack (Telegram)**
   - Проверка `auth_date` (не старше 24 часов)
   - Проверка подписи `hash`
   - Хранение использованных `hash` в Redis (TTL 24 часа)

3. **Account linking**
   - Возможность привязать несколько OAuth провайдеров к одному User
   - Проверка email при привязке (если email совпадает)

4. **Токены провайдеров**
   - Хранение `accessToken` и `refreshToken` в зашифрованном виде
   - Использование для получения дополнительной информации (если нужно)

### 2. Phone OTP Authentication

Аутентификация через подтверждение номера телефона без использования пароля.

#### Архитектурный принцип

**Принцип:**
- Phone login — это passwordless authentication
- OTP (One-Time Password) отправляется через SMS
- OTP имеет ограниченное время жизни (TTL)
- После подтверждения OTP создаётся стандартная сессия с токенами
- Поддерживается multi-device и multi-role (как обычный login)

#### Изменения в БД

**Новая таблица: `phone_verifications`**

```prisma
model PhoneVerification {
  id          String    @id @default(uuid())
  phone       String    @db.VarChar(20) // Формат: +79991234567
  code        String    @db.VarChar(6)  // 6-значный код
  attempts    Int       @default(0)     // Количество попыток
  expiresAt   DateTime  @map("expires_at")
  verifiedAt  DateTime? @map("verified_at")
  createdAt   DateTime  @default(now())

  @@unique([phone, code, expiresAt]) // Для быстрого поиска
  @@map("phone_verifications")
  @@index([phone])
  @@index([expiresAt])
}
```

**Альтернатива (через Redis):**
- Хранение OTP в Redis с TTL
- Ключ: `phone_otp:{phone}`
- Значение: `{code, attempts, expiresAt}`
- TTL: 5-10 минут

**Принципы:**
- Один OTP на один номер телефона (новый OTP инвалидирует старый)
- Ограничение количества попыток (max 3-5 попыток)
- Rate limiting на отправку SMS (1 SMS в минуту на номер)
- Автоматическая очистка истёкших OTP через cron

#### Библиотеки

```bash
# SMS провайдер (выбрать один)
npm install twilio              # Twilio
npm install @smsaero/smsaero    # SMS Aero
npm install @vonage/server-sdk  # Vonage (ex Nexmo)

# Redis для хранения OTP (опционально)
npm install ioredis
npm install @nestjs/redis

# Cron для очистки
npm install @nestjs/schedule
```

#### Phone Auth Service

```typescript
// application/services/phone-auth.service.ts
@Injectable()
export class PhoneAuthService {
  constructor(
    private readonly smsService: SmsService,
    private readonly redisService: RedisService, // или Prisma для БД
    private readonly userRepository: UserRepository,
    private readonly roleContextRepository: RoleContextRepository,
    private readonly tokenService: TokenService,
  ) {}

  async requestOtp(phone: string, deviceId: string): Promise<void> {
    // 1. Валидация номера телефона
    if (!this.isValidPhone(phone)) {
      throw new BadRequestException('Invalid phone number');
    }

    // 2. Rate limiting (проверка последней отправки)
    const lastSent = await this.redisService.get(`phone_rate:${phone}`);
    if (lastSent) {
      const timeDiff = Date.now() - parseInt(lastSent);
      if (timeDiff < 60000) { // 1 минута
        throw new TooManyRequestsException('Please wait before requesting new code');
      }
    }

    // 3. Генерация 6-значного кода
    const code = this.generateOtpCode();

    // 4. Сохранение OTP (Redis или БД)
    const expiresAt = new Date(Date.now() + 5 * 60 * 1000); // 5 минут
    await this.redisService.setex(
      `phone_otp:${phone}`,
      300, // 5 минут TTL
      JSON.stringify({
        code,
        attempts: 0,
        expiresAt: expiresAt.toISOString(),
        deviceId,
      }),
    );

    // 5. Отправка SMS
    await this.smsService.send(phone, `Your verification code: ${code}`);

    // 6. Сохранение timestamp для rate limiting
    await this.redisService.setex(`phone_rate:${phone}`, 60, Date.now().toString());
  }

  async verifyOtp(
    phone: string,
    code: string,
    deviceId: string,
    roleContextId?: string,
  ): Promise<{ accessToken: string; refreshToken: string }> {
    // 1. Получение OTP из Redis/БД
    const otpData = await this.redisService.get(`phone_otp:${phone}`);
    if (!otpData) {
      throw new UnauthorizedException('OTP not found or expired');
    }

    const { code: storedCode, attempts, expiresAt, deviceId: storedDeviceId } = JSON.parse(otpData);

    // 2. Проверка истечения
    if (new Date(expiresAt) < new Date()) {
      await this.redisService.del(`phone_otp:${phone}`);
      throw new UnauthorizedException('OTP expired');
    }

    // 3. Проверка количества попыток
    if (attempts >= 5) {
      await this.redisService.del(`phone_otp:${phone}`);
      throw new UnauthorizedException('Too many attempts. Please request new code');
    }

    // 4. Проверка кода
    if (code !== storedCode) {
      await this.redisService.setex(
        `phone_otp:${phone}`,
        300,
        JSON.stringify({
          code: storedCode,
          attempts: attempts + 1,
          expiresAt,
          deviceId: storedDeviceId,
        }),
      );
      throw new UnauthorizedException('Invalid code');
    }

    // 5. Удаление использованного OTP
    await this.redisService.del(`phone_otp:${phone}`);

    // 6. Поиск или создание User
    let user = await this.userRepository.findByPhone(phone);
    if (!user) {
      user = await this.userRepository.create({
        email: null, // Phone-only users не имеют email
        phone,
        password: null, // Phone-only users не имеют пароля
        isActivated: true, // Phone verified = activated
      });
    }

    // 7. Создание или выбор RoleContext
    let roleContext: RoleContext;
    if (roleContextId) {
      roleContext = await this.roleContextRepository.findById(roleContextId);
      if (!roleContext || roleContext.userId !== user.id) {
        throw new ForbiddenException('Invalid role context');
      }
    } else {
      // Если у пользователя нет ролей, создаём CANDIDATE по умолчанию
      const existingRoles = await this.roleContextRepository.findByUserId(user.id);
      if (existingRoles.length === 0) {
        import { UserRole } from '@auth/enums/user-role.enum';

        // userRole теперь enum, используем напрямую UserRole.CANDIDATE
        roleContext = await this.roleContextRepository.create({
          userId: user.id,
          userRole: UserRole.CANDIDATE, // Enum значение
          companyId: null,
          hrRoleId: null,
        });
      } else if (existingRoles.length === 1) {
        roleContext = existingRoles[0];
      } else {
        // Если несколько ролей, требуется выбор (возвращаем список)
        throw new BadRequestException('Multiple roles found. Please specify roleContextId');
      }
    }

    // 8. Генерация токенов
    const tokens = await this.tokenService.generateTokens(user, roleContext, deviceId);

    return tokens;
  }

  private generateOtpCode(): string {
    return Math.floor(100000 + Math.random() * 900000).toString();
  }

  private isValidPhone(phone: string): boolean {
    // Валидация формата: +79991234567
    return /^\+7\d{10}$/.test(phone);
  }
}
```

#### SMS Service

```typescript
// infrastructure/services/sms.service.ts
@Injectable()
export class SmsService {
  constructor(
    private readonly twilioClient: Twilio, // или другой провайдер
    private readonly configService: ConfigService,
  ) {}

  async send(phone: string, message: string): Promise<void> {
    try {
      await this.twilioClient.messages.create({
        body: message,
        to: phone,
        from: this.configService.get<string>('TWILIO_PHONE_NUMBER'),
      });
    } catch (error) {
      // Логирование ошибки
      throw new InternalServerErrorException('Failed to send SMS');
    }
  }
}
```

#### DTO

```typescript
// api/dto/phone-request-otp.dto.ts
export class PhoneRequestOtpDto {
  @IsString()
  @Matches(/^\+7\d{10}$/, {
    message: 'Phone must be in format +79991234567',
  })
  phone: string;

  @IsString()
  @IsNotEmpty()
  deviceId: string;
}

// api/dto/phone-verify-otp.dto.ts
export class PhoneVerifyOtpDto {
  @IsString()
  @Matches(/^\+7\d{10}$/)
  phone: string;

  @IsString()
  @Length(6, 6)
  @Matches(/^\d{6}$/)
  code: string;

  @IsString()
  @IsNotEmpty()
  deviceId: string;

  @IsString()
  @IsOptional()
  roleContextId?: string; // Если у пользователя несколько ролей
}
```

#### Flow: Phone Login

**Шаг 1: Запрос OTP**
```
POST /auth/phone/request
{
  "phone": "+79991234567",
  "deviceId": "device-123"
}

Response: 200 OK
{
  "message": "OTP sent to phone"
}
```

**Шаг 2: Подтверждение OTP**
```
POST /auth/phone/verify
{
  "phone": "+79991234567",
  "code": "123456",
  "deviceId": "device-123",
  "roleContextId": "optional-if-multiple-roles"
}

Response: 200 OK
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "abc123...",
  "user": {
    "id": "user-id",
    "phone": "+79991234567",
    "userRoleName": "CANDIDATE", // Значение из БД (совпадает с UserRole.CANDIDATE)
    "roleContextId": "rc-1"
  }
}
```

#### Security Considerations для Phone OTP

1. **Rate limiting**
   - Максимум 1 SMS в минуту на номер
   - Максимум 5 SMS в час на номер
   - Использование Redis для хранения timestamps

2. **OTP ограничения**
   - Максимум 5 попыток ввода кода
   - TTL: 5-10 минут
   - Одноразовый код (удаляется после использования)

3. **Валидация номера**
   - Проверка формата телефона
   - Проверка на валидность номера (если возможно)

4. **Хранение OTP**
   - Предпочтительно Redis (быстрое удаление по TTL)
   - Или БД с cron очисткой

5. **SMS провайдер**
   - Использование надёжного провайдера (Twilio, Vonage)
   - Обработка ошибок отправки
   - Логирование всех отправок

### 3. Account Linking (Привязка аккаунтов)

Возможность привязать несколько методов аутентификации к одному User.

#### Принцип

- Один User может иметь:
  - Email + Password
  - Google OAuth
  - Yandex OAuth
  - Telegram
  - Phone
- Все методы ведут к одному User
- Привязка происходит по email или phone

#### Flow привязки

1. Пользователь уже залогинен (например, через email/password)
2. Пользователь инициирует OAuth (например, Google)
3. Система проверяет:
   - Если OAuthAccount с таким email уже существует → привязка
   - Если нет → создаётся новый OAuthAccount для текущего User
4. Все методы аутентификации теперь доступны для этого User

#### Endpoints

```
POST /auth/link/google      # Привязать Google аккаунт
POST /auth/link/yandex      # Привязать Yandex аккаунт
POST /auth/link/telegram    # Привязать Telegram
POST /auth/link/phone       # Привязать телефон
GET  /auth/accounts         # Список привязанных аккаунтов
DELETE /auth/accounts/:id   # Отвязать аккаунт
```

### 4. Two-Factor Authentication (2FA)

Двухфакторная аутентификация для дополнительной безопасности.

#### Принцип

- После успешного login (email/password) требуется второй фактор
- Методы 2FA:
  - SMS OTP
  - TOTP (Time-based One-Time Password) через приложения (Google Authenticator, Authy)
  - Email OTP

#### Изменения в БД

```prisma
model User {
  // ... существующие поля
  twoFactorEnabled Boolean @default(false) @map("two_factor_enabled")
  twoFactorSecret  String? @map("two_factor_secret") // Для TOTP
  backupCodes      String[] @map("backup_codes") // Резервные коды
}

model TwoFactorVerification {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  method    TwoFactorMethod // SMS | TOTP | EMAIL
  code      String
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now())

  @@map("two_factor_verifications")
  @@index([userId])
  @@index([expiresAt])
}

enum TwoFactorMethod {
  SMS
  TOTP
  EMAIL
}
```

#### Flow

1. Login с email/password
2. Если `twoFactorEnabled = true`:
   - Генерируется временный токен (не полный access token)
   - Отправляется 2FA код (SMS/TOTP/Email)
   - Возвращается `requires2FA: true`
3. Клиент отправляет 2FA код
4. Проверка кода
5. Если валиден → выдаётся полный access + refresh токен

### 5. WebAuthn (Passkeys)

Аутентификация через биометрию или аппаратные ключи безопасности.

#### Принцип

- Использование Web Authentication API
- Поддержка:
  - Биометрия (Face ID, Touch ID, Windows Hello)
  - Аппаратные ключи (YubiKey, Titan Security Key)
  - Платформенные ключи (Passkeys)

#### Библиотеки

```bash
npm install @simplewebauthn/server
npm install @simplewebauthn/types
```

#### Изменения в БД

```prisma
model WebAuthnCredential {
  id              String   @id @default(uuid())
  userId          String   @map("user_id")
  credentialId   String   @unique @map("credential_id") @db.Text
  publicKey       String   @map("public_key") @db.Text
  counter         Int      @default(0)
  deviceName      String?  @map("device_name")
  createdAt       DateTime @default(now())
  lastUsedAt      DateTime? @map("last_used_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("webauthn_credentials")
  @@index([userId])
}
```

#### Flow

1. Регистрация: `/auth/webauthn/register/start` → `/auth/webauthn/register/finish`
2. Авторизация: `/auth/webauthn/login/start` → `/auth/webauthn/login/finish`
3. После успешной авторизации → стандартная сессия с токенами

## Интеграция с существующей системой

### Multi-Device поддержка

Все новые методы аутентификации (OAuth, Phone) должны:
- Создавать refresh token для конкретного устройства
- Сохранять `deviceId` в таблице `tokens`
- Сохранять `roleContextId` в таблице `tokens`
- Поддерживать несколько активных сессий на разных устройствах

### Multi-Role поддержка

Все новые методы должны:
- Поддерживать выбор роли при login (если у пользователя несколько RoleContext)
- Создавать RoleContext по умолчанию (CANDIDATE) если ролей нет
- Возвращать список доступных ролей если их несколько

### Security Considerations (общие)

1. **Rate limiting**
   - Ограничение количества запросов OTP/SMS
   - Ограничение количества OAuth попыток
   - Использование `@nestjs/throttler`

2. **Token security**
   - Refresh токены всегда хранятся в хэшированном виде
   - Access токены — stateless JWT
   - Привязка токенов к deviceId и roleContextId

3. **Account security**
   - Уведомления о новых методах аутентификации
   - Возможность отвязать аккаунты
   - История входов (опционально)

4. **Privacy**
   - Минимизация данных, запрашиваемых у OAuth провайдеров
   - Хранение только необходимых данных

## Необходимые библиотеки (сводка)

### OAuth
```bash
npm install @nestjs/passport passport
npm install passport-google-oauth20 @types/passport-google-oauth20
npm install passport-oauth2 @types/passport-oauth2
```

### Phone OTP
```bash
npm install twilio                    # или другой SMS провайдер
npm install ioredis @nestjs/redis    # для хранения OTP
npm install @nestjs/schedule         # для cron очистки
```

### WebAuthn
```bash
npm install @simplewebauthn/server
npm install @simplewebauthn/types
```

### Rate Limiting
```bash
npm install @nestjs/throttler
```

## Последовательность реализации

### Этап 1: OAuth (Google)
1. Установка библиотек
2. Создание таблицы `oauth_accounts`
3. Создание `GoogleStrategy`
4. Создание `OAuthService`
5. Добавление endpoints `/auth/google` и `/auth/google/callback`
6. Интеграция с существующей системой токенов
7. Тестирование

### Этап 2: OAuth (Yandex)
1. Создание `YandexStrategy`
2. Добавление endpoints
3. Тестирование

### Этап 3: Telegram Login
1. Создание `TelegramStrategy`
2. Реализация проверки подписи
3. Добавление endpoint `/auth/telegram`
4. Тестирование

### Этап 4: Phone OTP
1. Установка SMS провайдера
2. Настройка Redis (или создание таблицы `phone_verifications`)
3. Создание `PhoneAuthService`
4. Создание `SmsService`
5. Добавление endpoints `/auth/phone/request` и `/auth/phone/verify`
6. Реализация rate limiting
7. Тестирование

### Этап 5: Account Linking
1. Реализация логики привязки аккаунтов
2. Добавление endpoints для управления аккаунтами
3. Тестирование

### Этап 6: 2FA (опционально)
1. Создание таблицы `two_factor_verifications`
2. Реализация TOTP генерации
3. Интеграция с login flow
4. Тестирование

### Этап 7: WebAuthn (опционально)
1. Установка библиотек
2. Создание таблицы `webauthn_credentials`
3. Реализация регистрации и авторизации
4. Тестирование

## Примечания

- Все новые методы аутентификации должны интегрироваться с существующей системой токенов
- Multi-device и multi-role поддержка сохраняется для всех методов
- Security considerations должны быть реализованы с самого начала
- Рекомендуется поэтапная реализация (один метод за раз)
- Все изменения должны быть обратно совместимы с существующей системой
