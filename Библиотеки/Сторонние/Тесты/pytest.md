[← Оглавление](../../../README.md)

# pytest

> _Тест — это функция с `assert`. Всё остальное pytest берёт на себя_

---

**pytest** — стандартный фреймворк тестирования для [Python](../../../Python.md). Его сила в простоте: тест — обычная функция с `assert`, не нужны классы и специальные методы. Плюс мощные **фикстуры** (подготовка данных), **параметризация** (один тест на много входов) и богатая экосистема плагинов (`pytest-django`, `pytest-cov`).

```bash
pip install pytest
```

## Первый тест

Файлы и функции начинаются с `test_`; проверка — обычным `assert`:

```python
# test_math.py
def add(a, b):
    return a + b

def test_add():
    assert add(2, 3) == 5
```

Запуск — командой `pytest` (сам находит все `test_*`):

```bash
pytest              # запустить все тесты
pytest -v           # подробно, по каждому тесту
pytest test_math.py # конкретный файл
```

## assert с интроспекцией

В отличие от `unittest` с его `assertEqual`, pytest использует обычный `assert` — и при падении **сам показывает**, какие были значения:

```python
def test_example():
    result = add(2, 2)
    assert result == 5      # при падении: assert 4 == 5
```

## Фикстуры

**Фикстура** — функция, готовящая данные или ресурс для теста. Тест объявляет её как аргумент, pytest подставит результат (внедрение зависимостей):

```python
import pytest

@pytest.fixture
def sample_user():
    return {'name': 'Alex', 'age': 25}

def test_user_name(sample_user):        # фикстура приходит в аргумент
    assert sample_user['name'] == 'Alex'
```

Фикстуры умеют **освобождать ресурсы** через `yield` (код после `yield` — очистка):

```python
@pytest.fixture
def db_connection():
    conn = connect()        # setup — до yield
    yield conn              # отдать тесту
    conn.close()            # teardown — после теста
```

> Общие фикстуры выносят в файл **`conftest.py`** — pytest подхватывает их автоматически во всех тестах папки, без импорта.

## Параметризация

**`@pytest.mark.parametrize`** прогоняет один тест на множестве входов — вместо копипасты:

```python
@pytest.mark.parametrize('a, b, expected', [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected      # 3 отдельных теста
```

## Маркеры

**Маркеры** помечают тесты для особого поведения:

```python
@pytest.mark.skip(reason='пока не готово')      # пропустить
def test_feature(): ...

@pytest.mark.xfail                              # ожидаемо падает
def test_known_bug(): ...

@pytest.mark.slow                               # свой маркер: pytest -m slow
def test_heavy(): ...
```

## Проверка исключений

Что код **бросает** ошибку — проверяют через `pytest.raises`:

```python
import pytest

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        1 / 0
```

## Сквозной пример: тестируем функцию

```python
# calc.py
def divide(a, b):
    if b == 0:
        raise ValueError('деление на ноль')
    return a / b

# test_calc.py
import pytest
from calc import divide

@pytest.mark.parametrize('a, b, expected', [(6, 2, 3), (9, 3, 3), (1, 2, 0.5)])
def test_divide(a, b, expected):
    assert divide(a, b) == expected        # проверяем корректные входы

def test_divide_by_zero():
    with pytest.raises(ValueError):        # проверяем обработку ошибки
        divide(1, 0)
```

**Как это читать.** Параметризованный тест разом покрывает три случая деления; отдельный тест проверяет, что на ноль функция бросает `ValueError`. Запуск `pytest` найдёт оба и покажет результат по каждому входу. В Django этот же pytest расширяют плагином — см. [Тестирование Django приложений](../../../Фреймворки/DJANGO/Тестирование%20Django%20приложений.md).

## Связи

- [Юнит-тестирование Python](../../../Библиотеки/Сторонние/Тесты/Юнит-тестирование%20Python.md) — концепция юнит-тестов: изоляция, AAA, `unittest`, моки, TDD, пирамида;
- [Тестирование Django приложений](../../../Фреймворки/DJANGO/Тестирование%20Django%20приложений.md) — pytest + pytest-django: фикстуры, APIClient, покрытие;
- [Тестирование FastAPI-приложений](../../../Фреймворки/Тестирование%20FastAPI-приложений.md) — pytest + `TestClient`/`AsyncClient`, `dependency_overrides`, async-тесты;
- [CI CD](../../../DevOps/CI%20CD.md) — pytest запускается автоматически в пайплайне на каждый пуш;
- [Обработка ошибок Python](../../../Ядро%20языка/Обработка%20ошибок%20Python.md) — исключения, которые проверяют через `pytest.raises`.

## Источники

- [pytest documentation](https://docs.pytest.org/en/stable/)
- [Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
- [Parametrizing tests](https://docs.pytest.org/en/stable/how-to/parametrize.html)
