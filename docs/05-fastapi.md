# Интеграция с FastAPI

dishka предоставляет полноценную интеграцию с FastAPI: автоматическое управление скоупами, инъекция зависимостей в обработчики, поддержка WebSocket и обратная совместимость с `fastapi.Depends`.

## Установка

```bash
pip install dishka fastapi uvicorn
```

## Минимальный пример

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, APIRouter
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.fastapi import DishkaRoute, FromDishka, setup_dishka

# --- Зависимости ---
class Database:
    def query(self, sql: str): ...

class UserService:
    def __init__(self, db: Database): ...
    def get_users(self): return []

# --- Провайдер ---
class AppProvider(Provider):
    db = provide(Database, scope=Scope.APP)
    user_service = provide(UserService, scope=Scope.REQUEST)

# --- Роутер с автоинъекцией ---
router = APIRouter(route_class=DishkaRoute)

@router.get("/users")
async def get_users(service: FromDishka[UserService]) -> list:
    return service.get_users()

# --- Приложение ---
@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await app.state.dishka_container.close()

app = FastAPI(lifespan=lifespan)
app.include_router(router)

container = make_async_container(AppProvider())
setup_dishka(container=container, app=app)
```

---

## Пошаговое руководство

### Шаг 1. Импорты

```python
from dishka.integrations.fastapi import (
    DishkaRoute,      # route_class для автоинъекции
    FromDishka,       # маркер параметра для инъекции
    FastapiProvider,  # провайдер, который регистрирует Request/WebSocket
    inject,           # декоратор для ручной инъекции
    setup_dishka,     # подключение к FastAPI-приложению
)
from dishka import make_async_container, Provider, provide, Scope
```

### Шаг 2. Создание провайдера

Стандартный провайдер dishka. Можно использовать `fastapi.Request` как зависимость в провайдере — для этого передайте `FastapiProvider()` при создании контейнера:

```python
from fastapi import Request

class MyProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_gateway(self, request: Request) -> Gateway:
        # Request доступен, если добавлен FastapiProvider()
        token = request.headers.get("Authorization", "")
        return Gateway(token)
```

### Шаг 3. Создание контейнера

```python
container = make_async_container(
    MyProvider(),
    FastapiProvider(),  # нужен, если используете Request/WebSocket в провайдерах
)
```

### Шаг 4. Настройка lifespan (закрытие контейнера)

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await app.state.dishka_container.close()

app = FastAPI(lifespan=lifespan)
```

### Шаг 5. Подключение dishka к приложению

```python
setup_dishka(container=container, app=app)
```

Это добавляет middleware, который автоматически создаёт REQUEST-контейнер для каждого HTTP-запроса.

---

## Два способа инъекции

### Способ 1: DishkaRoute (рекомендуется для HTTP)

Задайте `route_class=DishkaRoute` на уровне роутера — все обработчики в нём получат автоинъекцию:

```python
router = APIRouter(route_class=DishkaRoute)

@router.get("/items/{item_id}")
async def get_item(
    item_id: int,                          # обычный параметр FastAPI
    service: FromDishka[ItemService],      # инъекция из dishka
) -> ItemResponse:
    return await service.get(item_id)
```

**Плюсы:** не нужен декоратор `@inject` на каждом обработчике.
**Ограничение:** работает только для HTTP-эндпоинтов, не для WebSocket.

### Способ 2: Декоратор `@inject`

Явная пометка конкретного обработчика:

```python
from dishka.integrations.fastapi import inject

router = APIRouter()  # обычный роутер без DishkaRoute

@router.get("/items")
@inject
async def list_items(
    service: FromDishka[ItemService],
) -> list[ItemResponse]:
    return await service.list_all()
```

**Используйте `@inject` для:**
- WebSocket-обработчиков
- Отдельных эндпоинтов, когда не хочется менять `route_class` у всего роутера

---

## WebSocket

Для WebSocket скоуп — `SESSION` (а не REQUEST):

