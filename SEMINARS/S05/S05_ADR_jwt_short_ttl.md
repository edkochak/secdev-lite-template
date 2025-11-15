# ADR: JWT Short TTL + Refresh + Key Rotation

Status: Proposed

## Context

Risk: R-03 "JWT token reuse/hijacking и подмена аутентификации" (L=4, I=4, Score=16)
DFD: Edge: U→A, Node: A (пользователь к API gateway)
NFR: NFR-003 (Security-AuthN)
Assumptions: публичная поверхность атаки, существующая JWT архитектура, stateless сервисы, HTTPS only, требования совместимости с мобильными клиентами

Риск высокого приоритета (Score=16) из-за публичной доступности токенов и серьёзных последствий их компрометации (доступ к пользовательским аккаунтам).

## Decision

Реализовать архитектуру с короткоживущими access токенами, refresh токенами для обновления и автоматической ротацией ключей подписи.

- Access Token TTL: 15 минут для минимизации окна атаки при компрометации
- Refresh Token: TTL=7 дней, secure+httponly cookies, возможность отзыва
- Key rotation: автоматическая ротация JWKS ключей каждые 24 часа
- Clock skew: tolerance ±60 секунд для синхронизации между сервисами
- Scope: все JWT токены в системе, все API эндпойнты требующие авторизации
- Layer: API Gateway + Authorization Service

## Alternatives

- **mTLS Client Authentication** - отклонено из-за отрицательного Net score (-4), высокой сложности PKI управления и проблем с мобильными клиентами
- **JWT + Device Fingerprinting + IP Binding** - отклонено из-за отрицательного Net score (-4), проблем с мобильностью пользователей и потенциальных нарушений приватности

## Consequences

**Положительные:**
+ Значительное сокращение окна атаки при краже токенов (15 минут vs часы/дни)
+ Возможность быстрого отзыва доступа через refresh token blacklist
+ Автоматическая ротация ключей снижает риски долгосрочной компрометации
+ Совместимость с существующей архитектурой и мобильными клиентами

**Негативные/издержки:**
- Увеличение количества запросов на обновление токенов
- Необходимость управления состоянием refresh токенов (blacklist)
- Дополнительная сложность в клиентском коде для обработки обновления токенов

## DoD / Acceptance

**Given** пользователь получает access token после аутентификации
**When** проходит 16 минут с момента выдачи токена  
**Then** токен становится невалидным и API возвращает 401 с корректным RFC7807 форматом

**Given** пользователь имеет валидный refresh token
**When** access token истекает и клиент запрашивает обновление
**Then** выдаётся новый access token с новым `exp` и обновлённым `kid` (key id)

Проверяемые критерии:
- test: `e2e-token-expiry-15min` - проверка истечения access токена
- test: `integration-refresh-flow` - тест refresh token механизма  
- test: `unit-key-rotation-24h` - проверка ротации ключей
- config: параметры TTL в конфигурации (ACCESS_TTL=15m, REFRESH_TTL=7d)
- log: события обновления токенов и ротации ключей

## Rollback / Fallback

Откат через environment переменные:
- `ACCESS_TOKEN_TTL` можно увеличить до предыдущих значений (например, 24h)
- `ENABLE_KEY_ROTATION=false` для отключения автоматической ротации
- Мониторинг: алерты на высокую частоту refresh запросов (может указывать на проблемы)

## Trace

- DFD: `/SEMINARS/S04/S04_dfd.md` - Edge: U→A, Node: A
- STRIDE: `/SEMINARS/S04/S04_stride_matrix.md` - строки с Spoofing угрозами
- Risk scoring: `/SEMINARS/S04/S04_risk_scoring.md` - R-03, Top-3 приоритет
- NFR: `/SEMINARS/S03/S03_register.md` - NFR-003 (Security-AuthN) 
- Issues: #AUTH-103, #AUTH-201

## Ownership & Dates

**Owner:** team-sec  
**Reviewers:** team-dev, team-ops  
**Date created:** 2025-11-15  
**Last updated:** 2025-11-15

## Open Questions

- Нужна ли дополнительная защита refresh токенов через fingerprinting?
- Как обрабатывать одновременные refresh запросы от одного пользователя?
- Политика хранения blacklisted refresh токенов (Redis vs Database)

## Appendix

```javascript
// Пример JWT payload для access token
{
  "sub": "user123",
  "iat": 1700000000,
  "exp": 1700000900,  // +15 минут
  "kid": "key-2025-11-15",
  "aud": "api.example.com",
  "scope": "read:profile write:orders"
}

// Пример refresh token flow
POST /auth/refresh
{
  "refresh_token": "rt_abc123...",
  "correlation_id": "xyz-456"
}

Response 200:
{
  "access_token": "ey...",
  "token_type": "Bearer", 
  "expires_in": 900,  // 15 минут в секундах
  "correlation_id": "xyz-456"
}
```

```yaml
# Environment configuration
ACCESS_TOKEN_TTL: "15m"
REFRESH_TOKEN_TTL: "7d"  
KEY_ROTATION_INTERVAL: "24h"
CLOCK_SKEW_TOLERANCE: "60s"
ENABLE_TOKEN_BLACKLIST: true
```