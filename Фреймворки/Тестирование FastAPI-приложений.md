[← Оглавление](../README.md)

# Тестирование FastAPI-приложений

> _Приложение не нужно запускать, чтобы его протестировать_

---

Главная особенность тестов в [FastAPI](../Фреймворки/FastAPI.md): чтобы отправить запрос в приложение, **не нужен работающий сервер**. Ни `uvicorn`, ни открытый порт, ни сеть — тестовый клиент вызывает ASGI-приложение напрямую, в том же процессе. Отсюда скорость (тысячи запросов в секунду на CI) и простота настройки: тест — обычная функция [pytest](../Библиотеки/Сторонние/Тесты/pytest.md). Общая теория тестирования — в [Юнит-тестирование Python](../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md); здесь — как это устроено именно во FastAPI.

## Почему сервер не нужен

Обычный HTTP-клиент открывает TCP-соединение и шлёт байты. Тестовый клиент делает иначе: он собирает **ASGI-словарь `scope`** (метод, путь, заголовки, query) и передаёт его прямо в `app`, минуя сокет.

```
Боевой запуск:  клиент ──TCP──▶ nginx ──▶ uvicorn ──[ASGI]──▶ app
Тест:           клиент ─────────────────────────────[ASGI]──▶ app
                        ↑ транспорт подменён: ни сокета, ни порта
```

Это не заглушка приложения: работают все роутеры, зависимости, middleware, валидация и обработчики ошибок — ровно та же цепочка, что в проде. Отличается только транспорт. Устройство ASGI и что такое `scope` — [WSGI и ASGI — интерфейс сервера и приложения](../WSGI%20и%20ASGI%20—%20интерфейс%20сервера%20и%20приложения.md).

> Побочный вывод: тесты **не проверяют** то, что живёт вне приложения, — конфиг nginx, лимиты uvicorn, TLS, таймауты. Это уровень e2e и нагрузочных тестов ([Нагрузочное тестирование — locust и k6](../Библиотеки/Сторонние/Тесты/Нагрузочное%20тестирование%20—%20locust%20и%20k6.md)).

## `TestClient` — синхронный путь

`fastapi.testclient.TestClient` — обёртка Starlette над HTTP-клиентом **httpx**, которая и подставляет ASGI-транспорт. Тесты пишутся обычными `def`, даже если все ручки асинхронные: клиент сам крутит event loop внутри. _(Раньше клиент строился на `requests`; сейчас Starlette переезжает на `httpx2` — обычный `httpx` ещё поддерживается, но объявлен устаревшим, поэтому в свежем проекте ставь `starlette[full]`.)_

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_create_and_get():
    r = client.post('/products', json={'name': 'Мышь', 'price': 999})
    assert r.status_code == 201
    pid = r.json()['id']

    assert client.get(f'/products/{pid}').json()['name'] == 'Мышь'

def test_validation_error():
    r = client.post('/products', json={'name': 'Без цены'})
    assert r.status_code == 422          # валидацию FastAPI делает сам
```

**Ловушка с `lifespan`.** События старта и остановки приложения **не выполняются** при создании клиента — только внутри `with`-блока. Если пул к БД или клиент [Redis](../Библиотеки/Сторонние/Redis.md) поднимаются в `lifespan`, без контекстного менеджера тест получит `None` вместо соединения:

```python
# НЕПРАВИЛЬНО — lifespan не отработал, app.state пуст:
client = TestClient(app)
client.get('/health')

# ПРАВИЛЬНО — старт и остановка выполняются:
with TestClient(app) as client:
    client.get('/health')
```

## Асинхронные тесты — `AsyncClient` + `ASGITransport`

`TestClient` синхронный: внутри теста нельзя сделать `await session.execute(...)`, чтобы проверить состояние базы. Как только тесту нужен собственный `await`, берут `httpx.AsyncClient` с ASGI-транспортом:

```python
import pytest
from httpx import ASGITransport, AsyncClient
from main import app

@pytest.mark.anyio
async def test_root():
    async with AsyncClient(transport=ASGITransport(app=app),
                           base_url='http://test') as ac:      # base_url любой, в сеть не идёт
        r = await ac.get('/')
    assert r.status_code == 200
