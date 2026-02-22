# Queue Processors - Обработчики задач

## Описание

Processors (обработчики) для очередей задач. **Важно:** Worker должен только обрабатывать задачу из очереди и эмитить бизнес-события. Вся бизнес-логика в сервисах, дополнительная логика через EventBus.

## Принципы

### ❌ Плохо: Бизнес-логика в Processor

```typescript
@Processor(MAIL_QUEUE_NAME)
export class MailProcessor extends WorkerHost {
  async process(job: Job<SendEmailJobData>): Promise<void> {
    const { to, subject, html } = job.data;

    // ❌ Бизнес-логика прямо в processor
    try {
      await this.smtpClient.send({
        to,
        subject,
        html,
      });
      this.logger.log(`Email sent to ${to.join(', ')}`);
    } catch (error) {
      this.logger.error(`Error: ${error.message}`);
      throw error;
    }

    // ❌ Дополнительная логика в processor
    if (emailType === EmailType.VERIFICATION) {
      await this.telegramService.sendEmailVerificationNotification({
        email: to[0],
        subject,
      });
    }
  }

  // ❌ Бизнес-логика в @OnWorkerEvent
  @OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
  async onCompleted(job: Job<SendEmailJobData>) {
    const { emailType } = job.data;
    // ❌ Запуск бизнес-workflow в worker event
    if (emailType === EmailType.VERIFICATION) {
      await this.telegramService.sendEmailVerificationNotification(...);
    }
  }
}
```

### ✅ Хорошо: Правильная архитектура

#### 1. Worker - только обработка очереди

```typescript
@Processor(MAIL_QUEUE_NAME)
export class MailProcessor extends WorkerHost {
  constructor(
    private readonly mailService: MailService,
    private readonly eventBus: AppEventBus,
  ) {
    super();
  }

  async process(job: Job<SendEmailJobData>): Promise<void> {
    const { to, subject, html, context, attachments } = job.data;

    // ✅ Только вызов сервиса для отправки email
    await this.mailService.sendEmail({
      subject,
      html,
      to,
      context: context || {},
      attachments,
    });
  }

  // ✅ @OnWorkerEvent - инфраструктурные вещи + бизнес-события
  @OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
  async onCompleted(job: Job<SendEmailJobData>) {
    // ✅ Логирование, метрики, мониторинг
    this.logger.log(`Email job completed: ${job.id}`);
    this.metrics.increment('email.sent');

    // ✅ Эмитим бизнес-событие после успешного завершения задачи
    this.eventBus.emit(AppEvent.EMAIL_SENT, {
      emailType: job.data.emailType,
      to: job.data.to,
      subject: job.data.subject,
    });
  }

  @OnWorkerEvent(MAIL_WORKER_EVENTS.FAILED)
  async onFailed(job: Job<SendEmailJobData>, error: Error) {
    // ✅ Логирование, метрики, алерты
    this.logger.error(`Email job failed: ${job.id}`, error.stack);
    this.metrics.increment('email.failed');
    this.sentry.captureException(error);

    // ✅ Может триггерить бизнес-событие для обработки ошибок
    // Но не содержит бизнес-логику напрямую
    this.eventBus.emit(AppEvent.EMAIL_SEND_FAILED, {
      email: job.data.to[0],
      error: error.message,
      jobId: job.id,
    });
  }
}
```

#### 2. Бизнес-событие через EventBus

После успешного завершения задачи (в `@OnWorkerEvent('completed')`) эмитится бизнес-событие:

```typescript
// В MailProcessor в @OnWorkerEvent('completed')
@OnWorkerEvent(MAIL_WORKER_EVENTS.COMPLETED)
async onCompleted(job: Job<SendEmailJobData>) {
  // Инфраструктурные вещи
  this.logger.log(`Email job completed: ${job.id}`);
  this.metrics.increment('email.sent');

  // ✅ Бизнес-событие после успешного завершения
  this.eventBus.emit(AppEvent.EMAIL_SENT, {
    emailType: job.data.emailType,
    to: job.data.to,
    subject: job.data.subject,
  });
}
```

Событие определено в `core/events/events.types.ts`:

```typescript
export enum AppEvent {
  EMAIL_SENT = 'email.sent',
  // ...
}

export interface EventPayloadMap {
  [AppEvent.EMAIL_SENT]: {
    emailType: EmailType;
    to: string[];
    subject: string;
  };
  // ...
}
```

#### 3. Отдельный Listener для бизнес-логики

```typescript
@Injectable()
export class EmailNotificationListener {
  constructor(
    private readonly telegramService: TelegramService,
  ) {}

  // ✅ Подписка на бизнес-событие через декоратор или onModuleInit
  @OnAppEvent(AppEvent.EMAIL_SENT)
  async handleEmailSent(payload: EventPayloadMap[AppEvent.EMAIL_SENT]) {
    // ✅ Бизнес-логика в отдельном listener
    if (payload.emailType === EmailType.VERIFICATION) {
      await this.telegramService.sendEmailVerificationNotification({
        email: payload.to[0],
        subject: payload.subject,
      });
    }
  }
}
```

## Разделение событий

### 🟢 Domain Events (бизнес-события)

Это ваши бизнес-события через `AppEventBus`:

- `USER_CREATED`
- `EMAIL_SENT`
- `APPLICATION_CREATED`
- `PAYMENT_CONFIRMED`

**Использование:**
- Публикация после бизнес-операций
- Подписка через listeners
- Обработка бизнес-логики

### 🔵 Infrastructure Events (Bull Worker Events)

Это события инфраструктуры от Bull:

