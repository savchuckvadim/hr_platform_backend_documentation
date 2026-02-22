# Swagger документация

## Описание

Требования к документации API через Swagger/OpenAPI.
Создание кастомных декораторов для упрощения документирования endpoints.
Интеграция с генерацией клиентского кода.

## Расположение

```
core/config/swagger/swagger.config.ts
core/decorators/api/
```

## Задачи

### ✅ Swagger Setup
**Статус**: Завершено
**Файл**: [swagger-setup.md](./swagger-setup.md)
**Описание**: Настройка Swagger в NestJS приложении

### ✅ Custom Decorators
**Статус**: Завершено
**Файл**: [custom-decorators.md](./custom-decorators.md)
**Описание**: Создание кастомных декораторов для API документации

### ✅ Response Documentation
**Статус**: Завершено
**Файл**: [response-documentation.md](./response-documentation.md)
**Описание**: Документирование ответов API (успешных и ошибок)

### ✅ DTO Documentation
**Статус**: Завершено
**Файл**: [dto-documentation.md](./dto-documentation.md)
**Описание**: Автоматическое документирование DTO через декораторы

## Ключевые концепции

- Автоматическая генерация Swagger документации
- Кастомные декораторы для упрощения документирования
- Документирование всех возможных ответов (включая ошибки)
- Примеры запросов и ответов
- Группировка endpoints по тегам
- Интеграция с генерацией TypeScript клиента

## Ссылки

- [Swagger Setup](./swagger-setup.md) - настройка Swagger
- [Custom Decorators](./custom-decorators.md) - кастомные декораторы
- [Response Documentation](./response-documentation.md) - документирование ответов
- [DTO Documentation](./dto-documentation.md) - документирование DTO
