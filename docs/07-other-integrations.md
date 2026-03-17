# Другие интеграции

Dishka поддерживает большое количество фреймворков. Все интеграции следуют одному и тому же паттерну:

1. Импортировать `FromDishka`, `inject`, `setup_dishka` из нужного модуля
2. Пометить параметры обработчиков через `FromDishka[Type]`
3. Вызвать `setup_dishka(container, ...)` при старте приложения

## Список всех интеграций

| Интеграция | Модуль | Тип контейнера |
|---|---|---|
| FastAPI | `dishka.integrations.fastapi` | Async |
| FastStream | `dishka.integrations.faststream` | Async |
| aiogram | `dishka.integrations.aiogram` | Async |
| aiogram-dialog | `dishka.integrations.aiogram_dialog` | Async |
| aiohttp | `dishka.integrations.aiohttp` | Async |
| Starlette | `dishka.integrations.starlette` | Async |
| Litestar | `dishka.integrations.litestar` | Async |
| Flask | `dishka.integrations.flask` | Sync |
| Sanic | `dishka.integrations.sanic` | Async |
| Celery | `dishka.integrations.celery` | Sync |
| Taskiq | `dishka.integrations.taskiq` | Async |
| Click | `dishka.integrations.click` | Sync |
| gRPC | `dishka.integrations.grpcio` | Sync/Async |
| telebot (PyTelegramBotAPI) | `dishka.integrations.telebot` | Sync/Async |
| arq | `dishka.integrations.arq` | Async |

---

## aiogram (Telegram-боты)

```python
import asyncio
from aiogram import Bot, Dispatcher, Router
from aiogram.types import Message, TelegramObject, User
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.aiogram import (
    AiogramMiddlewareData,  # dict с данными middleware aiogram
    AiogramProvider,        # регистрирует TelegramObject и AiogramMiddlewareData
    FromDishka,
    inject,
    setup_dishka,
)

# --- Провайдер ---
class BotProvider(Provider):
    @provide(scope=Scope.APP)
    async def get_some_value(self) -> int:
        return 42

    @provide(scope=Scope.REQUEST)
    async def get_user(self, obj: TelegramObject) -> User:
        # TelegramObject — текущий апдейт (Message, CallbackQuery и т.д.)
        return obj.from_user

# --- Роутер и обработчики ---
router = Router()

@router.message()
@inject  # или auto_inject=True в setup_dishka
async def handle_message(
    message: Message,
    user: FromDishka[User],
    value: FromDishka[int],
) -> None:
    await message.answer(f"Привет, {user.full_name}! Твоё число: {value}")

# --- Запуск ---
async def main():
    bot = Bot(token="YOUR_BOT_TOKEN")
    dp = Dispatcher()
    dp.include_router(router)

    container = make_async_container(BotProvider(), AiogramProvider())
    setup_dishka(container=container, router=dp, auto_inject=True)

    try:
        await dp.start_polling(bot)
    finally:
        await container.close()
        await bot.session.close()

asyncio.run(main())
```

**Скоупы в aiogram:**
- `APP` — Bot, Dispatcher, долгоживущие объекты
- `REQUEST` — один Telegram-апдейт (Message, CallbackQuery, etc.)

**Доступные данные через `AiogramProvider`:**
- `TelegramObject` — текущий апдейт
- `AiogramMiddlewareData` — словарь с данными middleware (event_chat, event_from_user и т.д.)

---

## aiohttp

```python
from aiohttp import web
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.aiohttp import (
    DISHKA_CONTAINER_KEY,
    AiohttpProvider,  # регистрирует Request в SESSION-скоупе
    FromDishka,
    inject,
    setup_dishka,
)

class AppProvider(Provider):
    @provide(scope=Scope.APP)
    def get_db(self) -> Database:
        return Database()

    user_service = provide(UserService, scope=Scope.REQUEST)

@inject
async def handle_request(
    request: web.Request,
    service: FromDishka[UserService],
) -> web.Response:
    users = await service.list()
    return web.json_response({"users": users})

async def on_startup(app: web.Application) -> None:
    container = make_async_container(AppProvider(), AiohttpProvider())
    setup_dishka(container=container, app=app)

async def on_cleanup(app: web.Application) -> None:
    await app[DISHKA_CONTAINER_KEY].close()

app = web.Application()
app.router.add_get("/users", handle_request)
app.on_startup.append(on_startup)
app.on_cleanup.append(on_cleanup)
```

**Особенность aiohttp:** WebSocket-запросы автоматически переходят в `SESSION`-скоуп, обычные HTTP — в `REQUEST`-скоуп.

---

## Flask

Flask — синхронный фреймворк, поэтому используется `make_container` (не async).

```python
from flask import Flask
from dishka import Provider, Scope, provide, make_container
from dishka.integrations.flask import (
    FlaskProvider,  # регистрирует flask.Request в REQUEST-скоупе
    FromDishka,
    inject,
    setup_dishka,
)

class AppProvider(Provider):
    db = provide(Database, scope=Scope.APP)
    user_service = provide(UserService, scope=Scope.REQUEST)

app = Flask(__name__)

@app.get("/users")
@inject
def list_users(service: FromDishka[UserService]) -> dict:
    return {"users": service.list()}

container = make_container(AppProvider(), FlaskProvider())
setup_dishka(container=container, app=app)

if __name__ == "__main__":
    app.run()
```

