# Mail Module - Модуль отправки email

## Описание

Модуль для отправки email сообщений с использованием React компонентов в качестве шаблонов. Все письма отправляются асинхронно через очередь задач (BullMQ). Используется для подтверждения email, восстановления паролей и других email уведомлений.

## Архитектурные принципы

### Ключевые принципы

1. **Обязательная отправка через очередь** - все письма отправляются асинхронно через BullMQ
2. **React Email шаблоны** - использование `@react-email/components` для создания шаблонов
3. **EventBus интеграция** - публикация событий после успешной отправки
4. **Типизация** - строгая типизация всех типов писем и данных
5. **Мультиязычность** - поддержка нескольких языков (ru, en, es)

## Структура модуля

```
mail/
├── api/
│   ├── controllers/
│   │   └── mail.controller.ts      # REST API для тестирования отправки
│   └── dto/
│       └── mail.dto.ts            # DTO для отправки писем
├── application/
│   └── services/
│       └── mail.service.ts        # Бизнес-логика отправки писем
├── domain/
│   └── interfaces/
│       └── mail.interface.ts      # Интерфейсы для типов писем
├── infrastructure/
│   └── processors/
│       └── mail.processor.ts      # Worker для обработки очереди
├── templates/                      # React Email шаблоны
│   ├── email-verification.template.tsx
│   ├── reset-password.template.tsx
│   └── ...
├── lib/
│   └── utils/
│       └── template.util.ts       # Утилиты для работы с шаблонами
├── consts/
│   ├── mail.constants.ts          # Константы модуля
│   └── verification-email.ts      # Тексты для verification email
├── events/
│   └── mail-events.constants.ts   # Константы событий
├── mail.module.ts
└── index.ts
```

## Задачи

### ✅ Mail Service
**Статус**: Завершено
**Файл**: [mail-service.md](./mail-service.md)
**Описание**: Сервис для отправки email через очередь

### ✅ React Email Templates
**Статус**: Завершено
**Файл**: [react-email-templates.md](./react-email-templates.md)
**Описание**: Создание React компонентов для email шаблонов

### ✅ Email Confirmation
**Статус**: Завершено
**Файл**: [email-confirmation.md](./email-confirmation.md)
**Описание**: Отправка писем для подтверждения email

### ✅ Password Recovery
**Статус**: Завершено
**Файл**: [password-recovery.md](./password-recovery.md)
**Описание**: Отправка писем для восстановления пароля

### ✅ Queue Integration
**Статус**: Завершено
**Файл**: [queue-integration.md](./queue-integration.md)
**Описание**: Интеграция с очередями для асинхронной отправки

### ✅ EventBus Integration
**Статус**: Завершено
**Файл**: [eventbus-integration.md](./eventbus-integration.md)
**Описание**: Публикация событий после отправки писем

## Ключевые концепции

- **Обязательная отправка через очередь** - все письма отправляются асинхронно
- **React Email компоненты** - использование `@react-email/components` для шаблонов
- **Рендеринг в HTML** - React компоненты рендерятся в HTML через `render()`
- **Мультиязычность** - поддержка нескольких языков
- **Типизация** - строгая типизация всех типов писем
- **EventBus** - публикация событий после успешной отправки

## Ссылки

- [Mail Service](./mail-service.md) - сервис для отправки email
- [React Email Templates](./react-email-templates.md) - создание шаблонов
- [Email Confirmation](./email-confirmation.md) - подтверждение email
- [Password Recovery](./password-recovery.md) - восстановление пароля
- [Queue Integration](./queue-integration.md) - интеграция с очередями
- [EventBus Integration](./eventbus-integration.md) - публикация событий