```python
from fastapi import WebSocket
from dishka.integrations.fastapi import inject

@router.websocket("/ws")
@inject  # обязательно для WebSocket
async def websocket_handler(
    ws: WebSocket,
    chat_service: FromDishka[ChatService],  # SESSION-скоуп
):
    await ws.accept()
    while True:
        data = await ws.receive_text()
        response = await chat_service.process(data)
        await ws.send_text(response)
```

Провайдер для WebSocket-зависимостей:

```python
from fastapi import WebSocket

class WsProvider(Provider):
    # WebSocket доступен, если добавлен FastapiProvider()
    @provide(scope=Scope.SESSION)
    def get_chat_service(self, ws: WebSocket) -> ChatService:
        return ChatService(ws.query_params.get("room"))
```

---

## Совместимость с fastapi.Depends

Если в части кода используется `Depends` — это работает параллельно с dishka:

```python
from fastapi import Depends

def get_current_user(token: str = Header(...)) -> str:
    return decode_token(token)

@router.get("/profile")
@inject
async def profile(
    user_id: str = Depends(get_current_user),     # через fastapi.Depends
    service: FromDishka[UserService] = ...,        # через dishka
) -> ProfileResponse:
    return await service.get_profile(user_id)
```

---

## Синхронный контейнер

Если по каким-то причинам используете `make_container` (не async):

```python
from dishka import make_container
from dishka.integrations.fastapi import inject_sync, DishkaSyncRoute, setup_dishka

router = APIRouter(route_class=DishkaSyncRoute)

@router.get("/")
@inject_sync  # вместо @inject
def sync_endpoint(service: FromDishka[SomeService]) -> str:
    return service.get_data()

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    app.state.dishka_container.close()  # не await!

app = FastAPI(lifespan=lifespan)
container = make_container(SyncProvider(), FastapiProvider())
setup_dishka(container=container, app=app)
```

---

## Реальный пример: слоистая архитектура

```python
# providers.py
from collections.abc import AsyncIterator
from sqlalchemy.ext.asyncio import AsyncSession, AsyncEngine, create_async_engine

class InfraProvider(Provider):
    @provide(scope=Scope.APP)
    def get_engine(self, settings: Settings) -> AsyncIterator[AsyncEngine]:
        engine = create_async_engine(settings.db_url)
        yield engine
        await engine.dispose()

    @provide(scope=Scope.REQUEST)
    async def get_session(self, engine: AsyncEngine) -> AsyncIterator[AsyncSession]:
        async with AsyncSession(engine) as session:
            yield session

class RepositoryProvider(Provider):
    scope = Scope.REQUEST
    user_repo = provide(UserRepository)
    order_repo = provide(OrderRepository)

class ServiceProvider(Provider):
    scope = Scope.REQUEST
    user_service = provide(UserService)
    order_service = provide(OrderService)

# main.py
@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await app.state.dishka_container.close()

def create_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.include_router(user_router)
    app.include_router(order_router)

    settings = Settings.from_env()
    container = make_async_container(
        InfraProvider(),
        RepositoryProvider(),
        ServiceProvider(),
        FastapiProvider(),
        context={Settings: settings},
    )
    setup_dishka(container=container, app=app)
    return app
```

```python
# routers/users.py
router = APIRouter(prefix="/users", tags=["users"], route_class=DishkaRoute)

@router.get("/")
async def list_users(service: FromDishka[UserService]) -> list[UserResponse]:
    return await service.list()

@router.post("/")
async def create_user(
    body: CreateUserRequest,
    service: FromDishka[UserService],
) -> UserResponse:
    return await service.create(body)
```

---

## Как работает интеграция внутри

1. `setup_dishka` добавляет middleware (`ContainerMiddleware`) к FastAPI-приложению
2. На каждый HTTP-запрос middleware создаёт `REQUEST`-контейнер (`async with app_container() as req_container`)
3. Контейнер сохраняется в `request.state.dishka_container`
4. `@inject` / `DishkaRoute` извлекают контейнер из `request.state` и вызывают `await container.get(Type)` для каждого `FromDishka[Type]`-параметра
5. При завершении запроса middleware закрывает REQUEST-контейнер → финализация зависимостей

---

## Следующий шаг

Интеграция с FastStream → [06-faststream.md](./06-faststream.md)
