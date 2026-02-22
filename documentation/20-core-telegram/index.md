# Core Telegram - Telegram сервис и модуль

## Описание

Глобальный Telegram сервис и модуль для отправки уведомлений в Telegram. Используется для логирования ошибок, мониторинга запросов и отправки административных уведомлений.

**Ключевые особенности:**
- Глобальный модуль (`@Global()`) - доступен во всех модулях без явного импорта
- Отправка сообщений в Telegram через Bot API
- Экранирование Markdown символов для безопасной отправки
- Ограничение длины сообщений (лимит Telegram: 4096 символов)
- Интеграция с GlobalExceptionFilter и LoggerMiddleware

## Расположение

```
core/telegram/
├── telegram.service.ts    # TelegramService - глобальный сервис
├── telegram.module.ts     # TelegramModule - глобальный модуль
├── telegram.controller.ts # API для отправки сообщений (опционально)
├── telegram.dto.ts        # DTO для API
└── index.ts               # Экспорты модуля
```

## Настройка и подключение

### 1. Установка зависимостей

```bash
npm install @nestjs/axios
npm install axios
```

### 2. Конфигурация

**Переменные окружения (`.env`):**
```env
TELEGRAM_BOT_TOKEN=your_bot_token_here
TELEGRAM_ADMIN_CHAT_ID=your_chat_id_here
```

