[← Оглавление](../../../README.md)

# Asyncio Event Loop и aiohttp

> _«Пока ждём ответа от сети — займёмся другим делом, а не сидим сложа руки»_

---

**Асинхронность** позволяет одному потоку обрабатывать тысячи операций, которые в основном **ждут** (запрос по сети, чтение диска, ответ базы), переключаясь между ними вместо простоя. **asyncio** — встроенный в Python фреймворк для такого кода, а **aiohttp** — построенная на нём библиотека для асинхронного HTTP: и клиент (замена [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md)), и сервер (замена Flask).

## Зачем асинхронность

Задачи делятся на два типа:

- **I/O-bound** (ограничены вводом-выводом) — большую часть времени **ждут** внешний ресурс: ответ API, запрос к базе, файл с диска. Процессор при этом простаивает.
- **CPU-bound** — упираются в вычисления: перемножение матриц, хеширование, обработка изображений.

Асинхронность выигрывает именно на **I/O-bound**: пока одна корутина ждёт ответа сети, event loop запускает другую. Для CPU-bound она бесполезна — там нужны реальные ядра ([multiprocessing](../../../Библиотеки/Модули/Параллелизм/multiprocessing.md)).

```python
import requests

def get_all():
    for i in range(10):
        requests.get(f'https://api.example.com/{i}')   # каждый запрос БЛОКИРУЕТ
#   10 запросов по 200 мс = 2 секунды: ждём последовательно
```

Синхронный код ждёт каждый ответ, прежде чем послать следующий запрос. Асинхронный отправит все сразу и подождёт их **одновременно** — те же 10 запросов уложатся в ~200 мс.

> **Async vs потоки vs процессы.** Потоки в Python ограничены GIL и тяжелее; процессы ([multiprocessing](../../../Библиотеки/Модули/Параллелизм/multiprocessing.md)) дают настоящий параллелизм, но дороги по памяти. Async — самый лёгкий способ **конкурентности** для I/O: всё в одном потоке, переключение почти бесплатно.

## Что асинхронность не ускоряет

Самое живучее заблуждение — «переписал на `async` и стало быстрее». Асинхронность ускоряет не код, а **обслуживание нескольких задач сразу**. Отсюда два практических следствия.

### Одну функцию `await` не ускоряет

Асинхронная функция, взятая отдельно, работает **ровно столько же, сколько синхронная, — и даже чуть дольше**: сам запрос быстрее не стал, а сверху добавились планирование корутины и переключения loop'а.

```
синхронный   get_profile():  ~200 мс   ← время ответа внешнего API
асинхронный  get_profile():  ~203 мс   ← то же ожидание + накладные расходы asyncio
```

Отдельно взятый пользователь ответ быстрее **не получит**. Выигрыш появляется, когда корутин минимум две: пока одна ждёт ответа от API, базы или кэша, интерпретатор не простаивает, а прогоняет вторую, третью, сотую. Ускоряется не функция, а **пропускная способность сервиса**.

> Именно поэтому async — вопрос денег, а не только скорости. Обслужить 1000 запросов в секунду умеют и потоки, и корутины, но корутине нужны килобайты вместо мегабайт стека, поэтому на асинхронное приложение железа уйдёт заметно меньше. До появления `asyncio` веб-приложения (Flask, Django) масштабировали именно потоками — работало, но дороже.

### Затык обычно не в асинхронности

Второе следствие первого: раз async не ускоряет отдельный запрос, то и «медленный эндпоинт» он не лечит. Простой `SELECT` по первичному ключу, выполняющийся 3 секунды, — проблема не корутин.

Типичные настоящие причины:

