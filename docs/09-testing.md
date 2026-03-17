# Тестирование приложений с dishka

Одно из главных преимуществ DI — лёгкость тестирования. Dishka спроектирована так, чтобы подмена зависимостей на моки была максимально простой.

## Ключевой принцип

**Контейнер не является глобальным синглтоном.** Это обычный объект, который вы создаёте и закрываете явно. В тестах просто создайте другой контейнер — с мок-провайдерами.

```
Продакшн: make_container(RealDBProvider(), RealCacheProvider(), ServiceProvider())
Тесты:    make_container(MockDBProvider(),  MockCacheProvider(), ServiceProvider())
```

---

## Стратегии подмены зависимостей

### 1. Полная замена провайдера

Самый явный способ — написать отдельный провайдер с мок-реализациями:

```python
from unittest.mock import AsyncMock, MagicMock
from dishka import Provider, Scope, provide, make_async_container

# Продакшн-провайдер
class DBProvider(Provider):
    @provide(scope=Scope.REQUEST)
    async def get_session(self, pool: AsyncEngine) -> AsyncIterator[AsyncSession]:
        async with AsyncSession(pool) as session:
            yield session

# Мок-провайдер для тестов
class MockDBProvider(Provider):
    @provide(scope=Scope.APP)
    def get_session(self) -> AsyncSession:
        mock = MagicMock(spec=AsyncSession)
        mock.execute = AsyncMock(return_value=...)
        return mock

# В тесте
container = make_async_container(MockDBProvider(), ServiceProvider())
```

### 2. Переопределение конкретной зависимости через override=True

Когда хочется взять основную конфигурацию, но заменить только одну зависимость:

```python
class MockUserRepoProvider(Provider):
    scope = Scope.APP

    @provide(override=True)  # переопределяет UserRepository из основного провайдера
    def get_user_repo(self) -> UserRepository:
        mock = MagicMock(spec=UserRepository)
        mock.find_by_id.return_value = User(id=1, name="Test User")
        return mock

container = make_async_container(
    MainProvider(),       # основные зависимости
    MockUserRepoProvider(), # переопределяем только UserRepository
)
```

### 3. Передача мока через from_context

Самый простой способ подменить одно значение без создания нового провайдера:

```python
from dishka import from_context, Provider, Scope, make_async_container

class TestableProvider(Provider):
    scope = Scope.APP
    # В тестах Config можно передать снаружи
    config = from_context(provides=Config, override=True)

test_config = Config(db_url="sqlite:///:memory:", secret_key="test")
container = make_async_container(
    MainProvider(),
    TestableProvider(),
    context={Config: test_config},
)
```

---

## Тестирование FastAPI-приложений

### Разделение фабрики приложения и настройки контейнера

Ключевой паттерн: **вынесите `setup_dishka` из основного кода** в отдельную функцию.

```python
# app/main.py
from fastapi import FastAPI
from dishka.integrations.fastapi import setup_dishka

def create_app() -> FastAPI:
    """Создаёт приложение БЕЗ настройки контейнера."""
    app = FastAPI()
    app.include_router(users_router)
    app.include_router(orders_router)
    return app

def create_production_app() -> FastAPI:
    """Создаёт приложение с продакшн-контейнером."""
    app = create_app()
    container = make_async_container(
        DBProvider(),
        CacheProvider(),
        ServiceProvider(),
    )
    setup_dishka(container=container, app=app)
    return app
```

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from unittest.mock import AsyncMock, MagicMock
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations.fastapi import setup_dishka
from app.main import create_app

class MockDBProvider(Provider):
    @provide(scope=Scope.APP)
    def get_db_session(self) -> AsyncSession:
        return MagicMock(spec=AsyncSession)

@pytest.fixture
def container():
    c = make_async_container(MockDBProvider(), ServiceProvider())
    yield c
    # Не забудьте закрыть контейнер после теста

@pytest.fixture
def client(container):
    app = create_app()
    setup_dishka(container, app)
    with TestClient(app) as client:
        yield client

# Если нужен прямой доступ к зависимости в тесте:
@pytest.fixture
async def db_session(container):
    return await container.get(AsyncSession)
```

```python
# tests/test_users.py
from unittest.mock import MagicMock

def test_list_users(client):
    response = client.get("/users")
    assert response.status_code == 200
    assert isinstance(response.json(), list)

def test_create_user(client, db_session):
    # Настраиваем мок перед тестом
    db_session.execute = MagicMock(return_value=...)

    response = client.post("/users", json={"name": "Alice"})
    assert response.status_code == 201
    db_session.execute.assert_called_once()
