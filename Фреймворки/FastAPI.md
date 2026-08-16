[← Оглавление](../README.md)

# FastAPI

> _Аннотации типов Python превращаются в валидацию, документацию и подсказки — бесплатно_

---

**FastAPI** — современный веб-фреймворк для API на [Python](../Python.md), построенный на трёх идеях: **аннотации типов** (из них рождаются валидация и документация), **асинхронность** (ASGI, конкурентные запросы) и **автодокументация** (интерактивная страница API из коробки). Быстрый в разработке и в рантайме, поэтому стал главной альтернативой [DRF](../Фреймворки/DJANGO/API.md) для чистых API.

## Три опоры FastAPI

- **Аннотации типов.** Ты объявляешь типы аргументов и тел запросов — FastAPI по ним **сам** валидирует входные данные, приводит типы и генерирует документацию. Один источник правды вместо трёх.
- **Асинхронность.** Построен на ASGI (Starlette + [asyncio](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md)): обработчики можно писать `async def` и держать тысячи одновременных I/O-запросов.
- **Автодокументация.** По коду строится схема **OpenAPI**, а из неё — интерактивные страницы Swagger UI и ReDoc, где API можно потыкать прямо в браузере.

## Минимальное приложение

```python
from fastapi import FastAPI

app = FastAPI()

@app.get('/')                      # маршрут + HTTP-метод одним декоратором
async def root():
    return {'status': 'ok'}        # dict сам сериализуется в JSON
```

Запуск — ASGI-сервером **uvicorn**:

```bash
pip install "fastapi[standard]"
uvicorn main:app --reload          # main.py, объект app; --reload для разработки
```

## Параметры пути и запроса

Тип аргумента — это и есть правило валидации. FastAPI сам разберёт URL и приведёт типы:

```python
@app.get('/items/{item_id}')            # item_id из пути
async def get_item(item_id: int, q: str | None = None):
    #   item_id: int  → часть пути, приводится к int (нет — 422 с понятной ошибкой)
    #   q: str = None → необязательный query-параметр  ?q=...
    return {'item_id': item_id, 'q': q}
```

Пришёл `/items/abc` вместо числа — FastAPI **сам** вернёт `422` с описанием, что не так. Ручных проверок не нужно.

## Pydantic-модели — тело запроса

Тело POST/PUT описывают классом [Pydantic](../Библиотеки/Сторонние/pydantic.md): поля с типами. FastAPI провалидирует входной JSON по этой модели:

```python
from pydantic import BaseModel

class Product(BaseModel):
    name: str
    price: float
    in_stock: bool = True          # значение по умолчанию → поле необязательно

@app.post('/products/')
async def create_product(product: Product):    # тело запроса валидируется по Product
    return {'created': product.name, 'price': product.price}
```

Клиент прислал `price: "дорого"` — вернётся 422 с точным указанием поля. Прислал корректно — в обработчик придёт готовый типизированный объект.

## Автодокументация

Пока ты писал код, FastAPI построил интерактивную документацию — открой в браузере при запущенном сервере:

- **`/docs`** — Swagger UI: список эндпоинтов, схемы, кнопка «выполнить запрос»;
- **`/redoc`** — ReDoc: та же схема в виде читаемого справочника.

Ничего дополнительно писать не нужно — документация всегда соответствует коду.

## Часто используемые возможности

Каталог того, что нужно почти в каждом проекте.

### Зависимости (`Depends`)

Внедрение зависимостей — самый используемый инструмент фреймворка после Pydantic-моделей. Функция-зависимость готовит ресурс (сессию БД, текущего пользователя, параметры пагинации), FastAPI вызывает её сам и подставляет результат в аргумент:

```python
from typing import Annotated
from fastapi import Depends

async def get_session():
    async with Session() as session:      # код до yield — перед ручкой
        yield session                      # после yield — когда ручка отработала

SessionDep = Annotated[AsyncSession, Depends(get_session)]

@app.get('/products')
async def list_products(session: SessionDep): ...
```

