# Интеграция с FastStream

> **Важно:** начиная с определённой версии dishka, интеграция с FastStream вынесена в отдельный пакет `dishka-faststream`.
>
> ```bash
> pip install dishka-faststream
> ```
>
> Тем не менее, в текущей версии репозитория интеграция ещё доступна напрямую через `dishka.integrations.faststream`. Проверьте актуальную версию.

FastStream — фреймворк для работы с брокерами сообщений (RabbitMQ, Kafka, NATS, Redis). Интеграция dishka с FastStream обеспечивает:

- Автоматическое управление REQUEST-скоупом для каждого сообщения
- Доступ к `StreamMessage` и `ContextRepo` в провайдерах
- Автоинъекцию зависимостей в обработчики сообщений

---

## Минимальный пример (RabbitMQ)

```python
from faststream import FastStream
from faststream.rabbit import RabbitBroker
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.faststream import FromDishka, setup_dishka

# --- Зависимости ---
class UserService:
    async def process(self, user_id: str) -> str:
        return f"Processed: {user_id}"

# --- Провайдер ---
class AppProvider(Provider):
    user_service = provide(UserService, scope=Scope.REQUEST)

# --- Брокер и обработчик ---
broker = RabbitBroker("amqp://guest:guest@localhost/")

@broker.subscriber("users.process")
async def handle_user(
    user_id: str,
    service: FromDishka[UserService],  # инъекция из dishka
) -> str:
    return await service.process(user_id)

# --- Приложение ---
container = make_async_container(AppProvider())
setup_dishka(container=container, app=FastStream(broker), auto_inject=True)

app = FastStream(broker)
```

---

## Пошаговое руководство

### Шаг 1. Импорты

```python
from dishka.integrations.faststream import (
    FromDishka,           # маркер параметра для инъекции
    inject,               # декоратор для ручной инъекции
    setup_dishka,         # подключение к брокеру/приложению
    FastStreamProvider,   # провайдер StreamMessage и ContextRepo
)
from dishka import make_async_container, Provider, provide, Scope
```

### Шаг 2. Провайдер

Обычный провайдер dishka. Если нужен доступ к объекту сообщения — добавьте `FastStreamProvider()`:

```python
from faststream.types import StreamMessage
from faststream import ContextRepo

class MessageProvider(Provider):
    @provide(scope=Scope.REQUEST)
    async def get_correlation_id(self, msg: StreamMessage) -> str:
        # StreamMessage доступен благодаря FastStreamProvider()
        return msg.correlation_id or ""

    @provide(scope=Scope.REQUEST)
    def get_processor(self, correlation_id: str) -> MessageProcessor:
        return MessageProcessor(correlation_id)
```

### Шаг 3. Контейнер

```python
container = make_async_container(
    MessageProvider(),
    FastStreamProvider(),  # если используете StreamMessage или ContextRepo
)
```

### Шаг 4. Подключение к брокеру

```python
from faststream import FastStream
from faststream.rabbit import RabbitBroker

broker = RabbitBroker("amqp://guest:guest@localhost/")
app = FastStream(broker)

setup_dishka(
    container=container,
    app=app,          # или broker=broker, если нет FastStream-приложения
    auto_inject=True, # автоматически инъецировать в подписчики (FastStream >= 0.5.0)
)
```

---

## Два способа инъекции

### auto_inject=True (рекомендуется для FastStream >= 0.5.0)

```python
setup_dishka(container=container, app=app, auto_inject=True)

# @inject не нужен — dishka автоматически инъецирует FromDishka-параметры
@broker.subscriber("orders.created")
async def handle_order(
    order_data: dict,
    service: FromDishka[OrderService],
) -> None:
    await service.process(order_data)
```

### Декоратор `@inject` (для всех версий, или выборочно)

```python
setup_dishka(container=container, app=app)  # без auto_inject

@broker.subscriber("orders.created")
@inject  # явная пометка
async def handle_order(
    order_data: dict,
    service: FromDishka[OrderService],
) -> None:
    await service.process(order_data)
```

---

## Использование роутеров