```

---

## Тестирование без фреймворка

Юнит-тесты часто не требуют контейнера вообще — используйте DI напрямую:

```python
# tests/test_user_service.py
from unittest.mock import AsyncMock

@pytest.mark.asyncio
async def test_get_user_returns_none_if_not_found():
    # Создаём объект напрямую — никакого контейнера не нужно
    mock_repo = AsyncMock(spec=UserRepository)
    mock_repo.find_by_id.return_value = None

    service = UserService(repo=mock_repo)
    result = await service.get_user(user_id=999)

    assert result is None
    mock_repo.find_by_id.assert_called_once_with(999)
```

Контейнер нужен в тестах только для **интеграционного тестирования** — когда проверяется работа нескольких слоёв вместе (обработчик → сервис → репозиторий).

---

## Получение мока из контейнера в тесте

Иногда удобно создать мок внутри провайдера, а потом получить его в тесте:

```python
class MockDBProvider(Provider):
    @provide(scope=Scope.APP)
    def get_connection(self) -> Connection:
        mock = MagicMock(spec=Connection)
        mock.execute = MagicMock(return_value="1")
        return mock

@pytest.fixture
def container():
    c = make_async_container(MockDBProvider())
    yield c

@pytest.fixture
async def connection(container):
    # Получаем мок напрямую из контейнера
    return await container.get(Connection)

async def test_handler_calls_db(client, connection):
    response = client.get("/")
    assert response.status_code == 200
    connection.execute.assert_called()  # проверяем, что мок был вызван
```

---

## Тестирование с реальными зависимостями (интеграционные тесты)

Для интеграционных тестов используют настоящие зависимости (тестовую БД, например):

```python
# tests/conftest.py
import pytest
from dishka import make_async_container

@pytest.fixture(scope="session")
def test_settings():
    return Settings(
        db_url="postgresql+asyncpg://user:pass@localhost/test_db",
        debug=True,
    )

@pytest.fixture(scope="session")
async def session_container(test_settings):
    """Контейнер на всю сессию тестов — для дорогих ресурсов."""
    c = make_async_container(
        InfraProvider(),
        context={Settings: test_settings},
    )
    yield c
    await c.close()

@pytest.fixture
async def db_session(session_container):
    """Свежая сессия на каждый тест."""
    async with session_container() as req_container:
        session = await req_container.get(AsyncSession)
        yield session
        await session.rollback()  # откат изменений после теста
```

---

## Тестирование FastStream с dishka

```python
import pytest
from faststream import FastStream, TestApp
from faststream.rabbit import RabbitBroker, TestRabbitBroker
from dishka import Provider, Scope, provide, make_async_container
from dishka.integrations import faststream as fs_integration
from dishka.integrations.base import FromDishka

# Роутер с обработчиком
@router.subscriber("orders.new")
async def handle_order(
    data: dict,
    service: FromDishka[OrderService],
) -> dict:
    return await service.process(data)

@pytest.fixture
def mock_provider():
    class MockOrderServiceProvider(Provider):
        @provide(scope=Scope.REQUEST)
        def get_service(self) -> OrderService:
            mock = MagicMock(spec=OrderService)
            mock.process = AsyncMock(return_value={"status": "ok"})
            return mock
    return MockOrderServiceProvider()

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
async def test_handle_order(client):
    result = await client.request({"item_id": 1}, "orders.new")
    decoded = await result.decode()
    assert decoded["status"] == "ok"
```

---

## Советы по тестированию с dishka

| Совет | Объяснение |
|---|---|
| Разделите `create_app()` и `setup_dishka()` | Это позволит тестам подключать свой контейнер |
| Не делайте контейнер глобальным | Каждый тест (или сессия) должен иметь свой контейнер |
| Закрывайте контейнер в fixture-teardown | Иначе ресурсы (соединения) не освободятся |
| Для юнит-тестов контейнер не нужен | Создавайте объекты напрямую через конструктор |
| `override=True` + отдельный провайдер | Лучший способ точечно заменить одну зависимость |
| Используйте `scope="session"` для дорогих ресурсов | Пул соединений не нужно пересоздавать для каждого теста |

---

## Что дальше

Вы прочитали всю документацию! Для углублённого изучения:

- Официальная документация: [dishka.readthedocs.io](https://dishka.readthedocs.io)
- Примеры в репозитории: папка `examples/`
- Real-world пример с FastAPI + aiogram: `examples/real_world/`
