[← Оглавление](../../README.md)

# pydantic

> _Аннотации типов начинают работать: объявил тип — данные проверены_

---

**pydantic** — библиотека валидации данных для [Python](../../Python.md) на основе аннотаций типов. Ты описываешь модель обычными подсказками типов, а pydantic в рантайме **проверяет** входные данные, **приводит** их к нужному типу и внятно **сообщает об ошибках**. Это ядро [FastAPI](../../Фреймворки/FastAPI.md) и стандарт для разбора конфигов, JSON и любых внешних данных. Ниже — актуальная версия **pydantic v2**.

```bash
pip install pydantic
```

## BaseModel — модель данных

Модель — класс-наследник `BaseModel` с полями-аннотациями:

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    name: str
    age: int = 18            # значение по умолчанию → поле необязательно

user = User(id=1, name='Alex', age=25)
user.name                    # 'Alex'
```

## Валидация и приведение типов

pydantic не просто проверяет типы — он **приводит** совместимые значения. Строку `"123"` в поле `int` превратит в число; неприводимое — отвергнет:

```python
User(id='1', name='Alex')        # id='1' → 1 (приведение), age=18 по умолчанию
```

Некорректные данные вызывают **`ValidationError`** с точным описанием, где и что не так:

```python
from pydantic import ValidationError

try:
    User(id='abc', name='Alex')
except ValidationError as e:
    print(e)     # 1 validation error for User → id: value is not a valid integer
```

## Ограничения полей — Field

Через `Field` задают дополнительные правила: границы чисел, длину строк, описание:

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)               # строго больше нуля
    tags: list[str] = Field(default_factory=list)   # пустой список по умолчанию
```

## Свои валидаторы

Когда встроенных правил мало — свои проверки декоратором `@field_validator` (одно поле) или `@model_validator` (вся модель):

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    name: str
    age: int

    @field_validator('age')
    @classmethod
    def check_age(cls, v: int) -> int:
        if v < 0:
            raise ValueError('возраст не может быть отрицательным')
        return v
```

## Сериализация

Модель легко превратить обратно в словарь или JSON:

```python
user.model_dump()          # → {'id': 1, 'name': 'Alex', 'age': 25} (dict)
user.model_dump_json()     # → '{"id":1,"name":"Alex","age":25}' (строка JSON)
User.model_validate(data)  # ← собрать модель из словаря (с валидацией)
```

## Где применяют

- **[FastAPI](../../Фреймворки/FastAPI.md)** — тела запросов и ответов описываются pydantic-моделями, валидация автоматическая;
- **Настройки** (`pydantic-settings`) — читать конфиг из переменных окружения с проверкой типов;
- **Разбор внешних данных** — JSON от API, содержимое файлов: сразу в типизированные объекты.

## Сквозной пример: валидация входных данных API

```python
from pydantic import BaseModel, Field, ValidationError

class Order(BaseModel):
    product: str = Field(min_length=1)
    quantity: int = Field(gt=0)               # только положительное
    price: float

# на входе — сырой словарь (например, из JSON запроса)
raw = {'product': 'Телефон', 'quantity': '2', 'price': 19999.9}

try:
    order = Order(**raw)          # 1. валидация + приведение ('2' → 2)
    print(order.quantity * order.price)      # 2. работаем с типизированными данными
except ValidationError as e:
    print(e)                      # 3. понятная ошибка, если данные кривые
```

**Как это читать.** Модель `Order` — это и схема, и валидатор: `quantity='2'` приводится к `2`, `quantity=0` было бы отвергнуто (`gt=0`). Дальше в коде работаешь с гарантированно корректными типизированными полями, а не с сырым словарём. Ровно так это делает [FastAPI](../../Фреймворки/FastAPI.md) под капотом.

## Связи

- [FastAPI](../../Фреймворки/FastAPI.md) — построен на pydantic: модели тел запросов и валидация из коробки;
- [typing](../../Библиотеки/Модули/Типы%20и%20объекты/typing.md) — аннотации типов, на которых держится pydantic;
- [dataclass](../../Библиотеки/Модули/Типы%20и%20объекты/dataclass.md) — похож синтаксисом, но без валидации в рантайме (сравни подход);
- Сериализация — model_dump/model_dump_json как её частный случай.

## Источники

- [Pydantic documentation](https://docs.pydantic.dev/latest/)
- [Models](https://docs.pydantic.dev/latest/concepts/models/)
- [Validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [Serialization](https://docs.pydantic.dev/latest/concepts/serialization/)
