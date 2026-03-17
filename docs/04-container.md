# Container: создание, API и управление жизненным циклом

Container — центральный объект во время работы приложения. Через него вы получаете зависимости, управляете их жизненным циклом и входите в дочерние скоупы.

## Создание контейнера

### Синхронный контейнер

```python
from dishka import make_container

container = make_container(ProviderA(), ProviderB(), ProviderC())
```

### Асинхронный контейнер

```python
from dishka import make_async_container

container = make_async_container(ProviderA(), ProviderB())
```

**Когда использовать async-контейнер:**
- Когда хотя бы одна фабрика — `async def` или `async` генератор
- При использовании async-фреймворков (FastAPI, FastStream, aiogram)

> `AsyncContainer` умеет вызывать и sync-, и async-фабрики. `SyncContainer` — только sync.

---

## Получение зависимостей

### Синхронно

```python
settings = container.get(Settings)
settings2 = container.get(Settings)  # тот же объект (кэш в рамках скоупа)
```

### Асинхронно

```python
settings = await container.get(Settings)
```

### Синхронный get в async-контейнере

Если известно, что у зависимости нет async-фабрик в цепочке:

```python
settings = container.get_sync(Settings)
```

---

## Передача контекста при создании

Данные, которые нужны провайдерам, но создаются вне контейнера:

```python
app_config = Config.from_env()

container = make_container(
    AppProvider(),
    context={Config: app_config},  # доступно для from_context(provides=Config, scope=Scope.APP)
)
```

---

## Вход в дочерние скоупы

Каждый вызов контейнера как функции создаёт дочерний контейнер следующего скоупа:

```python
container = make_container(provider)  # APP-скоуп

# Синхронно:
with container() as req_container:    # REQUEST-скоуп
    service = req_container.get(UserService)
# При выходе — финализация всех REQUEST-зависимостей

# Асинхронно:
async with container() as req_container:
    service = await req_container.get(UserService)
```

### Передача контекста при входе в скоуп

```python
with container(context={Request: http_request}) as req_container:
    user = req_container.get(CurrentUser)
```

### Вход в конкретный скоуп

Если нужно войти не в следующий по порядку, а в конкретный (например, пропустить SESSION и войти напрямую в REQUEST):

```python
with container(scope=Scope.REQUEST) as req_container:
    ...
```

### Несколько вложенных скоупов

```python
with container() as request_container:   # APP → REQUEST
    with request_container() as action_container:  # REQUEST → ACTION
        result = action_container.get(StepResult)
```

---

## Закрытие контейнера

APP-контейнер не является контекст-менеджером — его нужно закрывать явно:

```python
container = make_container(provider)

# ... работа приложения ...

container.close()       # синхронно
await container.close() # асинхронно
```

Закрытие выполняет финализацию всех APP-зависимостей в обратном порядке их создания.

---

## Потокобезопасность

По умолчанию APP-контейнер защищён блокировкой. Вложенные контейнеры — нет.

Если несколько потоков/задач могут параллельно входить в REQUEST-скоуп и создавать зависимости предыдущего скоупа — нужна явная блокировка:

```python
import threading
import asyncio

# Для многопоточных приложений
container = make_container(provider, lock_factory=threading.Lock)
with container(lock_factory=threading.Lock) as nested:
    ...

# Для async-приложений
container = make_async_container(provider, lock_factory=asyncio.Lock)
async with container(lock_factory=asyncio.Lock) as nested:
    ...
```

> **Практика:** для типичного FastAPI или Django приложения явные блокировки не нужны — каждый HTTP-запрос создаёт свой Request-контейнер.

---

## Компоненты в контейнере

Если провайдеры разбиты на компоненты — указывайте компонент при запросе:

```python
from dishka import FromComponent

value = container.get(SomeType, component="metrics")
```

---

## Полный пример жизненного цикла

### Синхронное приложение

```python
from dishka import make_container, Provider, Scope, provide
from collections.abc import Iterable

class AppProvider(Provider):
    @provide(scope=Scope.APP)
    def get_settings(self) -> Settings:
        return Settings.from_env()

    @provide(scope=Scope.APP)
    def get_db_pool(self, settings: Settings) -> DatabasePool:
        pool = DatabasePool(settings.db_url)
        yield pool
        pool.close()  # финализация при container.close()

class RequestProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_session(self, pool: DatabasePool) -> Iterable[Session]:
        session = pool.get_session()
        yield session
        session.close()

    user_service = provide(UserService, scope=Scope.REQUEST)

# Старт приложения
container = make_container(AppProvider(), RequestProvider())

# Обработка одного запроса
with container() as req:
    service = req.get(UserService)
    service.do_something()
# Session закрыта

# Ещё один запрос — новый Session, но тот же Pool
with container() as req:
    service = req.get(UserService)

# Остановка приложения
container.close()  # Pool закрыт
```

### Асинхронное приложение

```python
import asyncio
from dishka import make_async_container

async def main():
    container = make_async_container(AppProvider(), RequestProvider())

    async with container() as req:
        service = await req.get(UserService)
        await service.process()

    await container.close()

asyncio.run(main())
```

---

## Валидация графа зависимостей

При создании контейнера dishka автоматически проверяет:
- Все зависимости зарегистрированы
- Не нарушены правила скоупов
- Нет циклических зависимостей

Можно настроить строгость проверки:

```python
from dishka import ValidationSettings, STRICT_VALIDATION

# Ручная настройка:
container = make_container(
    provider,
    validation_settings=ValidationSettings(
        nothing_overridden=True,   # ошибка, если override=True, но нечего переопределять
        implicit_override=True,    # ошибка, если фабрика переопределяет без override=True
        nothing_decorated=True,    # ошибка, если @decorate не применился ни к одной фабрике
    )
)

# Готовый строгий пресет (все проверки включены):
container = make_container(provider, validation_settings=STRICT_VALIDATION)
```

Отключение валидации (только для отладки):

```python
container = make_container(provider, skip_validation=True)
```

---

## Следующий шаг

Интеграция с FastAPI → [05-fastapi.md](./05-fastapi.md)
