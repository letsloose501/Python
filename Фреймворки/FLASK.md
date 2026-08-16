[← Оглавление](../README.md)

# Flask

> _«Micro» — не про размер проекта, а про то, как мало навязано с самого начала_

---

**Flask** — микрофреймворк для веб-приложений на [Python](../Python.md). «Микро» означает, что ядро минимально: простое приложение помещается в один файл, а нужное (база, формы, авторизация) добавляется по мере роста расширениями. Противоположность подходу [Django](../Фреймворки/DJANGO/Django.md) с «батарейками в комплекте»: Flask ничего не навязывает — ты сам выбираешь компоненты.

## Минимальное приложение

```python
from flask import Flask

app = Flask('app')

@app.route('/')
def index():
    return 'Hello, world'

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

- **`Flask('app')`** — создаёт приложение.
- **`@app.route('/')`** — привязывает URL к функции-обработчику.
- **`app.run(...)`** — поднимает сервер разработки.

## Маршруты

Маршрут задают декоратором `@app.route` (с указанием методов) или явно через `add_url_rule`:

```python
@app.route('/hello/<name>', methods=['GET'])   # <name> — переменная пути
def hello(name):
    return f'Hello, {name}'

# то же самое без декоратора:
app.add_url_rule('/hello_world/', view_func=hello_world, methods=['POST'])
```

- **`<name>`** — динамическая часть URL, приходит в аргумент функции (`<int:id>` — с типом).
- **`methods`** — какие HTTP-методы обрабатывает маршрут (по умолчанию только `GET`).

## JSON-ответы

Для API Flask отдаёт JSON через `jsonify` — он сериализует словарь и ставит правильный заголовок:

```python
from flask import Flask, jsonify

app = Flask('app')

@app.route('/hello_world/', methods=['POST'])
def hello_world():
    return jsonify({'hello': 'world'})

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

Клиент на [requests](../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md):

```python
import requests

data = requests.post('http://127.0.0.1:5000/hello_world/')
print(data.status_code)     # 200
print(data.json())          # {'hello': 'world'}
```

## Расширения

Ядро Flask намеренно скромное — функциональность добавляют расширениями, каждое устанавливается отдельно:

- **Flask-SQLAlchemy** — ORM ([SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md)) для работы с БД;
- **Flask-Migrate** — миграции схемы;
- **Flask-RESTful** — построение REST API;
- **Flask-Login** — аутентификация пользователей.

Так собирается ровно тот стек, который нужен, без лишнего.

## Flask против Django

| | Flask | Django |
|---|---|---|
| **Философия** | микро, ничего не навязано | «батарейки в комплекте» |
| **ORM, админка, auth** | подключаешь расширениями | встроены |
| **Гибкость** | высокая (сам выбираешь всё) | по соглашениям фреймворка |
| **Порог входа** | простой старт, растёт вручную | больше правил сразу |
| **Когда брать** | небольшой сервис, микросервис, полный контроль | крупный сайт «из коробки» |

## Связи

- [Django](../Фреймворки/DJANGO/Django.md) — «тяжёлая» альтернатива с батарейками; сравнение выше;
- [FastAPI](../Фреймворки/FastAPI.md) — современный микрофреймворк с async и авто-документацией;
- [Asyncio Event Loop и aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — асинхронный веб-сервер (Flask по умолчанию синхронный);
- [requests](../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md) — типичный клиент для тестовых запросов к Flask;
- [WSGI и ASGI — интерфейс сервера и приложения](../WSGI%20и%20ASGI%20—%20интерфейс%20сервера%20и%20приложения.md) — интерфейс, по которому gunicorn зовёт Flask; там же мини-фреймворк с роутером-декоратором — Flask изнутри;
- [Развертывание проекта](../DevOps/Развертывание%20проекта.md) — Flask в прод выкатывают так же (gunicorn + nginx).

## Источники

- [Flask documentation](https://flask.palletsprojects.com/)
- [Flask — Quickstart](https://flask.palletsprojects.com/en/stable/quickstart/)
- [Flask extensions](https://flask.palletsprojects.com/en/stable/extensions/)
