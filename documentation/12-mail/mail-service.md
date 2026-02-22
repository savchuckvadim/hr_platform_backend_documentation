# Mail Service - Сервис отправки email

## Обзор

Сервис для отправки email сообщений. Все письма отправляются асинхронно через очередь задач (BullMQ). Сервис рендерит React компоненты в HTML и добавляет задачи в очередь.

## Расположение

```
mail/application/services/mail.service.ts
```

## Реализация

### MailService

```typescript
import { ISendMailOptions, MailerService } from '@nestjs-modules/mailer';
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { render } from '@react-email/components';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import { EmailVerificationTemplate } from '../../templates/email-verification.template';
import { ResetPasswordTemplate } from '../../templates/reset-password.template';
import {
    MAIL_QUEUE_NAME,
    MAIL_QUEUE_JOB_NAMES,
    EmailType
} from '../../events/mail-events.constants';
import {
    EMAIL_SUBJECTS,
    DEFAULT_EMAIL_FROM,
    DEFAULT_EMAIL_FROM_NAME,
    DEFAULT_LANGUAGE,
    JOB_OPTIONS,
    SupportedLanguage
} from '../../consts/mail.constants';

@Injectable()
export class MailService {
    private readonly logger = new Logger(MailService.name);
    private readonly smtpFrom: string;
    private readonly smtpFromName: string;
    private readonly authCookieSpaDomain: string;
    private readonly siteUrl: string;

    constructor(
        private readonly mailerService: MailerService,
        @InjectQueue(MAIL_QUEUE_NAME) private readonly queue: Queue,
        private readonly configService: ConfigService,
    ) {
        this.smtpFrom = this.configService.get<string>('SMTP_FROM') || DEFAULT_EMAIL_FROM;
        this.smtpFromName = this.configService.get<string>('SMTP_FROM_NAME') || DEFAULT_EMAIL_FROM_NAME;
        this.authCookieSpaDomain = this.configService.get<string>('AUTH_COOKIE_SPA_DOMAIN') || '';
        this.siteUrl = this.configService.get<string>('SITE_URL') || '';
    }

    /**
     * Отправка письма для подтверждения email
     * @param user Пользователь
     * @param token Токен подтверждения
     * @param language Язык письма
     */
    public async sendEmailVerification(
        user: { id: string; email: string; firstName?: string; lastName?: string },
        token: string,
        language: SupportedLanguage = DEFAULT_LANGUAGE,
    ) {
        const baseUrl = this.authCookieSpaDomain
            ? `https://${this.authCookieSpaDomain}`
            : this.siteUrl || '';

        // ✅ Рендеринг React компонента в HTML
        const html = await render(
            EmailVerificationTemplate({
                name: user.firstName || 'User',
                surname: user.lastName || null,
                token,
                language: language,
                baseUrl,
            }),
        );

        // ✅ Обязательная отправка через очередь
        await this.queue.add(
            MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
            {
                to: [user.email],
                subject: EMAIL_SUBJECTS.VERIFICATION[language],
                html,
                context: {
                    name: user.firstName || user.email,
                },
                emailType: EmailType.VERIFICATION,
            },
            {
                removeOnComplete: JOB_OPTIONS.REMOVE_ON_COMPLETE,
                removeOnFail: JOB_OPTIONS.REMOVE_ON_FAIL,
            },
        );

        this.logger.log(`📬 Email verification queued for ${user.email}`);
        return true;
    }

    /**
     * Отправка письма для восстановления пароля
     * @param user Пользователь
     * @param token Токен восстановления
     * @param language Язык письма
     */
    public async sendPasswordReset(
        user: { id: string; email: string; firstName?: string; lastName?: string },
        token: string,
        language: SupportedLanguage = DEFAULT_LANGUAGE,
    ) {
        const baseUrl = this.siteUrl || (this.authCookieSpaDomain ? `https://${this.authCookieSpaDomain}` : '');

        // ✅ Рендеринг React компонента в HTML
        const html = await render(ResetPasswordTemplate({
            user: { id: user.id, email: user.email } as any,
            name: user.firstName || 'User',
            surname: user.lastName || '',
            token,
            baseUrl,
            language,
        }));

        // ✅ Обязательная отправка через очередь
        await this.queue.add(
            MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
            {
                to: [user.email],
                subject: EMAIL_SUBJECTS.PASSWORD_RESET[language],
                html,
                context: {
                    name: user.firstName || user.email,
                },
                emailType: EmailType.PASSWORD_RESET,
            },
            {
                removeOnComplete: JOB_OPTIONS.REMOVE_ON_COMPLETE,
                removeOnFail: JOB_OPTIONS.REMOVE_ON_FAIL,
            },
        );

        this.logger.log(`📬 Password reset email queued for ${user.email}`);
        return true;
    }

    /**
     * Низкоуровневый метод для отправки email
     * Вызывается из MailProcessor после обработки очереди
     * @param params Параметры отправки
     */
    async sendEmail(params: {
        subject: string;
        html: string;
        to: string[];
        context: ISendMailOptions['context'];
        attachments?: Array<{
            filename: string;
            content: Buffer;
            cid?: string;
            contentType: string;
        }>;
    }) {
        try {
            const from = `"${this.smtpFromName}" <${this.smtpFrom}>`;

            const emailsList: string[] = params.to;

            if (!emailsList || emailsList.length === 0) {
                throw new Error(
                    `No recipients found, please check your email addresses`,
                );
            }

            const sendMailParams: ISendMailOptions = {
                to: emailsList,
                from: from,
                subject: params.subject,
                html: params.html,
                attachments: params.attachments,
            };

            const response = await this.mailerService.sendMail(sendMailParams);

            this.logger.log(
                `Email sent successfully to ${emailsList.join(', ')}`,
            );

            return {
                ...response,
                message: 'Email sent successfully',
            };
        } catch (error) {
            this.logger.error(
                `Error while sending mail: ${error instanceof Error ? error.message : String(error)}`,
                error instanceof Error ? error.stack : undefined,
            );
            throw error;
        }
    }
}
```

## Методы

### sendEmailVerification

Отправка письма для подтверждения email адреса.

**Параметры:**
- `user` - объект пользователя с email, firstName, lastName
- `token` - токен подтверждения
- `language` - язык письма (по умолчанию 'en')

**Возвращает:** `Promise<boolean>`

**Пример:**
```typescript
await mailService.sendEmailVerification(
    {
        id: 'user-123',
        email: 'user@example.com',
        firstName: 'Иван',
        lastName: 'Иванов',
    },
    'verification-token-123',
    'ru',
);
```

### sendPasswordReset

Отправка письма для восстановления пароля.

**Параметры:**
- `user` - объект пользователя с email, firstName, lastName
- `token` - токен восстановления
- `language` - язык письма (по умолчанию 'en')

**Возвращает:** `Promise<boolean>`

**Пример:**
```typescript
await mailService.sendPasswordReset(
    {
        id: 'user-123',
        email: 'user@example.com',
        firstName: 'Иван',
        lastName: 'Иванов',
    },
    'reset-token-123',
    'ru',
);
```

### sendEmail

Низкоуровневый метод для отправки email. Вызывается из MailProcessor после обработки очереди.

**Параметры:**
- `params.subject` - тема письма
- `params.html` - HTML содержимое
- `params.to` - массив email адресов получателей
- `params.context` - контекст для шаблона
- `params.attachments` - вложения (опционально)

**Возвращает:** `Promise<{ message: string }>`

## Принципы

1. **Всегда через очередь** - все письма отправляются асинхронно
2. **Рендеринг шаблонов** - React компоненты рендерятся в HTML через `render()`
3. **Типизация** - строгая типизация всех параметров
4. **Логирование** - все операции логируются
5. **Обработка ошибок** - ошибки логируются и пробрасываются дальше

## Best Practices

1. **Используйте типизированные методы** - `sendEmailVerification`, `sendPasswordReset`
2. **Всегда указывайте язык** - для мультиязычности
3. **Проверяйте email адреса** - перед добавлением в очередь
4. **Логируйте операции** - для отладки и мониторинга
5. **Не вызывайте sendEmail напрямую** - только через очередь