Так одна транзакция живёт ровно один HTTP-запрос и сама закрывается, а в тестах её подменяют одной строкой. Все формы (класс, вызываемый экземпляр, `dependencies=[...]` на роутере), кэш в пределах запроса и ловушки — [Depends — зависимости в FastAPI](../Фреймворки/Depends%20—%20зависимости%20в%20FastAPI.md).

### Валидация параметров: `Query`, `Path`, `Field`

Аннотации типа задают *тип*; `Query`/`Path`/`Field` добавляют **ограничения** и метаданные (для валидации и `/docs`):

```python
from fastapi import Query, Path
from pydantic import BaseModel, Field

@app.get('/items')
async def search(
    q: str = Query(min_length=3, max_length=50),        # длина строки
    page: int = Query(1, ge=1),                          # >= 1, по умолчанию 1
    item_id: int = Path(gt=0),                           # > 0 в пути
):
    ...

class Product(BaseModel):
    name: str = Field(min_length=1)
    price: float = Field(gt=0, description='Цена в рублях')   # > 0 + описание в доках
```

Не прошло ограничение — FastAPI сам вернёт `422` с указанием поля.

### Модель и код ответа

```python
@app.post('/products', response_model=ProductOut,          # форма ответа (лишние поля отсекутся)
          status_code=201,                                  # код успеха
          response_model_exclude_unset=True)                # не слать поля, оставшиеся по умолчанию
async def create(p: ProductIn): ...
```

`response_model` важен для безопасности: например, отдать `UserOut` без поля `password`, даже если оно есть в объекте.

### Формы и загрузка файлов

JSON — не всегда; для HTML-форм и файлов есть `Form` и `UploadFile`:

```python
from fastapi import Form, File, UploadFile

@app.post('/login')
async def login(username: str = Form(), password: str = Form()): ...

@app.post('/upload')
async def upload(file: UploadFile = File()):
    data = await file.read()          # UploadFile — стрим, не грузит всё в память сразу
    return {'filename': file.filename, 'size': len(data)}
```

> Сам файл обычно не держат на сервере и не пишут в БД, а заливают в объектное хранилище (S3) — в базе хранят лишь ключ. Готовый сценарий — в Сквозной пример: загрузка аватара через FastAPI.

### Заголовки и куки

```python
from fastapi import Header, Cookie

@app.get('/whoami')
async def whoami(user_agent: str = Header(None),   # заголовок User-Agent (снейк→кебаб авто)
                 session: str = Cookie(None)):      # cookie session
    ...
```

### Фоновые задачи (`BackgroundTasks`)

Лёгкая работа **после** ответа, без брокера: отправить письмо, записать лог. Для тяжёлого/долгого — [Celery](../Библиотеки/Сторонние/Celery.md).

```python
from fastapi import BackgroundTasks

@app.post('/register')
async def register(email: str, bg: BackgroundTasks):
    bg.add_task(send_welcome_email, email)   # выполнится после отправки ответа
    return {'ok': True}                       # клиент не ждёт письмо
```

**Куда попадёт задача — решает Starlette, не ты.** `BackgroundTasks` — механизм не FastAPI, а Starlette, поверх которого он построен. При добавлении задачи Starlette смотрит, корутина функция или нет:

- **`async def`-задача** — выполнится в том же **event loop**, когда до неё дойдёт очередь;
- **обычная `def`-задача** — уйдёт в **пул потоков**, чтобы не блокировать loop.

Отсюда частое заблуждение: «в фон выносят только синхронные функции». Нет — асинхронную длинную функцию (сходить в базу, потом в S3, потом в чужой API) выносить в фон можно и нужно.

> ⚠️ **Задачи выполняются последовательно, а не параллельно.** `BackgroundTasks.__call__` — обычный цикл `for task in tasks: await task()`. Три задачи по 10 секунд займут 30, а не 10. Нужна одновременность — либо `asyncio.gather` внутри одной задачи, либо очередь ([Celery](../Библиотеки/Сторонние/Celery.md)).