---

## Celery

```python
from celery import Celery
from dishka import Provider, Scope, provide, make_container
from dishka.integrations.celery import (
    FromDishka,
    inject,
    setup_dishka,
    DishkaTask,  # базовый класс для задач с автоинъекцией
)

class TaskProvider(Provider):
    db = provide(Database, scope=Scope.APP)
    email_service = provide(EmailService, scope=Scope.REQUEST)

celery_app = Celery("myapp", broker="redis://localhost/0")

# Способ 1: через DishkaTask (автоинъекция)
@celery_app.task(base=DishkaTask)
def send_email(user_id: int, service: FromDishka[EmailService]) -> None:
    service.send(user_id)

# Способ 2: через @inject
@celery_app.task
@inject
def process_order(order_id: int, db: FromDishka[Database]) -> None:
    db.process(order_id)

container = make_container(TaskProvider())
setup_dishka(container=container, app=celery_app)
```

**Скоупы в Celery:**
- `APP` — глобальные ресурсы (пул соединений, конфиг)
- `REQUEST` — один таск

---

## Click (CLI-приложения)

```python
import click
from dishka import Provider, Scope, provide, make_container
from dishka.integrations.click import FromDishka, inject, setup_dishka

class CliProvider(Provider):
    db = provide(Database, scope=Scope.APP)
    migrator = provide(Migrator, scope=Scope.REQUEST)

@click.group()
@click.pass_context
def cli(ctx: click.Context) -> None:
    container = make_container(CliProvider())
    setup_dishka(container=container, context=ctx, auto_inject=True)

@cli.command()
@inject
def migrate(migrator: FromDishka[Migrator]) -> None:
    migrator.run()

@cli.command()
@click.argument("table")
@inject
def show_table(table: str, db: FromDishka[Database]) -> None:
    db.show(table)

if __name__ == "__main__":
    cli()
```

---

## Litestar

```python
from litestar import Litestar, get
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.litestar import (
    FromDishka,
    inject,
    setup_dishka,
)

class AppProvider(Provider):
    service = provide(UserService, scope=Scope.REQUEST)

@get("/users")
@inject
async def list_users(service: FromDishka[UserService]) -> list:
    return await service.list()

container = make_async_container(AppProvider())
app = Litestar(route_handlers=[list_users])
setup_dishka(container=container, app=app)
```

---

## Starlette

```python
from starlette.applications import Starlette
from starlette.requests import Request
from starlette.responses import JSONResponse
from starlette.routing import Route
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.starlette import (
    FromDishka,
    inject,
    setup_dishka,
)

class AppProvider(Provider):
    service = provide(UserService, scope=Scope.REQUEST)

@inject
async def list_users(
    request: Request,
    service: FromDishka[UserService],
) -> JSONResponse:
    return JSONResponse({"users": await service.list()})

container = make_async_container(AppProvider())
app = Starlette(routes=[Route("/users", list_users)])
setup_dishka(container=container, app=app)
```

---

## Taskiq

```python
from taskiq import TaskiqScheduler
from taskiq_redis import RedisAsyncResultBackend, ListQueueBroker
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.taskiq import (
    FromDishka,
    inject,
    setup_dishka,
)

broker = ListQueueBroker("redis://localhost/0")

class TaskProvider(Provider):
    service = provide(ProcessingService, scope=Scope.REQUEST)

@broker.task
@inject
async def process_item(item_id: int, service: FromDishka[ProcessingService]) -> str:
    return await service.process(item_id)

container = make_async_container(TaskProvider())
setup_dishka(container=container, broker=broker)
```

---

## gRPC

```python
from grpc import server as grpc_server
from concurrent.futures import ThreadPoolExecutor
from dishka import Provider, Scope, provide, make_container
from dishka.integrations.grpcio import (
    DishkaInterceptor,   # sync-перехватчик
    FromDishka,
    GrpcioProvider,
    inject,
)

class AppProvider(Provider):
    service = provide(UserService, scope=Scope.REQUEST)

class UserServicer(UserServiceServicer):
    @inject
    def GetUser(
        self,
        request: GetUserRequest,
        context: ServicerContext,
        user_service: FromDishka[UserService],
    ) -> UserResponse:
        return user_service.get(request.user_id)

container = make_container(AppProvider(), GrpcioProvider())

interceptor = DishkaInterceptor(container)
server = grpc_server(
    ThreadPoolExecutor(),
    interceptors=[interceptor],
)
add_UserServiceServicer_to_server(UserServicer(), server)
```

Для async-gRPC используйте `DishkaAioInterceptor` и `make_async_container`.

---

## Следующий шаг

Продвинутые темы: компоненты, дженерики, визуализация → [08-advanced.md](./08-advanced.md)