**Получение Bot Token:**
1. Создайте бота через [@BotFather](https://t.me/botfather)
2. Получите токен бота
3. Добавьте токен в `.env` как `TELEGRAM_BOT_TOKEN`

**Получение Chat ID:**
1. Начните диалог с вашим ботом
2. Отправьте любое сообщение боту
3. Перейдите по ссылке: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
4. Найдите `chat.id` в ответе
5. Добавьте ID в `.env` как `TELEGRAM_ADMIN_CHAT_ID`

### 3. Реализация TelegramService

```typescript
// core/telegram/telegram.service.ts
import { HttpService } from '@nestjs/axios';
import { Global, Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { firstValueFrom } from 'rxjs';
import { TelegramSendMessageDto } from './telegram.dto';

@Global()
@Injectable()
export class TelegramService {
    private botToken: string;
    private adminChatId: string;

    constructor(
        private readonly httpService: HttpService,
        private readonly configService: ConfigService,
    ) {
        this.botToken = this.configService.get<string>(
            'TELEGRAM_BOT_TOKEN',
        ) as string;
        this.adminChatId = this.configService.get<string>(
            'TELEGRAM_ADMIN_CHAT_ID',
        ) as string;
    }

    /**
     * Отправка публичного сообщения (с дополнительными полями)
     */
    public async sendPublicMessage(dto: TelegramSendMessageDto) {
        const text = `\n💥 App:  ${dto.app}\n🌍 Domain:   ${dto.domain}\n🧭 UserId: ${dto.userId}\n\n ⚠️ Text:  ${dto.text}`;
        const cleanText = this.cleanText(text);

        const url = `https://api.telegram.org/bot${this.botToken}/sendMessage`;
        const payload = {
            chat_id: Number(this.adminChatId),
            text: `NEST from front ${cleanText}`,
            parse_mode: 'Markdown',
        };

        try {
            await firstValueFrom(this.httpService.post(url, payload));
        } catch (error) {
            console.error('Telegram error:', error.message);
        }
        return cleanText;
    }

    /**
     * Отправка обычного сообщения
     */
    async sendMessage(message: string) {
        const cleanText = this.cleanText(message);

        const url = `https://api.telegram.org/bot${this.botToken}/sendMessage`;
        const payload = {
            chat_id: Number(this.adminChatId),
            text: `NEST ${cleanText}`,
            parse_mode: 'Markdown',
        };

        try {
            await firstValueFrom(this.httpService.post(url, payload));
        } catch (error) {
            console.error('Telegram error:', error.message);
        }
    }

    /**
     * Отправка сообщения об ошибке администратору
     */
    async sendMessageAdminError(message: string) {
        const cleanText = this.cleanText(message);

        const url = `https://api.telegram.org/bot${this.botToken}/sendMessage`;
        const payload = {
            chat_id: this.adminChatId,
            text: `NEST ADMIN ERROR: ${cleanText}`,
            parse_mode: 'Markdown',
        };

        try {
            await firstValueFrom(this.httpService.post(url, payload));
        } catch (error) {
            console.error('Telegram error:', error.message);
        }
    }

    /**
     * Очистка текста от Markdown символов
     */
    private cleanText(text: string) {
        return text
            .replace(/_/g, '\\_')
            .replace(/\*/g, '\\*')
            .replace(/\[/g, '\\[')
            .replace(/`/g, '\\`')
            .replace(/[_*[\]()~`>#+=|{}.!\\]/g, '\\$&') // экранируем ВСЁ, что может сломать markdown
            .slice(0, 4000); // Telegram лимит: 4096 символов
    }
}
```

**Особенности:**
- Глобальный сервис (`@Global()`) - доступен во всех модулях
- Использует `HttpService` из `@nestjs/axios` для HTTP запросов
- Получает конфигурацию через `ConfigService`
- Экранирует Markdown символы для безопасной отправки
- Ограничивает длину сообщений до 4000 символов (лимит Telegram: 4096)
- Обрабатывает ошибки отправки (не прерывает выполнение)

### 4. Реализация TelegramModule

```typescript
// core/telegram/telegram.module.ts
import { Module } from '@nestjs/common';
import { HttpModule } from '@nestjs/axios';
import { ConfigModule } from '@nestjs/config';
import { TelegramService } from './telegram.service';
import { TelegramController } from './telegram.controller';

@Module({
    imports: [HttpModule, ConfigModule],
    controllers: [TelegramController], // Опционально
    providers: [TelegramService],
    exports: [TelegramService],
})
export class TelegramModule {}
```

**Особенности:**
- Импортирует `HttpModule` для HTTP запросов
- Импортирует `ConfigModule` для доступа к переменным окружения
- Экспортирует `TelegramService` для использования в других модулях
- Опционально включает `TelegramController` для API endpoints

### 5. Регистрация в AppModule

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { TelegramModule } from '@core/telegram/telegram.module';
import { ConfigModule } from '@nestjs/config';

@Module({
    imports: [
        ConfigModule.forRoot({
            isGlobal: true, // ConfigModule также должен быть глобальным
        }),
        TelegramModule, // ✅ Импортируем TelegramModule
        // ... другие модули
    ],
})
export class AppModule {}
```

**Важно:**
- `ConfigModule` должен быть импортирован **до** `TelegramModule`, так как `TelegramService` использует `ConfigService`
- `ConfigModule` должен быть глобальным (`isGlobal: true`) или явно импортирован в `TelegramModule`

### 6. Telegram Controller (опционально)

```typescript
// core/telegram/telegram.controller.ts
import { Body, Controller, Post } from '@nestjs/common';
import { ApiBody, ApiOperation, ApiTags } from '@nestjs/swagger';
import { TelegramSendMessageDto } from './telegram.dto';
import { TelegramService } from './telegram.service';

@ApiTags('Telegram')
@Controller('telegram')
export class TelegramController {
    constructor(private readonly telegramService: TelegramService) {}

    @ApiOperation({ summary: 'Send message to telegram' })
    @ApiBody({ type: TelegramSendMessageDto })
    @Post()
    async sendMessage(@Body() dto: TelegramSendMessageDto) {
        return await this.telegramService.sendPublicMessage(dto);
    }
}
```

**Использование:**
```bash
POST /telegram
{
  "app": "kpi_sales",
  "text": "Test message",
  "domain": "example.ru",
  "userId": "user-123"
}
```

### 7. DTO

```typescript
// core/telegram/telegram.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEnum, IsNotEmpty, IsString } from 'class-validator';

export enum EnumTelegramApp {
    KPI_SALES = 'kpi_sales',
    KONSTRUKTOR = 'konstruktor',
}

export class TelegramSendMessageDto {
    @ApiProperty({ enum: EnumTelegramApp })
    @IsEnum(EnumTelegramApp)
    @IsNotEmpty()
    app: EnumTelegramApp;

    @ApiProperty({ description: 'Text message' })
    @IsString()
    @IsNotEmpty()
    text: string;

    @ApiProperty({ description: 'Domain', example: 'example.ru' })
    @IsString()
    @IsNotEmpty()
    domain: string;

    @ApiProperty({ description: 'User ID' })
    @IsString()
    @IsNotEmpty()
    userId: string;
}
```

## Использование в модулях

### В GlobalExceptionFilter

```typescript
// core/filters/global-exception.filter.ts
import { TelegramService } from '@core/telegram/telegram.service';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
    constructor(private readonly telegram: TelegramService) {}

    async catch(exception: unknown, host: ArgumentsHost) {
        // ... обработка ошибки

        const message = `⚠️ Ошибка: ${error.name}\n\n📄 Файл: ${file}\n🔢 Строка: ${line}\n...`;
        await this.telegram.sendMessage(message);

        // ... отправка ответа
    }
}
```

### В LoggerMiddleware

```typescript
// core/middleware/logger.middleware.ts
import { TelegramService } from '@core/telegram/telegram.service';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
    constructor(private readonly telegram: TelegramService) {}

    async use(req: Request, res: Response, next: NextFunction) {
        // ... логирование запроса
        await this.telegram.sendMessage(message);

        res.on('finish', async () => {
            // ... логирование ответа
            await this.telegram.sendMessage(responseMessage);
        });

        next();
    }
}
```

### В других сервисах

```typescript
// application/services/notification.service.ts
import { Injectable } from '@nestjs/common';
import { TelegramService } from '@core/telegram/telegram.service';

@Injectable()
export class NotificationService {
    constructor(private readonly telegram: TelegramService) {}

    async notifyAdmin(message: string) {
        await this.telegram.sendMessage(message);
    }

    async notifyError(error: Error) {
        await this.telegram.sendMessageAdminError(
            `Error: ${error.message}\nStack: ${error.stack}`,
        );
    }
}
```

## Методы сервиса

### sendMessage(message: string)

Отправляет обычное сообщение в Telegram.

**Параметры:**
- `message` - текст сообщения (будет очищен от Markdown символов)

**Пример:**
```typescript
await telegramService.sendMessage('Привет, это тестовое сообщение!');
```

### sendPublicMessage(dto: TelegramSendMessageDto)

Отправляет сообщение с дополнительными полями (app, domain, userId).

**Параметры:**
- `dto` - объект с полями: `app`, `text`, `domain`, `userId`

**Пример:**
```typescript
await telegramService.sendPublicMessage({
    app: EnumTelegramApp.KPI_SALES,
    text: 'Важное уведомление',
    domain: 'example.ru',
    userId: 'user-123',
});
```

### sendMessageAdminError(message: string)

Отправляет сообщение об ошибке администратору с префиксом "ADMIN ERROR".

**Параметры:**
- `message` - текст сообщения об ошибке

**Пример:**
```typescript
await telegramService.sendMessageAdminError(
    'Критическая ошибка в системе!',
);
```

## Обработка ошибок

Сервис обрабатывает ошибки отправки сообщений и не прерывает выполнение приложения:

```typescript
try {
    await firstValueFrom(this.httpService.post(url, payload));
} catch (error) {
    console.error('Telegram error:', error.message);
    // Не выбрасывает исключение - приложение продолжает работу
}
```

**Причины ошибок:**
- Неверный `TELEGRAM_BOT_TOKEN`
- Неверный `TELEGRAM_ADMIN_CHAT_ID`
- Проблемы с сетью
- Превышение лимита запросов к Telegram API

## Best Practices

1. **Используйте для критических уведомлений** - ошибки, важные события
2. **Не злоупотребляйте отправкой** - Telegram имеет лимиты на количество запросов
3. **Обрабатывайте ошибки** - сервис не должен прерывать работу приложения
4. **Используйте разные методы** - `sendMessage` для обычных, `sendMessageAdminError` для критических
5. **Ограничивайте длину сообщений** - используйте `cleanText()` для автоматического ограничения
6. **Экранируйте Markdown** - используйте `cleanText()` для безопасной отправки

## Интеграция с другими модулями

- **[Response Filters, Interceptors & Middleware](./03-response-filters/index.md)** - использование в GlobalExceptionFilter и LoggerMiddleware
- **[Логирование и метрики](./09-logging-metrics/index.md)** - отправка уведомлений о метриках (опционально)

## Структура использования в проекте

```
core/telegram/                  # Глобальный Telegram модуль
├── telegram.service.ts         # TelegramService
├── telegram.module.ts          # TelegramModule
├── telegram.controller.ts      # API endpoints (опционально)
├── telegram.dto.ts             # DTO для API
└── index.ts

core/filters/                   # Использует TelegramService
└── global-exception.filter.ts

core/middleware/                # Использует TelegramService
└── logger.middleware.ts
```

## Примечания

- `TelegramService` является глобальным и доступен во всех модулях
- Все методы асинхронные и не блокируют выполнение
- Ошибки отправки логируются, но не прерывают работу приложения
- Сообщения автоматически очищаются от Markdown символов и ограничиваются по длине
- Для production рекомендуется настроить rate limiting для предотвращения превышения лимитов Telegram API