```

Два нюанса, на которых спотыкаются:

- **`AsyncClient` не запускает `lifespan`** — в отличие от `TestClient` даже в `async with`. Нужны события старта — берут `LifespanManager` из пакета `asgi-lifespan`.
- **Асинхронные тесты не запустятся сами.** Нужен плагин: `anyio` (его использует документация FastAPI) или `pytest-asyncio`. У второго удобно включить `asyncio_mode = auto` в конфиге — тогда `async def`-тесты не надо помечать декоратором:

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

| | `TestClient` | `AsyncClient` + `ASGITransport` |
|---|---|---|
| Тест объявляется | `def` | `async def` (нужен плагин) |
| Свой `await` в тесте | нельзя | можно (проверить БД, дождаться задачу) |
| `lifespan` | в `with`-блоке | только через `asgi-lifespan` |
| Когда брать | большинство случаев | нужен доступ к async-ресурсам из теста |

## Подмена зависимостей — `dependency_overrides`

Причина, по которой в FastAPI всё выносят в [зависимости](../Фреймворки/Depends%20—%20зависимости%20в%20FastAPI.md): их можно подменить снаружи, не трогая код приложения. `app.dependency_overrides` — словарь «настоящая зависимость → подставная»:

```python
async def override_session():                       # сессия к тестовой базе
    async with TestSession() as session:
        yield session

async def override_user():                          # «залогиненный» пользователь
    return User(id=1, role='admin')

app.dependency_overrides[get_session] = override_session
app.dependency_overrides[get_current_user] = override_user
```

Ключ — **объект функции**, не строка. Подменяется та зависимость, которую импортировал код приложения, поэтому импортировать надо из того же модуля, откуда её берут роутеры.

> **Сбрасывай подмены после теста** (`app.dependency_overrides.clear()`), иначе они утекут в соседние тесты и те начнут проходить по чужой причине. Надёжнее всего делать это в фикстуре с `yield`.

**Что подменять, а что нет.** Подмена пользователя и внешних API — норма. Подмена **базы** — только на другую настоящую базу, а не на мок: смысл теста API в том, чтобы пройти насквозь через все слои до SQL.

## Тестовая база: чем платить за скорость

Видео-советы «поднимите SQLite для тестов» звучат заманчиво, но SQLite — **другая СУБД**: нет `JSONB`, массивов, `ON CONFLICT` ведёт себя иначе, ограничения проверяются не так. Зелёный тест на SQLite не гарантирует ничего про PostgreSQL. Разбор всех четырёх способов поднять базу — [Интеграционные тесты и testcontainers](../Библиотеки/Сторонние/Тесты/Интеграционные%20тесты%20и%20testcontainers.md); коротко:

- **testcontainers** — настоящий PostgreSQL в контейнере, поднимается из кода теста. Рабочий вариант по умолчанию;
- **отдельная тестовая база** рядом с dev — дёшево, но состояние «плывёт» и нельзя гнать тесты параллельно;
- **SQLite** — уместен разве что для учебного проекта, где SQLite и в проде.

**Изоляция между тестами.** Пересоздавать схему перед каждым тестом медленно. Стандартный приём — каждый тест внутри транзакции, которая в конце откатывается: база после теста в исходном состоянии, а `ROLLBACK` стоит миллисекунды.

## Что тестировать в API-сервисе

Пирамида ([Юнит-тестирование Python](../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md#пирамида-тестирования)) для FastAPI распадается так:

| Уровень | Что проверяет | Чем |
|---|---|---|
| Юнит | чистая логика: расчёты, правила, валидаторы Pydantic | `pytest`, без клиента |
| Интеграционный | слой доступа к данным против настоящей БД | сессия + testcontainers |
| API | контракт: коды ответов, форма JSON, права, ошибки | `TestClient` / `AsyncClient` |
| e2e | связка с nginx, брокером, внешними сервисами | поднятое окружение |

Приоритет при нехватке времени — **тесты API**: они проходят насквозь через все слои и охраняют контракт, который видит пользователь. Что всё-таки тестируют отдельно на уровне БД — миграции, уникальные ограничения, каскады, сложные запросы.

Обязательный минимум на каждую ручку: успешный сценарий, `404` на отсутствующий объект, `422` на кривое тело, `401`/`403` без прав.

## Сквозной пример: `conftest.py` + тест роутера

```python
# conftest.py — общая обвязка для всех тестов
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
from testcontainers.postgres import PostgresContainer

from main import app
from deps import get_session, get_current_user
from models import Base, User

@pytest.fixture(scope='session')                 # 1. база одна на весь прогон
def engine():
    with PostgresContainer('postgres:16') as pg:
        url = pg.get_connection_url().replace('psycopg2', 'asyncpg')
        yield create_async_engine(url)

@pytest.fixture(scope='session', autouse=True)   # 2. схема — один раз
async def schema(engine):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

