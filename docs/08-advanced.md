# Продвинутые темы

## Компоненты (Components)

Компоненты позволяют создавать изолированные группы провайдеров внутри одного контейнера. Это полезно, когда разные части приложения используют одинаковые типы, но с разными реализациями.

### Когда нужны компоненты

Представьте приложение, которое работает с несколькими базами данных — для пользователей и для комментариев. Оба используют тип `DBConnection`, но разные реализации:

```python
from typing import Annotated
from dishka import Provider, Scope, provide, make_container, FromComponent

class DBConnection(Protocol): ...
class UserDBConnection(DBConnection): ...
class CommentDBConnection(DBConnection): ...

class UserDAO:
    def __init__(self, db: DBConnection): ...

class CommentDAO:
    def __init__(self, db: DBConnection): ...

class UserProvider(Provider):
    component = "user"          # все фабрики этого провайдера — в компоненте "user"
    scope = Scope.APP

    db = provide(UserDBConnection, provides=DBConnection)
    dao = provide(UserDAO)

class CommentProvider(Provider):
    component = "comment"       # отдельный изолированный компонент
    scope = Scope.APP

    db = provide(CommentDBConnection, provides=DBConnection)
    dao = provide(CommentDAO)

container = make_container(UserProvider(), CommentProvider())

# Получение по компоненту
user_db = container.get(DBConnection, component="user")     # UserDBConnection
comment_db = container.get(DBConnection, component="comment")  # CommentDBConnection
```

### Связывание компонентов через FromComponent

Провайдер из одного компонента может запросить зависимость из другого:

```python
from typing import Annotated
from dishka import FromComponent

class MainProvider(Provider):
    @provide(scope=Scope.APP)
    def get_ratio(
        self,
        value: Annotated[int, FromComponent("metrics")]  # берём int из компонента "metrics"
    ) -> float:
        return value / 100.0

class MetricsProvider(Provider):
    component = "metrics"

    @provide(scope=Scope.APP)
    def get_value(self) -> int:
        return 42

container = make_container(MainProvider(), MetricsProvider())
container.get(float)  # 0.42
```

### Назначение компонента через Annotated в возвращаемом типе

```python
class MainProvider(Provider):
    @provide(scope=Scope.APP)
    def foobar(self, a: Annotated[int, FromComponent("X")]) -> float:
        return a / 10

    # Регистрируем int прямо в компоненте "X" через return type
    @provide(scope=Scope.APP)
    def foo(self) -> Annotated[int, FromComponent("X")]:
        return 1

container = make_container(MainProvider())
container.get(float)  # 0.1
```

> **Предупреждение:** смешивать фабрики разных компонентов в одном провайдере — плохая практика. Используйте это только если компонент состоит из 1–2 фабрик.

### Один провайдер — несколько компонентов

```python
base_provider = MyProvider()

# Использовать один и тот же провайдер в разных компонентах
provider_for_a = base_provider.to_component("component_a")
provider_for_b = base_provider.to_component("component_b")

container = make_container(provider_for_a, provider_for_b)
```

### Alias между компонентами

```python
# Создать alias для int из компонента "X" в текущем компоненте
a = alias(int, component="X")
```

### Использование компонентов в интеграциях

В обработчиках фреймворков `FromDishka[T]` работает с дефолтным компонентом. Для другого компонента:

```python
from typing import Annotated
from dishka import FromComponent

# В FastAPI
@router.get("/")
async def endpoint(
    service: Annotated[UserService, FromComponent("user")],
) -> str:
    ...
```

---

## Дженерики (Generic Types)

dishka поддерживает работу с Generic-классами и TypeVar.

### Создание Generic-зависимостей

```python
from typing import Generic, TypeVar
from dishka import Provider, Scope, provide, make_container

T = TypeVar("T", bound=int)

class Repository(Generic[T]):
    def __init__(self, model_type: type[T]):
        self.model_type = model_type

class MyProvider(Provider):
    @provide(scope=Scope.APP)
    def make_repo(self, t: type[T]) -> Repository[T]:
        # t — конкретный тип, переданный при запросе
        print(f"Creating repository for {t}")
        return Repository(t)

container = make_container(MyProvider())
int_repo = container.get(Repository[int])    # Creating repository for <class 'int'>
bool_repo = container.get(Repository[bool])  # Creating repository for <class 'bool'>

# Repository[int] и Repository[bool] — разные объекты с разным кэшем
int_repo is bool_repo  # False
```

### Generic-декораторы

```python
from typing import TypeVar
from collections.abc import Iterator
from dishka import Provider, Scope, provide, decorate, make_container

T = TypeVar("T")

class MyProvider(Provider):
    scope = Scope.APP

    @provide
    def make_int(self) -> int:
        return 1

    @provide
    def make_str(self) -> str:
        return "hello"

    # Этот декоратор применяется ко ВСЕМ зависимостям
    @decorate
    def log_all(self, a: T, t: type[T]) -> Iterator[T]:
        print(f"Получено {t.__name__} = {a}")
        yield a
        print(f"Финализация {t.__name__}")

container = make_container(MyProvider())
container.get(int)   # Получено int = 1
container.get(str)   # Получено str = hello
container.close()    # Финализация str; Финализация int
```

### Ограничения Generic

- Нельзя использовать `TypeVar` с bound на Generic-тип
- Generic-декораторы применяются только к конкретным фабрикам или к фабрикам с более узкими TypeVar

---

## Сбор нескольких объектов одного типа (collect)

По умолчанию, если несколько фабрик создают один тип — выигрывает последняя. `collect` позволяет получить **все** зарегистрированные объекты типа в виде списка:

