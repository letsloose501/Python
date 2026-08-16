[← Оглавление](../../../README.md)

# SQLAlchemy

> _SQL, когда нужен контроль; объекты, когда нужна скорость — в одной библиотеке_

---

**SQLAlchemy** — главный SQL-инструментарий и [ORM](../../../Фреймворки/DJANGO/ORM%20Django.md) для [Python](../../../Python.md), не привязанный к какому-либо фреймворку (в отличие от встроенного ORM Django). У него два уровня: **Core** — конструктор SQL-запросов на Python, и **ORM** — отображение классов на таблицы. Работает и синхронно, и асинхронно (async-версия используется с [aiohttp](../../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md)).

## Два уровня: Core и ORM

- **Core** — низкий уровень: таблицы и SQL-выражения как объекты Python. Больше контроля, ближе к SQL.
- **ORM** — высокий уровень: классы-модели ↔ таблицы, строки ↔ объекты. Пишешь объектами, SQLAlchemy генерирует SQL.

Чаще используют ORM; Core нужен, когда важен точный контроль над запросом. Дальше — про ORM (стиль SQLAlchemy 2.0).

## Engine — подключение

**Engine** — точка входа к базе: пул соединений и диалект под конкретную СУБД. Создаётся из строки подключения (DSN):

```python
from sqlalchemy import create_engine

engine = create_engine('postgresql+psycopg2://user:secret@localhost:5432/mydb')
#   postgresql — СУБД, psycopg2 — драйвер (psycopg2), дальше — user:pass@host:port/db
```

## Модели — классы как таблицы

Модели наследуют общий `DeclarativeBase`; поля объявляют через `Mapped` и `mapped_column`:

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(unique=True)
    age: Mapped[int]

Base.metadata.create_all(engine)      # создать таблицы в БД по моделям
```

## Session — единица работы

**Session** — «зона хранения» объектов и разговор с базой: через неё добавляют, читают, меняют и удаляют записи, а `commit` фиксирует изменения в одной транзакции.

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    user = User(name='Alex', age=25)
    session.add(user)          # пометить для вставки
    session.commit()           # зафиксировать (INSERT уходит в БД)
```

## Запросы — select()

В стиле 2.0 запросы строят функцией `select()`, а выполняют через сессию. `scalars()` возвращает сами объекты:

```python
from sqlalchemy import select

with Session(engine) as session:
    user = session.get(User, 1)                       # по первичному ключу

    stmt = select(User).where(User.age >= 18)         # SELECT ... WHERE age >= 18
    adults = session.scalars(stmt).all()              # список объектов User

    for u in session.scalars(select(User).order_by(User.name)):
        print(u.name)
```

## Обязательные и необязательные поля

Тип в аннотации решает, будет ли столбец `NOT NULL`. Обычный `Mapped[str]` даёт **обязательный** столбец, `Mapped[Optional[str]]` — допускающий `NULL`:

```python
from typing import Optional

class User(Base):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]                          # NOT NULL
    phone: Mapped[Optional[str]]               # NULL разрешён
    age: Mapped[int | None]                    # то же самое, современный синтаксис
```

Это ровно та разница между `NOT NULL` и его отсутствием, что и в самой базе, — просто выраженная через аннотацию типа.

## Изменение и удаление

Записи меняют **не отдельным запросом, а прямо на объекте**: сессия отслеживает загруженные объекты и на `commit` сама сформирует `UPDATE` для изменившихся полей. Это и называется unit of work.

```python
with Session(engine) as session:
    user = session.get(User, 1)
    user.age = 26                  # просто меняем атрибут
    session.commit()               # UPDATE users SET age=26 WHERE id=1

    session.delete(user)           # пометить на удаление
    session.commit()               # DELETE FROM users WHERE id=1
```

Массовое изменение без загрузки объектов — через `update()` / `delete()`:

```python
from sqlalchemy import update

session.execute(update(User).where(User.age < 18).values(active=False))
session.commit()
```

> **`merge()` — не способ «обновить запись».** Его задача другая: взять **отсоединённый** объект (например, собранный из внешних данных или переживший закрытие сессии) и слить его состояние с тем, что есть в базе, вернув привязанный к сессии экземпляр. Для обычного «поменять поле у загруженной записи» он не нужен и делает лишний `SELECT`.

## Откат при ошибке

Если внутри транзакции что-то упало, сессия остаётся в нерабочем состоянии — до явного отката ни один следующий запрос не выполнится:

```python
with Session(engine) as session:
    try:
        session.add(User(name='Alex', age=25))
        session.commit()
    except IntegrityError:              # нарушено UNIQUE, NOT NULL, CHECK…
        session.rollback()              # обязательно: вернуть сессию в рабочее состояние
        raise
```

`IntegrityError` — это как раз сработавшее ограничение в базе: SQLAlchemy не проверяет их сама, а транслирует ошибку СУБД.

## Связи между таблицами

Связь задаётся `ForeignKey` + `relationship` — тогда доступ к связанным объектам идёт как к атрибуту:

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class Post(Base):
    __tablename__ = 'posts'
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    author_id: Mapped[int] = mapped_column(ForeignKey('users.id'))
    author: Mapped['User'] = relationship(back_populates='posts')

class User(Base):
    # ... поля выше ...
    posts: Mapped[list['Post']] = relationship(back_populates='author')
