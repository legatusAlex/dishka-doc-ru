# Документация dishka на русском языке

**dishka** — библиотека для Dependency Injection (DI) в Python. Название происходит от русского слова «дишка» (милая маленькая штука). Библиотека решает те же задачи, что и `dependency-injector`, `lagom`, `python-inject` и аналоги, но с упором на явность, модульность и удобство работы со скоупами.

## Для кого эта документация

Для Python-разработчиков, которые:
- уже знакомы с концепцией DI (из других языков или библиотек)
- переходят на dishka с `dependency-injector` или другого DI-фреймворка
- хотят подробного и практичного объяснения на русском языке

## Структура документации

| Файл | Содержание |
|------|------------|
| [01-concepts.md](./01-concepts.md) | Ключевые концепции: Provider, Container, Scope, Component |
| [02-providers.md](./02-providers.md) | Инструменты провайдера: `provide`, `alias`, `decorate`, `from_context`, `provide_all` |
| [03-scopes.md](./03-scopes.md) | Скоупы: встроенные, пропускаемые, кастомные |
| [04-container.md](./04-container.md) | Контейнер: создание, API, потокобезопасность, async |
| [05-fastapi.md](./05-fastapi.md) | Интеграция с FastAPI — подробно |
| [06-faststream.md](./06-faststream.md) | Интеграция с FastStream — подробно |
| [07-other-integrations.md](./07-other-integrations.md) | Другие интеграции: aiogram, aiohttp, Flask, Celery и др. |
| [08-advanced.md](./08-advanced.md) | Продвинутые темы: компоненты, дженерики, визуализация |
| [09-testing.md](./09-testing.md) | Тестирование: моки, фикстуры, замена зависимостей |

## Быстрый старт

```bash
pip install dishka
```

```python
from dishka import Provider, Scope, make_container, provide

# 1. Опишите свои классы обычным Python
class Database:
    def __init__(self, url: str): ...

class UserRepository:
    def __init__(self, db: Database): ...

class UserService:
    def __init__(self, repo: UserRepository): ...

# 2. Создайте провайдер — опишите, как создавать зависимости
class AppProvider(Provider):
    scope = Scope.APP

    @provide
    def get_db(self) -> Database:
        return Database("sqlite:///app.db")

    user_repo = provide(UserRepository)
    user_service = provide(UserService)

# 3. Создайте контейнер и получайте зависимости
container = make_container(AppProvider())
service = container.get(UserService)
container.close()
```

## Ключевые отличия от dependency-injector

| Возможность | dependency-injector | dishka |
|---|---|---|
| Скоупы | 2 (singleton + factory) | Любое количество |
| Финализация зависимостей | Через `closing()` | Через `yield` в фабрике |
| Модульность провайдеров | Контейнеры с наследованием | Отдельные `Provider`-классы |
| Маркировка зависимостей | `Provide[...]` в классе | Не нужно — только type hints |
| Интеграции с фреймворками | Есть | Есть (FastAPI, FastStream, aiogram и др.) |
