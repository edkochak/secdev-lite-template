# S04 - Приоритизация рисков L×I (risk scoring)

Этот файл агрегирует риски из `S04_stride_matrix.md`, объединяет дубликаты и приоритизирует по L×I (1-5).

---

## Шкала L×I (1-5)

**Likelihood (вероятность):**
1 - маловероятно (редкая поверхность, сложная эксплуатация, хорошая детекция)
2 - низкая
3 - средняя
4 - высокая
5 - очень высокая (публичная поверхность, типовая уязвимость, слабая детекция)

**Impact (ущерб):**
1 - низкий (локальный эффект, без PII/денег/доступности)
2 - умеренный
3 - значимый (PII/деньги одного клиента/подсистемы)
4 - высокий (существенный ущерб/простой)
5 - критичный (массовая утечка, компрометация прав, длительный простой)

**Score = L × I** → сортируем по убыванию → отмечаем **Top-5**

---

## Таблица приоритизации

| Risk ID | Source (DFD/Row) | Consolidated Description | Threat (S/T/R/I/D/E) | NFR link (ID) | L (1-5) | Rationale-L | I (1-5) | Rationale-I | **Score (=L×I)** | Decision (Top-5?) | ADR candidate |
| ------- | ---------------- | ------------------------ | -------------------- | ------------- | ------: | ----------- | ------: | ----------- | ---------------: | ----------------- | ------------- |
| **R-01** | Edge: U→A, S→P | PII утечка в логах и сообщениях об ошибках | I | NFR-005 (Privacy/PII) | 4 | Публичная поверхность, типичная ошибка разработки | 5 | Массовая утечка PII, GDPR нарушения | **20** | ✅ Top-1 | PII Masking + RFC7807 |
| **R-02** | Edge: S→D, Node: D | SQL injection и несанкционированный доступ к данным | T, I | NFR-008 (Data-Integrity) | 4 | Типичная уязвимость, публичная поверхность | 5 | Массовая утечка данных, компрометация БД | **20** | ✅ Top-2 | Parameterized Queries + ORM |
| **R-03** | Edge: U→A, Node: A | JWT token reuse/hijacking и подмена аутентификации | S | NFR-003 (AuthN) | 4 | Публичная поверхность, типичная атака | 4 | Компрометация аутентификации, доступ к аккаунтам | **16** | ✅ Top-3 | JWT TTL+Refresh + Rate Limiting |
| **R-04** | Node: S, Edge: S→D | RBAC bypass и доступ к чужим данным | E | NFR-006 (AuthZ/RBAC) | 3 | Средняя сложность эксплуатации | 5 | Критичная компрометация прав, доступ к чужим данным | **15** | ✅ Top-4 | Tenant Isolation + RBAC |
| **R-05** | Edge: S→P | External service DoS и залипание без circuit breaker | D | NFR-009 (Timeouts/Retry/CB) | 4 | Высокая вероятность сбоев внешних сервисов | 3 | Значимый простой платежей | **12** | ✅ Top-5 | Circuit Breaker + Timeouts |
| **R-06** | Edge: U→A, A→S | DoS через большие payload и resource exhaustion | D | NFR-002 (RateLimiting), NFR-007 (InputValidation) | 3 | Средняя вероятность, есть защита | 4 | Высокий ущерб от недоступности | **12** | | Payload Limits + Rate Limiting |
| **R-07** | Node: A, Edge: A→S | Configuration tampering и injection через невалидированные поля | T | NFR-002 (RateLimiting), NFR-007 (InputValidation) | 3 | Средняя сложность, нужен доступ к конфигу | 3 | Значимый ущерб от изменения настроек | **9** | | Config Validation + Input Sanitization |
| **R-08** | Edge: S→D, Node: D | Database data tampering без аудита | T, R | NFR-008 (Data-Integrity), NFR-010 (Auditability) | 2 | Низкая вероятность, нужен доступ к БД | 4 | Высокий ущерб от изменения данных | **8** | | Database Triggers + Audit Logging |
| **R-09** | Edge: S→P | Payment request modification и утечка платежных данных | T, I | NFR-009 (Timeouts/Retry/CB), NFR-005 (Privacy/PII) | 2 | Низкая вероятность, защищённые каналы | 4 | Высокий ущерб от компрометации платежей | **8** | | Request Signing + Payment Masking |
| **R-10** | Edge: S→E | Email spam/DoS и утечка email адресов | D, I | NFR-002 (RateLimiting), NFR-005 (Privacy/PII) | 2 | Низкая вероятность, есть защита | 3 | Значимый ущерб от спама и утечки email | **6** | | Email Rate Limiting + Masking |

---

## Сводка Top-5 (итог блока)

После сортировки по **Score** выпишем итоговый список для S05:

* **Top-1:** `R-01` - **PII утечка в логах** (Score: 20) - L=4 (публичная поверхность), I=5 (массовая утечка PII, GDPR) → **ADR candidate**: PII Masking + RFC7807
* **Top-2:** `R-02` - **SQL injection и несанкционированный доступ** (Score: 20) - L=4 (типичная уязвимость), I=5 (массовая утечка данных) → **ADR candidate**: Parameterized Queries + ORM
* **Top-3:** `R-03` - **JWT token reuse/hijacking** (Score: 16) - L=4 (публичная поверхность), I=4 (компрометация аутентификации) → **ADR candidate**: JWT TTL+Refresh + Rate Limiting
* **Top-4:** `R-04` - **RBAC bypass и доступ к чужим данным** (Score: 15) - L=3 (средняя сложность), I=5 (критичная компрометация прав) → **ADR candidate**: Tenant Isolation + RBAC
* **Top-5:** `R-05` - **External service DoS и залипание** (Score: 12) - L=4 (высокая вероятность сбоев), I=3 (значимый простой платежей) → **ADR candidate**: Circuit Breaker + Timeouts

---

## Анализ по категориям угроз

### По типу угрозы:
- **Information Disclosure (I)**: 2 из Top-5 (40%)
- **Tampering (T)**: 1 из Top-5 (20%)
- **Spoofing (S)**: 1 из Top-5 (20%)
- **Elevation of Privilege (E)**: 1 из Top-5 (20%)
- **Denial of Service (D)**: 1 из Top-5 (20%)

### По NFR категориям:
- **Privacy/PII (NFR-005)**: 1 из Top-5
- **Data-Integrity (NFR-008)**: 1 из Top-5
- **Security-AuthN (NFR-003)**: 1 из Top-5
- **Security-AuthZ/RBAC (NFR-006)**: 1 из Top-5
- **Timeouts/Retry/CircuitBreaker (NFR-009)**: 1 из Top-5

### По критичности:
- **Score 20**: 2 риска (критичные)
- **Score 16**: 1 риск (высокие)
- **Score 15**: 1 риск (высокие)
- **Score 12**: 1 риск (средние)

---

## Готовность к S05 (ADR)

✅ **Top-5 рисков** определены и приоритизированы
✅ **ADR candidates** подготовлены для каждого риска
✅ **Обоснования L×I** документированы
✅ **Связи с NFR** из S03 установлены

**Следующий шаг**: Создание ADR (Architecture Decision Records) для Top-5 рисков на семинаре S05.