```

```python
post.author.name        # объект пользователя через связь
user.posts              # список постов пользователя
```

## SQLAlchemy против ORM Django

| | SQLAlchemy | [ORM Django](../../../Фреймворки/DJANGO/ORM%20Django.md) |
|---|---|---|
| **Привязка** | независимая библиотека | часть Django |
| **Уровни** | Core (SQL) + ORM | только ORM |
| **Запросы** | `select(...).where(...)` | `Model.objects.filter(...)` |
| **Сессия** | явная (`Session`, `commit`) | неявная (автосохранение) |
| **С чем** | Flask, FastAPI, любой Python | Django |

> Django ORM удобнее «из коробки» и проще, SQLAlchemy — гибче и работает где угодно. Выбор часто определяет фреймворк: Django → его ORM; [Flask](../../../Фреймворки/FLASK.md)/[FastAPI](../../../Фреймворки/FastAPI.md) → SQLAlchemy.

## Асинхронный режим и параллельные запросы

Для асинхронных приложений есть async-вариант: `create_async_engine`, `AsyncSession`, `await session.commit()`. Полный пример — в [Asyncio Event Loop и aiohttp](../../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#сквозной-пример-async-rest-api-на-aiohttp--sqlalchemy).

### Ловушка: одна сессия на несколько корутин

Как только независимые запросы к базе решают [распараллелить через `gather`](../../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#последовательный-await--самая-частая-ошибка-в-продакшене), всплывает ограничение, о котором документация говорит прямо: **один экземпляр `AsyncSession` небезопасен для нескольких конкурентных задач**.

Причина не в капризе библиотеки. Сессия — изменяемый объект с состоянием, который ведёт **одну транзакцию**: у неё свой набор отслеживаемых объектов, свой буфер незаписанных изменений, своё соединение. Две корутины, дёргающие её одновременно, попадают в середину чужой операции.

```python
# НЕПРАВИЛЬНО — одна сессия шарится между тремя корутинами
async def get_dashboard(session: AsyncSession, user_id: int):
    return await asyncio.gather(
        fetch_user(session, user_id),      # все три лезут в одну транзакцию
        fetch_orders(session, user_id),    # → состояние сессии рвётся,
        fetch_feed(session, user_id),      #   падает с ошибкой состояния
    )

# ПРАВИЛЬНО — каждая задача открывает свою сессию из фабрики
async_session = async_sessionmaker(engine, expire_on_commit=False)

async def fetch_user(factory, user_id: int):
    async with factory() as session:                # своя сессия и своё соединение
        return await session.get(User, user_id)

async def get_dashboard(user_id: int):
    return await asyncio.gather(
        fetch_user(async_session, user_id),
        fetch_orders(async_session, user_id),
        fetch_feed(async_session, user_id),
    )
```

Передавать между корутинами нужно **фабрику** (`async_sessionmaker`), а не готовую сессию: каждая задача берёт себе свежую.

> ⚠️ Цена этого решения — соединения. Три параллельные корутины занимают **три** соединения вместо одного, и при высоком RPS сервис быстрее упирается в `max_connections` (по умолчанию около 100). Распараллеливание запросов к базе всегда размен: меньше латентность одного эндпоинта — больше давление на пул.

**Что это ломает в архитектуре.** Обвязку над репозиториями (`Unit of Work`, у части команд — «database manager») обычно пишут из расчёта «одна сессия на запрос»: она открывает сессию, раздаёт её репозиториям и управляет транзакцией. Под параллельные запросы её приходится дорабатывать — учить выдавать отдельную сессию на каждую ветку. Это типовая точка боли: эндпоинт с четырьмя обращениями к базе часто оказывается самым медленным в сервисе (500 мс против 100–200 у остальных), а починить его «в лоб» через `gather` мешает именно общая сессия. → Чистая архитектура и DDD

## Сквозной пример: пользователи от модели до запроса

```python
from sqlalchemy import create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

# 1. подключение
engine = create_engine('sqlite:///app.db')      # SQLite для примера

# 2. модель
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    age: Mapped[int]

# 3. создать таблицы
Base.metadata.create_all(engine)

# 4. записать и прочитать
with Session(engine) as session:
    session.add_all([User(name='Alex', age=25), User(name='Ivan', age=17)])
    session.commit()

    adults = session.scalars(select(User).where(User.age >= 18)).all()
    print([u.name for u in adults])          # ['Alex']
```

**Как это читать.** Engine знает, как подключиться; модель `User` описывает таблицу; `create_all` создаёт её; Session добавляет записи и фиксирует их `commit`; `select(...).where(...)` строит запрос, `scalars(...).all()` возвращает объекты. Сменить SQLite на PostgreSQL — поменять только строку подключения.

## Связи

- [ORM Django](../../../Фреймворки/DJANGO/ORM%20Django.md) — встроенный ORM Django; сравнение подходов выше;
- [psycopg2](../../../Библиотеки/Сторонние/Работа%20с%20БД/psycopg2.md) — драйвер, через который SQLAlchemy общается с PostgreSQL;
- [Asyncio Event Loop и aiohttp](../../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — async-SQLAlchemy под aiohttp/FastAPI; там же — как правильно распараллеливать запросы;
- [Depends — зависимости в FastAPI](../../../Фреймворки/Depends%20—%20зависимости%20в%20FastAPI.md) — как сессию отдают в ручку: одна транзакция на запрос через зависимость с `yield`;
- PostgreSQL — типичная боевая база под SQLAlchemy; её `max_connections` ограничивает параллелизм корутин;
- [pandas](../../../Библиотеки/Сторонние/pandas.md) — данные из БД грузят в DataFrame (`pd.read_sql`) для анализа.

## Источники

- [SQLAlchemy 2.0 documentation](https://docs.sqlalchemy.org/en/20/)
- [ORM Quick Start](https://docs.sqlalchemy.org/en/20/orm/quickstart.html)
- [Session Basics](https://docs.sqlalchemy.org/en/20/orm/session_basics.html)
- [SQLAlchemy Unified Tutorial](https://docs.sqlalchemy.org/en/20/tutorial/)
- [Asyncio Integration — «A single instance of AsyncSession is not safe for use in multiple, concurrent tasks»](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
