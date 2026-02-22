# Тестирование модулей

## Описание

Каждый модуль должен быть покрыт тестами. Тесты должны покрывать все слои модуля: domain, application, infrastructure.

## Структура тестов

### Расположение

```
module-name/
├── __tests__/
│   ├── domain/
│   │   └── entity.spec.ts
│   ├── application/
│   │   ├── services/
│   │   │   └── service.spec.ts
│   │   └── use-cases/
│   │       └── use-case.spec.ts
│   ├── infrastructure/
│   │   ├── repositories/
│   │   │   └── repository.spec.ts
│   │   └── processors/
│   │       └── processor.spec.ts
│   └── api/
│       └── controllers/
│           └── controller.spec.ts
```

## Примеры тестов

### Use Case тест

```typescript
import { CreateApplicationUseCase } from '@application/application/use-cases/create-application.use-case';
import { ApplicationService } from '@application/application/services/application.service';
import { AppEvent } from '@core/events/events.types';
import { Test, TestingModule } from '@nestjs/testing';

describe('CreateApplicationUseCase', () => {
  let useCase: CreateApplicationUseCase;
  const mockApplicationService = {
    create: jest.fn(),
  };
  const mockEventBus = {
    emit: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CreateApplicationUseCase,
        {
          provide: ApplicationService,
          useValue: mockApplicationService,
        },
        {
          provide: 'AppEventBus',
          useValue: mockEventBus,
        },
      ],
    }).compile();

    useCase = module.get<CreateApplicationUseCase>(CreateApplicationUseCase);
    jest.clearAllMocks();
  });

  describe('execute', () => {
    it('should create application and publish event', async () => {
      const dto = { vacancyId: 'vacancy-123', resumeId: 'resume-123' };
      const candidateId = 'candidate-123';
      const mockApplication = {
        id: 'application-123',
        candidateId,
        vacancyId: 'vacancy-123',
        status: 'NEW',
      };

      mockApplicationService.create.mockResolvedValue(mockApplication);
      mockEventBus.emit.mockResolvedValue(undefined);

      const result = await useCase.execute(dto, candidateId);

      expect(mockApplicationService.create).toHaveBeenCalledWith(dto, candidateId);
      expect(mockEventBus.emit).toHaveBeenCalledWith(
        AppEvent.APPLICATION_CREATED,
        expect.objectContaining({
          applicationId: mockApplication.id,
          candidateId,
          vacancyId: dto.vacancyId,
        }),
      );
      expect(result).toEqual(mockApplication);
    });

    it('should handle service errors', async () => {
      mockApplicationService.create.mockRejectedValue(new Error('Service error'));

      await expect(
        useCase.execute({ vacancyId: 'vacancy-123' }, 'candidate-123'),
      ).rejects.toThrow('Service error');
      expect(mockEventBus.emit).not.toHaveBeenCalled();
    });
  });
});
```

### Service тест

```typescript
import { ApplicationService } from '@application/application/services/application.service';
import { ApplicationRepository } from '@application/infrastructure/repositories/application.repository';
import { Test, TestingModule } from '@nestjs/testing';

describe('ApplicationService', () => {
  let service: ApplicationService;
  const mockRepository = {
    create: jest.fn(),
    findById: jest.fn(),
    findByCandidateId: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ApplicationService,
        {
          provide: ApplicationRepository,
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<ApplicationService>(ApplicationService);
    jest.clearAllMocks();
  });

  describe('create', () => {
    it('should create application', async () => {
      const dto = { vacancyId: 'vacancy-123', resumeId: 'resume-123' };
      const candidateId = 'candidate-123';
      const mockApplication = {
        id: 'application-123',
        candidateId,
        vacancyId: 'vacancy-123',
        status: 'NEW',
      };

      mockRepository.create.mockResolvedValue(mockApplication);

      const result = await service.create(dto, candidateId);

      expect(mockRepository.create).toHaveBeenCalledWith(
        expect.objectContaining({
          candidateId,
          vacancyId: dto.vacancyId,
        }),
      );
      expect(result).toEqual(mockApplication);
    });
  });
});
```

### Controller тест

```typescript
import { ApplicationsController } from '@application/api/controllers/applications.controller';
import { CreateApplicationUseCase } from '@application/application/use-cases/create-application.use-case';
import { Test, TestingModule } from '@nestjs/testing';

describe('ApplicationsController', () => {
  let controller: ApplicationsController;
  const mockUseCase = {
    execute: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [ApplicationsController],
      providers: [
        {
          provide: CreateApplicationUseCase,
          useValue: mockUseCase,
        },
      ],
    }).compile();

    controller = module.get<ApplicationsController>(ApplicationsController);
    jest.clearAllMocks();
  });

  describe('create', () => {
    it('should create application', async () => {
      const dto = { vacancyId: 'vacancy-123', resumeId: 'resume-123' };
      const candidateId = 'candidate-123';
      const mockApplication = {
        id: 'application-123',
        candidateId,
        vacancyId: 'vacancy-123',
        status: 'NEW',
      };

      mockUseCase.execute.mockResolvedValue(mockApplication);

      const result = await controller.create(dto, { userId: candidateId } as any);

      expect(mockUseCase.execute).toHaveBeenCalledWith(dto, candidateId);
      expect(result).toEqual(mockApplication);
    });
  });
});
```

## Принципы тестирования

1. **Изоляция**: Каждый тест изолирован, использует моки
2. **Покрытие**: Тестируются все публичные методы
3. **Граничные случаи**: Тестируются успешные и ошибочные сценарии
4. **Читаемость**: Тесты должны быть понятными и самодокументируемыми
5. **Скорость**: Тесты должны выполняться быстро

## Покрытие

Каждый модуль должен иметь:
- ✅ Тесты для Use Cases
- ✅ Тесты для Services
- ✅ Тесты для Controllers
- ✅ Тесты для Processors (если есть)
- ✅ Тесты для Repositories (интеграционные)

## Пример структуры тестов модуля

См. примеры тестов в существующих модулях:
- Use Case тесты - проверка координации сервиса и публикации событий
- Service тесты - проверка бизнес-логики
- Controller тесты - проверка HTTP endpoints
- Processor тесты - проверка обработки очередей (если есть)
