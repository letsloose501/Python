[← Оглавление](../../../README.md)

# psycopg2

> _Прямой провод от Python к PostgreSQL — ты сам пишешь SQL_

---

**psycopg2** — самый популярный драйвер PostgreSQL для [Python](../../../Python.md): низкоуровневый мост, через который программа шлёт SQL-запросы в базу и получает результаты. В отличие от [ORM](../../../Библиотеки/Сторонние/ORM/SQLAlchemy.md), здесь ты пишешь SQL руками — больше контроля, но и больше ответственности. Именно psycopg2 работает «под капотом» у [Django ORM](../../../Фреймворки/DJANGO/ORM%20Django.md) и SQLAlchemy при подключении к PostgreSQL.

> **psycopg2 vs psycopg2-binary.** `psycopg2` компилируется под систему (нужен компилятор и заголовки libpq), `psycopg2-binary` — готовая сборка с бинарниками, ставится сразу. Для разработки берут `binary`, для прода часто собирают из исходников.

```bash
pip install psycopg2-binary
```

## Подключение и курсор

Работа идёт в два шага: **соединение** (`connect`) с базой и **курсор** (`cursor`) — объект, через который выполняют запросы и читают результат.

```python
import psycopg2

conn = psycopg2.connect(
    host='localhost', port=5432,
    dbname='mydb', user='myuser', password='secret',
)
cur = conn.cursor()          # курсор — через него шлём SQL
```

## Выполнение запросов

`execute` отправляет SQL; результат читают методами `fetch*`:

```python
cur.execute("SELECT id, name FROM users")
cur.fetchone()      # одна строка (кортеж) или None
cur.fetchall()      # все строки списком кортежей
cur.fetchmany(10)   # следующие 10 строк
```

## Параметризованные запросы — защита от SQL-инъекций

> **Главное правило psycopg2.** Никогда не подставляй данные в SQL через f-строки или конкатенацию — это открывает **SQL-инъекцию**. Передавай значения **вторым аргументом** `execute`, драйвер сам их безопасно экранирует.

```python
# ОПАСНО — SQL-инъекция:
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")     # НЕЛЬЗЯ

# ПРАВИЛЬНО — плейсхолдеры %s, значения отдельно:
cur.execute("SELECT * FROM users WHERE name = %s", (name,))
cur.execute("INSERT INTO users (name, age) VALUES (%s, %s)", (name, age))
```

Плейсхолдер всегда `%s` (независимо от типа), значения — кортежем. Так вредоносный ввод вроде `'; DROP TABLE users; --` попадёт в базу как обычная строка, а не как команда.

## Транзакции: commit и rollback

Изменения (`INSERT`, `UPDATE`, `DELETE`) не попадут в базу, пока не вызван **`commit`**. При ошибке — **`rollback`** откатывает всё с начала транзакции:

```python
try:
    cur.execute("INSERT INTO users (name, age) VALUES (%s, %s)", ('Alex', 25))
    conn.commit()          # зафиксировать изменения
except Exception:
    conn.rollback()        # откатить при ошибке
```

## Контекстный менеджер

`with` избавляет от ручного закрытия и упрощает транзакции: у соединения выход из блока делает `commit` (или `rollback` при исключении), курсор закрывается сам:

```python
with psycopg2.connect(dsn) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM users")
        rows = cur.fetchall()
# коммит и закрытие — автоматически
```

## Сквозной пример: создать таблицу и прочитать

```python
import psycopg2

with psycopg2.connect(host='localhost', dbname='mydb',
                      user='myuser', password='secret') as conn:
    with conn.cursor() as cur:
        # 1. создать таблицу
        cur.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                name TEXT,
                age INTEGER
            )
        """)
        # 2. вставить (параметризованно!)
        cur.execute("INSERT INTO users (name, age) VALUES (%s, %s)", ('Alex', 25))
        # 3. прочитать
        cur.execute("SELECT id, name, age FROM users WHERE age >= %s", (18,))
        for row in cur.fetchall():
            print(row)          # (1, 'Alex', 25)
    # commit на выходе из with conn
```

