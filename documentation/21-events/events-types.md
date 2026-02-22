# Events Types - Контракты событий

## Описание

Централизованное определение всех событий системы. Один источник правды для типов событий и их payload.

## Структура

### Файл: `src/core/events/events.types.ts`

```typescript
/**
 * Enum всех событий приложения
 * Централизованный источник правды
 */
export enum AppEvent {
  // User events
  USER_CREATED = 'user.created',
  USER_UPDATED = 'user.updated',
  USER_DELETED = 'user.deleted',

  // Candidate events
  CANDIDATE_PROFILE_CREATED = 'candidate.profile.created',
  CANDIDATE_PROFILE_UPDATED = 'candidate.profile.updated',
  RESUME_CREATED = 'resume.created',
  RESUME_UPDATED = 'resume.updated',
  RESUME_DELETED = 'resume.deleted',

  // Employer events
  EMPLOYER_PROFILE_CREATED = 'employer.profile.created',
  EMPLOYER_PROFILE_UPDATED = 'employer.profile.updated',
  VACANCY_CREATED = 'vacancy.created',
  VACANCY_UPDATED = 'vacancy.updated',
  VACANCY_DELETED = 'vacancy.deleted',

  // Application events
  APPLICATION_CREATED = 'application.created',
  APPLICATION_STATUS_CHANGED = 'application.status.changed',
  APPLICATION_VIEWED = 'application.viewed',

  // Presence events
  USER_ONLINE = 'user.online',
  USER_OFFLINE = 'user.offline',

  // Chat events
  MESSAGE_CREATED = 'message.created',
  MESSAGE_READ = 'message.read',
  CHAT_CREATED = 'chat.created',

  // Mail events
  EMAIL_SENT = 'email.sent',
  EMAIL_SEND_FAILED = 'email.send.failed',
}

/**
 * Типизированная мапа payload для каждого события
 * Гарантирует типобезопасность при публикации и подписке
 */
export interface EventPayloadMap {
  [AppEvent.USER_CREATED]: {
    userId: string;
    email: string;
    role: 'CANDIDATE' | 'EMPLOYER';
  };

  [AppEvent.USER_UPDATED]: {
    userId: string;
    changes: Record<string, any>;
  };

  [AppEvent.CANDIDATE_PROFILE_CREATED]: {
    userId: string;
    profileId: string;
    firstName?: string;
    lastName?: string;
  };

  [AppEvent.RESUME_CREATED]: {
    resumeId: string;
    candidateId: string;
    title: string;
  };

  [AppEvent.VACANCY_CREATED]: {
    vacancyId: string;
    employerId: string;
    title: string;
  };

  [AppEvent.APPLICATION_CREATED]: {
    applicationId: string;
    candidateId: string;
    vacancyId: string;
    resumeId?: string;
  };

  [AppEvent.APPLICATION_STATUS_CHANGED]: {
    applicationId: string;
    oldStatus: string;
    newStatus: string;
    changedBy: string;
  };

  [AppEvent.USER_ONLINE]: {
    userId: string;
    socketId: string;
    timestamp: Date;
  };

  [AppEvent.USER_OFFLINE]: {
    userId: string;
    timestamp: Date;
  };

  [AppEvent.MESSAGE_CREATED]: {
    messageId: string;
    chatId: string;
    userId: string;
    content: string;
  };

  [AppEvent.CHAT_CREATED]: {
    chatId: string;
    applicationId?: string;
    userIds: string[];
  };

  [AppEvent.EMAIL_SENT]: {
    emailType: string;
    to: string[];
    subject: string;
  };

  [AppEvent.EMAIL_SEND_FAILED]: {
    email: string;
    error: string;
    jobId?: string;
  };
}
```

## Принципы

1. **Один источник правды**: Все события определены в одном месте
2. **Типизация**: Каждое событие имеет строго типизированный payload
3. **Именование**: Используется dot notation (module.action)
4. **Расширяемость**: Легко добавлять новые события

## Использование

```typescript
import { AppEvent, EventPayloadMap } from '@core/events/events.types';

// Тип события автокомплитится
const event: AppEvent = AppEvent.USER_CREATED;

// Payload строго проверяется
const payload: EventPayloadMap[AppEvent.USER_CREATED] = {
  userId: '123',
  email: 'user@example.com',
  role: 'CANDIDATE',
};
```
