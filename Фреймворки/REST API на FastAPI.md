[← Оглавление](../README.md)

# REST API на FastAPI (асинхронный)

> Как собрать полноценный **REST-сервис** на асинхронном фреймворке: ресурсы и правильные HTTP-методы/коды, разбивка по роутерам, внедрение зависимостей (`Depends`), async-работа с БД и тесты. Механику самого [FastAPI](../Фреймворки/FastAPI.md) и стиль REST ([принципы](../Фреймворки/DJANGO/API.md#что-такое-rest-api)) — в отдельных заметках; здесь они соединяются в рабочее приложение.

---

REST-сервис отдаёт клиентам данные (JSON) и операции над ними по HTTP. **Асинхронный** фреймворк ([FastAPI](../Фреймворки/FastAPI.md) на ASGI) особенно уместен: API — это I/O-bound нагрузка (ждём БД, внешние сервисы), а `async` позволяет держать тысячи одновременных запросов на одном процессе, не блокируясь на ожидании. Ниже — как спроектировать ресурс и довести его до тестируемого CRUD.

## Ресурс, методы, коды ответов

REST-принципы (ресурсы, URI, stateless, единый интерфейс) разобраны в [API](../Фреймворки/DJANGO/API.md#что-такое-rest-api). Для проектирования эндпоинтов важны две вещи: **какой метод** и **какой код ответа** вернуть.

| Метод | Операция над ресурсом | Успешный код | Идемпотентный | Безопасный |
|---|---|---|---|---|
| `GET /items` / `GET /items/{id}` | получить | `200 OK` | да | да (не меняет данные) |
| `POST /items` | создать | `201 Created` | нет | нет |
| `PUT /items/{id}` | заменить целиком | `200 OK` | да | нет |
| `PATCH /items/{id}` | обновить частично | `200 OK` | нет* | нет |
| `DELETE /items/{id}` | удалить | `204 No Content` | да | нет |

- **Безопасный** — не меняет состояние (только `GET`). **Идемпотентный** — повтор того же запроса даёт тот же результат (`PUT`/`DELETE` можно слать дважды без вреда, `POST` — создаст дубль).
- Ошибки клиента: `400` (плохой запрос), `404` (ресурс не найден), `422` (не прошла валидация — FastAPI отдаёт сам), `401`/`403` (нет аутентификации/прав).

> Возвращать правильный код — часть контракта REST. Создал ресурс → `201`, не `200`; удалил → `204` (без тела); нет объекта → `404`, а не пустой `200`.

## URL-схема ресурса

Ресурс — существительное во множественном числе, действие задаёт **метод**, а не URL:

```
GET    /products        # список
POST   /products        # создать
GET    /products/{id}   # один
PUT    /products/{id}   # заменить
DELETE /products/{id}   # удалить
```
```
# антипаттерн — глагол в URL:
GET /getAllProducts     # ✗ — операцию задаёт метод, а не путь
POST /products/create   # ✗
```

### Список — это всегда страница

`GET /products` без ограничений — мина замедленного действия: на тестовых данных вернёт двадцать строк, на боевых попытается отдать миллион. Списочная ручка обязана иметь:

- **потолок размера страницы** — `limit: int = Query(20, ge=1, le=100)`, иначе `?limit=1000000` кладёт сервис;
- **однозначную сортировку** — с уникальным столбцом в хвосте `ORDER BY`, иначе строки кочуют между страницами;
- **признак продолжения** — `has_more` или `next_cursor`; `total` считать дорого и обычно не нужно.

Механика и выбор подхода — Пагинация — offset, keyset и гибрид, форма ответа и курсор — Курсор в API — контракт пагинации.

## Полный CRUD-ресурс

Собираем ресурс `products` со всеми методами и корректными кодами. Модели входа/выхода — [Pydantic](../Библиотеки/Сторонние/pydantic.md); `response_model` фиксирует форму ответа; `status_code` — код успеха; `HTTPException` — ошибки.

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

app = FastAPI()
db: dict[int, dict] = {}                       # хранилище-заглушка (вместо БД)

class ProductIn(BaseModel):                    # что принимаем (тело запроса)
    name: str
    price: float

class ProductOut(ProductIn):                   # что отдаём (+ id)
    id: int

@app.get("/products", response_model=list[ProductOut])
async def list_products():
    return list(db.values())                   # 200 OK

@app.get("/products/{pid}", response_model=ProductOut)
async def get_product(pid: int):
    if pid not in db:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Товар не найден")
    return db[pid]

@app.post("/products", response_model=ProductOut,
          status_code=status.HTTP_201_CREATED)     # создание → 201
async def create_product(product: ProductIn):
    pid = max(db, default=0) + 1
    db[pid] = {"id": pid, **product.model_dump()}
    return db[pid]

@app.put("/products/{pid}", response_model=ProductOut)
async def replace_product(pid: int, product: ProductIn):
    if pid not in db:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Товар не найден")
    db[pid] = {"id": pid, **product.model_dump()}
    return db[pid]

@app.delete("/products/{pid}", status_code=status.HTTP_204_NO_CONTENT)  # → 204, без тела
async def delete_product(pid: int):
    if pid not in db:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Товар не найден")
    del db[pid]
```

Как читать: типы аргументов и Pydantic-модели дают валидацию бесплатно (неверное тело → `422`); `response_model` отсекает лишние поля и документирует ответ; каждый метод возвращает свой код. Открыв `/docs`, весь CRUD можно потыкать в браузере.

## Роутеры — разбить по ресурсам

Держать всё в одном файле не масштабируется. **`APIRouter`** — под-приложение для одного ресурса; главный `app` их подключает с общим префиксом:

```python
# routers/products.py
from fastapi import APIRouter

router = APIRouter(prefix="/products", tags=["products"])

@router.get("")                       # → GET /products
async def list_products(): ...

@router.get("/{pid}")                 # → GET /products/{pid}
async def get_product(pid: int): ...

# main.py
from fastapi import FastAPI
from routers import products

app = FastAPI()
app.include_router(products.router)   # подключить ресурс
```

`prefix` задаёт общий путь, `tags` группирует эндпоинты в `/docs`.

## Depends — внедрение зависимостей

**`Depends`** — механизм DI: функция-зависимость подготавливает ресурс (сессию БД, текущего пользователя), FastAPI вызывает её и подставляет результат в обработчик. Общая логика в одном месте, а не в каждом эндпоинте.

```python
from fastapi import Depends, HTTPException, status

async def get_current_user(token: str = ""):      # зависимость: достать пользователя
    if not token:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Нужна авторизация")
    return {"user": "Иван"}

@app.get("/me")
async def me(user: dict = Depends(get_current_user)):   # FastAPI сам вызовет зависимость
    return user
```

Зависимости **вкладываются** (одна зависит от другой) и переиспользуются — так подключают сессию БД, проверку прав, пагинацию.

> Это только базовая форма. Классы-зависимости, запись через `Annotated`, кэш в пределах запроса, `dependencies=[...]` на роутере и подмена в тестах — в отдельной заметке [Depends — зависимости в FastAPI](../Фреймворки/Depends%20—%20зависимости%20в%20FastAPI.md).

## Async-работа с БД

Чтобы `async def`-обработчик реально не блокировал event loop, драйвер БД тоже должен быть асинхронным ([async SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md) + `asyncpg`). Сессию отдают через `Depends` с `yield` — как фикстуру: открыть до, закрыть после запроса.

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
Session = async_sessionmaker(engine, expire_on_commit=False)

async def get_session():                 # зависимость-сессия
    async with Session() as session:     # открыть
        yield session                    # отдать обработчику
        # закрытие — автоматически на выходе из with

@app.get("/products/{pid}")
async def get_product(pid: int, session: AsyncSession = Depends(get_session)):
    product = await session.get(Product, pid)     # await — не блокирует loop
    if product is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND)
    return product
```

> **Ловушка async.** Синхронный вызов (обычный драйвер БД, `requests`, `time.sleep`) внутри `async def` блокирует весь loop — вся выгода теряется. Подробно — в [FastAPI](../Фреймворки/FastAPI.md#ловушка-блокирующий-код-в-async-def).

## Тестирование REST-эндпоинтов

FastAPI даёт **`TestClient`** (поверх httpx) — поднимает приложение в памяти, шлёт запросы без реального сервера. Проверяют статус-код и тело. Это [тесты](../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md) на уровне API (ближе к интеграционным):

```python
# test_api.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_create_and_get():
    # создать → 201
    r = client.post("/products", json={"name": "Мышь", "price": 999})
    assert r.status_code == 201
    pid = r.json()["id"]

    # получить → 200
    r = client.get(f"/products/{pid}")
    assert r.status_code == 200
    assert r.json()["name"] == "Мышь"

def test_not_found():
    assert client.get("/products/99999").status_code == 404

def test_validation_error():
    r = client.post("/products", json={"name": "Без цены"})   # нет price
    assert r.status_code == 422        # FastAPI сам валидирует
```

Для по-настоящему асинхронных тестов берут `httpx.AsyncClient` с `ASGITransport` и [pytest](../Библиотеки/Сторонние/Тесты/pytest.md)-плагин `pytest-asyncio`. `TestClient` синхронный и покрывает большинство случаев.

> Почему сервер не нужен вовсе, ловушка с `lifespan`, подмена зависимостей через `dependency_overrides`, тестовая база и готовый `conftest.py` — в [Тестирование FastAPI-приложений](../Фреймворки/Тестирование%20FastAPI-приложений.md).

## Деплой FastAPI

FastAPI — **ASGI**-приложение, поэтому боевой сервер — не gunicorn-как-у-Django, а **uvicorn** (ASGI-сервер). Общая трёхслойная схема `nginx → сервер приложений → приложение` и обвязка (systemd, `.env`, Docker) — та же, что в [Развертывание проекта](../DevOps/Развертывание%20проекта.md); здесь — FastAPI-специфика.

**Чем деплой FastAPI отличается от Django:**

- точка входа — ASGI-объект `app` (`main:app`), а не WSGI `wsgi.py`;
- сервер приложений — **uvicorn**, а не голый gunicorn;
- нет `collectstatic` и шаблонов — это чистый API (статику, если есть, раздаёт nginx или CDN);
- миграции — своим инструментом ([Alembic](../Библиотеки/Сторонние/ORM/SQLAlchemy.md)), а не `manage.py migrate`.

**Запуск в несколько воркеров (задействовать все ядра):**

```bash
# вариант 1 — команда fastapi (из fastapi[standard]), проще всего
fastapi run main.py --workers 4

# вариант 2 — uvicorn напрямую, больше контроля
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

Ориентир по числу воркеров — как у gunicorn: `2 × ядра + 1`.

### Uvicorn и Gunicorn — кто из них зачем

Частый вопрос на собеседовании: «зачем и uvicorn, и gunicorn, если и то и то запускает приложение?» Ответ — **у них разные роли**:

- **Uvicorn — ASGI-сервер (воркер).** Принимает соединение, переводит HTTP в ASGI-вызов приложения и отдаёт ответ обратно. Он *обрабатывает запросы*.
- **Gunicorn — менеджер процессов (супервизор).** Сам HTTP-запрос он не обслуживает: поднимает N воркеров, следит, что они живы, перезапускает упавший, умеет плавно перекатывать их по сигналу. Он *присматривает за воркерами*.

Отсюда классическая связка «gunicorn с uvicorn-воркерами»: обработку берёт uvicorn, надзор — gunicorn. Голые `uvicorn --workers` раньше означали, что упавший процесс никто не поднимет.

> ⚠️ **Совет «бери gunicorn -k uvicorn.workers.UvicornWorker» устарел дважды.** Сначала в uvicorn **0.30** (май 2024) появился собственный менеджер процессов: `--workers` сам следит за воркерами и перезапускает их, а модуль `uvicorn.workers` объявлен устаревшим (класс переехал в отдельный пакет `uvicorn-worker`). Документация FastAPI на странице про воркеров gunicorn больше не упоминает вовсе.
>
> А затем **у gunicorn появился свой ASGI-воркер** — в версии **24.0.0** бетой, стабильный с **25.1.0**. Уводить ASGI через uvicorn-воркер больше не нужно:
>
> ```bash
> gunicorn main:app -k asgi -w 4
> ```
>
> Настройки, логирование и надзор за процессами при этом гуникорновские — [подробнее](../DevOps/Развертывание%20проекта.md#классы-воркеров).

**В Kubernetes воркеры не нужны совсем.** Роль «следить, что процессы живы, и поднимать упавшие» там уже выполняет сам кластер: `Deployment` держит заданное число реплик, `livenessProbe` перезапускает зависший под. Ставить внутрь пода ещё один супервизор — дублировать оркестратор и портить себе метрики и лимиты памяти (они считаются на под). Поэтому в контейнере — **один процесс uvicorn**, а масштабирование задаёт `replicas`.

| Где крутится | Что запускать | Кто перезапускает упавшее |
|---|---|---|
| Один сервер, systemd | `uvicorn --workers N` или `gunicorn -k asgi -w N` | менеджер процессов uvicorn/gunicorn |
| Docker без оркестратора | один uvicorn на контейнер + `restart: always` | демон Docker |
| [Kubernetes](../DevOps/Kubernetes.md) | один uvicorn на контейнер, `replicas: N` | сам кластер (`Deployment`, `livenessProbe`) |

**В контейнере — наоборот, один процесс на контейнер.** Не поднимай `--workers` внутри Docker: пусть контейнер запускает **один** uvicorn, а масштабированием (репликами) занимается оркестратор ([Kubernetes](../DevOps/Kubernetes.md)/Docker Swarm). Так проще рестартить, следить за памятью и катить обновления.

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
# один процесс на контейнер; реплики — на уровне оркестратора
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

> Воркеры и контейнеры дают только **репликацию**. HTTPS, автозапуск и мониторинг памяти — отдельно: TLS и проксирование — nginx ([Развертывание проекта](../DevOps/Развертывание%20проекта.md#шаг-7-nginx--обратный-прокси-и-статика)), автоподъём — systemd или оркестратор.

## Связи

- [FastAPI](../Фреймворки/FastAPI.md) — механика фреймворка: типы→валидация, автодокументация, uvicorn, ловушка блокировки loop
- [Depends — зависимости в FastAPI](../Фреймворки/Depends%20—%20зависимости%20в%20FastAPI.md) — все формы зависимостей, `yield`, кэш запроса, подмена в тестах
- [Тестирование FastAPI-приложений](../Фреймворки/Тестирование%20FastAPI-приложений.md) — `TestClient`, async-тесты, `dependency_overrides`, тестовая БД
- [API](../Фреймворки/DJANGO/API.md#что-такое-rest-api) — принципы REST (ресурсы, stateless, единый интерфейс); DRF как sync-альтернатива
- [Asyncio Event Loop и aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — event loop и `async/await`, на которых стоит FastAPI
- [pydantic](../Библиотеки/Сторонние/pydantic.md) — модели запроса/ответа и валидация
- [Юнит-тестирование Python](../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md) · [pytest](../Библиотеки/Сторонние/Тесты/pytest.md) — как тестировать эндпоинты
- [Разделение доступа в DRF](../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md) · [Oauth 2.0](../Библиотеки/Сторонние/Работа%20с%20WEB/Oauth%202.0.md) — аутентификация/JWT (те же идеи для FastAPI через `Depends`)
- [Развертывание проекта](../DevOps/Развертывание%20проекта.md) — деплой ASGI-приложения (uvicorn под gunicorn)

## Источники

- [FastAPI — Bigger Applications (APIRouter)](https://fastapi.tiangolo.com/tutorial/bigger-applications/)
- [FastAPI — Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [FastAPI — Response Status Code](https://fastapi.tiangolo.com/tutorial/response-status-code/)
- [FastAPI — Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [FastAPI — SQL (Relational) Databases](https://fastapi.tiangolo.com/tutorial/sql-databases/)
- [FastAPI — Server Workers](https://fastapi.tiangolo.com/deployment/server-workers/)
- [FastAPI — FastAPI in Containers (Docker)](https://fastapi.tiangolo.com/deployment/docker/)
- [Uvicorn 0.30.0 — новый менеджер процессов и устаревание `uvicorn.workers`](https://marcelotryle.com/blog/2024/05/28/uvicorn-0300-release/)
- [uvicorn-worker — вынесенный класс воркера для gunicorn](https://pypi.org/project/uvicorn-worker/)
- [MDN — HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [REST API: принципы работы и методы — Яндекс Практикум](https://practicum.yandex.ru/blog/chto-takoe-rest-api-i-kak-rabotaet/)
