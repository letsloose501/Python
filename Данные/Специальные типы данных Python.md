[← Оглавление](../README.md)

# Специальные типы данных Python

> Специальные типы не хранят «данные» в обычном смысле — они выражают **отсутствие значения**, **тип объекта**, **аннотации** и **управляющие конструкции**. О примитивных типах — в [Примитивные типы данных Python](../Данные/Примитивные%20типы%20данных%20Python.md). О составных — в [Составные типы данных Python](../Данные/Составные%20типы%20данных%20Python.md).

---

## NoneType — отсутствие значения

**Что это:** единственное значение — `None`. Означает «ничего», «не задано», «пусто». Аналог `null` в других языках.

`None` — **синглтон**: существует ровно один объект `None` в памяти. Поэтому проверяется через `is`, а не ` == `.

**Магические методы:** `__bool__` возвращает `False`, `__repr__` возвращает `'None'`.

```python
x = None
type(None)          # → <class 'NoneType'>

# правильная проверка:
if x is None:
    print("не задано")

if x is not None:
    print("задано")

# неправильно (работает, но не идиоматично):
if x == None:
    ...
```

### Когда появляется None

```python
# функция без return:
def ничего():
    pass

результат = ничего()    # → None

# функция с return без значения:
def тоже_ничего():
    return

# методы изменяющие объект на месте возвращают None:
lst = [3, 1, 2]
результат = lst.sort()  # → None! sort() меняет lst, не возвращает новый
print(результат)         # → None — распространённая ошибка

# аналогично: append, extend, reverse, update, add
```

### Использование None как значения по умолчанию

```python
# идиома: None как дефолтный аргумент (вместо изменяемого объекта)
def добавить(элемент, список=None):
    if список is None:
        список = []         # создаётся новый список при каждом вызове
    список.append(элемент)
    return список

# НЕПРАВИЛЬНО — общий список для всех вызовов:
def добавить(элемент, список=[]):   # список создаётся один раз при определении функции!
    список.append(элемент)
    return список
```

### Optional — None в аннотациях типов

