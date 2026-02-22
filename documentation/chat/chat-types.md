# Chat Types - Типы чатов

## Обзор

Типы чатов на HR Platform и их использование в различных сценариях.

## Типы чатов

### DIRECT - Прямой чат

Прямой чат между двумя пользователями (кандидат ↔ кандидат, кандидат ↔ HR, HR ↔ HR).

**Характеристики:**
- Ровно 2 участника
- Не привязан к отклику
- Может быть создан вручную

**Пример использования:**
```typescript
// Создание прямого чата между кандидатом и HR
const chat = await chatsService.createChat(userId, {
    type: ChatType.DIRECT,
    memberIds: [otherUserId],
});
```

### APPLICATION - Чат по отклику

Чат, автоматически создаваемый при отклике кандидата на вакансию.

**Характеристики:**
- Привязан к Application
- Участники: кандидат и HR компании (root или нанятый)
- Автоматическое создание при отклике

**Пример использования:**
```typescript
// Автоматическое создание чата при отклике
// В ApplicationService после создания отклика
const chat = await chatsService.createChat(candidateId, {
    type: ChatType.APPLICATION,
    applicationId: application.id,
    memberIds: [hrUserId], // HR компании
});
```

### GROUP - Групповой чат

Групповой чат для нескольких участников (на будущее).

**Характеристики:**
- Более 2 участников
- Имеет название и описание
- Может иметь аватар

## Создание чатов

### Прямой чат

```typescript
// Кандидат создает чат с другим кандидатом
await chatsService.createChat(candidateId, {
    type: ChatType.DIRECT,
    memberIds: [otherCandidateId],
});
```

### Чат по отклику

```typescript
// Автоматическое создание при отклике
// В ApplicationService
async createApplication(candidateId: string, dto: CreateApplicationDto) {
    // Создаем отклик
    const application = await this.applicationRepository.create({
        candidateId,
        vacancyId: dto.vacancyId,
        resumeId: dto.resumeId,
        coverLetter: dto.coverLetter,
    });

    // Получаем HR компании для чата
    const vacancy = await this.vacancyRepository.findById(dto.vacancyId);
    const companyId = vacancy.project.companyId;

    // Получаем первого доступного HR (root или нанятый)
    const hrRoleContext = await this.roleContextRepository.findFirstByCompanyId(companyId);

    if (hrRoleContext) {
        // Создаем чат по отклику
        await this.chatsService.createChat(candidateId, {
            type: ChatType.APPLICATION,
            applicationId: application.id,
            memberIds: [hrRoleContext.userId],
        });
    }

    return application;
}
```

## Best Practices

1. **Автоматическое создание** - для APPLICATION чатов
2. **Проверяйте участников** - перед созданием
3. **Используйте правильный тип** - для разных сценариев
4. **Связывайте с Application** - для APPLICATION чатов
