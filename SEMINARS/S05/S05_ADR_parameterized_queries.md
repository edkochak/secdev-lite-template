# ADR: Parameterized Queries + DAO Input Validation

Status: Proposed

## Context

Risk: R-02 "SQL injection и несанкционированный доступ к данным" (L=4, I=5, Score=20)
DFD: Edge: S→D, Node: D (сервис к базе данных)
NFR: NFR-008 (Data-Integrity)
Assumptions: существующая архитектура с прямыми SQL-запросами, публичные API эндпойнты, критичные данные в БД, частичное использование ORM

Риск критичен (Score=20) из-за высокой вероятности SQL injection атак на публичных поверхностях и катастрофического воздействия (массовая утечка данных, компрометация БД).

## Decision

Заменить все прямые SQL-запросы на parameterized prepared statements и добавить валидацию входных данных на уровне DAO с канонизацией.

- Database queries: только prepared statements с параметрами, запрет на string concatenation в SQL
- Input validation: проверка и нормализация всех входных параметров на DAO уровне  
- Data canonicalization: стандартизация строк, дат, номеров телефонов перед записью в БД
- Scope: все операции чтения/записи в БД через DAO слой, все публичные API эндпойнты
- Layer: Data Access Object (DAO) level

## Alternatives

- **ORM с параметризованными запросами + миграция** - отклонено из-за нулевого Net score, требует масштабный рефакторинг всей архитектуры
- **Database WAF + Query Pattern Blocking** - отклонено из-за отрицательного Net score (-3), высоких внешних зависимостей и неполной защиты

## Consequences

**Положительные:**
+ Полное устранение SQL injection уязвимостей (Security impact=5)
+ Улучшение целостности данных через валидацию и канонизацию
+ Сохранение существующей архитектуры с минимальными изменениями
+ Быстрая реализация с использованием знакомых технологий

**Негативные/издержки:**
- Необходимость рефакторинга всех существующих SQL-запросов
- Дополнительная валидация может незначительно увеличить латентность
- Требует дисциплины команды при написании нового кода

## DoD / Acceptance

**Given** пользовательский ввод с потенциальными SQL injection попытками (напр., `'; DROP TABLE users; --`)
**When** выполняется операция CREATE/READ/UPDATE через API эндпойнт (например, POST `/api/orders`)
**Then**
- Все запросы к БД используют prepared statements с параметрами
- Входные данные валидированы и канонизированы
- SQL injection попытки не влияют на структуру запроса
- Данные записаны в канонизированном виде

Проверяемые критерии:
- test: `unit-sql-injection-attempts` - тесты на SQL injection попытки
- test: `integration-dao-validation` - проверка валидации на DAO уровне
- code: примеры prepared statements в коде
- scan: SAST правила для детекции небезопасных SQL запросов
- policy: code review checklist с проверкой SQL безопасности

## Rollback / Fallback

Постепенный rollback через feature flag `ENABLE_PARAMETERIZED_QUERIES=false` для каждого DAO класса.
Fallback на существующие SQL запросы с дополнительным логированием.
Monitoring: метрики ошибок валидации и алерты на подозрительные SQL паттерны.

## Trace

- DFD: `/SEMINARS/S04/S04_dfd.md` - Edge: S→D, Node: D
- STRIDE: `/SEMINARS/S04/S04_stride_matrix.md` - строки с Tampering угрозами к данным
- Risk scoring: `/SEMINARS/S04/S04_risk_scoring.md` - R-02, Top-2 приоритет  
- NFR: `/SEMINARS/S03/S03_register.md` - NFR-008 (Data-Integrity)
- Issues: #DATA-601, #SEC-401

## Ownership & Dates

**Owner:** team-dev  
**Reviewers:** team-sec, DBA team  
**Date created:** 2025-11-15  
**Last updated:** 2025-11-15

## Open Questions

- Нужно ли добавить дополнительную валидацию на уровне БД через stored procedures?
- Как обрабатывать legacy код с dynamic SQL в отчётах?
- Политика обработки ошибок валидации (fail fast vs graceful degradation)

## Appendix

```sql
-- До: небезопасный подход
String query = "SELECT * FROM users WHERE id = " + userId;

-- После: безопасный подход  
PreparedStatement stmt = connection.prepareStatement(
    "SELECT * FROM users WHERE id = ?"
);
stmt.setInt(1, userId);
```

```java
// Пример DAO валидации
public class UserDAO {
    public User findById(String userId) {
        // Валидация входных данных
        if (!isValidUserId(userId)) {
            throw new ValidationException("Invalid user ID format");
        }
        
        // Канонизация
        String canonicalUserId = canonicalizeUserId(userId);
        
        // Безопасный запрос
        return jdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new Object[]{canonicalUserId},
            userRowMapper
        );
    }
}
```