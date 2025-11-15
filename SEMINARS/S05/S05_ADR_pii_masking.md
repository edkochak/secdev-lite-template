# ADR: PII Masking + RFC7807 Error Format

Status: Proposed

## Context

Risk: R-01 "PII утечка в логах и сообщениях об ошибках" (L=4, I=5, Score=20)
DFD: Edge: U→A, S→P (пользовательские данные в логах и ошибках)
NFR: NFR-005 (Privacy/PII)
Assumptions: JSON-логи, существующий middleware для ошибок, GDPR compliance требования, structured logging

Риск относится к топ-приоритетным (Score=20) из-за высокой вероятности утечки PII через логи (публичная поверхность, типичная ошибка разработки) и критичного воздействия (массовые GDPR нарушения).

## Decision

Реализовать middleware для автоматического маскирования PII в логах и стандартизировать формат ошибок по RFC7807 без утечки чувствительных данных.

- Middleware PII masking: обработка полей email, phone, name в логах через allowlist разрешённых полей
- Error formatting: все API ошибки в формате RFC7807 (application/problem+json) без стэктрейсов и PII
- Retention policy: сырые PII данные хранятся не дольше 365 дней с автоматической очисткой
- Scope: все API эндпойнты (/api/*), все уровни логирования (INFO, ERROR, DEBUG)
- Layer: application middleware level

## Alternatives

- **Structured Logging + Field Filtering** - отклонено из-за высокой сложности рефакторинга (Net=-1), требует изменения всех точек логирования
- **Centralized Log Processing + PII Detection** - отклонено из-за высокой стоимости (Net=-3) и внешних зависимостей от log processing платформы

## Consequences

**Положительные:**
+ Полное устранение риска утечки PII в логах (Security impact=5)
+ GDPR compliance и снижение регуляторных рисков
+ Стандартизированный формат ошибок упрощает клиентскую интеграцию
+ Минимальная сложность реализации через middleware паттерн

**Негативные/издержки:**
- Потеря части диагностической информации в логах (требует баланс между безопасностью и отладкой)
- Необходимость обучения команды новому формату логирования
- Дополнительная нагрузка на performance из-за обработки каждого лог-события

## DoD / Acceptance

**Given** DTO с персональными данными (email: user@example.com, phone: +71234567890, name: "Иван Иванов")
**When** происходит логирование операции PUT `/api/profile` и возникает ошибка
**Then** 
- Поля PII маскированы в логах (email: u***@***.com, phone: +7***7890, name: И*** И***)
- Ошибка возвращается в формате RFC7807 с полями type, title, status, detail, correlation_id
- Стэктрейсы и внутренние детали не попадают в ответ клиенту

Проверяемые критерии:
- test: `integration-pii-masking-logs` - проверка маскирования в логах
- test: `e2e-error-format-rfc7807` - проверка формата ошибок  
- log: примеры замаскированных записей в production логах
- policy: retention schedule для PII данных 365 дней
- scan: SAST правила для детекции PII в новом коде

## Rollback / Fallback

Откат через feature flag `ENABLE_PII_MASKING=false` в environment переменных.
Fallback на стандартное логирование без маскирования.
Monitoring: алерты на увеличение размера логов (может указывать на проблемы с маскированием).

## Trace

- DFD: `/SEMINARS/S04/S04_dfd.md` - Edge: U→A, S→P
- STRIDE: `/SEMINARS/S04/S04_stride_matrix.md` - строки с Information Disclosure угрозами  
- Risk scoring: `/SEMINARS/S04/S04_risk_scoring.md` - R-01, Top-1 приоритет
- NFR: `/SEMINARS/S03/S03_register.md` - NFR-005 (Privacy/PII)
- Issues: #PRIV-301, #OBS-201

## Ownership & Dates

**Owner:** team-dev  
**Reviewers:** team-sec, team-ops  
**Date created:** 2025-11-15  
**Last updated:** 2025-11-15

## Open Questions

- Какие дополнительные поля считать PII (IP адреса, user agents)?
- Нужно ли маскировать PII в metrics/traces или только в логах?
- Настройка retention policy для разных типов PII данных

## Appendix

```javascript
// Пример middleware для маскирования PII
const piiMaskingMiddleware = (req, res, next) => {
  const allowedFields = ['user_id', 'timestamp', 'action', 'correlation_id'];
  // Маскирование логики...
  req.logContext = maskPiiFields(req.logContext, allowedFields);
  next();
};

// Пример RFC7807 ответа
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation failed", 
  "status": 400,
  "detail": "The request body contains invalid fields",
  "correlation_id": "abc-123-def"
}
```