**Ловушка: ресурс из зависимости.** Сессия БД, полученная через `Depends` с `yield`, живёт ровно столько, сколько живёт запрос, — и её время жизни относительно фоновой задачи **менялось от версии к версии** (в 0.106.0 выход из зависимости перенесли до отправки ответа, в 0.118.0 вернули обратно). Надёжный приём, не зависящий от версии: передавать в задачу **id объекта**, а внутри задачи открывать своё соединение.

```python
# ХРУПКО — сессия принадлежит запросу, а не задаче:
@app.post('/orders')
async def create(bg: BackgroundTasks, session: SessionDep):
    order = await repo.create(session, ...)
    bg.add_task(send_receipt, session, order)      # сессия может быть уже закрыта

# НАДЁЖНО — задача сама заводит себе ресурс:
async def send_receipt(order_id: int):
    async with Session() as session:               # своё соединение
        order = await session.get(Order, order_id)
        ...

bg.add_task(send_receipt, order.id)                # передаём только id
```

**Минусы — почему не всё через `BackgroundTasks`:**

- **Тот же процесс.** Задача крутится внутри воркера приложения. Упал или перезапустился процесс до её выполнения — задача **потеряна безвозвратно** (нет персистентности).
- **Нет очереди, ретраев и расписания.** Упала с ошибкой — никто не повторит; отложить «через час» нельзя.
- **Не масштабируется между машинами.** Выполняется только на том же сервере — нельзя раскидать нагрузку на пул отдельных воркеров.
- **CPU-задача заблокирует воркер.** `sync`-функция с тяжёлым счётом займёт поток, а `async`-функция — вовсе весь event loop (см. [ниже](#cpu-задачи-io-bound-против-cpu-bound)).
- **Нет наблюдаемости.** Не видно, выполнилась задача или нет, — ни статуса, ни результата.

> Правило: `BackgroundTasks` — для **лёгкого, некритичного, «выстрелил и забыл»** (лог, уведомление). Нужна гарантия выполнения, ретраи, тяжёлый счёт или несколько машин — это [Celery](../Библиотеки/Сторонние/Celery.md) с брокером (RabbitMQ/[Redis](../Библиотеки/Сторонние/Redis.md)).

### CORS и middleware

**Middleware** оборачивает каждый запрос/ответ. Самый частый — CORS (пустить фронтенд с другого домена; механика — Политика одного источника и CORS):

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(CORSMiddleware,
                   allow_origins=['https://myfront.com'],
                   allow_methods=['*'], allow_headers=['*'])
```

### Ограничение частоты запросов (rate limiting)

Защита от перебора и злоупотреблений: сколько запросов в единицу времени можно с одного клиента. Главный вопрос — **на каком уровне** ставить ограничитель:

| Уровень | Чем | Когда |
|---|---|---|
| **Периметр** (reverse-proxy / API-gateway) | nginx `limit_req`, gateway, Cloudflare | Грубый заслон от флуда/DDoS **до** того, как запрос дойдёт до Python. Дёшево, но не знает бизнес-логики (кто пользователь, какой тариф) |
| **Приложение** (FastAPI) | `slowapi` (+ [Redis](../Библиотеки/Сторонние/Redis.md)) | Лимиты по пользователю/эндпоинту/тарифу — там, где нужна логика. Дороже: запрос уже добрался до приложения |

Обычно совмещают: грубый заслон на периметре + точные лимиты в приложении.

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

# storage_uri в Redis → счётчик ОБЩИЙ для всех воркеров/машин (без него — свой на процесс)
limiter = Limiter(key_func=get_remote_address, storage_uri='redis://localhost:6379')
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)  # при превышении → 429

@app.get('/search')
@limiter.limit('5/minute')        # не больше 5 запросов в минуту с одного IP
async def search(request: Request):   # параметр request обязателен — slowapi берёт из него IP
    ...
```

> В кластере из нескольких воркеров/подов счётчик должен быть **общим** — иначе каждый процесс считает свой лимит, и реальный порог кратно выше заданного. Поэтому хранилище — [Redis](../Библиотеки/Сторонние/Redis.md), а не память процесса. В DRF та же идея зовётся throttling ([Разделение доступа в DRF](../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md)).

**Частота против квоты — это разные ограничения**, и их постоянно смешивают под одним словом «rate limit»:

| | Ограничение частоты | Квота |
|---|---|---|
| Что считает | запросы **за окно времени** («5 в минуту») | одновременные или суммарные операции («не больше 3 генераций разом», «100 отчётов в месяц») |
| Зачем | защита от флуда и перегрузки | бизнес-модель: тариф, себестоимость ресурса |
| Где живёт | периметр и приложение | **только приложение** — периметр не знает про тарифы |
| Чем считается | счётчик с TTL | состояние в БД (сколько задач у пользователя сейчас в работе) |

Пример квоты: в тарифе указано «не более трёх одновременных генераций видео». На nginx это не выразить — он не знает ни пользователя, ни его тариф, ни сколько задач у него уже запущено. Значит, проверка идёт в приложении, перед постановкой задачи в очередь, и опирается на реальное состояние, а не на счётчик запросов.

### Свои обработчики ошибок

Превратить исключение в аккуратный JSON-ответ единообразно по всему API. Работает и для **своих**, и для **стандартных** исключений — регистрируешь обработчик на класс через `@app.exception_handler(...)`.

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class NotFound(Exception): ...

@app.exception_handler(NotFound)                      # свой класс
async def not_found_handler(request: Request, exc: NotFound):
    return JSONResponse(status_code=404, content={'detail': 'Не найдено'})
```

**Перехватить стандартную ошибку и отдать свой формат.** Две встроенные, которые переопределяют чаще всего:

- **`RequestValidationError`** — FastAPI кидает её сам, когда вход не прошёл валидацию (по умолчанию `422` с подробным `detail`). Перехватываешь — отдаёшь свою форму и код.
- **`HTTPException`** — то, что ты кидаешь через `raise HTTPException(...)`. Обработчик регистрируют на **`StarletteHTTPException`** (базовый класс), хотя в коде кидают `fastapi.HTTPException`.

```python
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)        # переопределить стандартный 422
async def validation_handler(request, exc: RequestValidationError):
    return JSONResponse(status_code=400,              # свой код и форма
        content={'error': 'bad_request', 'fields': exc.errors()})

@app.exception_handler(StarletteHTTPException)         # все HTTPException разом
async def http_handler(request, exc):
    return JSONResponse(status_code=exc.status_code,
        content={'error': exc.detail})
```

> Обработчик вешают **на класс** исключения — FastAPI ищет его по типу с учётом наследования (поэтому регистрация на базовый `StarletteHTTPException` ловит и `fastapi.HTTPException`). Нужно лишь **обернуть** стандартное поведение (залогировать и отдать дефолт) — импортируй готовые `http_exception_handler` / `request_validation_exception_handler` из `fastapi.exception_handlers` и вызови их из своего.

### События старта и остановки (`lifespan`)

Открыть пул соединений к БД/[Redis](../Библиотеки/Сторонние/Redis.md) при старте и закрыть при остановке — один раз, а не на каждый запрос:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis = await connect_redis()   # старт
    yield
    await app.state.redis.close()             # остановка

app = FastAPI(lifespan=lifespan)
```

### Настройки через `pydantic-settings`

Конфиг из переменных окружения с валидацией типов — та же идея, что у [Pydantic](../Библиотеки/Сторонние/pydantic.md):

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    redis_url: str = 'redis://localhost'      # значение по умолчанию
    debug: bool = False

settings = Settings()          # читает DATABASE_URL, REDIS_URL, DEBUG из окружения/.env
```

### WebSocket

Двусторонний канал реального времени (чат, уведомления) — поверх того же приложения:

```python
from fastapi import WebSocket

@app.websocket('/ws')
async def ws(websocket: WebSocket):
    await websocket.accept()
    while True:
        msg = await websocket.receive_text()
        await websocket.send_text(f'эхо: {msg}')
```

## FastAPI против Flask и Django

| | FastAPI | [Flask](../Фреймворки/FLASK.md) | [Django](../Фреймворки/DJANGO/Django.md) + DRF |
|---|---|---|---|
| **Модель** | async (ASGI) | sync (WSGI) | sync (DRF) |
| **Валидация** | из типов (Pydantic) | вручную | сериализаторы DRF |
| **Автодокументация** | из коробки (OpenAPI) | нет | сторонние пакеты |
| **Что внутри** | только API | микро, всё сам | «батарейки»: ORM, админка |
| **Когда брать** | быстрый async API | простой сервис | большой сайт с админкой |

## Ловушка: блокирующий код в async def

Обработчик `async def` крутится в **одном** event loop. Синхронный вызов, который *ждёт* (обычная БД, `requests.get`, `time.sleep`), блокирует весь loop — и сервер не обрабатывает другие запросы, пока не дождётся. Вся выгода асинхронности теряется.

```python
# НЕПРАВИЛЬНО — блокирует event loop:
@app.get('/data')
async def get_data():
    r = requests.get('https://slow-api.com')    # синхронный, ЖДЁТ → loop стоит
    return r.json()

# ПРАВИЛЬНО (1) — асинхронные инструменты с await:
@app.get('/data')
async def get_data():
    async with aiohttp.ClientSession() as s:     # не блокирует loop
        async with s.get('https://slow-api.com') as r:
            return await r.json()

# ПРАВИЛЬНО (2) — обычный def: FastAPI сам вынесет его в пул потоков:
@app.get('/data')
def get_data():                                  # sync-обработчик → threadpool
    return requests.get('https://slow-api.com').json()
```

> **Правило.** В `async def` — только неблокирующие (`await`) вызовы. Есть лишь синхронный код (обычный драйвер БД, `requests`) — объявляй обработчик обычным `def`, и FastAPI выполнит его в отдельном потоке, не блокируя loop. Худший вариант — синхронный `requests` внутри `async def`.

## CPU-задачи: I/O-bound против CPU-bound

Ловушка выше — про **I/O-bound** (ждём сеть, БД, диск): там помогает `await` или вынос `sync`-кода в пул потоков. **CPU-bound** нагрузка (хеширование, обработка картинок, парсинг, ML-инференс) — другая история: она не *ждёт*, а *считает*, и потоки её не ускоряют из-за GIL.

**Почему threadpool не спасает CPU.** Обычный `def`-обработчик FastAPI выносит в пул потоков — это снимает блокировку loop для *ожидающего* кода. Но **GIL** пускает в интерпретатор Python только **один поток за раз**: пока один считает, остальные стоят. Для вычислений потоки не дают параллелизма — нужен **отдельный процесс**.

```python
# ПЛОХО — тяжёлый счёт прямо в async-обработчике:
@app.post('/hash')
async def hash_pw(data: bytes):
    return {'h': bcrypt(data)}   # блокирует ВЕСЬ event loop на всё время счёта
```

**Куда выносить CPU-задачу — по нарастанию тяжести:**

- **Короткий счёт (десятки мс)** — в пул процессов через event loop, не блокируя его:

  ```python
  import asyncio
  from concurrent.futures import ProcessPoolExecutor

  pool = ProcessPoolExecutor()          # процессы обходят GIL

  @app.post('/hash')
  async def hash_pw(data: bytes):
      loop = asyncio.get_running_loop()
      h = await loop.run_in_executor(pool, bcrypt, data)   # считает в другом процессе
      return {'h': h}
  ```

- **Долгий или массовый счёт (секунды и больше)** — вынести из веб-процесса совсем, в [Celery](../Библиотеки/Сторонние/Celery.md): обработчик ставит задачу в очередь и сразу отвечает `202 Accepted` с id, а воркеры считают отдельно.

> **Правило.** **I/O-bound** → `async` + `await` (или `sync def` → threadpool). **CPU-bound** → отдельные процессы: `ProcessPoolExecutor` для короткого, [Celery](../Библиотеки/Сторонние/Celery.md)/[multiprocessing](../Библиотеки/Модули/Параллелизм/multiprocessing.md) для тяжёлого. Threadpool из-за GIL вычисления не ускорит. Само деление и как его определять — в [CPU-bound против IO-bound](../Библиотеки/Модули/Параллелизм/CPU-bound%20против%20IO-bound.md).

## Сквозной пример: мини-API товаров

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()
products: dict[int, dict] = {}          # хранилище в памяти (вместо БД для примера)

class Product(BaseModel):               # 1. модель тела запроса
    name: str
    price: float

@app.post('/products/{pid}')            # 2. создать
async def create(pid: int, product: Product):
    products[pid] = product.model_dump()
    return {'created': pid}

@app.get('/products/{pid}')             # 3. прочитать (404, если нет)
async def read(pid: int):
    if pid not in products:
        raise HTTPException(status_code=404, detail='Не найдено')
    return products[pid]
```

**Как это читать.** Типы аргументов (`pid: int`) и модель `Product` задают валидацию — FastAPI проверяет вход сам и отдаёт понятные ошибки. `HTTPException` — стандартный способ вернуть код ошибки. Открыв `/docs`, оба эндпоинта можно сразу протестировать в браузере. В реальном проекте `dict` заменяют на [async SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md) + PostgreSQL.

## Связи

- [REST API на FastAPI](../Фреймворки/REST%20API%20на%20FastAPI.md) — сборка полного REST-сервиса: CRUD со статус-кодами, роутеры, `Depends`, async-БД, деплой;
- [Depends — зависимости в FastAPI](../Фреймворки/Depends%20—%20зависимости%20в%20FastAPI.md) — зависимости целиком: формы, `yield`, кэш запроса, подмена в тестах;
- [Тестирование FastAPI-приложений](../Фреймворки/Тестирование%20FastAPI-приложений.md) — `TestClient` и `AsyncClient`, `dependency_overrides`, тестовая БД;
- [Asyncio Event Loop и aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — async-модель и event loop, на которых стоит FastAPI;
- [pydantic](../Библиотеки/Сторонние/pydantic.md) — валидация и модели данных, ядро FastAPI (и `pydantic-settings` для конфига);
- [Celery](../Библиотеки/Сторонние/Celery.md) — тяжёлые фоновые задачи с брокером; `BackgroundTasks` — для лёгких, без брокера;
- [Redis](../Библиотеки/Сторонние/Redis.md) — кеш и сессии; пул соединений поднимают в `lifespan`;
- [API](../Фреймворки/DJANGO/API.md) — DRF как «тяжёлая» альтернатива для API поверх Django;
- [FLASK](../Фреймворки/FLASK.md) — синхронный микрофреймворк, идейный предшественник;
- [WSGI и ASGI — интерфейс сервера и приложения](../WSGI%20и%20ASGI%20—%20интерфейс%20сервера%20и%20приложения.md) — что такое ASGI под FastAPI: `scope/receive/send`, откуда взялся `lifespan` и почему middleware устроены как декораторы;
- [Развёртывание](../DevOps/Развертывание%20проекта.md#шаг-5-gunicorn--сервер-приложений) — FastAPI катят через ASGI-воркер uvicorn под gunicorn.

## Источники

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [FastAPI — Tutorial: First steps](https://fastapi.tiangolo.com/tutorial/first-steps/)
- [FastAPI — Request body (Pydantic)](https://fastapi.tiangolo.com/tutorial/body/)
- [FastAPI — Background Tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/)
- [Starlette — исходник `background.py`](https://github.com/encode/starlette/blob/master/starlette/background.py) — sync→пул потоков, async→event loop, задачи в цикле по очереди
- [FastAPI — Handling Errors](https://fastapi.tiangolo.com/tutorial/handling-errors/)
- [FastAPI — Concurrency and async / await](https://fastapi.tiangolo.com/async/)
- [SlowAPI — rate limiter for Starlette and FastAPI](https://slowapi.readthedocs.io/)
- [nginx — ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [Uvicorn](https://www.uvicorn.org/)
- [FastAPI — что это и зачем нужен — Яндекс Практикум](https://practicum.yandex.ru/blog/fastapi-chto-eto-i-zachem-nuzhen/) — русскоязычное введение
