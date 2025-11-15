# S07 - Containerization Summary

## Выполненные задачи:

### 1. Базовая контейнеризация
- ✅ Dockerfile готов в корне проекта
- ✅ docker-compose.yml настроен
- ✅ One-liner из S06 переносится в контейнерный сценарий

### 2. Улучшения безопасности (создан Dockerfile.secure)
- ✅ **Non-root пользователь:** создание `appuser` и переключение с `USER appuser`
- ✅ **Ограничение прав:** `chown -R appuser:appuser /app`
- ✅ **Health check:** проверка доступности на `/` каждые 30 секунд
- ✅ **Минимальная поверхность атаки:** использование `python:3.11-slim`

### 3. Команды запуска:

**Обычная сборка:**
```bash
docker build -t secdev-seed:latest .
docker run --rm -p 8000:8000 secdev-seed:latest
```

**Безопасная сборка:**
```bash
docker build -f Dockerfile.secure -t secdev-seed:secure .
docker run --rm -p 8000:8000 secdev-seed:secure
```

**Docker Compose:**
```bash
docker compose up --build
```

### 4. Проверки безопасности контейнера:

- **Пользователь:** приложение НЕ запускается от root
- **Healthcheck:** контейнер самодиагностируется
- **Порты:** только необходимый 8000 экспозирован
- **Окружение:** переменные среды без секретов

### Артефакты S07:
- `Dockerfile` - базовая версия
- `Dockerfile.secure` - улучшенная безопасная версия
- `docker-compose.yml` - оркестрация
- `EVIDENCE/S07/containerization_summary.md` - данный файл