- `completed` - задача завершена
- `failed` - задача провалилась
- `stalled` - задача зависла
- `progress` - прогресс выполнения

**Использование:**
- ✅ Логирование
- ✅ Метрики
- ✅ Retry-контроль
- ✅ Мониторинг
- ✅ Алерты
- ✅ Эмит бизнес-событий (после успешного завершения или ошибки)
- ❌ НЕ для прямой бизнес-логики
- ❌ НЕ для изменения состояния домена напрямую
- ❌ НЕ для запуска бизнес-workflow напрямую

## Правильная архитектура

### Поток обработки

```mermaid
graph TD
    UserService[UserService.createUser]
    UserService -->|emit| USER_CREATED[USER_CREATED Event]
    USER_CREATED -->|subscribe| EmailListener[EmailListener]
    EmailListener -->|queue.add| MailQueue[Mail Queue]
    MailQueue -->|process| MailProcessor[MailProcessor.process]
    MailProcessor -->|call| MailService[MailService.sendEmail]
    MailService -->|success| WorkerCompleted[@OnWorkerEvent completed]
    WorkerCompleted -->|emit| EmitEmailSent[emit EMAIL_SENT]
    EmitEmailSent -->|subscribe| TelegramListener[TelegramListener]
    TelegramListener -->|call| TelegramService[TelegramService]

    WorkerCompleted -->|also| LogMetrics[Логирование, метрики]
    MailProcessor -->|@OnWorkerEvent failed| WorkerFailed[Логирование, метрики, emit EMAIL_SEND_FAILED]
```

**Описание потока:**
1. `UserService` создает пользователя и эмитит `USER_CREATED`
2. `EmailListener` подписан на событие, отправляет задачу в очередь (тяжелая операция)
3. `MailProcessor.process()` обрабатывает задачу из очереди, вызывает `MailService.sendEmail()`
4. После успешного завершения задачи срабатывает `@OnWorkerEvent('completed')`
5. В `@OnWorkerEvent('completed')` выполняется:
   - Логирование и метрики (инфраструктурные вещи)
   - Эмит бизнес-события `EMAIL_SENT`
6. `TelegramListener` обрабатывает событие `EMAIL_SENT`, отправляет в Telegram

### Разделение ответственности

1. **MailProcessor.process()** - только обработка очереди, вызов сервиса
2. **MailProcessor.@OnWorkerEvent('completed')** - логирование, метрики, эмит бизнес-события
3. **MailService** - бизнес-логика отправки email
4. **EventBus** - публикация бизнес-событий
5. **EmailNotificationListener** - обработка бизнес-событий (Telegram и т.д.)

## Структура сервисов

### MailService

```typescript
@Injectable()
export class MailService {
  constructor(
    private readonly smtpClient: SmtpClient,
    private readonly logger: Logger,
  ) {}

  async sendEmail(data: SendEmailDto): Promise<void> {
    try {
      await this.smtpClient.send({
        to: data.to,
        subject: data.subject,
        html: data.html,
        context: data.context,
        attachments: data.attachments,
      });

      this.logger.log(`Email sent to ${data.to.join(', ')}`);
    } catch (error) {
      this.logger.error(
        `Error sending email: ${error instanceof Error ? error.message : String(error)}`,
        error instanceof Error ? error.stack : undefined,
      );
      throw error;
    }
  }
}
```

## Listeners и очереди

### Когда listener отправляет в очередь?

Listeners на глобальные события могут:
1. **Выполнять логику сразу** (синхронно) - для быстрых операций
2. **Отправлять в очередь** - для тяжелых операций (когда нужен retry, долгое выполнение)

**Примеры тяжелых операций:**
- Отправка email
- Генерация отчетов
- Обработка файлов
- Интеграция с внешними API

**Пример listener который отправляет в очередь:**

```typescript
@Injectable()
export class UserCreatedListener {
  constructor(
    @InjectQueue(MAIL_QUEUE_NAME) private readonly mailQueue: Queue,
  ) {}

  @OnAppEvent(AppEvent.USER_CREATED)
  async handleUserCreated(payload: EventPayloadMap[AppEvent.USER_CREATED]) {
    // ✅ Прямое добавление задачи в очередь (без dispatcher)
    await this.mailQueue.add(
      MAIL_QUEUE_JOB_NAMES.SEND_EMAIL,
      {
        to: [payload.email],
        subject: 'Welcome!',
        html: '<h1>Welcome to HR Platform</h1>',
        emailType: EmailType.WELCOME,
      },
    );
  }
}
```


## Преимущества подхода

1. **Разделение ответственности**: Worker не знает про Telegram, Telegram не знает про Bull
2. **Тестируемость**: Каждый компонент тестируется изолированно
3. **Переиспользование**: Логику можно использовать не только из очереди
4. **Чистота кода**: Worker только обрабатывает очередь, бизнес-логика через события
5. **Масштабируемость**: Легко добавлять новые обработчики событий
6. **Гибкость**: Listeners могут выбирать - выполнить сразу или отправить в очередь

## Пример полного модуля

```
mail/
├── application/
│   └── services/
│       └── mail.service.ts          # Бизнес-логика отправки
├── infrastructure/
│   ├── processors/
│   │   └── mail.processor.ts        # Worker - только обработка очереди
│   └── listeners/
│       └── email-notification.listener.ts  # Обработка EMAIL_SENT события
└── mail.module.ts
```

## Ссылки

- [Core Events Module](../../15-events/index.md) - централизованная система событий
- [Event Decorators](../../15-events/event-decorators.md) - декораторы для подписки на события
