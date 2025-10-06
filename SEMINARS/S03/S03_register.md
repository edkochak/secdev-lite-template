# S03 - Реестр NFR (Given-When-Then)

Этот файл содержит **реестр NFR** для выбранных User Stories.

---

## Выбранные User Stories

1. **US-002** - Вход в систему (Login)
   - Вход, выдача JWT/сессии, защита от перебора
   - API: `POST /api/auth/login`

2. **US-004** - Редактирование профиля
   - Изменение имени, контактов; маскирование PII в логах
   - API: `GET/PUT /api/profile`

3. **US-008** - Оформление заказа/покупки
   - Корзина → заказ → платёжная сессия → вебхук статуса
   - API: `POST /api/orders`, `POST /api/payments/session`, `POST /api/payments/webhook`

---

## Поля реестра (data dictionary)

* **ID** - короткий идентификатор, например `NFR-001`.
* **User Story / Feature** - к какой истории/фиче относится требование.
* **Category** - категория из банка (Performance, Security-AuthZ/RBAC, RateLimiting и т.д.).
* **Requirement (NFR)** - *измеримое* требование (числа/пороги/границы действия).
* **Rationale / Risk** - зачем это нужно, какой риск/ценность покрываем.
* **Acceptance (G-W-T)** - проверяемая формулировка: *Given … When … Then …*.
* **Evidence (test/log/scan/policy)** - чем подтвердим выполнение.
* **Trace (issue/link)** - ссылка на задачу, обсуждение, артефакт.
* **Owner** - ответственный.
* **Status** - `Draft` | `Proposed` | `Approved` | `Implemented` | `Verified`.
* **Priority** - `P1 - High` | `P2 - Medium` | `P3 - Low`.
* **Severity** - `S1 - Critical` | `S2 - Major` | `S3 - Minor`.
* **Tags** - произвольные метки (через запятую).

---

## Таблица реестра

