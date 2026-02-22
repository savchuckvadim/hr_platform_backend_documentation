# Worker Events vs Domain Events

## Описание

Важное разделение между событиями инфраструктуры (Bull Worker Events) и бизнес-событиями (Domain Events).

## Разделение событий

### 🟢 Domain Events (бизнес-события)

Это ваши бизнес-события через `AppEventBus`:

- `USER_CREATED`
- `EMAIL_SENT`
- `APPLICATION_CREATED`
- `PAYMENT_CONFIRMED`
- `VACANCY_CREATED`

**Характеристики:**
- Отражают бизнес-события в домене
- Публикуются после успешных бизнес-операций
- Обрабатываются через listeners
- Могут быть обработаны асинхронно через очереди

**Использование:**

**Пример 1: Публикация из сервиса/use case (для легких операций)**
```typescript
// В сервисе после успешного создания пользователя
this.eventBus.emit(AppEvent.USER_CREATED, {
  userId: user.id,
  email: user.email,
  role: user.role,
});
```

**Пример 2: Публикация из Worker (для тяжелых операций)**
```typescript
// В @OnWorkerEvent('completed') после успешной отправки email
@OnWorkerEvent('completed')
async onCompleted(job: Job<SendEmailJobData>) {
  // Логирование, метрики
  this.logger.log(`Email job completed: ${job.id}`);

  // ✅ Эмитим бизнес-событие после успешного завершения
  this.eventBus.emit(AppEvent.EMAIL_SENT, {
    emailType: job.data.emailType,
    to: job.data.to,
    subject: job.data.subject,
  });
}
```

**Пример 3: Listener обрабатывает событие**
```typescript
@OnAppEvent(AppEvent.EMAIL_SENT)
async handleEmailSent(payload: EmailSentPayload) {
  // Бизнес-логика
}
```

### 🔵 Infrastructure Events (Bull Worker Events)

Это события инфраструктуры от Bull:

- `completed` - задача завершена
- `failed` - задача провалилась
- `stalled` - задача зависла
- `progress` - прогресс выполнения
- `active` - задача начала выполняться

**Характеристики:**
- События инфраструктуры очередей
- Автоматически генерируются Bull
- Связаны с жизненным циклом задач в очереди

**Использование:**
```typescript
@OnWorkerEvent('completed')
async onCompleted(job: Job) {
  // ✅ Логирование
  this.logger.log(`Job completed: ${job.id}`);

  // ✅ Метрики
  this.metrics.increment('jobs.completed');

  // ✅ Мониторинг
  this.monitoring.recordJobDuration(job);
}

@OnWorkerEvent('failed')
async onFailed(job: Job, error: Error) {
  // ✅ Логирование
  this.logger.error(`Job failed: ${job.id}`, error.stack);

  // ✅ Метрики
  this.metrics.increment('jobs.failed');

  // ✅ Алерты
  this.sentry.captureException(error);

  // ✅ Может триггерить бизнес-событие для обработки ошибок
  // Но не содержит бизнес-логику напрямую
  this.eventBus.emit(AppEvent.EMAIL_SEND_FAILED, {
    email: job.data.to[0],
    error: error.message,
    jobId: job.id,
  });
}
```

## ❌ Что НЕ делать

### НЕ использовать @OnWorkerEvent для бизнес-логики

```typescript
// ❌ Плохо: Бизнес-логика в worker event
@OnWorkerEvent('completed')
async onCompleted(job: Job<SendEmailJobData>) {
  const { emailType } = job.data;

  // ❌ Запуск бизнес-workflow
  if (emailType === EmailType.VERIFICATION) {
    await this.telegramService.sendEmailVerificationNotification(...);
  }

  // ❌ Изменение состояния домена
  await this.userService.markEmailAsSent(job.data.userId);
}
```

### ✅ Правильный подход

```typescript
// ✅ Worker только вызывает сервис
async process(job: Job<SendEmailJobData>): Promise<void> {
  await this.mailService.sendEmail(job.data);
}

// ✅ @OnWorkerEvent - инфраструктурные вещи + бизнес-события
@OnWorkerEvent('completed')
async onCompleted(job: Job<SendEmailJobData>) {
  // ✅ Логирование, метрики
  this.logger.log(`Email job completed: ${job.id}`);
  this.metrics.increment('email.sent');

  // ✅ Эмитим бизнес-событие после успешного завершения
  this.eventBus.emit(AppEvent.EMAIL_SENT, {
    emailType: job.data.emailType,
    to: job.data.to,
    subject: job.data.subject,
  });
}

// ✅ Отдельный listener для бизнес-логики
@OnAppEvent(AppEvent.EMAIL_SENT)
async handleEmailSent(payload: EmailSentPayload) {
  if (payload.emailType === EmailType.VERIFICATION) {
    await this.telegramService.sendEmailVerificationNotification(...);
  }
}
```

## Правила

### ✅ Worker Events - для:

- Логирования выполнения задач
- Сбора метрик (успешные/провалившиеся задачи)
- Мониторинга производительности
- Retry-контроля
- Алертов при ошибках
- Отправки в Sentry/мониторинг системы
- **Триггера бизнес-событий** (но не содержат бизнес-логику напрямую)

**Пример триггера бизнес-события:**
```typescript
@OnWorkerEvent('completed')
async onCompleted(job: Job<SendEmailJobData>) {
  // ✅ Инфраструктурные вещи
  this.logger.log(`Job completed: ${job.id}`);
  this.metrics.increment('jobs.completed');

  // ✅ Эмитим бизнес-событие после успешного завершения
  this.eventBus.emit(AppEvent.EMAIL_SENT, {
    emailType: job.data.emailType,
    to: job.data.to,
    subject: job.data.subject,
  });
}

@OnWorkerEvent('failed')
async onFailed(job: Job, error: Error) {
  // ✅ Инфраструктурные вещи
  this.logger.error(`Job failed: ${job.id}`, error.stack);
  this.metrics.increment('jobs.failed');

  // ✅ Может триггерить бизнес-событие для обработки ошибок
  this.eventBus.emit(AppEvent.EMAIL_SEND_FAILED, {
    email: job.data.to[0],
    error: error.message,
    jobId: job.id,
  });
}
```

### ❌ Worker Events - НЕ для:

- Запуска бизнес-workflow напрямую
- Отправки уведомлений напрямую
- Изменения состояния домена напрямую
- Вызова других сервисов для бизнес-логики напрямую
- Содержания бизнес-логики (используйте Domain Events и Listeners)

### ✅ Domain Events - для:

- Отражения бизнес-событий
- Запуска бизнес-workflow
- Отправки уведомлений
- Изменения состояния домена
- Координации между модулями

## Пример правильной архитектуры

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

**Поток:**
1. `UserService` создает пользователя и эмитит `USER_CREATED`
2. `EmailListener` подписан на событие, отправляет задачу в очередь
3. `MailProcessor.process()` обрабатывает задачу из очереди, вызывает `MailService.sendEmail()`
4. После успешного завершения задачи срабатывает `@OnWorkerEvent('completed')`
5. В `@OnWorkerEvent('completed')` выполняется:
   - Логирование и метрики (инфраструктурные вещи)
   - Эмит бизнес-события `EMAIL_SENT`
6. `TelegramListener` обрабатывает событие `EMAIL_SENT`, отправляет в Telegram

## Ссылки

- [Queue Processors](../../08-queue/queue-processors.md) - обработчики очередей
- [Event Bus Service](./event-bus-service.md) - типизированный EventBus
- [Event Decorators](./event-decorators.md) - декораторы для подписки
