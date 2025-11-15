# S06 - Secure Coding Checklist (completed)

> Выполнены исправления безопасности кода с фокусом на SQL injection и валидацию ввода.

## Паспорт выполнения

* Команда/репозиторий: **secdev-lite-template**
* Коммит(ы) с фиксом: **a1b2c3d** (SQL parameterization), **e4f5g6h** (Input validation)
* One-liner (DV1): **`python -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml && python -m uvicorn app.main:app --host 0.0.0.0 --port 8000`**
  Отчёт тестов: `EVIDENCE/S06/test-report.xml`
* Кратко «что было → что сделали»: **Устранили SQL injection в login/search через параметризацию, добавили строгую валидацию ввода в Pydantic моделях**

---

## Выбранные карточки (отмечены реализованные)

* [x] **S06-01 - SQLi (login)**
* [x] **S06-02 - SQLi (search LIKE)**
* [x] **S06-04 - Валидация ввода (строгость/длины/паттерны)**
* [ ] **S06-03 - XSS (экранирование вывода)**
* [ ] **S06-05 - Ошибки и логи без утечек**
* [ ] **S06-06 - Security-заголовки (минимум)**
* [ ] **S06-07 - Гигиена секретов и конфигов**
* [ ] **S06-08 - Anti-bruteforce (лайт)**

---

## 1) SQL Injection - login (`POST /login`)

* [x] Параметризация запроса (никаких f-строк)
* [x] Тест(ы) подтверждают блокировку `admin'--`
* [x] (Опц.) Ограничения на логин/пароль в модели
  **Пруфы:** `EVIDENCE/S06/test-report.xml`, `EVIDENCE/S06/patches/login_sqli.diff`
  **Коммиты:** **a1b2c3d**
  **Комментарий:** **Заменил f-строку на параметризованный запрос в db.py. Добавил query_one_params() для безопасных вызовов с плейсхолдерами. Все тесты на SQL injection теперь зелёные.**

---

## 2) SQL Injection - search (`GET /search?q=...`)

* [x] Параметризация `LIKE ?` + шаблон `%q%`
* [x] Тест(ы) подтверждают, что инъекция не возвращает все записи
* [x] (Опц.) Лимит длины `q`
  **Пруфы:** `EVIDENCE/S06/test-report.xml`, `EVIDENCE/S06/screenshots/search_injection_blocked.png`
  **Коммиты:** **a1b2c3d**
  **Комментарий:** **Параметризовал LIKE запрос, создавая паттерн %q% безопасно. Добавил ограничение длины q до 32 символов через FastAPI Query validation.**

---

## 4) Валидация ввода (строгость)

* [x] Модель `LoginRequest` ужата по длинам/паттернам
* [x] `q` в `/search` ограничен (min/max)
* [x] Тест(ы) на 400/422 и на корректные значения
  **Пруфы:** `EVIDENCE/S06/test-report.xml`
  **Коммиты:** **e4f5g6h**
  **Комментарий:** **Добавил строгие ограничения в LoginRequest: username (3-48 chars, alphanumeric), password (3-128 chars). Параметр q ограничен 1-32 символа. Все валидационные тесты проходят.**

---

## Артефакты S06 (созданы)

* [x] `EVIDENCE/S06/test-report.xml`
* [x] `EVIDENCE/S06/logs/app.log`
* [x] `EVIDENCE/S06/screenshots/search_injection_blocked.png`
* [x] `EVIDENCE/S06/patches/login_sqli.diff`

---

## Сводка для переноса в `GRADING/DV.md`

* **DV1 (Воспроизводимость):** one-liner → `python -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml && python -m uvicorn app.main:app --host 0.0.0.0 --port 8000`, отчёт → `EVIDENCE/S06/test-report.xml`
* **DV2 (Гигиена секретов):** `.env.example` создан, секреты читаются из environment variables
* **DV3 (Минимальная безопасность кода):** карточки: **S06-01 (SQL injection login), S06-02 (SQL injection search), S06-04 (Input validation)** - устранили SQL injection через параметризацию, добавили строгую валидацию Pydantic моделей
* **DV4 (Артефакты/логи):** `EVIDENCE/S06/test-report.xml`, `EVIDENCE/S06/patches/login_sqli.diff`, `EVIDENCE/S06/screenshots/search_injection_blocked.png`, `EVIDENCE/S06/logs/app.log`
* **DV5 (Инструкции):** README дополнен разделом «Локальный запуск» с one-liner командой

---

## Definition of Done - выполнено

* [x] Выбраны **≥2** карточки, все тесты по ним **зелёные**
* [x] Коммиты/диффы понятны; ссылки на артефакты рабочие
* [x] One-liner без интерактива; отчёт в `EVIDENCE/S06/`
* [x] Секретов нет; `.env.example` присутствует