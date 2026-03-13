# Скоупы (Scopes)

Скоуп определяет **время жизни** зависимости. Это главная особенность dishka по сравнению с другими DI-библиотеками.

## Встроенные скоупы

Полная цепочка встроенных скоупов (в порядке убывания времени жизни):

```
[RUNTIME] → APP → [SESSION] → REQUEST → ACTION → STEP
```

Квадратные скобки означают, что скоуп по умолчанию **пропускается** (skipped). Подробнее — ниже.

### APP — синглтон на всё приложение

```python
class InfraProvider(Provider):
    @provide(scope=Scope.APP)
    def get_settings(self) -> Settings:
        return Settings.from_env()

    @provide(scope=Scope.APP)
    def get_db_pool(self, settings: Settings) -> DatabasePool:
        return DatabasePool(settings.db_url)
```

Объект создаётся один раз при старте и уничтожается при `container.close()`.

### REQUEST — на один запрос/событие

```python
class RequestProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_session(self, pool: DatabasePool) -> Iterable[Session]:
        session = pool.acquire()
        yield session
        session.release()

    user_service = provide(UserService, scope=Scope.REQUEST)
```

Объект создаётся при входе в `with container() as req_container` и уничтожается при выходе.

### ACTION и STEP — дополнительные короткие скоупы

Используются редко — для случаев, когда внутри одного запроса нужны ещё более короткоживущие объекты:

```python
class WorkflowProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_job_context(self) -> JobContext:
        return JobContext()

    @provide(scope=Scope.ACTION)
    def get_step_processor(self, ctx: JobContext) -> StepProcessor:
        return StepProcessor(ctx)
```

```python
with container() as request_container:
    with request_container() as action_container:  # входим в ACTION
        processor = action_container.get(StepProcessor)
```

---

## Пропускаемые скоупы (Skipped Scopes)

`RUNTIME` и `SESSION` по умолчанию пропускаются — то есть при стандартном использовании вы их не замечаете, но они существуют и в них можно размещать зависимости.

### Как это работает

При вызове `make_container(provider)` вы автоматически входите в `APP`-скоуп. `RUNTIME` при этом тоже активен — его зависимости доступны, но явно в него входить не нужно.

```python
container = make_container(provider)
# RUNTIME активен неявно, APP — основной скоуп
```

Если нужен явный доступ к `RUNTIME`:

```python
container = make_container(provider, start_scope=Scope.RUNTIME)
with container() as app_container:
    # RUNTIME → APP
    ...
```

### SESSION — для WebSocket и долгих соединений

```python
container = make_container(provider)

# Обычный HTTP-запрос: APP → REQUEST (SESSION пропускается)
with container() as request_container:
    ...

# WebSocket: APP → SESSION → REQUEST
with container(scope=Scope.SESSION) as session_container:
    with session_container() as request_container:
        # WebSocket-соединение → отдельный REQUEST для каждого сообщения
        ...
```

> **Практика:** `RUNTIME` удобен для зависимостей, которые должны жить между пересозданиями приложения в тестах. `SESSION` — для данных WebSocket-сессии.

---

## Правила зависимостей между скоупами

```
✅ Можно:    REQUEST зависит от APP
✅ Можно:    REQUEST зависит от REQUEST
✅ Можно:    ACTION зависит от APP или REQUEST

❌ Нельзя:   APP зависит от REQUEST
❌ Нельзя:   APP зависит от ACTION
```

Если нарушить это правило — dishka выбросит ошибку при создании контейнера.

```python
class WrongProvider(Provider):
    config = provide(Config, scope=Scope.APP)
    service = provide(UserService, scope=Scope.APP)

class RequestProvider(Provider):
    request = from_context(provides=Request, scope=Scope.REQUEST)

    @provide(scope=Scope.REQUEST)
    def get_user(self, req: Request) -> User:
        return req.user

# UserService в APP не может зависеть от User в REQUEST
# → NoFactoryError или ScopeViolationError при make_container
```

---

## Конкурентный доступ и блокировки

По умолчанию `APP`-контейнер **потокобезопасен** (lock установлен автоматически). Вложенные контейнеры (REQUEST и ниже) не защищены блокировкой — каждый запрос должен иметь свой контейнер.

Если нужна параллельная работа с вложенными скоупами:

```python
import asyncio

container = make_async_container(
    provider,
    lock_factory=asyncio.Lock,  # для APP-уровня
)

async with container(lock_factory=asyncio.Lock) as req_container:
    # этот контейнер тоже будет защищён блокировкой
    ...
```

Для многопоточности (синхронный контейнер):

```python
import threading

container = make_container(provider, lock_factory=threading.Lock)
with container(lock_factory=threading.Lock) as req_container:
    ...
```

---

## Кастомные скоупы

Если стандартная цепочка `APP → REQUEST → ACTION → STEP` не подходит — создайте свою:

```python
from dishka import Provider, make_container, BaseScope, new_scope

class MyScope(BaseScope):
    # Порядок объявления важен — от широкого к узкому
    APP = new_scope("app")
    COMMAND = new_scope("command")
    HANDLER = new_scope("handler")

class MyProvider(Provider):
    @provide(scope=MyScope.APP)
    def get_bot(self) -> Bot:
        return Bot(token="...")

    @provide(scope=MyScope.COMMAND)
    def get_command_context(self) -> CommandContext:
        return CommandContext()

    @provide(scope=MyScope.HANDLER)
    def get_handler(self, ctx: CommandContext) -> Handler:
        return Handler(ctx)

container = make_container(MyProvider(), scopes=MyScope)
with container() as command_container:   # APP → COMMAND
    with command_container() as handler_container:  # COMMAND → HANDLER
        handler = handler_container.get(Handler)
```

---

## Типичные паттерны распределения по скоупам

### Веб-приложение (FastAPI, Flask)

| Объект | Скоуп | Причина |
|--------|-------|---------|
| `Settings` / `Config` | APP | Один конфиг на всё приложение |
| `DatabasePool` / `Engine` | APP | Пул соединений — один |
| `HTTPClient` (httpx, aiohttp) | APP | Переиспользуем сессию |
| `DatabaseSession` | REQUEST | Новая сессия на каждый запрос |
| `UnitOfWork` | REQUEST | Транзакция на запрос |
| `UserService`, `OrderService` | REQUEST | Бизнес-логика привязана к запросу |
| `CurrentUser` | REQUEST | Из контекста запроса (`from_context`) |

### Telegram-бот (aiogram)

| Объект | Скоуп | Причина |
|--------|-------|---------|
| `Bot` | APP | Один экземпляр бота |
| `Dispatcher` | APP | Один диспетчер |
| `SessionPool` | APP | Пул соединений с БД |
| `Session` | REQUEST | Одна сессия на один апдейт |
| `UserService` | REQUEST | Обработка одного апдейта |
| `Message` / `Update` | REQUEST | Через `from_context` |

---

## Следующий шаг

Подробнее о контейнере и его API → [04-container.md](./04-container.md)
