# Ключевые концепции dishka

## Что такое Dependency Injection

**Dependency Injection (DI)** — паттерн, при котором объект получает свои зависимости извне, а не создаёт их сам.

```python
# Плохо — объект создаёт зависимости сам
class UserService:
    def __init__(self):
        self.db = Database("sqlite:///app.db")  # жёстко зашито
        self.cache = Redis("localhost:6379")     # жёстко зашито

# Хорошо — зависимости передаются снаружи
class UserService:
    def __init__(self, db: Database, cache: Redis):
        self.db = db
        self.cache = cache
```

Во втором случае `UserService` можно тестировать с mock-объектами, легко переключать реализации и переиспользовать код.

DI-контейнер — это инструмент, который автоматизирует передачу зависимостей: вместо того чтобы вручную собирать граф объектов, вы описываете **как** создавать каждый объект, а контейнер делает это за вас.

---

## Четыре ключевые сущности dishka

### 1. Dependency (Зависимость)

Зависимость — это любой объект, который нужен другому объекту для работы. В dishka зависимости описываются через **аннотации типов** в `__init__` или в параметрах фабричных функций.

```python
class APIClient:
    def __init__(self, settings: Settings):  # Settings — зависимость APIClient
        self.base_url = settings.api_url

class OrderService:
    def __init__(self, client: APIClient, db: Database):  # две зависимости
        ...
```

Dishka анализирует аннотации типов и сам разрешает весь граф зависимостей.

---

### 2. Provider (Провайдер)

**Provider** — класс, который описывает, *как* создавать зависимости. Это центральный объект при настройке dishka.

```python
from dishka import Provider, Scope, provide

class InfrastructureProvider(Provider):
    scope = Scope.APP  # скоуп по умолчанию для всех фабрик этого провайдера

    @provide
    def get_settings(self) -> Settings:
        return Settings.from_env()

    @provide
    def get_db(self, settings: Settings) -> Database:
        return Database(settings.db_url)

    # Если никакой логики нет — передаём класс напрямую
    api_client = provide(APIClient)
```

**Правила:**
- Наследуйтесь от `Provider`
- Помечайте фабрики декоратором `@provide` или функцией `provide(SomeClass)`
- Аннотация возвращаемого типа обязательна — она говорит контейнеру, что именно создаёт этот метод
- Параметры метода — это зависимости, которые контейнер разрешит автоматически

Провайдеры можно (и нужно) разбивать на логические группы:

```python
class DatabaseProvider(Provider):
    """Всё, что связано с базой данных"""
    ...

class CacheProvider(Provider):
    """Redis и кэширование"""
    ...

class ServiceProvider(Provider):
    """Бизнес-логика"""
    ...

container = make_container(DatabaseProvider(), CacheProvider(), ServiceProvider())
```

---

### 3. Scope (Скоуп)

**Scope** определяет время жизни зависимости. Это главное, чем dishka отличается от большинства DI-библиотек.

Стандартные скоупы (по убыванию времени жизни):

```
APP → REQUEST → ACTION → STEP
```

- **APP** — объект создаётся один раз при старте приложения и живёт до его остановки (аналог singleton)
- **REQUEST** — объект создаётся для каждого HTTP-запроса, Telegram-апдейта или другого события
- **ACTION** — более короткий скоуп, например для отдельного действия внутри запроса
- **STEP** — самый короткий, для отдельного шага

**Правило:** объект может зависеть от объектов из того же или более широкого скоупа, но не из более узкого.

```python
# Правильно: REQUEST-объект зависит от APP-объекта
class RequestService:
    def __init__(self, db: Database, settings: Settings):  # оба из APP — OK
        ...

# Неправильно: APP-объект зависит от REQUEST-объекта
class AppConfig:
    def __init__(self, request: Request):  # Request — из REQUEST, это ошибка
        ...
```

Пример назначения скоупов:

```python
class MyProvider(Provider):
    # Создаётся один раз на всё приложение
    settings = provide(Settings, scope=Scope.APP)
    db_pool = provide(DatabasePool, scope=Scope.APP)

    # Создаётся для каждого запроса
    db_session = provide(DatabaseSession, scope=Scope.REQUEST)
    user_service = provide(UserService, scope=Scope.REQUEST)
```

> **Важно:** если один и тот же объект запрашивается несколько раз в рамках одного скоупа — возвращается одна и та же инстанция (кэширование включено по умолчанию). Можно отключить: `provide(MyClass, cache=False)`.

---

### 4. Container (Контейнер)

**Container** — объект, через который вы получаете зависимости во время работы приложения.

```python
container = make_container(MyProvider())

# Получить APP-зависимость
settings = container.get(Settings)

# Войти в REQUEST-скоуп
with container() as request_container:
    service = request_container.get(UserService)
    # Здесь доступны REQUEST-зависимости и все выше

# При выходе из with-блока — все REQUEST-зависимости финализируются
container.close()  # финализация APP-зависимостей
```

Контейнер **не создаёт объекты сам** — он управляет жизненным циклом и делегирует создание провайдерам.

---

### 5. Component (Компонент)

**Component** — именованная изолированная группа провайдеров внутри одного контейнера. Используется, когда в одном контейнере нужно иметь несколько независимых наборов зависимостей одного типа.

```python
from typing import Annotated
from dishka import Provider, Scope, provide, make_container, FromComponent

class MainProvider(Provider):
    @provide(scope=Scope.APP)
    def get_ratio(self, value: Annotated[int, FromComponent("metrics")]) -> float:
        return value / 100.0

class MetricsProvider(Provider):
    component = "metrics"  # изолированная группа

    @provide(scope=Scope.APP)
    def get_value(self) -> int:
        return 42

container = make_container(MainProvider(), MetricsProvider())
ratio = container.get(float)  # 0.42
```

Компоненты полезны при построении микросервисных архитектур или при подключении сторонних модулей, которые используют те же типы данных.

---

## Как всё связано

```
┌─────────────────────────────────────────┐
│              Container                   │
│                                         │
│  ┌──────────┐  ┌──────────────────────┐ │
│  │ APP scope│  │  Provider(s)         │ │
│  │          │  │  - provide(Service)  │ │
│  │  ┌────┐  │  │  - alias(...)        │ │
│  │  │ 🔵 │  │  │  - from_context(...) │ │
│  │  └────┘  │  │  - decorate(...)     │ │
│  │    ↓     │  └──────────────────────┘ │
│  │ REQUEST  │                           │
│  │  scope   │                           │
│  │  ┌────┐  │                           │
│  │  │ 🟢 │  │                           │
│  │  └────┘  │                           │
│  └──────────┘                           │
└─────────────────────────────────────────┘
```

1. Вы создаёте **провайдеры** — описываете, как создавать каждую зависимость
2. Передаёте провайдеры в **контейнер**
3. Контейнер строит граф зависимостей и валидирует его
4. Вы входите в нужный **скоуп** и запрашиваете объекты через `container.get(Type)`
5. Контейнер находит нужную фабрику, разрешает все вложенные зависимости и возвращает объект
6. При выходе из скоупа — все зависимости **финализируются** (если использовался `yield`)

---

## Следующий шаг

Подробнее об инструментах провайдера (`provide`, `alias`, `decorate`, `from_context`) → [02-providers.md](./02-providers.md)
