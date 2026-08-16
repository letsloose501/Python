[← Оглавление](../README.md)

# Depends — зависимости в FastAPI

> _Общее для многих ручек пишется один раз и подставляется само_

---

**`Depends`** — механизм внедрения зависимостей ([DI](../ООП/Принципы/Принципы%20разработки%20—%20DRY,%20KISS,%20DI.md#di-против-dip--техника-против-принципа)) в [FastAPI](../Фреймворки/FastAPI.md): ты объявляешь функцию, которая готовит нужный ресурс (сессию БД, текущего пользователя, параметры пагинации), а фреймворк сам вызывает её перед обработчиком и подставляет результат в аргумент. Это самая частая тема на собеседовании по FastAPI — и самый используемый инструмент фреймворка после Pydantic-моделей.

## Какую боль решает

Без зависимостей одинаковый код расползается по каждой ручке:

```python
# НЕПРАВИЛЬНО — три ручки, три копии одного и того же:
@app.get('/products')
async def list_products(page: int = 1, limit: int = 20):
    async with Session() as session:            # 1. открыли сессию
        offset = (page - 1) * limit             # 2. посчитали смещение
        ...                                     # 3. и не забыть закрыть

@app.get('/orders')
async def list_orders(page: int = 1, limit: int = 20):
    async with Session() as session:            # то же самое
        offset = (page - 1) * limit
        ...
```

Меняется формула пагинации или настройка сессии — правь во всех местах, забыл одно → баг. Это прямое нарушение [DRY](../ООП/Принципы/Принципы%20разработки%20—%20DRY,%20KISS,%20DI.md#dry--dont-repeat-yourself). Зависимость выносит общий кусок в одно место, а ручка получает готовый результат.

## Как это работает

FastAPI смотрит на **подпись** функции-зависимости так же, как на подпись обработчика: `page: int = 1` станет query-параметром, `Product` — телом запроса, `token: str = Header()` — заголовком. Всё это попадёт и в валидацию, и в `/docs`.

```
Запрос ──▶ FastAPI читает подпись ручки
              │
              ├─ видит Depends(get_session) ──▶ вызывает get_session()
              │                                   (её параметры тоже разбирает из запроса)
              └─ подставляет результат в аргумент ──▶ вызывает ручку
```

```python
from fastapi import Depends

async def pagination(page: int = 1, limit: int = 20) -> dict:
    return {'limit': limit, 'offset': (page - 1) * limit}

@app.get('/products')
async def list_products(p: dict = Depends(pagination)):   # p = {'limit': 20, 'offset': 0}
    ...
```

`GET /products?page=3&limit=50` — FastAPI разберёт query-параметры, отдаст их в `pagination`, а её результат положит в `p`. Оба параметра при этом видны в Swagger — как будто объявлены прямо в ручке.

> Здесь пагинация взята как самый узнаваемый пример общей зависимости, а не как образец для боевого списка: `offset = (page - 1) * limit` линейно дорожает с глубиной, а `limit` без верхней границы позволяет запросить миллион строк одной ссылкой. Что с этим делают — Пагинация — offset, keyset и гибрид и Курсор в API — контракт пагинации.

## Четыре формы зависимости

### 1. Функция — базовый случай

Как в примере выше. Годится, пока результат — простое значение или словарь.

### 2. `Annotated` — современный способ записи

Запись `p: dict = Depends(pagination)` смешивает **тип** и **значение по умолчанию**: формально у аргумента появляется дефолт, которого нет. С Python 3.9+ и FastAPI 0.95+ правильнее класть `Depends` внутрь `Annotated`, а саму связку сохранять в псевдоним типа:

```python
from typing import Annotated

PaginationDep = Annotated[dict, Depends(pagination)]      # объявили один раз

@app.get('/products')
async def list_products(p: PaginationDep): ...            # и переиспользуем везде

@app.get('/orders')
async def list_orders(p: PaginationDep): ...
```

Плюсы: тип аргумента честный (`dict`, а не «`dict` со значением по умолчанию»), функцию можно вызвать напрямую в тесте, и один псевдоним вместо повтора `Depends(...)` в каждой ручке. Официальная документация с 0.95 пишет примеры именно так.

### 3. Класс — когда параметров много

Если общих query-параметров не два, а пять-десять, удобнее описать их классом: FastAPI читает подпись **`__init__`** ровно так же, как подпись функции, а в ручку приходит готовый экземпляр с полями и методами.

```python
from fastapi import Query

class Pagination:
    def __init__(self,
                 page: int = Query(1, ge=1),                 # валидация как обычно
                 limit: int = Query(20, ge=1, le=100)):
        self.page = page
        self.limit = limit
        self.offset = (page - 1) * limit                     # вычисляемое поле

PaginationDep = Annotated[Pagination, Depends(Pagination)]

@app.get('/products')
async def list_products(p: PaginationDep):
    return await repo.list(limit=p.limit, offset=p.offset)   # автодополнение в IDE работает
```

> Класс — сам себе зависимость: `Depends(Pagination)` без аргументов равносильно `Depends()` с тем же типом в аннотации. Выигрыш против словаря — типизация полей и место, куда положить вычисления (`offset`) и методы.

### 4. Вызываемый экземпляр — параметризованная зависимость

Зависимость нельзя «вызвать с аргументом» — `Depends(check_role('admin'))` работать не будет, потому что FastAPI ждёт саму функцию, а не её результат. Приём: класс с `__init__` для настройки и `__call__` как самой зависимостью — FastAPI читает подпись `__call__`.

```python
class RoleRequired:
    def __init__(self, *roles: str):
        self.roles = roles                                   # настройка на этапе создания

    def __call__(self, user: UserDep) -> User:               # это и есть зависимость
        if user.role not in self.roles:
            raise HTTPException(403, 'Недостаточно прав')
        return user

admin_only = RoleRequired('admin')                           # один экземпляр — одна настройка
manager_or_admin = RoleRequired('admin', 'manager')

@app.delete('/users/{uid}')
async def delete_user(uid: int, user: Annotated[User, Depends(admin_only)]): ...
```

## Зависимость с `yield` — ресурс, который надо закрыть

Главный сценарий из практики: одна сессия БД на один HTTP-запрос. Код **до** `yield` выполняется перед ручкой, код **после** — когда ручка отработала. По сути это фикстура на уровне запроса, а не теста.

```python
async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with Session() as session:       # открыли соединение из пула
        yield session                      # отдали в ручку
        # выход из with — сессия закрыта, соединение вернулось в пул

SessionDep = Annotated[AsyncSession, Depends(get_session)]

@app.get('/products/{pid}')
async def get_product(pid: int, session: SessionDep):
    return await session.get(Product, pid)
```

Чем это лучше `async with` внутри каждой ручки: сессия гарантированно закроется даже при исключении, а в тестах её легко подменить (см. ниже). Так же выносят [Redis](../Библиотеки/Сторонние/Redis.md)-клиент, HTTP-клиент, файловые дескрипторы.

**Когда именно выполняется код после `yield`** — вопрос, на котором ломались проекты, и версия FastAPI тут принципиальна:

| Версии | Выход из зависимости | Последствие |
|---|---|---|
| до 0.106.0 | **после** отправки ответа | сессию можно было использовать в фоновой задаче |
| 0.106.0 – 0.117.x | **до** отправки ответа | сессия в `BackgroundTasks` уже закрыта; `StreamingResponse` не может читать из БД |
| **0.118.0 и новее** | снова **после** отправки | поведение вернули, попутно починив краевые случаи |

> Даже на свежей версии **не полагайся** на сессию из `Depends` внутри фоновой задачи: передавай в неё **id объекта**, а не сам объект, и открывай своё соединение внутри задачи. Так задача не зависит от версии фреймворка и от того, жив ли ещё запрос. Это же рекомендует официальная документация.

## Вложенные зависимости и кэш запроса

Зависимость может зависеть от другой — FastAPI разрешит всю цепочку сама:

```python
async def get_token(authorization: str = Header()) -> str: ...

async def get_current_user(token: Annotated[str, Depends(get_token)],
                           session: SessionDep) -> User: ...

UserDep = Annotated[User, Depends(get_current_user)]

@app.get('/me')
async def me(user: UserDep): ...
```

**В пределах одного запроса зависимость вызывается один раз.** Если `get_current_user` нужна и самой ручке, и проверке прав, и логгеру — функция отработает единожды, а результат раздастся всем из кэша. Это не оптимизация «на всякий случай», а гарантия: один запрос — одна сессия, один поход за пользователем.

Нужен свежий результат на каждое обращение — отключи кэш явно:

```python
Annotated[str, Depends(get_value, use_cache=False)]   # вызывать каждый раз заново
```

## Зависимость без результата — на ручке, роутере, приложении

Иногда зависимость нужна ради **побочного эффекта**: проверить токен, залогировать, кинуть `403`. Возвращаемое значение никому не нужно — тогда её вешают в `dependencies=[...]`, и аргумент в ручке не появляется:

```python
@app.get('/admin', dependencies=[Depends(verify_admin)])          # одна ручка
async def admin_panel(): ...

router = APIRouter(prefix='/admin', dependencies=[Depends(verify_admin)])   # весь роутер

app = FastAPI(dependencies=[Depends(verify_api_key)])             # всё приложение
```

Уровень выбирают по охвату правила: проверка API-ключа для всего сервиса — на приложении, права администратора — на роутере, редкая частность — на ручке. Результат такой зависимости отбрасывается, но исключение внутри неё останавливает запрос.

## Синхронные и асинхронные зависимости

Правило то же, что для обработчиков: `async def`-зависимость выполняется в event loop, обычная `def`-зависимость — **в пуле потоков**, чтобы блокирующий вызов не остановил цикл. Значит, синхронную зависимость с обычным драйвером БД писать можно — она не заблокирует сервер, но займёт поток из ограниченного пула.

> Худший вариант — блокирующий вызов внутри `async def`-зависимости: он останавливает весь event loop до самых первых строк обработки запроса. Механика и пороги — [Блокирующий код в event loop](../Библиотеки/Модули/Параллелизм/Блокирующий%20код%20в%20event%20loop.md).

## Подмена в тестах — `dependency_overrides`

Главная практическая причина вообще выносить всё в зависимости. `app.dependency_overrides` — словарь «исходная зависимость → подменная»; в тестах туда кладут сессию к тестовой базе, фейкового пользователя, заглушку внешнего API:

```python
async def override_session():
    async with TestSession() as session:
        yield session

app.dependency_overrides[get_session] = override_session   # ключ — сама функция
...
app.dependency_overrides.clear()                            # обязательно сбросить после теста
```

Ключ — **объект функции**, а не строка с именем, поэтому подменяется именно то, что импортировано в код. Подробнее с фикстурами и тестовой БД — [Тестирование FastAPI-приложений](../Фреймворки/Тестирование%20FastAPI-приложений.md).

> Обратная сторона: подменить можно только то, что **пришло снаружи**. Если сессия или HTTP-клиент создаются внутри функции, заменить их нечем, и тест обречён ходить в настоящую базу. Тестируемость — это и есть главный аргумент в пользу DI.

## Ловушки

- **Вызов вместо передачи.** `Depends(get_session())` вместо `Depends(get_session)` — в скобках уже результат, а FastAPI ждёт саму функцию. Параметризация делается вызываемым экземпляром (форма 4).
- **Тяжёлый ресурс на каждый запрос.** Зависимость выполняется **на каждый** запрос: создавать в ней пул соединений или HTTP-клиент — значит платить за это тысячи раз. Долгоживущие объекты поднимают один раз в `lifespan` ([FastAPI](../Фреймворки/FastAPI.md)), а зависимость лишь достаёт готовый.
- **`Depends` без `yield` там, где нужен `yield`.** `return session` вместо `yield session` — закрывать соединение будет некому, пул кончится под нагрузкой.
- **Зависимость, которая молча меняет ответ.** Зависимость может кинуть `HTTPException`, но не может «дописать» тело ответа — для сквозной логики над ответом нужен middleware ([FastAPI](../Фреймворки/FastAPI.md#cors-и-middleware)).

## Сравнение форм

| Форма | Когда брать | Что приходит в ручку |
|---|---|---|
| Функция | 1–2 параметра, простой результат | значение (обычно `dict`) |
| `Annotated`-псевдоним | всегда, как способ записи | то же, но без фальшивого дефолта |
| Класс | много общих параметров, нужны вычисления и методы | экземпляр с полями |
| Вызываемый экземпляр | зависимость надо настроить (`RoleRequired('admin')`) | результат `__call__` |
| `yield`-генератор | ресурс с открытием/закрытием | ресурс (сессия, клиент) |
| `dependencies=[...]` | нужен только побочный эффект | ничего |

## Сквозной пример: сессия, пользователь, пагинация

```python
# deps.py — все зависимости проекта в одном месте
from typing import Annotated
from fastapi import Depends, Header, HTTPException, Query
from sqlalchemy.ext.asyncio import AsyncSession

async def get_session():                                   # 1. ресурс с закрытием
    async with Session() as session:
        yield session

SessionDep = Annotated[AsyncSession, Depends(get_session)]

async def get_current_user(                                # 2. зависит от (1)
    session: SessionDep,
    authorization: str = Header(),
) -> User:
    user = await auth.resolve(session, authorization)
    if user is None:
        raise HTTPException(401, 'Нужна авторизация')
    return user

UserDep = Annotated[User, Depends(get_current_user)]

class Pagination:                                          # 3. класс-зависимость
    def __init__(self, page: int = Query(1, ge=1),
                 limit: int = Query(20, ge=1, le=100)):
        self.limit, self.offset = limit, (page - 1) * limit

PaginationDep = Annotated[Pagination, Depends(Pagination)]

# routers/orders.py
@router.get('/orders')
async def my_orders(user: UserDep, p: PaginationDep, session: SessionDep):
    #  session здесь — ТА ЖЕ, что получил get_current_user: кэш запроса
    return await repo.orders_of(session, user.id, p.limit, p.offset)
```

**Как это читать.** Ручка объявляет три аргумента и не содержит ни строчки инфраструктуры: авторизация, транзакция и разбор параметров разрешились до её вызова. `session` в `get_current_user` и в самой ручке — один объект, потому что в пределах запроса зависимость вызывается один раз. В тесте достаточно подменить `get_session` и `get_current_user` — и весь роутер проверяется без базы и без токенов.

## Связи

- [FastAPI](../Фреймворки/FastAPI.md) — механика фреймворка, куда `Depends` встроен;
- [REST API на FastAPI](../Фреймворки/REST%20API%20на%20FastAPI.md) — сборка сервиса, где зависимости раздают сессию и пользователя;
- [Тестирование FastAPI-приложений](../Фреймворки/Тестирование%20FastAPI-приложений.md) — `dependency_overrides`, тестовая БД, фикстуры;
- [Принципы разработки — DRY, KISS, DI](../ООП/Принципы/Принципы%20разработки%20—%20DRY,%20KISS,%20DI.md) — DI как техника и DIP как принцип: `Depends` — реализация первой;
- [SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md) — асинхронная сессия, которую чаще всего и отдают через `Depends`;
- [Блокирующий код в event loop](../Библиотеки/Модули/Параллелизм/Блокирующий%20код%20в%20event%20loop.md) — почему sync-зависимость уходит в пул потоков и чем это ограничено;
- [Разделение доступа в DRF](../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md) — та же задача (права, аутентификация) в Django-мире решается классами разрешений.

## Источники

- [FastAPI — Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [FastAPI — Classes as Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/classes-as-dependencies/)
- [FastAPI — Sub-dependencies (кэш и `use_cache`)](https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/)
- [FastAPI — Dependencies with yield](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/)
- [FastAPI — Advanced Dependencies (параметризация, изменение 0.106.0)](https://fastapi.tiangolo.com/advanced/advanced-dependencies/)
- [FastAPI — Testing Dependencies with Overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/)
- [FastAPI — Dependencies in path operation decorators](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-in-path-operation-decorators/)
