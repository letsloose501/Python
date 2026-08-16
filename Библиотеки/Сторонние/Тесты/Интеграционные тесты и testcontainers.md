[← Оглавление](../../../README.md)

# Интеграционные тесты и testcontainers

Юнит-тест проверяет функцию в изоляции, подменяя базу моком ([Юнит-тестирование Python](../../../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md)). Проблема в том, что **мок отвечает так, как ты придумал**, а настоящий PostgreSQL — так, как считает нужным. Мок не знает про уникальные индексы, каскадные удаления, типы колонок и уровни изоляции.

**Интеграционный тест** — проверка того, что твой код правильно работает с **реальной** внешней системой: базой, брокером, кэшем. Именно этот вопрос задают мидлу: «как ты тестируешь код, который ходит в БД и во внешний API?»

---

## Почему моки врут

```python
# Юнит-тест с моком: зелёный
def test_create_user(mocker):
    db = mocker.Mock()
    db.insert.return_value = 1
    assert create_user(db, email="a@b.c") == 1     # ✅

# В проде: падает
# IntegrityError: duplicate key value violates unique constraint "users_email_key"
```

Мок подтвердил ровно то, что ты в него заложил. Реальная база знает про уникальный индекс — а тест про него не знал.

Что ловится **только** интеграционным тестом:

- нарушения ограничений: `UNIQUE`, `FOREIGN KEY`, `NOT NULL`, `CHECK`;
- реальное поведение транзакций и откатов (Программирование/Базы данных/Уровни изоляции и аномалии);
- корректность самих SQL-запросов и миграций;
- расхождение типов Python ↔ БД (`Decimal` против `float`, часовые пояса);
- ленивая загрузка в ORM и проблема N+1.

---

## Четыре способа поднять зависимость

| Способ | Плюсы | Минусы |
|---|---|---|
| **SQLite в памяти** | Мгновенно, ноль настройки | 🔴 Другая СУБД: нет `JSONB`, других типов и блокировок. Зелёный тест ничего не гарантирует |
| **Общая dev-база** | Похожа на прод | 🔴 Тесты мешают друг другу, состояние «плывёт», нельзя гонять параллельно |
| **docker-compose вручную** | Настоящая СУБД | Надо не забыть поднять; порты конфликтуют; CI требует отдельной настройки |
| **testcontainers** | Настоящая СУБД, поднимается и гасится сама | Нужен Docker, старт занимает секунды |

> 🔴 **SQLite вместо PostgreSQL — самая распространённая ошибка.** Соблазн понятен: быстро и без Docker. Но ты тестируешь **другую базу**. Запрос с `JSONB`, оконной функцией или `ON CONFLICT` в SQLite либо не работает, либо работает иначе. Тест зелёный, прод падает.

---

## testcontainers

**Testcontainers** — библиотека, которая поднимает докер-контейнер прямо из кода теста и гасит его после. Настоящий PostgreSQL нужной версии, изолированный, на случайном порту.

```python
from testcontainers.postgres import PostgresContainer

with PostgresContainer("postgres:16") as pg:
    url = pg.get_connection_url()      # postgresql+psycopg2://...:РАНДОМНЫЙ_ПОРТ/test
    # внутри блока работает настоящая база; на выходе контейнер удалён
```

Случайный порт — важная деталь: тесты можно гонять параллельно и на своей машине, и в CI, не согласовывая порты.

---

## Сборка с pytest: два уровня фикстур

Ключ к быстрым интеграционным тестам — **поднимать контейнер один раз на прогон, а изоляцию давать транзакцией**.

```python
# conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")           # ← один контейнер на весь прогон
def engine():
    with PostgresContainer("postgres:16") as pg:
        engine = create_engine(pg.get_connection_url())
        Base.metadata.create_all(engine)   # или прогнать миграции Alembic
        yield engine

@pytest.fixture                            # ← своя транзакция на каждый тест
def session(engine):
    connection = engine.connect()
    transaction = connection.begin()
    session = sessionmaker(bind=connection)()
    yield session
    session.close()
    transaction.rollback()                 # откат: база снова чистая
    connection.close()
```

**Как это читает pytest.** `scope="session"` поднимает контейнер один раз — старт стоит несколько секунд, платить за него в каждом тесте нельзя. Фикстура `session` открывает транзакцию перед тестом и **откатывает** после: что бы тест ни записал, следующий получит чистую базу. Это быстрее пересоздания схемы на порядок.

```python
def test_unique_email(session):
    session.add(User(email="a@b.c")); session.flush()
    session.add(User(email="a@b.c"))
    with pytest.raises(IntegrityError):     # настоящее ограничение настоящей базы
        session.flush()
```