```python
from dishka import Provider, Scope, provide, collect, make_container

class Handler(Protocol):
    async def handle(self, event: Event) -> None: ...

class LoggingHandler(Handler): ...
class MetricsHandler(Handler): ...
class NotificationHandler(Handler): ...

class HandlerProvider(Provider):
    scope = Scope.APP

    logging_handler = provide(LoggingHandler, provides=Handler)
    metrics_handler = provide(MetricsHandler, provides=Handler)
    notification_handler = provide(NotificationHandler, provides=Handler)

    # Собираем все Handler в список
    all_handlers = collect(Handler)

container = make_container(HandlerProvider())
handlers = container.get(list[Handler])
# [LoggingHandler(), MetricsHandler(), NotificationHandler()]

# Можно использовать как зависимость
class EventBus:
    def __init__(self, handlers: list[Handler]):
        self.handlers = handlers
```

### Параметры collect

```python
all_handlers = collect(
    Handler,
    provides=Sequence[Handler],  # изменить тип результата (но список создаётся всегда)
    scope=Scope.REQUEST,          # явный скоуп
    cache=True,                   # кэшировать (по умолчанию True)
)
```

> **Важно:** `override=True` на фабрике заставляет контейнер игнорировать все предыдущие фабрики этого типа — они не попадут в `collect`.

---

## Визуализация графа зависимостей (Plotter)

dishka умеет генерировать диаграммы графа зависимостей — полезно для документирования архитектуры и отладки.

### Mermaid (HTML с интерактивной диаграммой)

```python
from dishka.plotter import render_mermaid

container = make_container(AppProvider(), ServiceProvider())

# Генерируем HTML с Mermaid-диаграммой
html_content = render_mermaid(container)

with open("dependency_graph.html", "w") as f:
    f.write(html_content)
```

Открой `dependency_graph.html` в браузере — увидишь интерактивную диаграмму зависимостей.

### D2 (текстовый формат для d2lang.com)

```python
from dishka.plotter import render_d2

d2_content = render_d2(container)

with open("dependency_graph.d2", "w") as f:
    f.write(d2_content)
# Затем: d2 dependency_graph.d2 graph.svg
```

### Пример диаграммы

Диаграмма показывает:
- Все зарегистрированные типы и их скоупы
- Стрелки зависимостей (кто от кого зависит)
- Группировку по скоупам (APP, REQUEST и т.д.)

---

## `when` — условные фабрики

Позволяет регистрировать несколько фабрик для одного типа и активировать нужную в зависимости от условия.

Работает через два элемента:
- `Marker("name")` — метка, которую указывают в `when=` у фабрики
- `@activate(Marker("name"))` — метод провайдера, возвращающий `bool`: активна ли метка

```python
from dishka import Provider, Scope, provide, activate, Marker, from_context

class Config:
    debug: bool = False

class MyProvider(Provider):
    scope = Scope.APP
    config = from_context(provides=Config, scope=Scope.APP)

    # Базовая реализация — используется всегда, когда нет активной метки
    @provide
    def get_prod_cache(self) -> Cache:
        return NormalCacheImpl()

    # Эта фабрика активна только когда Marker("debug") == True
    @provide(when=Marker("debug"))
    def get_debug_cache(self) -> Cache:
        return DebugCacheImpl()

    # Логика активации метки
    @activate(Marker("debug"))
    def is_debug(self, config: Config) -> bool:
        return config.debug

debug_config = Config()
debug_config.debug = True

container = make_container(MyProvider(), context={Config: debug_config})
cache = container.get(Cache)  # DebugCacheImpl, потому что config.debug=True
```

Для повторного использования одной логики активации на нескольких метках — создайте подкласс `Marker`:

```python
class EnvMarker(Marker):
    pass

class MyProvider(Provider):
    config = from_context(Config, scope=Scope.APP)

    @provide
    def get_default(self) -> Service: ...

    @provide(when=EnvMarker("staging"))
    def get_staging(self) -> Service: ...

    @provide(when=EnvMarker("production"))
    def get_production(self) -> Service: ...

    # Один активатор на все EnvMarker: marker.value сравнивается с config.env
    @activate(EnvMarker)
    def check_env(self, marker: EnvMarker, config: Config) -> bool:
        return config.environment == marker.value
```

---

## `from_context` в продвинутых сценариях

### Несколько контекстных переменных

```python
class WebProvider(Provider):
    # APP-уровень — передаём при make_container
    app_config = from_context(provides=AppConfig, scope=Scope.APP)

    # REQUEST-уровень — передаём при container()
    http_request = from_context(provides=Request, scope=Scope.REQUEST)
    current_user = from_context(provides=User, scope=Scope.REQUEST)

container = make_container(WebProvider(), context={AppConfig: config})

with container(context={Request: req, User: user}) as req_container:
    ...
```

### Использование from_context для разделения тестовой и боевой конфигурации

```python
class ProdProvider(Provider):
    scope = Scope.APP
    config = provide(Config)  # создаётся из кода

class TestProvider(Provider):
    scope = Scope.APP
    # В тестах конфиг передаётся снаружи
    config = from_context(provides=Config, override=True)

# Продакшн
prod_container = make_container(ProdProvider())

# Тесты
test_config = Config(db_url="sqlite:///:memory:", debug=True)
test_container = make_container(
    ProdProvider(),
    TestProvider(),
    context={Config: test_config},
)
```

---

## Валидация при создании контейнера

Дополнительные проверки при `make_container`:

```python
from dishka.entities.validation_settings import ValidationSettings

container = make_container(
    provider,
    validation_settings=ValidationSettings(
        nothing_overridden=True,   # ошибка, если override=True, но нет что переопределять
    )
)
```

Отключение валидации (только для отладки):

```python
container = make_container(provider, skip_validation=True)
```

---

## Следующий шаг

Тестирование приложений с dishka → [09-testing.md](./09-testing.md)
