# Storage Module - Модуль хранения файлов

## Описание

Модуль для работы с хранением файлов через AWS S3. Обеспечивает загрузку, удаление и получение файлов для HR Platform. Storage модуль использует S3 сервис и предоставляет CRUD операции для документов с поддержкой различных путей через enums.

## Архитектурные принципы

### Ключевые принципы

1. **Storage модуль** - использует S3 сервис для хранения
2. **CRUD операции** - универсальные методы для работы с документами
3. **Enums путей** - централизованное управление путями файлов
4. **Валидация файлов** - утилиты для проверки типа и размера
5. **Не плодить методы** - один метод для всех типов файлов
6. **S3 не знает о типах** - только уровни доступа (PUBLIC, PRIVATE, APP)

## Структура модуля

### Storage Module

```
core/storage/
├── api/
│   ├── controllers/
│   │   └── storage.controller.ts
│   └── dto/
│       ├── upload-file.dto.ts
│       └── file.dto.ts
├── application/
│   └── services/
│       └── storage.service.ts
├── domain/
│   └── enums/
│       └── file-path.enum.ts
├── infrastructure/
│   └── utils/
│       ├── file-validation.util.ts
│       └── file-size.util.ts
├── storage.module.ts
└── index.ts
```

### Core S3 Module

```
core/s3/
├── s3.module.ts              # Модуль S3
├── s3.service.ts             # Сервис для работы с S3
├── enums/
│   └── access-level.enum.ts  # Уровни доступа (PUBLIC, PRIVATE, APP)
└── index.ts
```

## Задачи

### ✅ S3 Configuration
**Статус**: Завершено
**Файл**: [s3-configuration.md](./s3-configuration.md)
**Описание**: Настройка AWS S3 клиента и конфигурации

### ✅ Storage Module
**Статус**: Завершено
**Файл**: [storage-module.md](./storage-module.md)
**Описание**: Storage модуль с CRUD операциями и enums путей

### ✅ File Validation
**Статус**: Завершено
**Файл**: [file-validation.md](./file-validation.md)
**Описание**: Валидация типов и размеров файлов

### ✅ Integration Examples
**Статус**: Завершено
**Файл**: [integration-examples.md](./integration-examples.md)
**Описание**: Примеры интеграции в другие модули

## Ключевые концепции

- **S3 Service** - низкоуровневый сервис для работы с S3 (только уровни доступа)
- **Storage Service** - высокоуровневый сервис с CRUD операциями
- **File Path Enums** - централизованное управление путями
- **File Validation** - утилиты для проверки файлов
- **Уровни доступа** - PUBLIC, PRIVATE, APP
- **CRUD операции** - универсальные методы для всех типов файлов

## Ссылки

- [S3 Configuration](./s3-configuration.md) - настройка AWS S3
- [Storage Module](./storage-module.md) - Storage модуль с CRUD операциями
- [File Validation](./file-validation.md) - валидация файлов
- [Integration Examples](./integration-examples.md) - примеры интеграции