> **Прогоняй в тестах те же миграции, что и в проде** (Миграции и zero-downtime). `create_all()` строит схему из моделей, а прод строится миграциями — они могут разойтись, и тест этого не заметит.

---

## Не только база

```python
from testcontainers.redis import RedisContainer
from testcontainers.rabbitmq import RabbitMqContainer
from testcontainers.core.container import DockerContainer   # любой образ
```

Так тестируют [кэш](../../../Библиотеки/Сторонние/Redis.md), [очередь](../../../Библиотеки/Сторонние/RabbitMQ.md) и даже собственные сервисы-заглушки.

---

## Внешние HTTP-API — не поднимать, а подменять

Чужой API в тестах поднимать нельзя: нет доступа, есть лимиты, ответы меняются. Три подхода:

- **Мок транспорта** (`respx` для httpx, `responses` для requests) — быстро, но снова «отвечает как придумал».
- **Запись и воспроизведение** (`vcrpy`) — один раз записали настоящий ответ в файл, дальше играем его. Ближе к правде, но записи устаревают.
- **Контрактные тесты** — обе стороны проверяют соответствие общей схеме. Единственный способ поймать, что чужой API изменился (Версионирование API).

```python
import respx, httpx

@respx.mock
def test_charge_retries_on_503():
    route = respx.post("https://billing/charge").mock(
        side_effect=[httpx.Response(503), httpx.Response(200, json={"ok": True})]
    )
    assert charge("order-1", 100)["ok"] is True
    assert route.call_count == 2          # проверили, что ретрай сработал
```

Здесь мок уместен: мы тестируем **свою** логику ретраев (Устойчивость — таймауты, ретраи, circuit breaker), а не чужой сервис.

---

## Сколько интеграционных тестов

Пирамида тестирования: юнитов много, интеграционных меньше, e2e совсем мало. Причина — время: юнит-тест миллисекунды, интеграционный десятки миллисекунд плюс старт контейнера.

**Практическое правило:** интеграционным тестом покрывай **каждый запрос к базе, который написан руками**, и **каждый сценарий с транзакцией**. Чистую бизнес-логику без похода наружу оставь юнит-тестам — она и должна быть отделена от ввода-вывода (Чистая архитектура и DDD).

---

## Сквозной пример: тест сценария заказа

**Шаг 1. Фикстуры** — как выше: контейнер на сессию, транзакция на тест.

**Шаг 2. Тест счастливого пути.**

```python
def test_order_reduces_stock(session):
    product = Product(id=1, остаток=5)
    session.add(product); session.flush()

    create_order(session, product_id=1, qty=2)

    session.refresh(product)
    assert product.остаток == 3
```

**Шаг 3. Тест граничного случая** — того самого, который мок бы не поймал:

```python
def test_order_fails_when_out_of_stock(session):
    session.add(Product(id=1, остаток=0)); session.flush()
    with pytest.raises(OutOfStock):
        create_order(session, product_id=1, qty=1)
```

**Шаг 4. Тест конкурентности** — две параллельные сессии на один товар проверяют, что не случилось потерянного обновления. Такой тест **невозможен** ни с моком, ни с SQLite.

**Шаг 5. В CI** — Docker доступен в GitHub Actions из коробки, отдельная настройка сервисов не нужна: testcontainers поднимет всё сам ([CI CD](../../../DevOps/CI%20CD.md)).

---

## Связи

- [Юнит-тестирование Python](../../../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md) — уровень ниже: моки, стабы, пирамида тестирования
- [pytest](../../../Библиотеки/Сторонние/Тесты/pytest.md) — фикстуры, параметризация, запуск
- [Тестирование FastAPI-приложений](../../../Фреймворки/Тестирование%20FastAPI-приложений.md) — где эта база подставляется в приложение через `dependency_overrides`
- [Нагрузочное тестирование — locust и k6](../../../Библиотеки/Сторонние/Тесты/Нагрузочное%20тестирование%20—%20locust%20и%20k6.md) — уровень выше: поведение под нагрузкой
- Миграции и zero-downtime — в тестах гоняем те же миграции, что в проде
- [Docker](../../../DevOps/Docker.md) — testcontainers работает поверх него
- [CI CD](../../../DevOps/CI%20CD.md) — где эти тесты запускаются на каждом пул-реквесте

## Источники

- [testcontainers-python — Documentation](https://testcontainers-python.readthedocs.io/)
- [Testcontainers — Getting started with Python](https://testcontainers.com/guides/getting-started-with-testcontainers-for-python/)
- [pytest — Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
- [respx — Documentation](https://lundberg.github.io/respx/)
