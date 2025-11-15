# S06 - Secure Coding One-liner

## One-liner команда для полного цикла сборки и тестирования:

```bash
python3 scripts/init_db.py && python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v && python3 -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```

## Требования:
- Python 3.9+
- Зависимости: `pip install -r requirements.txt`

## Что делает команда:
1. `python3 scripts/init_db.py` - инициализирует SQLite базу данных с тестовыми данными
2. `python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v` - запускает тесты, генерирует XML отчёт
3. `python3 -m uvicorn app.main:app --host 0.0.0.0 --port 8000` - запускает веб-сервер

## Результат:
- Отчёт тестов: `EVIDENCE/S06/test-report.xml`
- Веб-приложение доступно на http://localhost:8000