**Как это читать.** Соединение открывает доступ к базе, курсор выполняет SQL. Значения везде идут через `%s` — это защита от инъекций. Блок `with` сам фиксирует транзакцию и закрывает ресурсы. Когда ручной SQL надоедает — переходят на [SQLAlchemy](../../../Библиотеки/Сторонние/ORM/SQLAlchemy.md) или [ORM Django](../../../Фреймворки/DJANGO/ORM%20Django.md), которые генерируют его сами.

## Приёмы из практики — RETURNING, ON CONFLICT, динамический UPDATE

**`RETURNING` + `fetchone()` — получить сгенерированный id** и сразу вставить связанные строки (один-ко-многим: пользователь → телефоны):

```python
cur.execute("""INSERT INTO users(name, last_name, email)
               VALUES (%s, %s, %s) RETURNING id;""", (name, last_name, email))
user_id = cur.fetchone()[0]                       # id только что созданной строки
for phone in phones:
    cur.execute("INSERT INTO phones(user_id, phone) VALUES (%s, %s);", (user_id, phone))
```

**`ON CONFLICT ... DO NOTHING` + `cur.rowcount`** — вставить, только если не дубликат, и понять, была ли вставка:

```python
cur.execute("""INSERT INTO phones(user_id, phone) VALUES (%s, %s)
               ON CONFLICT (phone) DO NOTHING;""", (user_id, phone))
if cur.rowcount == 0:                             # 0 строк затронуто → конфликт
    raise ValueError("Номер уже существует")
```

**Динамический `UPDATE` через `psycopg2.sql`.** Имя столбца нельзя подставить через `%s` (это для значений). Для безопасной подстановки **идентификаторов** есть `psycopg2.sql.Identifier` — он экранирует имена, защищая от инъекции в структуре запроса:

```python
from psycopg2.sql import SQL, Identifier

fields, values = [], []
for col, val in [("name", name), ("last_name", last_name), ("email", email)]:
    if val is not None:                           # обновляем только переданные поля
        fields.append(SQL("{} = %s").format(Identifier(col)))
        values.append(val)

query = SQL("UPDATE users SET {} WHERE id = %s;").format(SQL(", ").join(fields))
values.append(user_id)
cur.execute(query, values)                        # значения по-прежнему через %s
```

**Обработка нарушения ограничений.** Уникальность/внешние ключи ловят через `psycopg2.IntegrityError` → откат → понятная ошибка приложению:

```python
try:
    cur.execute("INSERT INTO users(name, last_name, email) VALUES (%s, %s, %s);",
                (name, last_name, email))
    conn.commit()
except psycopg2.IntegrityError:
    conn.rollback()                               # обязателен перед новыми запросами
    raise ValueError("Пользователь с таким email уже существует")
```

> `%s` — только для **значений**; для имён таблиц/столбцов используй `sql.Identifier`, а не f-строки, иначе вернёшь ту же SQL-инъекцию, от которой защищались.

## Связи

- PostgreSQL — база, к которой psycopg2 подключается;
- [SQLAlchemy](../../../Библиотеки/Сторонние/ORM/SQLAlchemy.md) — ORM поверх psycopg2 (`postgresql+psycopg2://...`);
- [ORM Django](../../../Фреймворки/DJANGO/ORM%20Django.md) — Django ORM использует psycopg2 как драйвер PostgreSQL;
- [Работа с сервером](../../../DevOps/Работа%20с%20сервером.md#postgresql-на-сервере) — установка и настройка самой базы.

## Источники

- [psycopg2 documentation](https://www.psycopg.org/docs/)
- [Basic module usage](https://www.psycopg.org/docs/usage.html)
- [Passing parameters to SQL queries](https://www.psycopg.org/docs/usage.html#passing-parameters-to-sql-queries)
