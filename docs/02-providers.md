# Provider: инструменты регистрации зависимостей

Provider — это класс, в котором вы описываете, как создавать зависимости. Для этого в dishka есть пять основных инструментов.

## Обзор инструментов

| Инструмент | Когда использовать |
|---|---|
| `provide` | Основной способ регистрации фабрики |
| `alias` | Получение одного объекта по нескольким типам |
| `decorate` | Обёртка/модификация существующей зависимости |
| `from_context` | Зависимость передаётся вручную при входе в скоуп |
| `provide_all` | Регистрация нескольких классов за раз |

---

## `provide` — регистрация фабрики

Основной инструмент. Используется как декоратор метода или как функция для передачи класса.

### Декоратор метода

```python
from dishka import Provider, Scope, provide

class MyProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_service(self) -> UserService:
        return UserService()
```

Аннотация возвращаемого типа обязательна — это ключ, по которому контейнер найдёт фабрику.

### Параметры метода — это зависимости

```python
class MyProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_service(self, db: Database, cache: Redis) -> UserService:
        # db и cache будут автоматически разрешены контейнером
        return UserService(db, cache)
```

### Скоуп по умолчанию для всего провайдера

Чтобы не указывать `scope=` у каждой фабрики — задайте его на уровне класса:

```python
class MyProvider(Provider):
    scope = Scope.REQUEST  # скоуп по умолчанию

    @provide  # использует Scope.REQUEST
    def get_service(self) -> UserService:
        return UserService()

    @provide(scope=Scope.APP)  # явный скоуп переопределяет дефолтный
    def get_settings(self) -> Settings:
        return Settings.from_env()
```

### Передача класса напрямую

Если никакой логики создания нет — dishka сам вызовет `__init__` и разрешит все параметры:

```python
class MyProvider(Provider):
    scope = Scope.REQUEST

    # Эти две записи эквивалентны:
    service = provide(UserService)

    @provide
    def get_service(self) -> UserService:
        return UserService()  # UserService.__init__ без аргументов
```

### Регистрация через интерфейс (provides=)

Когда нужно зарегистрировать конкретную реализацию по абстрактному типу:

```python
class UserRepository(Protocol): ...
class SQLUserRepository(UserRepository): ...

class MyProvider(Provider):
    scope = Scope.REQUEST

    # container.get(UserRepository) вернёт SQLUserRepository
    user_repo = provide(SQLUserRepository, provides=UserRepository)
```

### Финализация через yield

Для ресурсов, которые нужно освобождать (соединения, сессии, файлы):

```python
from collections.abc import Iterable
import sqlite3

class MyProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_connection(self) -> Iterable[sqlite3.Connection]:
        conn = sqlite3.connect(":memory:")
        yield conn          # ← отдаём объект
        conn.close()        # ← выполняется при выходе из скоупа
```

Для async-версии:

```python
from collections.abc import AsyncIterator
from sqlalchemy.ext.asyncio import AsyncSession

class MyProvider(Provider):
    @provide(scope=Scope.REQUEST)
    async def get_session(self, pool: AsyncEngine) -> AsyncIterator[AsyncSession]:
        async with AsyncSession(pool) as session:
            yield session
        # session.close() вызывается автоматически через контекст-менеджер
```

### Обработка ошибок в генераторе

Если в процессе обработки запроса произошла ошибка — она передаётся в генератор:

```python
class MyProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def get_session(self) -> Iterable[Session]:
        session = Session()
        exc = yield session
        if exc:
            session.rollback()
        else:
            session.commit()
        session.close()
```

### Отключение кэширования

По умолчанию зависимость создаётся один раз в рамках скоупа. Чтобы создавать новый объект при каждом запросе:

```python
class MyProvider(Provider):
    @provide(scope=Scope.REQUEST, cache=False)
    def get_uuid(self) -> UUID:
        return uuid4()  # новый UUID на каждый container.get(UUID)
```

### AnyOf — один объект по нескольким типам

```python
from dishka import AnyOf, provide, Provider, Scope

class UserDAOImpl(UserReader, UserWriter): ...

class MyProvider(Provider):
    @provide(scope=Scope.APP)
    def get_user_dao(self) -> AnyOf[UserDAOImpl, UserReader, UserWriter]:
        return UserDAOImpl()
```

Теперь `container.get(UserDAOImpl)`, `container.get(UserReader)` и `container.get(UserWriter)` вернут один и тот же объект.

### WithParents — автоматически регистрировать все родительские типы

```python
from dishka import WithParents, provide, Provider, Scope

class UserDAOImpl(UserReader, UserWriter, UserDeleter): ...

class MyProvider(Provider):
    @provide(scope=Scope.APP)
    def get_user_dao(self) -> WithParents[UserDAOImpl]:
        return UserDAOImpl()
# Эквивалентно AnyOf[UserDAOImpl, UserReader, UserWriter, UserDeleter]
# Игнорирует: object, type, ABC, Protocol, Generic, Exception, Enum
```

### override=True — переопределение фабрики

Используется при тестировании или расширении поведения:

```python
class MockProvider(Provider):
    scope = Scope.APP

    user_dao_mock = provide(UserDAOMock, provides=UserDAO, override=True)
```

### recursive=True — автоматическая регистрация зависимостей

```python
class MyProvider(Provider):
    external_client = provide(
        ExternalAPIClientImpl,
        provides=ExternalAPIClient,
        scope=Scope.REQUEST,
        recursive=True,  # все зависимости ExternalAPIClientImpl регистрируются автоматически
    )
```