`None` в аннотациях выражается через `Optional[X]` или (Python 3.10+) `X | None` — «значение типа `X` или `None`». Полный разбор вместе с `Union` и `Any` — ниже в разделе [Аннотации типов](#аннотации-типов).

### Плюсы

- Явное обозначение отсутствия значения — лучше чем `0`, `""`, `-1`
- Синглтон — проверка через `is` быстрее чем ` == `

### Минусы

- `None` — false, но `0`, `""`, `[]` тоже false → не путать `if x is None` и `if not x`

```python
x = 0
if not x:           # → True — выполнится, хотя x задан!
if x is None:       # → False — правильная проверка на отсутствие
```

---

## type — тип объекта

**Что это:** каждый объект в Python имеет тип, и этот тип сам является объектом класса `type`. Это метакласс — класс всех классов.

```python
type(42)            # → <class 'int'>
type("hello")       # → <class 'str'>
type([1, 2])        # → <class 'list'>
type(None)          # → <class 'NoneType'>

# type самого type:
type(int)           # → <class 'type'>
type(type)          # → <class 'type'> — type — экземпляр самого себя
```

### isinstance и issubclass

```python
isinstance(42, int)             # → True
isinstance(True, int)           # → True — bool — подкласс int
isinstance(42, (int, float))    # → True — проверка нескольких типов

issubclass(bool, int)           # → True
issubclass(int, object)         # → True — всё наследует object
```

### Динамическое создание классов через type

`type` можно вызвать с тремя аргументами — создаёт новый класс:

```python
# type(имя, базы, атрибуты)
Животное = type("Животное", (), {"говорит": lambda self: "..."})
Кот = type("Кот", (Животное,), {"говорит": lambda self: "Мяу"})

кот = Кот()
кот.говорит()   # → "Мяу"

# эквивалентно:
class Животное:
    def говорит(self):
        return "..."

class Кот(Животное):
    def говорит(self):
        return "Мяу"
```

> Раз классы создаёт `type`, можно объявить **свой метакласс** и вмешаться в создание классов (регистрация подклассов, валидация, ORM). Что это, можно ли менять и миксовать — в [Метаклассы Python](../Ядро%20языка/Метаклассы%20Python.md).

---

## Константы и Final

В Python **нет настоящих констант** — любое имя можно переприсвоить. Есть договорённость `UPPER_SNAKE_CASE`, аннотация `typing.Final` (её проверяет mypy, а не рантайм) и по-настоящему неизменяемые объекты — `tuple`, `frozenset`, `@dataclass(frozen=True)`, [enum](../Библиотеки/Модули/Типы%20и%20объекты/enum.md).

```python
from typing import Final

MAX_SIZE: Final = 100      # обещание «не менять»
MAX_SIZE = 200             # рантайм молчит; mypy: "Cannot assign to final name"
```

> Полный разбор — с уровнями защиты (соглашение → `Final` → неизменяемые объекты → `Enum`) и ловушками — в [Константы в Python](../Ядро%20языка/Константы%20в%20Python.md).

---

## Аннотации типов

**Что это:** синтаксис для указания ожидаемых типов переменных, аргументов и возвращаемых значений. Python **не проверяет** их во время выполнения — только для статических анализаторов (mypy, pyright, IDE).

Добавлены в Python 3.5+, активно развиваются.

### Базовый синтаксис

```python
# переменные
имя: str = "Иван"
возраст: int = 25
цена: float = 99.9
активен: bool = True

# функции
def приветствие(имя: str) -> str:
    return f"Привет, {имя}!"

def ничего_не_возвращает(x: int) -> None:
    print(x)
```

### Коллекции

```python
from typing import List, Dict, Tuple, Set  # Python 3.8 и ниже

def обработать(данные: List[int]) -> Dict[str, int]:
    ...

# Python 3.9+: встроенные типы напрямую
def обработать(данные: list[int]) -> dict[str, int]:
    ...

# вложенные:
матрица: list[list[float]]
конфиг: dict[str, list[int]]
```

### Optional, Union, Any

```python
from typing import Optional, Union, Any

# Optional[X] — значение типа X или None:
def найти(ключ: str) -> Optional[str]:
    ...

# Python 3.10+: X | None
def найти(ключ: str) -> str | None:
    ...

# Union — одно из нескольких типов:
def обработать(x: Union[int, str]) -> str:
    ...

# Python 3.10+:
def обработать(x: int | str) -> str:
    ...

# Any — любой тип (отключает проверку):
def принять_что_угодно(x: Any) -> None:
    ...
```

### Callable, Generator, Iterator

```python
from typing import Callable, Generator, Iterator

# функция как аргумент:
def применить(fn: Callable[[int, int], int], a: int, b: int) -> int:
    return fn(a, b)

# генератор:
def числа() -> Generator[int, None, None]:
    yield 1
    yield 2

# итератор:
def итератор() -> Iterator[str]:
    ...
```

### TypeVar — обобщённые типы

```python
from typing import TypeVar

T = TypeVar("T")

def первый(lst: list[T]) -> T:
    return lst[0]

первый([1, 2, 3])       # возвращает int
первый(["a", "b"])      # возвращает str
```

### TypedDict — словарь с известной структурой

```python
from typing import TypedDict

class Пользователь(TypedDict):
    имя: str
    возраст: int
    email: str

def создать(данные: Пользователь) -> None:
    ...

создать({"имя": "Иван", "возраст": 25, "email": "i@i.ru"})
```

### dataclass — класс-данные

```python
from dataclasses import dataclass, field

@dataclass
class Точка:
    x: float
    y: float
    z: float = 0.0              # значение по умолчанию

@dataclass
class Студент:
    имя: str
    оценки: list[int] = field(default_factory=list)  # изменяемый дефолт

p = Точка(1.0, 2.0)
p.x             # → 1.0
print(p)        # → Точка(x=1.0, y=2.0, z=0.0) — __repr__ автоматически

@dataclass(frozen=True)   # неизменяемый dataclass
class НеизменяемаяТочка:
    x: float
    y: float
```

---

## ellipsis — `...`

**Что это:** специальный синглтон `Ellipsis` типа `ellipsis`. Используется как заглушка в аннотациях, стабах и NumPy-срезах.

```python
type(...)           # → <class 'ellipsis'>
... is Ellipsis     # → True

# заглушка для тела функции (вместо pass):
def функция() -> None:
    ...             # допустимо вместо pass

# в аннотациях Callable: любые аргументы
from typing import Callable
fn: Callable[..., int]  # функция с любыми аргументами, возвращает int

# в NumPy: многомерные срезы
import numpy as np
arr = np.zeros((3, 4, 5))
arr[..., 0]         # → все измерения кроме последнего, первый элемент
```

---

## NotImplemented

**Что это:** специальный синглтон, который возвращается из [магических методов](../Ядро%20языка/Магические%20методы%20Python.md) сравнения и арифметики, когда операция не определена для данных типов. Не путать с `NotImplementedError`.

```python
class Число:
    def __init__(self, n):
        self.n = n

    def __add__(self, other):
        if isinstance(other, Число):
            return Число(self.n + other.n)
        return NotImplemented       # не raise, а return!
        # Python попробует other.__radd__(self)

# NotImplementedError — исключение, другое:
def абстрактный_метод(self):
    raise NotImplementedError("Подклассы должны реализовать этот метод")
```

---

## Специальные значения float

Технически это `float`, но семантически — специальные:

```python
inf = float("inf")          # положительная бесконечность
neg_inf = float("-inf")     # отрицательная бесконечность (в -inf присвоить нельзя — SyntaxError)
nan = float("nan")          # Not a Number

# арифметика с inf:
inf + 1         # → inf
inf * 2         # → inf
inf - inf       # → nan
1 / inf         # → 0.0
1 / 0           # → ZeroDivisionError (не inf!)

# nan — заражает всё:
nan + 1         # → nan
nan == nan      # → False (!)
nan is float("nan")  # → False (не синглтон)

import math
math.isnan(nan)         # → True  — правильная проверка
math.isinf(inf)         # → True
math.isfinite(42.0)     # → True  — не nan и не inf
```

---

## Сводная таблица

|Тип|Значение|Синглтон|Истинность|Применение|
|---|---|---|---|---|
|`NoneType`|`None`|Да|`False`|Отсутствие значения|
|`type`|`int`, `str`, ...|—|`True`|Тип объекта, метакласс|
|`ellipsis`|`...`|Да|`True`|Заглушки, аннотации, NumPy|
|`NotImplemented`|`NotImplemented`|Да|`True`|Возврат из магических методов|
|`float('inf')`|`inf`|Нет|`True`|Математическая бесконечность|
|`float('nan')`|`nan`|Нет|`True`|Неопределённость в числах|

> Примитивные типы (`int`, `float`, `str`...) — в [Примитивные типы данных Python](../Данные/Примитивные%20типы%20данных%20Python.md). Составные (`list`, `dict`, `set`...) — в [Составные типы данных Python](../Данные/Составные%20типы%20данных%20Python.md).
