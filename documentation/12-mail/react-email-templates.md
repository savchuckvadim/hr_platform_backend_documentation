# React Email Templates - React Email шаблоны

## Обзор

Использование библиотеки `@react-email/components` для создания email шаблонов в виде React компонентов. Компоненты рендерятся в HTML для отправки через SMTP.

## Библиотека

### Установка

```bash
npm install @react-email/components react
```

### Основные компоненты

Библиотека предоставляет набор компонентов для создания email шаблонов:

- `Html` - корневой HTML элемент
- `Head` - заголовок документа
- `Body` - тело письма
- `Container` - контейнер для контента
- `Section` - секция контента
- `Text` - текстовый блок
- `Heading` - заголовок
- `Button` - кнопка-ссылка
- `Img` - изображение
- `Tailwind` - поддержка Tailwind CSS
- `Font` - подключение шрифтов
- `Preview` - превью письма в почтовых клиентах

## Структура шаблона

### Базовый шаблон

```typescript
// templates/email-verification.template.tsx
import React from 'react';
import {
    Body,
    Button,
    Container,
    Font,
    Head,
    Heading,
    Html,
    Img,
    Preview,
    Section,
    Tailwind,
    Text,
} from '@react-email/components';

interface EmailVerificationTemplateProps {
    name: string;
    surname?: string | null;
    token: string;
    language?: 'ru' | 'en' | 'es';
    baseUrl: string;
}

export function EmailVerificationTemplate({
    name,
    surname = '',
    token,
    language = 'en',
    baseUrl,
}: EmailVerificationTemplateProps) {
    const logo = `${baseUrl}/touch-icons/512x512.png`;
    const verifyLink = `${baseUrl}/auth/confirm/${token}`;
    const texts = getVerificationEmailTexts(language);
    const currentYear = new Date().getFullYear();

    return (
        <Tailwind>
            <Html>
                <Head>
                    <Font
                        fontFamily="Geist"
                        fallbackFontFamily="Arial"
                        webFont={{
                            url: 'https://fonts.googleapis.com/css2?family=Geist:wght@300;500;700&display=swap',
                            format: 'woff2',
                        }}
                    />
                </Head>

                <Body style={{ backgroundColor: '#f8f9fa', fontFamily: 'Inter, Arial, sans-serif' }}>
                    <Preview>{texts.preview}</Preview>

                    <Container className="mx-auto my-10 max-w-[500px] rounded-lg bg-white p-8 shadow-lg">
                        <Section className="text-center">
                            <Img
                                src={logo}
                                width="100"
                                height="100"
                                alt="HR Platform"
                                className="mx-auto mb-4"
                            />

                            <Heading className="text-2xl font-bold text-blue-600">
                                {texts.heading}
                            </Heading>

                            <Text className="mb-6 text-gray-500">
                                {texts.greeting(name, surname)}
                            </Text>

                            <Section className="mb-8 rounded-lg border border-blue-100 bg-blue-50 p-6">
                                <Text className="mb-4 text-gray-800">
                                    {texts.instruction}
                                </Text>

                                <Button
                                    href={verifyLink}
                                    className="inline-flex items-center justify-center rounded-full bg-blue-600 px-8 py-3 text-sm font-medium text-white hover:bg-blue-600/90"
                                >
                                    {texts.buttonText}
                                </Button>
                            </Section>

                            <Text className="text-sm text-gray-500">
                                {texts.ignoreText}
                            </Text>

                            <Text className="mt-6 text-sm text-gray-400">
                                {texts.copyright(currentYear)}
                            </Text>
                        </Section>
                    </Container>
                </Body>
            </Html>
        </Tailwind>
    );
}
```

## Рендеринг шаблона

### Использование render()

```typescript
import { render } from '@react-email/components';
import { EmailVerificationTemplate } from './templates/email-verification.template';

// Рендеринг React компонента в HTML
const html = await render(
    EmailVerificationTemplate({
        name: 'Иван',
        surname: 'Иванов',
        token: 'verification-token-123',
        language: 'ru',
        baseUrl: 'https://hr-platform.com',
    }),
);
```

## Расположение шаблонов

```
mail/
└── templates/
    ├── email-verification.template.tsx
    ├── reset-password.template.tsx
    └── ...
```

## Мультиязычность

### Утилита для текстов

```typescript
// lib/utils/template.util.ts
import {
    Language,
    VerificationEmailTexts,
    VERIFICATION_EMAIL_TEXTS,
} from '../../consts/verification-email';
import { DEFAULT_LANGUAGE } from '../../consts/mail.constants';

export const getVerificationEmailTexts = (
    language: Language = DEFAULT_LANGUAGE,
): VerificationEmailTexts => {
    return VERIFICATION_EMAIL_TEXTS[language] || VERIFICATION_EMAIL_TEXTS[DEFAULT_LANGUAGE];
};
```

### Константы текстов

```typescript
// consts/verification-email.ts
export type Language = 'ru' | 'en' | 'es';

export interface VerificationEmailTexts {
    preview: string;
    heading: string;
    greeting: (name: string, surname?: string | null) => string;
    instruction: string;
    buttonText: string;
    ignoreText: string;
    copyright: (year: number) => string;
}

export const VERIFICATION_EMAIL_TEXTS: Record<Language, VerificationEmailTexts> = {
    ru: {
        preview: 'Подтвердите ваш email адрес',
        heading: 'Подтверждение email',
        greeting: (name, surname) => `Привет, ${name}${surname ? ` ${surname}` : ''}!`,
        instruction: 'Нажмите на кнопку ниже, чтобы подтвердить ваш email адрес.',
        buttonText: 'Подтвердить email',
        ignoreText: 'Если вы не регистрировались, просто проигнорируйте это письмо.',
        copyright: (year) => `© ${year} HR Platform. Все права защищены.`,
    },
    en: {
        preview: 'Confirm your email address',
        heading: 'Email Verification',
        greeting: (name, surname) => `Hello, ${name}${surname ? ` ${surname}` : ''}!`,
        instruction: 'Click the button below to verify your email address.',
        buttonText: 'Verify Email',
        ignoreText: 'If you did not sign up, please ignore this email.',
        copyright: (year) => `© ${year} HR Platform. All rights reserved.`,
    },
    es: {
        preview: 'Confirma tu dirección de correo electrónico',
        heading: 'Verificación de correo',
        greeting: (name, surname) => `¡Hola, ${name}${surname ? ` ${surname}` : ''}!`,
        instruction: 'Haz clic en el botón de abajo para verificar tu dirección de correo electrónico.',
        buttonText: 'Verificar correo',
        ignoreText: 'Si no te registraste, ignora este correo.',
        copyright: (year) => `© ${year} HR Platform. Todos los derechos reservados.`,
    },
};
```

## Best Practices

1. **Используйте Tailwind** - для стилизации компонентов
2. **Добавляйте Preview** - для превью в почтовых клиентах
3. **Используйте Font** - для подключения кастомных шрифтов
4. **Мультиязычность** - через утилиты и константы
5. **Типизация** - строгая типизация всех props
6. **Адаптивность** - учитывайте разные почтовые клиенты
7. **Тестирование** - проверяйте рендеринг в разных клиентах

## Ограничения

1. **Не все CSS поддерживается** - используйте inline стили и Tailwind
2. **Изображения** - должны быть доступны по URL или вложены
3. **JavaScript** - не поддерживается в email
4. **Шрифты** - только через webFont в компоненте Font