---

## `alias` — один объект по другому типу

Позволяет получать один и тот же объект по нескольким типам без создания нового объекта.

```python
from dishka import alias, provide, Provider, Scope

class UserRepository(Protocol): ...
class UserDAOImpl(UserRepository): ...

class MyProvider(Provider):
    scope = Scope.REQUEST

    user_dao = provide(UserDAOImpl)

    # container.get(UserRepository) вернёт тот же объект, что и container.get(UserDAOImpl)
    user_dao_alias = alias(source=UserDAOImpl, provides=UserRepository)
```

**Отличие от `provides=`:** `alias` — это отдельная запись, которую можно добавить из другого провайдера; `provides=` задаётся прямо в фабрике.

### Переопределение alias

```python
class TestProvider(Provider):
    scope = Scope.APP

    mock_dao = provide(UserDAOMock)
    # Переопределяем существующий alias для UserRepository
    override_alias = alias(UserDAOMock, provides=UserRepository, override=True)
```

---

## `decorate` — обёртка существующей зависимости

Применяется, когда нужно добавить поведение (логирование, метрики, кэш) к уже зарегистрированной зависимости из другого провайдера.

```python
from dishka import decorate, provide, Provider, Scope

class UserRepository(Protocol): ...
class UserDAOImpl(UserRepository): ...
class UserDAOWithMetrics(UserRepository):
    def __init__(self, dao: UserRepository, metrics: PrometheusClient):
        self.dao = dao
        self.metrics = metrics

class BaseProvider(Provider):
    scope = Scope.REQUEST
    user_dao = provide(UserDAOImpl, provides=UserRepository)

class MetricsProvider(Provider):
    prometheus = provide(PrometheusClient, scope=Scope.APP)

    @decorate
    def add_metrics(self, dao: UserRepository, metrics: PrometheusClient) -> UserRepository:
        return UserDAOWithMetrics(dao, metrics)
```

**Правило:** `decorate` нельзя применять в том же провайдере, где объявлена фабрика. Это инструмент для постобработки зависимостей из *внешних* провайдеров.

### Изменение скоупа через decorate

```python
class ExtendedProvider(Provider):
    # Сужаем скоуп зависимости
    @decorate(scope=Scope.REQUEST)
    def wrap_service(self, svc: SomeService) -> SomeService:
        return SomeService(svc)
```

---

## `from_context` — зависимость из контекста запроса

Позволяет передавать данные в провайдер при входе в скоуп — например, объект HTTP-запроса, Telegram-апдейт и т.п.

```python
from dishka import from_context, provide, Provider, Scope

class RequestProvider(Provider):
    # Объявляем, что Request будет передан снаружи при входе в REQUEST-скоуп
    request = from_context(provides=Request, scope=Scope.REQUEST)

    @provide(scope=Scope.REQUEST)
    def get_user(self, request: Request) -> User:
        return request.user

container = make_container(RequestProvider(), context={App: app_instance})

# При входе в скоуп передаём данные через context=
with container(context={Request: current_request}) as req_container:
    user = req_container.get(User)
```

> **Важно:** ключи в словаре `context` — это именно типы (не строки), без `Annotated` или алиасов.

### from_context для APP-скоупа

```python
class AppProvider(Provider):
    # Config передаётся при создании контейнера
    config = from_context(provides=Config, scope=Scope.APP)

container = make_container(AppProvider(), context={Config: my_config})
```

### Переопределение фабрики через from_context (для тестов)

```python
class TestProvider(Provider):
    scope = Scope.APP
    # Переопределяем фабрику — берём Config из контекста, а не из провайдера
    config = from_context(provides=Config, override=True)

test_container = make_container(
    MainProvider(),
    TestProvider(),
    context={Config: test_config}
)
```

---

## `provide_all` — регистрация нескольких классов сразу

Удобный помощник, когда нужно зарегистрировать много классов с одинаковым скоупом:

```python
from dishka import provide_all, Provider, Scope

class ServiceProvider(Provider):
    scope = Scope.REQUEST

    # Вместо этого:
    # register_user = provide(RegisterUserInteractor)
    # update_profile = provide(UpdateProfileInteractor)
    # delete_user = provide(DeleteUserInteractor)

    # Пишем так:
    interactors = provide_all(
        RegisterUserInteractor,
        UpdateProfileInteractor,
        DeleteUserInteractor,
    )
```

Или через метод:

```python
provider = Provider(scope=Scope.REQUEST)
provider.provide_all(
    RegisterUserInteractor,
    UpdateProfileInteractor,
)
```

---

## Комбинирование через сложение

Все инструменты провайдера можно комбинировать через `+`:

```python
class MyProvider(Provider):
    scope = Scope.APP

    provides = (
        provide(UserDAOImpl, provides=UserDAO)
        + provide(PostDAOImpl, provides=PostDAO)
        + alias(source=PostDAOImpl, provides=PostReader)
        + decorate(SomeDecorator, provides=SomeClass)
        + from_context(Config)
    )
```

---

## Провайдер без класса — через экземпляр

Если нужна лёгкая конфигурация без наследования:

```python
provider = Provider(scope=Scope.APP)
provider.provide(Settings, provides=Settings)
provider.provide(Database)

container = make_container(provider)
```

---

## Следующий шаг

Подробнее о скоупах и времени жизни объектов → [03-scopes.md](./03-scopes.md)