@pytest.fixture                                  # 3. сессия в транзакции с откатом
async def session(engine):
    async with engine.connect() as conn:
        trx = await conn.begin()
        async with async_sessionmaker(bind=conn, expire_on_commit=False)() as s:
            yield s
        await trx.rollback()                     # состояние базы вернулось

@pytest.fixture                                  # 4. клиент с подменёнными зависимостями
async def client(session):
    app.dependency_overrides[get_session] = lambda: session
    app.dependency_overrides[get_current_user] = lambda: User(id=1, role='admin')
    async with AsyncClient(transport=ASGITransport(app=app),
                           base_url='http://test') as ac:
        yield ac
    app.dependency_overrides.clear()             # 5. подмены не утекают в соседние тесты
```

```python
# tests/test_products.py
async def test_create_returns_201_and_saves(client, session):
    r = await client.post('/products', json={'name': 'Мышь', 'price': 999})
    assert r.status_code == 201

    saved = await session.get(Product, r.json()['id'])   # свой await — проверяем БД напрямую
    assert saved.price == 999

async def test_unknown_product_404(client):
    assert (await client.get('/products/99999')).status_code == 404

async def test_price_must_be_positive(client):
    r = await client.post('/products', json={'name': 'Мышь', 'price': -5})
    assert r.status_code == 422
```

**Как это читать.** Фикстуры выстроены по времени жизни: контейнер и схема — раз на прогон, транзакция и клиент — на каждый тест. Ручка получает сессию не из своего `get_session`, а ту, что открыл тест, — поэтому после отката база чистая, а тест видит записи, которые сделал сам запрос. `dependency_overrides` подставляет и пользователя, так что авторизацию не надо имитировать токенами.

## Ловушки

- **Фикстура `client` шире, чем `session`.** Клиент с подменой на сессию из фикстуры теста должен жить не дольше самой сессии, иначе после отката запрос уйдёт в закрытое соединение.
- **`app` — глобальный объект.** `dependency_overrides` живёт на приложении, а не на клиенте: любая забытая подмена видна всем последующим тестам в прогоне.
- **Порядок тестов как скрытая зависимость.** Тест, который проходит только после соседнего, — сломанная изоляция. Проверяется запуском в случайном порядке (`pytest-randomly`).
- **Фоновая задача не дождалась.** `BackgroundTasks` выполняются после ответа: `TestClient` их дождётся (запрос завершается вместе с ними), а вот проверять результат сразу после `await ac.post(...)` в асинхронном клиенте надо осознанно.
- **Мок вместо базы в тестах API.** Замоканный репозиторий превращает тест API в проверку собственных фантазий — см. [Интеграционные тесты и testcontainers](../Библиотеки/Сторонние/Тесты/Интеграционные%20тесты%20и%20testcontainers.md#почему-моки-врут).

## Связи

- [FastAPI](../Фреймворки/FastAPI.md) · [REST API на FastAPI](../Фреймворки/REST%20API%20на%20FastAPI.md) — приложение, которое тестируем;
- [Depends — зависимости в FastAPI](../Фреймворки/Depends%20—%20зависимости%20в%20FastAPI.md) — почему DI и есть предпосылка тестируемости;
- [pytest](../Библиотеки/Сторонние/Тесты/pytest.md) — фикстуры, параметризация, конфиг;
- [Юнит-тестирование Python](../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md) — AAA, моки и стабы, пирамида;
- [Интеграционные тесты и testcontainers](../Библиотеки/Сторонние/Тесты/Интеграционные%20тесты%20и%20testcontainers.md) — настоящая БД в тестах и почему не SQLite;
- [Тестирование Django приложений](../Фреймворки/DJANGO/Тестирование%20Django%20приложений.md) — та же задача в Django: `APIClient`, `pytest-django`, `model_bakery`;
- [WSGI и ASGI — интерфейс сервера и приложения](../WSGI%20и%20ASGI%20—%20интерфейс%20сервера%20и%20приложения.md) — что за `scope` подсовывает тестовый клиент;
- [CI CD](../DevOps/CI%20CD.md) — где эти тесты прогоняются автоматически.

## Источники

- [FastAPI — Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [FastAPI — Async Tests](https://fastapi.tiangolo.com/advanced/async-tests/)
- [FastAPI — Testing Dependencies with Overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/)
- [FastAPI — Testing Events: startup - shutdown](https://fastapi.tiangolo.com/advanced/testing-events/)
- [Starlette — TestClient](https://www.starlette.io/testclient/)
- [HTTPX — ASGITransport](https://www.python-httpx.org/async/#calling-into-python-web-apps)
- [pytest-asyncio — Modes](https://pytest-asyncio.readthedocs.io/en/latest/concepts.html)
