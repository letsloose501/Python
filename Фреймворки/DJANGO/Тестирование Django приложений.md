[← Оглавление](../../README.md)

# Тестирование Django приложений

> _Тест — это код, который проверяет, что твой код работает. И не ломается завтра_

---

**Тестирование** — неотъемлемая часть разработки: автотесты проверяют, что код делает то, что должен, и не ломается при изменениях. Во многих командах есть требование по **покрытию** — например, не ниже 80% строк. У Django есть встроенный модуль на `unittest`, но мощнее и удобнее **[pytest](../../Библиотеки/Сторонние/Тесты/pytest.md)** — его и используют, не теряя при этом инструментов Django для тестов.

## Настройка pytest-django

```bash
pip install pytest-django
```

Чтобы pytest подхватил настройки Django, в корне проекта — файл `pytest.ini`:

```ini
[pytest]
DJANGO_SETTINGS_MODULE = drf_test.settings   # путь к settings проекта
```

Тесты кладут в пакеты `tests/` внутри приложений; файлы и функции начинаются с `test_`. Запуск — командой `pytest`.

## Структура теста: AAA

Тело теста делят на три части — паттерн **Arrange–Act–Assert**:

```python
def test_something():
    # Arrange — подготовка данных и окружения
    # Act     — вызов тестируемого функционала
    # Assert  — проверка, что результат ожидаемый
    assert 2 == 2
```

`assert` — обычное утверждение Python: если условие ложно, тест падает.

> Тесту, который ходит в базу, нужен доступ к тестовой БД. Его открывает декоратор **`@pytest.mark.django_db`** — без него обращение к моделям упадёт с ошибкой.

## APIClient — тестируем API

Для запросов к DRF-эндпоинтам DRF даёт **`APIClient`** — он имитирует HTTP-клиента внутри теста:

```python
import pytest
from rest_framework.test import APIClient
from django.contrib.auth.models import User

@pytest.mark.django_db
def test_get_messages():
    # Arrange
    client = APIClient()
    user = User.objects.create_user('admin')
    Message.objects.create(user_id=user.id, text='test')
    # Act
    response = client.get('/messages/')
    # Assert
    assert response.status_code == 200
    data = response.json()
    assert len(data) == 1
    assert data[0]['text'] == 'test'
```

## Фикстуры

**Фикстура** — функция, которая готовит и возвращает данные для тестов. Тест принимает её как аргумент — pytest подставит результат сам. Так убирают повторяющуюся подготовку:

```python
@pytest.fixture
def client():
    return APIClient()

@pytest.fixture
def user():
    return User.objects.create_user('admin')

@pytest.mark.django_db
def test_create_message(client, user):          # фикстуры приходят в аргументы
    count = Message.objects.count()
    response = client.post('/messages/', data={'user': user.id, 'text': 'test text'})
    assert response.status_code == 201
    assert Message.objects.count() == count + 1  # запись реально создалась
```

> Чтобы не писать `format='json'` в каждом запросе, задай формат по умолчанию в `settings.py`:
> `REST_FRAMEWORK = {'TEST_REQUEST_DEFAULT_FORMAT': 'json'}`.

## model_bakery — генерация данных

Готовить объекты руками утомительно. **model_bakery** создаёт записи по модели с автозаполнением полей:

```bash
pip install model_bakery
```

```python
from model_bakery import baker

@pytest.fixture
def message_factory():
    def factory(**kwargs):
        return baker.make(Message, **kwargs)     # создать запись(и) Message
    return factory

@pytest.mark.django_db
def test_get_messages(client, user, message_factory):
    messages = message_factory(_quantity=10)     # сразу 10 объектов
    response = client.get('/messages/')
    assert response.status_code == 200
    assert len(response.json()) == len(messages)
```

**Фабрика-фикстура** (функция, возвращающая функцию) удобна, когда в разных тестах нужно разное число объектов.

## Покрытие кода

**Покрытие** (coverage) показывает, какие строки затронуты тестами, а какие нет. Считает пакет `pytest-cov`:

```bash
pip install pytest-cov
```

Какие файлы не учитывать — в `.coveragerc`:

```ini
[run]
omit = tests/*, .venv/*, manage.py, drf_test/*, *test.py
```

```bash
pytest --cov=.                      # процент покрытия в консоли
pytest --cov=. --cov-report=html    # наглядный HTML-отчёт (папка htmlcov/)
```

> Покрытие — ориентир, а не цель. 100% покрытых строк не значит «нет багов»: строка *исполнена* тестом ≠ её поведение *проверено* ассертами.

## Сквозной пример: тесты для API сообщений

```python
# tests/test_messages.py
import pytest
from rest_framework.test import APIClient
from django.contrib.auth.models import User
from app.models import Message

@pytest.fixture
def client():
    return APIClient()

@pytest.fixture
def user():
    return User.objects.create_user('admin')

@pytest.mark.django_db
def test_list(client, user):
    Message.objects.create(user_id=user.id, text='hi')   # Arrange
    response = client.get('/messages/')                  # Act
    assert response.status_code == 200                   # Assert
    assert len(response.json()) == 1

@pytest.mark.django_db
def test_create(client, user):
    count = Message.objects.count()
    response = client.post('/messages/', {'user': user.id, 'text': 'new'})
    assert response.status_code == 201
    assert Message.objects.count() == count + 1
```

**Как это читать.** Фикстуры `client` и `user` готовят окружение; `@pytest.mark.django_db` даёт тестовую БД (она откатывается после каждого теста); каждый тест следует AAA — создаёт данные, дёргает эндпоинт, проверяет статус и результат.

## Связи

- [pytest](../../Библиотеки/Сторонние/Тесты/pytest.md) — сам фреймворк тестирования: фикстуры, параметризация, ассерты;
- [CRUD В DRF](../../Фреймворки/DJANGO/CRUD%20В%20DRF.md) — эндпоинты, которые здесь тестируем через APIClient;
- [API](../../Фреймворки/DJANGO/API.md) — DRF-основы (сериализаторы, view);
- [CI CD](../../DevOps/CI%20CD.md) — тесты гоняются автоматически в пайплайне на каждый пуш.

## Источники

- [pytest-django documentation](https://pytest-django.readthedocs.io/)
- [Django — Testing overview](https://docs.djangoproject.com/en/stable/topics/testing/overview/)
- [DRF — Testing (APIClient)](https://www.django-rest-framework.org/api-guide/testing/)
- [model_bakery documentation](https://model-bakery.readthedocs.io/)
