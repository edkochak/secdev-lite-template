# S04 - STRIDE per element (матрица)

Этот файл содержит **STRIDE анализ** для каждого элемента и потока из DFD, с связями к NFR из S03.

---

## Легенда STRIDE

* **S - Spoofing:** подмена идентичности/токена
* **T - Tampering:** изменение данных/запросов/конфигурации
* **R - Repudiation:** отрицание действий (нет аудита/трассировки)
* **I - Information disclosure:** утечка конфиденциальных данных (PII/секреты)
* **D - Denial of service:** отказ в обслуживании (ресурсное истощение/«залипание»)
* **E - Elevation of privilege:** повышение привилегий/обход RBAC/тенант-изоляции

---

## STRIDE Matrix

| Element | Data/Boundary | Threat (S/T/R/I/D/E) | Description | NFR link (ID) | Mitigation idea (ADR later) |
| ------- | ------------- | -------------------- | ----------- | ------------- | --------------------------- |
| **Edge: U → A** | JWT/HTTPS | S | Повтор/подмена токена, reuse истёкшего/украденного JWT | NFR-003 (AuthN) | JWT TTL+Refresh, rate limit на `/auth/*` |
| **Edge: U → A** | JWT/HTTPS | T | Модификация JWT payload, подделка claims | NFR-003 (AuthN) | JWT signature validation, HMAC/RSA |
| **Edge: U → A** | PII/Profile Data | I | PII в логах и сообщениях об ошибках | NFR-005 (Privacy/PII) | Маскирование PII, RFC7807 без стэктрейсов |
| **Edge: U → A** | PII/Profile Data | D | DoS через большие payload, rate limiting bypass | NFR-002 (RateLimiting), NFR-007 (InputValidation) | Payload size limits, rate limiting per IP |
| **Node: A (API Gateway)** | Logs/Requests | R | Отсутствие аудита действий пользователей | NFR-010 (Auditability) | Structured logging с correlation_id |
| **Node: A (API Gateway)** | Configuration | T | Изменение rate limits, security config | NFR-002 (RateLimiting) | Immutable config, config validation |
| **Edge: A → S** | DTO/Requests | T | Injection через невалидированные поля | NFR-007 (InputValidation) | Schema validation, input sanitization |
| **Edge: A → S** | Rate Limited | D | Resource exhaustion через множество запросов | NFR-002 (RateLimiting) | Rate limiting, request queuing |
| **Node: S (Service)** | Business Logic | E | Обход RBAC, доступ к чужим данным | NFR-006 (AuthZ/RBAC) | Tenant isolation, role-based access |
| **Node: S (Service)** | Session/State | S | Session hijacking, token reuse | NFR-003 (AuthN) | Secure session management, token rotation |
| **Edge: S → D** | SQL/ORM | T | SQL injection, data tampering | NFR-008 (Data-Integrity) | Parameterized queries, ORM validation |
| **Edge: S → D** | SQL/ORM | I | Утечка данных через SQL injection | NFR-008 (Data-Integrity) | Query parameterization, access controls |
| **Edge: S → D** | Audit Events | R | Отсутствие audit trail для критичных операций | NFR-010 (Auditability) | Immutable audit log, event sourcing |
| **Node: D (Database)** | Stored Data | I | Несанкционированный доступ к PII/финансовым данным | NFR-005 (Privacy/PII), NFR-006 (AuthZ/RBAC) | Database encryption, access controls |
| **Node: D (Database)** | Stored Data | T | Изменение данных без аудита | NFR-008 (Data-Integrity), NFR-010 (Auditability) | Database triggers, audit logging |
| **Edge: S → P** | HTTP/gRPC | D | Залипание без timeout/retry/circuit breaker | NFR-009 (Timeouts/Retry/CB) | Timeout≤3s, retry≤3 с джиттером, CB |
| **Edge: S → P** | HTTP/gRPC | T | Модификация платежных запросов | NFR-009 (Timeouts/Retry/CB) | Request signing, TLS, idempotency |
| **Edge: S → P** | HTTP/gRPC | I | Утечка платежных данных в логах | NFR-005 (Privacy/PII) | Payment data masking, secure logging |
| **Node: P (Payment API)** | External Service | D | Недоступность внешнего сервиса | NFR-009 (Timeouts/Retry/CB) | Circuit breaker, fallback mechanisms |
| **Edge: S → E** | SMTP/API | I | Утечка email адресов и PII | NFR-005 (Privacy/PII) | Email masking, secure transmission |
| **Edge: S → E** | SMTP/API | D | DoS через спам, rate limiting bypass | NFR-002 (RateLimiting) | Email rate limiting, spam protection |

---

## Сводка по категориям угроз

### Spoofing (S) - 3 угрозы
- JWT token reuse/hijacking
- Session hijacking
- External service impersonation

### Tampering (T) - 5 угроз
- JWT payload modification
- Configuration tampering
- SQL injection/data tampering
- Payment request modification
- Database data modification

### Repudiation (R) - 2 угрозы
- Missing audit trail for user actions
- Missing audit trail for critical operations

### Information Disclosure (I) - 5 угроз
- PII in logs and error messages
- SQL injection data leakage
- Unauthorized access to PII/financial data
- Payment data in logs
- Email addresses and PII exposure

### Denial of Service (D) - 5 угроз
- DoS via large payloads
- Resource exhaustion through requests
- External service unavailability
- Email spam/DoS
- Rate limiting bypass

### Elevation of Privilege (E) - 1 угроза
- RBAC bypass, access to other users' data

---

## Связи с NFR из S03

| NFR ID | Категория | Количество угроз | Критичность |
|--------|-----------|------------------|-------------|
| **NFR-002** | RateLimiting | 3 | Высокая (DoS protection) |
| **NFR-003** | Security-AuthN | 2 | Критичная (Authentication) |
| **NFR-005** | Privacy/PII | 4 | Критичная (Data protection) |
| **NFR-006** | Security-AuthZ/RBAC | 2 | Критичная (Authorization) |
| **NFR-007** | Security-InputValidation | 2 | Высокая (Input security) |
| **NFR-008** | Data-Integrity | 3 | Критичная (Data integrity) |
| **NFR-009** | Timeouts/Retry/CircuitBreaker | 2 | Высокая (Resilience) |
| **NFR-010** | Auditability | 2 | Высокая (Compliance) |

---

## Готовность к приоритизации

✅ **Все элементы DFD проанализированы** (узлы и рёбра)
✅ **Каждая угроза связана с NFR** из реестра S03
✅ **Mitigation ideas** подготовлены для будущих ADR
✅ **20 уникальных угроз** выявлено и документировано

**Следующий шаг**: Переход к `S04_risk_scoring.md` для приоритизации L×I и выбора Top-5.