| ID      | User Story / Feature                    | Category                 | Requirement (NFR)                                                                                                    | Rationale / Risk                                         | Acceptance (G-W-T)                                                                                                                                                                                                 | Evidence (test/log/scan/policy)                      | Trace (issue/link) | Owner    | Status   | Priority    | Severity     | Tags                    |
| ------- | --------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- | ------------------ | -------- | -------- | ----------- | ------------ | ----------------------- |
| NFR-001 | US-002: Login                           | Performance              | P95 латентность для POST `/api/auth/login` ≤ 200 мс при 100 RPS в течение 5 минут                                   | UX требования и SLO для критичного эндпойнта             | **Given** сервис развернут и здоров<br>**When** на `/api/auth/login` подается 100 RPS в течение 5 минут<br>**Then** P95 ≤ 200 мс и доля ошибок ≤ 1%                                                              | test: `load-login-100rps`; метрика: p95_latency      | #AUTH-101          | team-sec | Proposed | P1 - High   | S1 - Critical| perf,login,slo          |
| NFR-002 | US-002: Login                           | RateLimiting             | На `/api/auth/login` действует лимит 10 req/min на IP; превышение → 429 + Retry-After                               | Защита от brute-force атак и перебора паролей            | **Given** клиент с IP X<br>**When** выполняется 11+ запросов к `/api/auth/login` за 60 секунд<br>**Then** лишние запросы получают 429 и корректный заголовок Retry-After (60-120 сек)                            | test: `e2e-ratelimit-login`; log: 429 responses      | #AUTH-102          | team-sec | Proposed | P1 - High   | S1 - Critical| security,ratelimit,auth |
| NFR-003 | US-002: Login                           | Security-AuthN           | Все write-эндпойнты требуют валидный JWT; истёкший/некорректный токен → 401                                         | Базовая линия безопасности для защищённых операций       | **Given** истекший или некорректный JWT<br>**When** POST `/api/auth/login` (или любой write-эндпойнт)<br>**Then** 401 с телом в RFC 7807 (application/problem+json)                                              | test: `integration-auth-401`; example: error response| #AUTH-103          | team-sec | Approved | P1 - High   | S1 - Critical| security,authn,jwt      |
| NFR-004 | US-002: Login                           | Observability/Logging    | Все запросы логируются в JSON и содержат correlation_id на каждом этапе                                              | Трассировка запросов и разбор инцидентов                 | **Given** запрос с заголовком X-Correlation-ID=abc123<br>**When** он проходит через сервис login<br>**Then** во всех логах появляется correlation_id=abc123 и ключевые поля (user_id, ip, timestamp, status)     | log: structured JSON logs; query: correlation_id     | #OBS-201           | team-ops | Proposed | P2 - Medium | S2 - Major   | observability,logging   |
| NFR-005 | US-004: Edit Profile                    | Privacy/PII              | PII (email, phone, name) не попадает в логи в открытом виде; сырые PII хранятся не дольше 365 дней                  | Соответствие GDPR/приватность и минимизация утечек       | **Given** DTO с персональными данными (email, phone)<br>**When** происходит логирование операции PUT `/api/profile`<br>**Then** поля PII маскированы (e***@***.com, +7***1234); план ретенции 365 дней применён | log: masked PII examples; policy: retention schedule | #PRIV-301          | team-dev | Proposed | P1 - High   | S2 - Major   | privacy,pii,gdpr        |
| NFR-006 | US-004: Edit Profile                    | Security-AuthZ/RBAC      | Пользователь может редактировать только свой профиль; попытка доступа к чужому → 403                                 | Изоляция данных пользователей, наименьшие привилегии     | **Given** пользователь user_A аутентифицирован<br>**When** он пытается PUT `/api/profile/{user_B_id}`<br>**Then** ответ 403 (или 404 по политике) без утечки данных user_B                                       | test: `integration-authz-profile`; policy: RBAC rules| #AUTHZ-401         | team-sec | Approved | P1 - High   | S1 - Critical| security,authz,rbac     |
| NFR-007 | US-004: Edit Profile                    | Security-InputValidation | Для PUT `/api/profile`: размер тела ≤ 1 MiB, extra поля запрещены, валидация email/phone по стандартам              | Защита от DoS, инъекций и грязных данных                 | **Given** тело запроса 2 MiB или с неизвестными полями<br>**When** PUT `/api/profile`<br>**Then** 413 (если >1 MiB) или 400 (если extra поля) с телом ошибки в RFC 7807                                          | test: `e2e-validation-profile`; schema: DTO validator| #VAL-501           | team-dev | Proposed | P2 - Medium | S2 - Major   | validation,security     |
| NFR-008 | US-008: Order/Payment                   | Data-Integrity           | Все SQL-запросы параметризованы; транзакции для заказов используют SERIALIZABLE или READ COMMITTED с явными блокировками | Защита от SQL-инъекций и race conditions при оплате      | **Given** создание заказа с одновременными запросами<br>**When** выполняется POST `/api/orders`<br>**Then** используется параметризация; транзакция изолирована; нет дублей заказов                               | code: ORM config/query examples; test: concurrency   | #DATA-601          | team-dev | Approved | P1 - High   | S1 - Critical| integrity,sql,orders    |
| NFR-009 | US-008: Order/Payment                   | Timeouts/Retry/CircuitBreaker | Исходящие вызовы к платёжному API: timeout ≤ 3s, retry ≤ 3 с экспоненциальным backoff; circuit breaker при ≥50% ошибок за 1 мин | Устойчивость к недоступности внешних зависимостей        | **Given** недоступность платёжного API<br>**When** сервис вызывает `/api/payments/session`<br>**Then** суммарное ожидание ≤ 9s; выполняются не более 3 retry с джиттером; после порога включается circuit breaker | config: HTTP client settings; test: failure scenarios| #RESIL-701         | team-ops | Proposed | P1 - High   | S2 - Major   | resilience,timeout,cb   |
| NFR-010 | US-008: Order/Payment                   | Auditability             | Для операций создания/изменения заказа создаётся audit-событие (actor, action, order_id, amount, timestamp, result) | Соответствие финансовым политикам, расследование споров  | **Given** пользователь создаёт заказ<br>**When** операция POST `/api/orders` завершается<br>**Then** создается audit-запись с actor, order_id, amount, временем и результатом; запись неизменяема (append-only) | log: audit trail examples; policy: immutable storage | #AUDIT-801         | team-ops | Proposed | P1 - High   | S2 - Major   | audit,compliance,orders |

---

## Проверка непротиворечивости

✅ **Timeouts и Performance:** NFR-001 требует P95 ≤ 200 мс для login, NFR-009 устанавливает timeout 3s для внешних вызовов - не конфликтуют.

✅ **Rate Limiting и SLO:** NFR-002 ограничивает 10 req/min на IP для login, NFR-001 тестирует 100 RPS общую нагрузку - покрывают разные аспекты (защита vs производительность).

✅ **Privacy и Logging:** NFR-005 требует маскирования PII в логах, NFR-004 требует correlation_id в логах - дополняют друг друга.

✅ **Security layers:** NFR-003 (AuthN), NFR-006 (AuthZ), NFR-007 (Input Validation) - многоуровневая защита без конфликтов.

---

## Итоги

* ✅ **10 NFR** с измеримыми формулировками
* ✅ Каждое NFR имеет **Acceptance в формате G-W-T**
* ✅ Определены **Evidence** и **Trace**
* ✅ Покрыты критичные категории: Performance, Security (AuthN/AuthZ/Input), Privacy, Data Integrity, Resilience, Observability, Auditability
* ✅ Нет противоречий между требованиями

---

## Следующие шаги (после семинара)

* Перенести эти 10 NFR в раздел **NFR** файла `GRADING/TM.md`
* На S04-S05 связать NFR с угрозами (STRIDE) и архитектурными решениями (ADR) по ID
* Подготовить Evidence (тесты, логи, политики) для TM/DV/DS