```python
from faststream.rabbit import RabbitRouter

orders_router = RabbitRouter()
payments_router = RabbitRouter()

@orders_router.subscriber("orders.new")
async def handle_new_order(
    body: dict,
    service: FromDishka[OrderService],
) -> None:
    await service.create(body)

@payments_router.subscriber("payments.confirm")
async def handle_payment(
    body: dict,
    service: FromDishka[PaymentService],
) -> None:
    await service.confirm(body)

broker = RabbitBroker("amqp://...")
broker.include_router(orders_router)
broker.include_router(payments_router)

app = FastStream(broker)
setup_dishka(container=container, app=app, auto_inject=True)
```

---

## Совместное использование FastStream + FastAPI

Можно использовать **один контейнер** для обоих фреймворков:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, APIRouter
from faststream import FastStream
from faststream.rabbit import RabbitBroker, RabbitRouter
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations import fastapi as fastapi_integration
from dishka.integrations import faststream as faststream_integration
from dishka.integrations.fastapi import DishkaRoute, FromDishka as FastApiFD
from dishka.integrations.faststream import inject as fs_inject, FromDishka as FastStreamFD

class SomeDependency:
    async def do_something(self) -> int:
        return 42

class AppProvider(Provider):
    dep = provide(SomeDependency, scope=Scope.REQUEST)

# FastAPI часть
http_router = APIRouter(route_class=DishkaRoute)

@http_router.get("/")
async def http_handler(dep: FastApiFD[SomeDependency]) -> int:
    return await dep.do_something()

# FastStream часть
amqp_router = RabbitRouter()

@amqp_router.subscriber("task-queue")
@fs_inject
async def amqp_handler(msg: str, dep: FastStreamFD[SomeDependency]) -> int:
    return await dep.do_something()

# Сборка приложения
def create_app() -> FastAPI:
    container = make_async_container(AppProvider())

    broker = RabbitBroker("amqp://guest:guest@localhost/")
    broker.include_router(amqp_router)
    faststream_integration.setup_dishka(container, broker=broker)

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        await broker.start()
        yield
        await broker.stop()
        await container.close()

    app = FastAPI(lifespan=lifespan)
    app.include_router(http_router)
    fastapi_integration.setup_dishka(container, app)
    return app
```

> **Важно:** `FromDishka` из `dishka.integrations.fastapi` и из `dishka.integrations.faststream` — это одно и то же (`dishka.integrations.base.FromDishka`). Можно импортировать из любого места или напрямую:
> ```python
> from dishka.integrations.base import FromDishka
> ```

---

## Тестирование FastStream с dishka

```python
import pytest
from faststream import FastStream, TestApp
from faststream.rabbit import RabbitBroker, TestRabbitBroker, RabbitRouter
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations import faststream as fs_integration
from dishka.integrations.base import FromDishka

router = RabbitRouter()

@router.subscriber("test-queue")
async def handler(msg: str, service: FromDishka[UserService]) -> str:
    return await service.process(msg)

@pytest.fixture
def mock_provider():
    class MockProvider(Provider):
        @provide(scope=Scope.REQUEST)
        async def get_service(self) -> UserService:
            return MockUserService()  # мок-реализация
    return MockProvider()

@pytest.fixture
def container(mock_provider):
    return make_async_container(mock_provider)

@pytest.fixture
async def app(container):
    broker = RabbitBroker()
    broker.include_router(router)
    faststream_app = FastStream(broker)
    fs_integration.setup_dishka(container, faststream_app, auto_inject=True)
    return faststream_app

@pytest.fixture
async def client(app):
    async with TestRabbitBroker(app.broker) as br, TestApp(app):
        yield br

@pytest.mark.asyncio
async def test_handler(client):
    result = await client.request("hello", "test-queue")
    assert await result.decode() == "processed: hello"
```

---

## Как работает интеграция внутри

1. `setup_dishka` добавляет middleware к брокеру FastStream
2. Для каждого входящего сообщения middleware создаёт REQUEST-контейнер
3. `StreamMessage` и `ContextRepo` автоматически передаются в контекст контейнера (через `FastStreamProvider`)
4. `@inject` / `auto_inject` инъецируют `FromDishka`-параметры из контейнера
5. После обработки сообщения REQUEST-контейнер закрывается → финализация зависимостей

---

## Следующий шаг

Другие интеграции (aiogram, aiohttp, Flask, Celery и др.) → [07-other-integrations.md](./07-other-integrations.md)
