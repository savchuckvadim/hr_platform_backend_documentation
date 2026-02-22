# Presence Module - Модуль присутствия

## Описание

Модуль для отслеживания статусов присутствия пользователей (онлайн/оффлайн) в реальном времени. Использует Redis с TTL для автоматического определения оффлайн статуса и WebSocket для real-time обновлений.

## Архитектурные принципы

### Ключевые принципы

1. **Redis TTL** — автоматическое определение оффлайн статуса через истечение ключей
2. **Heartbeat механизм** — клиент отправляет ping для поддержания онлайн статуса
3. **EventBus интеграция** — события публикуются через EventBus, отправка через WebSocket listeners
4. **Multi-node поддержка** — работает с несколькими инстансами через Redis
5. **Таргетированная рассылка** — только заинтересованным пользователям

## Структура модуля

```
presence/
├── api/
│   └── controllers/
│       └── presence.controller.ts      # REST API для получения статусов
├── application/
│   └── services/
│       └── presence.service.ts         # Бизнес-логика presence
├── infrastructure/
│   ├── repositories/
│   │   └── presence.repository.ts     # Работа с Redis
│   └── listeners/
│       └── presence-websocket.listener.ts  # WebSocket listeners
├── domain/
│   └── interfaces/
│       └── presence.interface.ts      # Интерфейсы
├── presence.module.ts
└── index.ts
```

## Задачи

### ✅ Presence Service
**Статус**: Завершено
**Файл**: [presence-service.md](./presence-service.md)
**Описание**: Сервис для управления статусами присутствия

### ✅ Redis Storage
**Статус**: Завершено
**Файл**: [redis-storage.md](./redis-storage.md)
**Описание**: Хранение статусов в Redis с TTL и keyspace events

### ✅ WebSocket Integration
**Статус**: Завершено
**Файл**: [websocket-integration.md](./websocket-integration.md)
**Описание**: Интеграция с WebSocket для real-time обновлений

### ✅ Presence API
**Статус**: Завершено
**Файл**: [presence-api.md](./presence-api.md)
**Описание**: REST API endpoints для получения статусов

## Ключевые концепции

- **TTL механизм** — автоматическое определение оффлайн через истечение ключей Redis
- **Heartbeat** — клиент отправляет ping каждые 25 секунд
- **Keyspace events** — подписка на события истечения ключей
- **Multi-node** — синхронизация между инстансами через Redis
- **EventBus** — события публикуются через EventBus, отправка через listeners

## Ссылки

- [Presence Service](./presence-service.md) - сервис для управления статусами
- [Redis Storage](./redis-storage.md) - хранение в Redis с TTL
- [WebSocket Integration](./websocket-integration.md) - real-time обновления
- [Presence API](./presence-api.md) - REST API endpoints