- **Пул соединений к БД мал.** У PostgreSQL `max_connections` по умолчанию — около 100. При 500–1000 запросов в секунду свободных соединений просто нет, и корутины дружно ждут в очереди за пулом, а не за данными.
- **Нет кэша там, где он напрашивается.** Одни и те же данные каждый раз считаются базой заново. [Redis](../../../Библиотеки/Сторонние/Redis.md) отдаёт их из оперативной памяти без нагрузки на CPU базы — см. Стратегии кэширования.
- **Запрос плохо написан.** Нет индекса, [N+1](../../../Фреймворки/DJANGO/ORM%20Django.md#ловушка-n1-запросов), лишние джойны — асинхронность ускорит доставку плохого плана, но не сам план.
- **Внешний сервис отдаёт медленно** или режет по лимиту (`429`) — тогда нужны таймауты, ретраи и circuit breaker, а не корутины.

> Вывод, который стоит держать в голове: **асинхронность решает многие проблемы, но далеко не все.** Перед тем как переписывать код, надо понять систему целиком и [замерить, где именно упирается](../../../Библиотеки/Сторонние/Тесты/Нагрузочное%20тестирование%20—%20locust%20и%20k6.md). Иначе оптимизируется то, что и так не было узким местом.

## Event loop — сердце asyncio

**Event loop** (цикл событий) — ядро любой asyncio-программы. Это бесконечный цикл, который держит очередь задач, запускает их по одной и, когда задача натыкается на `await` (начинает чего-то ждать), **откладывает** её и берёт следующую готовую. Как только ожидаемое приходит — возвращается к отложенной.

```
        ┌─────────────── Event Loop ───────────────┐
        │  очередь готовых задач                    │
        │  ┌────┐ ┌────┐ ┌────┐                     │
        │  │ T1 │ │ T2 │ │ T3 │  ← выполняются по   │
        │  └─┬──┘ └────┘ └────┘    очереди          │
        │    │ await (ждёт сеть)                    │
        │    └──▶ отложена, пока не придёт ответ ───┘
        └───────────────────────────────────────────┘
```

Запускает и закрывает цикл одна команда:

```python
import asyncio

asyncio.run(main())   # создать event loop, выполнить корутину main(), закрыть loop
```

## Корутина и async/await

**Корутина** — функция, объявленная через `async def`. Её особенность: вызов **не запускает** её сразу, а возвращает объект-корутину, который нужно **ждать** через `await`.

```python
import asyncio

async def get_people(people_id):          # объявляем корутину
    await asyncio.sleep(1)                # имитируем ожидание ответа сети
    return {'id': people_id}

async def main():
    coroutine = get_people(1)             # ← корутина СОЗДАНА, но НЕ выполнена
    result = await coroutine              # ← await запускает и ждёт результат
    print(result)

asyncio.run(main())
```

- **`async def`** — делает функцию корутиной.
- **`await`** — «дождись этого и верни результат»; на этом месте loop может переключиться на другую задачу. `await` работает только внутри `async def`.

Ошибка новичка №1 — забыть `await` перед корутиной:

```python
# НЕПРАВИЛЬНО — корутина создана, но не запущена
async def main():
    get_people(1)          # RuntimeWarning: coroutine was never awaited
    # тело корутины даже не выполнилось, результат потерян

# ПРАВИЛЬНО — await запускает корутину и ждёт результат
async def main():
    result = await get_people(1)
```

## Задачи (Task) и конкурентность

Если просто писать `await` подряд, корутины выполнятся **последовательно** — никакого выигрыша:

```python
async def main():
    await get_people(1)   # ждём 1 секунду
    await get_people(2)   # потом ещё 1 — итого 2 секунды
```

Чтобы запустить их **одновременно**, корутину заворачивают в **Task** через `create_task` — loop планирует её выполнение сразу, не дожидаясь `await`:

```python
async def main():
    t1 = asyncio.create_task(get_people(1))   # ← стартует сразу, в фоне
    t2 = asyncio.create_task(get_people(2))   # ← тоже сразу
    r1, r2 = await t1, await t2               # ждём уже запущенные — шли параллельно
    print(r1, r2)                             # ~1 секунда вместо 2
```

Для пачки корутин удобнее `asyncio.gather` — он сам заворачивает их в Task, запускает конкурентно и собирает результаты по порядку:

```python
async def main():
    coros = [get_people(i) for i in range(1, 11)]   # 10 корутин
    results = await asyncio.gather(*coros)          # запустить ВСЕ разом и дождаться
    print(results)                                  # ~1 секунда вместо 10
```

- **`asyncio.create_task(coro)`** — обернуть корутину в Task и сразу поставить в очередь loop.
- **`asyncio.gather(*coros)`** — запустить много корутин конкурентно, вернуть список результатов **в порядке аргументов**.

### Последовательный `await` — самая частая ошибка в продакшене

Код ниже встречается даже в боевых проектах. Три запроса выполняются по очереди, хотя ни один не нуждается в результате предыдущего:

```python
# НЕПРАВИЛЬНО — 3 независимых запроса подряд: 100 + 100 + 100 = 300 мс
async def get_dashboard(user_id: int):
    user = await fetch_user(user_id)             # зависит только от аргумента
    orders = await fetch_orders(user_id)         # тоже только от аргумента
    feed = await fetch_recommendations(user_id)  # и этот
    return user, orders, feed

# ПРАВИЛЬНО — те же три запроса конкурентно: ≈100 мс
async def get_dashboard(user_id: int):
    user, orders, feed = await asyncio.gather(
        fetch_user(user_id),                     # ← передаём объекты корутин,
        fetch_orders(user_id),                   #   а не результаты: без await!
        fetch_recommendations(user_id),
    )
    return user, orders, feed
```

**Признак, по которому это видно:** все аргументы каждой строки известны ещё на входе в функцию. Значит, порядок выполнения не важен и строки можно запускать разом.

> ⚠️ В `gather` передают **корутины, а не вызовы с `await`**. Написать `await fetch_user(...)` внутри `gather` — значит снова выполнить всё последовательно, ещё до того как `gather` начнёт работу.

### Когда распараллелить нельзя — и это нормально

Обратный случай выглядит почти так же, но каждая строка нуждается в предыдущей:

```python
user = await fetch_user(user_id)               # нужен user…
company = await fetch_company(user.company_id)  # …чтобы узнать компанию
account = await fetch_account(company.account_id)  # …и только потом счёт
```

Такую цепочку `gather` не спасёт: это не ошибка, а природа задачи. Ускорять её надо не конкурентностью, а сокращением числа обращений:

- **`JOIN` вместо трёх запросов** — база отдаст всё разом, и три похода превратятся в один;
- **кэш**, если данные часто повторяются и занимают немного памяти.

> Что именно окажется быстрее — вопрос замеров, а не правил. Один запрос на 100 мс и два по 5 мс требуют разных решений, чем три по 50 мс. Сначала измерь, потом переписывай.

## TaskGroup — современный способ (3.11+)

С Python 3.11 предпочтительна **`asyncio.TaskGroup`** — «структурная конкурентность»: группа задач с гарантией, что все дождутся, а если одна упадёт — остальные корректно отменятся.

```python
async def main():
    async with asyncio.TaskGroup() as tg:       # блок ждёт все задачи на выходе
        for i in range(1, 11):
            tg.create_task(get_people(i))
#   после выхода из блока все 10 задач гарантированно завершены
```

## Три вида awaitable

`await` можно применять к трём типам объектов:

| Объект | Что это | Когда встречаешь |
|---|---|---|
| **Корутина** | результат вызова `async def` | пишешь сам |
| **Task** | корутина, запланированная в loop | `create_task`, `gather` |
| **Future** | низкоуровневое «обещание результата» | внутри библиотек, редко вручную |

Task — это подкласс Future, приспособленный оборачивать корутины. В обычном коде хватает корутин и Task.

## Таймауты и отмена

Корутина, ждущая сеть, без таймаута ждёт **вечно**. Внешний сервис завис — задача висит вместе с ним, и висят все, кто её ждёт. Поэтому у любого сетевого ожидания должен быть предел.

С Python 3.11 предпочтительный способ — **`asyncio.timeout()`**, асинхронный менеджер контекста:

```python
async def main():
    try:
        async with asyncio.timeout(10):        # 10 секунд на весь блок
            await long_running_task()
    except TimeoutError:                        # ловить СНАРУЖИ блока
        print('не уложились')
```

> Обрати внимание на две вещи. Во-первых, исключение — **встроенный `TimeoutError`**, а не `asyncio.TimeoutError` (с 3.11 это одно и то же). Во-вторых, `except` должен стоять **вне** блока `async with`: внутри менеджер ещё не превратил отмену в `TimeoutError`.

Старший вариант, оборачивающий одну корутину, — **`asyncio.wait_for(aw, timeout)`**. По истечении срока он **отменяет** задачу и бросает `TimeoutError`:

```python
try:
    result = await asyncio.wait_for(fetch(url), timeout=1.0)
except TimeoutError:
    result = None
```

### Отмена

Таймаут внутри устроен на **отмене**: задаче бросают `CancelledError` в точке ближайшего `await`.

```python
task = asyncio.create_task(work())
task.cancel()                    # попросить отмениться
```

Корутина может перехватить `CancelledError`, чтобы прибрать за собой (закрыть файл, откатить транзакцию), но обязана **пробросить его дальше**:

```python
async def work():
    try:
        await asyncio.sleep(3600)
    except asyncio.CancelledError:
        await cleanup()          # прибрались…
        raise                    # …и обязательно пробросили
```

> **Главная ловушка отмены: `CancelledError` наследуется от `BaseException`, а не от `Exception`.** Значит, привычное `except Exception:` его **не поймает** — и это сделано намеренно. Но обратное тоже верно: широкий `except BaseException:` или проглоченный `CancelledError` ломают отмену, таймауты и `TaskGroup`, потому что все они стоят на ней.

Если задачу нужно уберечь от отмены снаружи — **`asyncio.shield(aw)`**: ждущий получит `CancelledError`, а сама задача продолжит работать.

## Ограничение конкурентности: Semaphore

`asyncio.gather` на десяти запросах — хорошо. На десяти тысячах — способ уронить чужой сервис и получить бан по лимиту: все корутины стартуют разом, никакого «по чуть-чуть» loop не делает.

**`asyncio.Semaphore(n)`** — счётчик, пускающий внутрь не больше `n` задач одновременно. Остальные ждут на входе.

```python
sem = asyncio.Semaphore(10)          # не больше 10 запросов одновременно

async def fetch_one(session, url):
    async with sem:                   # занять слот; на выходе освободить
        async with session.get(url) as resp:
            return await resp.json()

async def main():
    async with aiohttp.ClientSession() as session:
        coros = [fetch_one(session, u) for u in urls]   # хоть 10 000 корутин
        return await asyncio.gather(*coros)             # но в полёте — 10
```

Корутин по-прежнему создаётся сколько угодно, но **одновременно работающих** ровно десять. Это же лекарство от «слишком много соединений к базе».

**`asyncio.BoundedSemaphore`** — тот же семафор, но бросает `ValueError`, если освобождений оказалось больше, чем захватов; так ловятся ошибки в ручном `acquire`/`release`.

Остальные примитивы синхронизации: `Lock` (один за раз, см. ловушку гонки ниже), `Event` (сигнал «можно продолжать» многим сразу), `Condition` (замок плюс ожидание условия), `Barrier` (ждать, пока соберётся N задач; с 3.11).

## asyncio.Queue — производитель и потребитель

Когда задачи не известны заранее, а появляются по ходу, конкурентность строят не на `gather`, а на **очереди**: одни корутины кладут работу, другие разбирают.

```python
async def worker(queue):
    while True:
        url = await queue.get()          # ждём задачу
        try:
            await handle(url)
        finally:
            queue.task_done()            # отметить, что эта задача обработана

async def main():
    queue = asyncio.Queue(maxsize=100)   # ограничение = защита от переполнения памяти
    workers = [asyncio.create_task(worker(queue)) for _ in range(5)]

    for url in urls:
        await queue.put(url)             # при заполненной очереди ЖДЁТ — это и есть backpressure

    await queue.join()                   # дождаться, пока разберут всё
    for w in workers:
        w.cancel()                       # воркеры бесконечны — снять вручную
```

Пять воркеров разбирают общую очередь — та же модель, что у [воркеров Celery](../../../Библиотеки/Сторонние/Celery.md#модели-конкурентности-пулы), только в одном процессе. `maxsize` даёт **backpressure**: производитель притормаживает, когда потребители не успевают, вместо того чтобы копить задачи в памяти.

## Ловушка: гонка данных через `await`

Частое заблуждение: раз asyncio — один поток, гонок данных не бывает и замки не нужны. Это верно **только между** точками `await`. Переключение происходит именно на `await` — и если общий изменяемый стейт читается до `await`, а пишется после, между ними успевает влезть другая корутина.

Классический пример — счётчик, который должен дойти до 100, но не доходит:

```python
import asyncio

counter = 0

async def increment():
    global counter
    value = counter              # 1. прочитали текущее значение
    await asyncio.sleep(0)       # 2. ← отдали управление loop'у, ещё НЕ записав
    counter = value + 1          # 3. пишем value+1 (а counter за это время изменился!)

async def main():
    await asyncio.gather(*[increment() for _ in range(100)])
    print(counter)               # → 1, а не 100

asyncio.run(main())
```

Почему `1`, а не `100`:

```
counter = 0
корутина 1: value = 0 ──await──▶ ... counter = 0+1 = 1
корутина 2: value = 0 ──await──▶ ... counter = 0+1 = 1
   ...все 100 прочитали 0, пока никто не записал...
итог: counter = 1
```

На `await asyncio.sleep(0)` каждая корутина уступает очередь — loop успевает прогнать **первые половины всех** остальных (все читают `value = 0`), и только потом доходит до записи. Все пишут `0 + 1`.

> **Убери `await` из середины** — и `counter` дойдёт до 100: без точки приостановки чтение-запись атомарно, переключаться loop'у негде. Гонку создаёт именно разрыв критической секции через `await`, а не сам по себе конкурентный запуск.

### Как чинить

**Способ 1 — не разрывать критическую секцию `await`'ом.** Читай и меняй стейт единым куском, а ожидание держи снаружи секции.

**Способ 2 — `asyncio.Lock`**, если `await` внутри секции неизбежен (например, запрос к БД под изменение общего стейта):

```python
lock = asyncio.Lock()

async def increment():
    global counter
    async with lock:             # в секцию пускается одна корутина за раз
        value = counter
        await asyncio.sleep(0)   # даже с await внутри — остальные ждут на входе
        counter = value + 1
#   теперь counter == 100
```

**`asyncio.Lock`** — асинхронный аналог [`threading.Lock`](../../../Библиотеки/Модули/Параллелизм/threading.md#ловушка-гонки-и-lock): `async with lock` пускает в блок одну корутину, остальные ждут на входе. Разница: поток на замке *блокируется*, а корутина — *уступает* loop'у, поэтому остальные задачи продолжают крутиться.

## aiohttp — асинхронный HTTP

**aiohttp** — HTTP поверх asyncio: асинхронный **клиент** (вместо [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md)) и асинхронный **сервер** (вместо Flask).

```bash
pip install aiohttp[speedups]   # + ускорители (aiodns, Brotli) — желательно
```

### Клиент

Запросы идут через **`ClientSession`** — переиспользуемую сессию (пул соединений). Её открывают через `async with`, чтобы гарантированно закрыть:

```python
import asyncio
import aiohttp

async def get_people(session, people_id):
    async with session.get(f'https://api.example.com/{people_id}') as response:
        return await response.json()          # await — тело ответа тоже приходит асинхронно

async def main():
    async with aiohttp.ClientSession() as session:      # одна сессия на все запросы
        coros = [get_people(session, i) for i in range(1, 11)]
        results = await asyncio.gather(*coros)          # 10 запросов одновременно
        for item in results:
            print(item)

asyncio.run(main())
```

- **`ClientSession`** создают **одну** на много запросов, а не по сессии на запрос — иначе теряется весь смысл пула соединений.
- `session.get/post(...)` возвращает ответ через `async with`; `await response.json()` / `.text()` — читает тело.

#### Параметры запроса и чтение ответа

```python
async with session.get(url, params={'page': 2}, headers={'X-Token': tok}) as resp:
    resp.status                 # код ответа (не корутина)
    resp.raise_for_status()     # бросить исключение на 4xx/5xx
    data = await resp.json()    # тело: .json() / .text() / .read() (bytes)

await session.post(url, json={'name': 'user'})    # тело как JSON
await session.post(url, data={'name': 'user'})    # тело как форма
```

#### Таймауты — задавать обязательно

```python
timeout = aiohttp.ClientTimeout(
    total=10,          # на всю операцию целиком
    connect=3,         # на установку соединения
    sock_read=5,       # на промежуток между порциями данных
)

async with aiohttp.ClientSession(timeout=timeout) as session:   # на всю сессию
    async with session.get(url, timeout=aiohttp.ClientTimeout(total=2)) as resp:
        ...                                                      # или на один запрос
```

> **Таймаут по умолчанию есть, но он огромный: `total=300`, то есть пять минут.** Остальные составляющие (`connect`, `sock_read`) по умолчанию не ограничены вовсе. Для веб-сервиса, который сам должен ответить за секунды, это всё равно что без таймаута: клиент отвалится задолго до того, как сработает защита. Свои значения ставят всегда — и лучше на уровне сессии, чтобы не забыть в отдельном запросе.

Что делать с упавшим по таймауту запросом — Устойчивость — таймауты, ретраи, circuit breaker.

#### Большие ответы — читать потоком

`await resp.read()` затягивает **весь** ответ в память. Для файлов и больших выгрузок читают порциями:

```python
async with session.get(url) as resp:
    with open('big.zip', 'wb') as f:
        async for chunk in resp.content.iter_chunked(64 * 1024):   # по 64 КБ
            f.write(chunk)
```

`resp.content` — асинхронный поток; память под весь файл не выделяется. Запись на диск здесь синхронная и блокирует loop — почему это важно и как чинить, см. [Асинхронная работа с файлами](../../../Библиотеки/Модули/Параллелизм/Асинхронная%20работа%20с%20файлами.md).

### Сервер

Маршруты описывают через **`RouteTableDef`** и декораторы, ответы возвращают объектами `web.Response` / `web.json_response`:

```python
from aiohttp import web

routes = web.RouteTableDef()

@routes.get('/')
async def hello(request):
    return web.Response(text="Hello, world")

@routes.get('/{name}')                        # часть пути → переменная
async def hello_name(request):
    name = request.match_info['name']         # достать её из запроса
    return web.Response(text=f"Hello, {name}")

app = web.Application()
app.add_routes(routes)
web.run_app(app)                              # поднять сервер на 0.0.0.0:8080
```

**Class-based views** — когда на один путь несколько методов (GET/POST/…), их собирают в класс-наследник `web.View`:

```python
@routes.view('/{name}')
class NameView(web.View):
    async def get(self):
        name = self.request.match_info['name']
        return web.Response(text=f"Hello, {name}")
```

**Middleware** — обёртка вокруг каждого обработчика (логирование, обработка ошибок, сессии). Через неё пропускается любой запрос до и после хендлера:

```python
@web.middleware
async def error_middleware(request, handler):
    try:
        return await handler(request)                    # вызвать сам обработчик
    except web.HTTPException as ex:
        return web.json_response({'error': ex.reason}, status=ex.status)

app = web.Application(middlewares=[error_middleware])
```

## Сквозной пример: async REST API на aiohttp + SQLAlchemy

CRUD-сервис пользователей: асинхронный сервер aiohttp + асинхронная [SQLAlchemy](../../../Библиотеки/Сторонние/ORM/SQLAlchemy.md) + PostgreSQL в [Docker](../../../DevOps/Docker.md). Собираем по файлам.

**Шаг 1. `docker-compose.yml`** — поднимаем базу одной командой (`docker compose up`):

```yaml
services:
  db:
    image: postgres:16
    ports: ["5431:5432"]         # наружу 5431, чтобы не конфликтовать с локальным postgres
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
```

**Шаг 2. `server.py`** — сервер. Комментарии-цифры показывают порядок запуска:

```python
import datetime
import json
from aiohttp import web
from bcrypt import hashpw, gensalt
from sqlalchemy import func
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.exc import IntegrityError

# 1. Подключение к базе (asyncpg — async-драйвер PostgreSQL)
PG_DSN = 'postgresql+asyncpg://app:secret@127.0.0.1:5431/app'
engine = create_async_engine(PG_DSN)
Session = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

# 2. Модель таблицы (стиль SQLAlchemy 2.0: Mapped + mapped_column)
class User(Base):
    __tablename__ = 'app_users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(unique=True, index=True)
    password: Mapped[str]                                    # Mapped[str] → NOT NULL
    creation_time: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

app = web.Application()

# 3. Жизненный цикл: создать таблицы на старте, закрыть движок на выходе
async def orm_context(app):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield                                    # до yield — startup, после — cleanup
    await engine.dispose()

# 4. Middleware: на каждый запрос открывает сессию БД и кладёт в request
@web.middleware
async def session_middleware(request, handler):
    async with Session() as session:
        request['session'] = session
        return await handler(request)

app.cleanup_ctx.append(orm_context)
app.middlewares.append(session_middleware)

# 5. Вспомогательные функции
def hash_password(password: str) -> str:
    return hashpw(password.encode(), gensalt()).decode()

async def get_user(user_id: int, session: AsyncSession) -> User:
    user = await session.get(User, user_id)
    if user is None:
        raise web.HTTPNotFound(
            text=json.dumps({'status': 'error', 'message': 'user not found'}),
            content_type='application/json',
        )
    return user

# 6. View с методами под каждый HTTP-глагол
class UserView(web.View):
    async def get(self):
        session = self.request['session']
        user_id = int(self.request.match_info['user_id'])
        user = await get_user(user_id, session)
        return web.json_response({
            'id': user.id,
            'name': user.name,
            'creation_time': user.creation_time.isoformat(),
        })

    async def post(self):
        session = self.request['session']
        json_data = await self.request.json()
        json_data['password'] = hash_password(json_data['password'])
        user = User(**json_data)
        session.add(user)
        try:
            await session.commit()
        except IntegrityError:
            raise web.HTTPConflict(
                text=json.dumps({'status': 'error', 'message': 'user already exists'}),
                content_type='application/json',
            )
        return web.json_response({'id': user.id})

    async def delete(self):
        session = self.request['session']
        user_id = int(self.request.match_info['user_id'])
        user = await get_user(user_id, session)
        await session.delete(user)
        await session.commit()
        return web.json_response({'status': 'success'})

# 7. Маршруты → View
app.add_routes([
    web.get(r'/users/{user_id:\d+}', UserView),
    web.post('/users/', UserView),
    web.delete(r'/users/{user_id:\d+}', UserView),
])

if __name__ == '__main__':
    web.run_app(app)
```

**Шаг 3. `client.py`** — асинхронный клиент, создаёт пользователя:

```python
import asyncio
from aiohttp import ClientSession

async def main():
    async with ClientSession() as session:
        response = await session.post(
            'http://127.0.0.1:8080/users/',
            json={'name': 'user_1', 'password': '1234'},
        )
        print(response.status)
        print(await response.json())

asyncio.run(main())
```

**Как это читать.** `docker compose up` поднимает базу; `python server.py` создаёт таблицы (`orm_context`) и слушает `:8080`. На каждый запрос `session_middleware` открывает сессию БД, `UserView` выбирает метод по HTTP-глаголу, `get_user` кидает 404 через исключение. Пароли хешируются `bcrypt` — в базе не хранится открытый текст. Клиент делает те же запросы асинхронно.

## Связи

- [Блокирующий код в event loop](../../../Библиотеки/Модули/Параллелизм/Блокирующий%20код%20в%20event%20loop.md) — что считать «слишком долго», как выносить в поток или процесс и когда рефакторить не надо;
- [Декораторы для асинхронных функций](../../../Ядро%20языка/Декораторы%20для%20асинхронных%20функций.md) — как обернуть корутину логом, таймером или retry и почему `@lru_cache` на ней ломается;
- [Асинхронная работа с файлами](../../../Библиотеки/Модули/Параллелизм/Асинхронная%20работа%20с%20файлами.md) — почему `aiofiles` под капотом создаёт поток, а не ждёт асинхронно;
- [CPU-bound против IO-bound](../../../Библиотеки/Модули/Параллелизм/CPU-bound%20против%20IO-bound.md) — почему async годится для I/O, но не для счёта; как различать нагрузку;
- Асинхронность и многопоточность — базовая теория: I/O- vs CPU-bound, GIL, конкурентность vs параллелизм;
- [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md) — синхронный аналог клиента aiohttp; async-версия не блокирует loop;
- [multiprocessing](../../../Библиотеки/Модули/Параллелизм/multiprocessing.md) — параллелизм для CPU-bound; asyncio — конкурентность для I/O-bound;
- [SQLAlchemy](../../../Библиотеки/Сторонние/ORM/SQLAlchemy.md) — ORM; в async-режиме (`create_async_engine`, `AsyncSession`) работает под aiohttp;
- [aiogram](../../../Фреймворки/aiogram.md) — фреймворк Telegram-ботов поверх asyncio/aiohttp: обработчики — это корутины;
- [Docker](../../../DevOps/Docker.md) — база данных для примера поднимается через `docker-compose.yml`;
- [WSGI/ASGI](../../../DevOps/Развертывание%20проекта.md#шаг-4-wsgiasgi--мост-приложение--сервер) — async-приложения деплоят через ASGI (uvicorn), а не WSGI.

## Источники

- [asyncio — Asynchronous I/O (оглавление)](https://docs.python.org/3/library/asyncio.html)
- [asyncio — Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html) (таймауты, отмена, `shield`)
- [asyncio — Synchronization Primitives](https://docs.python.org/3/library/asyncio-sync.html) (`Semaphore`, `Lock`, `Event`)
- [asyncio — Queues](https://docs.python.org/3/library/asyncio-queue.html)
- [asyncio — Event Loop](https://docs.python.org/3/library/asyncio-eventloop.html)
- [aiohttp — Client Quickstart](https://docs.aiohttp.org/en/stable/client_quickstart.html)
- [aiohttp — Server (web) & Advanced](https://docs.aiohttp.org/en/stable/web_advanced.html)
- [PostgreSQL — Connection Settings (`max_connections`, «The default is typically 100 connections»)](https://www.postgresql.org/docs/current/runtime-config-connection.html)
- [SQLAlchemy — Asyncio Integration (сессия на задачу)](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
