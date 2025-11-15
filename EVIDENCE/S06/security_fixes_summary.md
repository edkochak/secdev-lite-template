# S06 Security Fixes Summary

## Применённые исправления безопасности:

### 1. SQL Injection - Login (S06-01)
**Проблема:** f-строка в SQL запросе `app/main.py:34`
**Исправление:** 
- Добавлена функция `query_one_params()` в `app/db.py:24-28`
- Заменён опасный запрос на параметризованный: `"SELECT id, username FROM users WHERE username = ? AND password = ?"`
- Тесты: `test_login_should_not_allow_sql_injection` и `test_login_valid_credentials_format` ✅

### 2. SQL Injection - Search (S06-02)  
**Проблема:** f-строка в LIKE запросе `app/main.py:26`
**Исправление:**
- Добавлена функция `query_params()` в `app/db.py:30-34`
- Безопасная параметризация: `"SELECT id, name, description FROM items WHERE name LIKE ?"` с `pattern = f"%{q}%"`
- Тесты: `test_search_should_not_return_all_on_injection` ✅

### 3. XSS - Небезопасный вывод (S06-03)
**Проблема:** `{{ message|safe }}` в шаблоне `app/templates/index.html:5`
**Исправление:** 
- Убран фильтр `|safe`, включено автоэкранирование Jinja2
- Тесты: `test_echo_should_escape_script_tags` ✅

### 4. Валидация ввода (S06-04)
**Проблема:** Отсутствие ограничений в модели `LoginRequest`
**Исправление:**
- Добавлены строгие constraints в `app/models.py:6-7`:
  - `username`: 3-48 символов, только alphanumeric + `.` + `-` + `_`
  - `password`: 3-128 символов
- Параметр `q` в search ограничен 1-32 символами через `Query` валидацию
- Тесты: проверка 422 ошибок при превышении лимитов

## Итоговая статистика тестов:
```
========================= 4 passed, 1 warning in 0.19s =========================
```

## Карточки безопасности (реализованы):
- ✅ S06-01 - SQL Injection (login) 
- ✅ S06-02 - SQL Injection (search LIKE)
- ✅ S06-03 - XSS (экранирование вывода)
- ✅ S06-04 - Валидация ввода (строгость/длины)

## Артефакты:
- `EVIDENCE/S06/test-report.xml` - JUnit XML отчёт тестов
- `EVIDENCE/S06/one-liner.md` - документация команды сборки
- `EVIDENCE/S06/security_fixes_summary.md` - данный файл