[← Оглавление](../README.md)

# Ошибки Python

> В Python два класса проблем: **синтаксические ошибки** (до запуска) и **исключения** (во время выполнения). Исключение — это объект. Каждое исключение — экземпляр класса, наследующего `BaseException`. Обработка — в [Обработка ошибок Python](../Ядро%20языка/Обработка%20ошибок%20Python.md).

---

### Что такое исключение

Когда Python встречает недопустимую операцию — он **бросает** (raises) исключение: создаёт объект-ошибку и начинает искать обработчик вверх по стеку вызовов. Если обработчика нет — программа падает и печатает Traceback.

```python
100 / 0
# Traceback (most recent call last):
#   File "script.py", line 1, in <module>
#     100 / 0
# ZeroDivisionError: division by zero
```

Разбор сообщения:

- `Traceback (most recent call last)` — список вызовов, последний внизу
- `File "script.py", line 1, in <module>` — файл и строка
- `100 / 0` — выражение где упало
- `ZeroDivisionError` — класс исключения
- `division by zero` — текст сообщения

Читать Traceback **снизу вверх**: последняя строка — тип и причина, выше — путь до места падения.

---

### Синтаксические ошибки

Обнаруживаются **до запуска** на этапе разбора кода. `try/except` их не поймает — программа вообще не стартует.

```python
if True
    print("ой")
# SyntaxError: expected ':'

x = (1 + 2
# SyntaxError: '(' was never closed

def f():
print("ой")
# IndentationError: expected an indented block

def g():
	x = 1
        y = 2   # смешаны табы и пробелы
# TabError: inconsistent use of tabs and spaces in indentation
```

IDE подсвечивает их красным ещё до запуска.

---

### Иерархия исключений

Все исключения наследуют `BaseException`. Перехватывая родителя — ловишь и всех потомков.

```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── ArithmeticError
    │   ├── FloatingPointError
    │   ├── OverflowError
    │   └── ZeroDivisionError
    ├── AssertionError
    ├── AttributeError
    ├── BufferError
    ├── EOFError
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── MemoryError
    ├── NameError
    │   └── UnboundLocalError
    ├── OSError
    │   ├── FileNotFoundError
    │   ├── FileExistsError
    │   ├── PermissionError
    │   ├── TimeoutError
    │   ├── IsADirectoryError
    │   ├── NotADirectoryError
    │   └── ConnectionError
    │       ├── BrokenPipeError
    │       ├── ConnectionAbortedError
    │       ├── ConnectionRefusedError
    │       └── ConnectionResetError
    ├── RuntimeError
    │   ├── NotImplementedError
    │   └── RecursionError
    ├── StopIteration
    ├── StopAsyncIteration
    ├── SyntaxError
    │   └── IndentationError
    │       └── TabError
    ├── TypeError
    ├── ValueError
    │   └── UnicodeError
    │       ├── UnicodeDecodeError
    │       ├── UnicodeEncodeError
    │       └── UnicodeTranslateError
    └── Warning
        ├── DeprecationWarning
        ├── RuntimeWarning
        └── UserWarning
```

---

### Системные исключения — не перехватывать

|Исключение|Когда|Почему не трогать|
|---|---|---|
|`SystemExit`|`sys.exit()`|Это нормальный выход из программы|
|`KeyboardInterrupt`|Ctrl+C|Пользователь сознательно остановил|
|`GeneratorExit`|`.close()` у генератора|Часть протокола генератора|

`except Exception` их не поймает — они наследуют `BaseException` напрямую, минуя `Exception`. Голый `except:` или `except BaseException:` поймает — но это сломает нормальное завершение.

```python
# ПЛОХО — программа не реагирует на Ctrl+C:
while True:
    try:
        делать_что_то()
    except:          # ловит и KeyboardInterrupt
        pass

# ХОРОШО:
while True:
    try:
        делать_что_то()
    except Exception:  # SystemExit и KeyboardInterrupt пройдут насквозь
        pass
```

---

### Частые исключения подробно

#### ZeroDivisionError

**Что значит:** попытка делить на ноль.

```python
10 / 0      # ZeroDivisionError: division by zero
10 // 0     # ZeroDivisionError: integer division or modulo by zero
10 % 0      # ZeroDivisionError: integer division or modulo by zero
```

**Применение:** проверять делитель или ловить исключение.

```python
def безопасное_деление(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        return None

# или inline:
результат = a / b if b != 0 else None
```

---

#### TypeError

**Что значит:** операция применена к объекту неподходящего типа. «Ты пытаешься сложить число и строку, я не знаю как это делать».

```python
"2" + 2          # TypeError: can only concatenate str (not "int") to str
len(42)          # TypeError: object of type 'int' has no len()
None + 1         # TypeError: unsupported operand type(s) for +: 'NoneType' and 'int'
[1, 2] < "abc"   # TypeError: '<' not supported between instances of 'list' and 'str'

def f(a, b): pass
f(1)             # TypeError: f() missing 1 required positional argument: 'b'
f(1, 2, 3)       # TypeError: f() takes 2 positional arguments but 3 were given
```

**Магические методы:** `TypeError` возникает когда не реализован нужный метод — `__add__`, `__len__`, `__lt__` и т.д. Реализуешь метод — ошибки не будет.

```python
class Деньги:
    def __init__(self, сумма):
        self.сумма = сумма

    def __add__(self, other):
        if not isinstance(other, Деньги):
            return NotImplemented    # Python сам превратит в TypeError
        return Деньги(self.сумма + other.сумма)

    def __str__(self):
        return f"{self.сумма}₽"

print(Деньги(100) + Деньги(50))   # → 150₽
Деньги(100) + 50                   # TypeError: unsupported operand type(s)
```

---

#### ValueError

**Что значит:** тип правильный, но значение не имеет смысла для операции. «Я знаю что это строка, но `"abc"` не может стать числом».

```python
int("abc")           # ValueError: invalid literal for int() with base 10: 'abc'
int("3.14")          # ValueError — нельзя напрямую, нужно float() → int()
float("нет")         # ValueError: could not convert string to float: 'нет'
[1, 2, 3].index(99)  # ValueError: 99 is not in list
[1, 2, 3].remove(9)  # ValueError: list.remove(x): x not in list
import math
math.sqrt(-1)         # ValueError: math domain error
```

**Применение:** часто возникает при парсинге пользовательского ввода.

```python
def получить_возраст():
    while True:
        try:
            возраст = int(input("Возраст: "))
            if возраст < 0 or возраст > 150:
                raise ValueError("возраст должен быть от 0 до 150")
            return возраст
        except ValueError as e:
            print(f"Ошибка: {e}. Попробуй ещё раз.")
```

---

#### NameError / UnboundLocalError

**Что значит:** переменная с таким именем не существует в текущей области видимости.

```python
print(x)      # NameError: name 'x' is not defined
print(math)   # NameError: name 'math' is not defined (забыли import)
```

`UnboundLocalError` — подвид `NameError`. Возникает когда Python видит присвоение переменной **где-то ниже** в функции и считает её локальной, но выше ты её уже используешь.

```python
x = 10

def f():
    print(x)   # UnboundLocalError: local variable 'x' referenced before assignment
    x = 20     # Python видит это → x считается локальной для всей функции

# исправление через global или nonlocal:
def f():
    global x
    print(x)   # OK — берёт глобальную
    x = 20
```

---

#### IndexError / KeyError

**Что значит:** обращение по несуществующему индексу или ключу.

```python
lst = [1, 2, 3]
lst[5]    # IndexError: list index out of range
lst[-10]  # IndexError: list index out of range

d = {"a": 1}
d["b"]    # KeyError: 'b'
```

**Магические методы:** `__getitem__` — именно его вызывает `obj[key]`. Должен бросать `IndexError` или `KeyError` если ключ не найден.

```python
class МойКонтейнер:
    def __init__(self):
        self._data = {}

    def __getitem__(self, key):
        if key not in self._data:
            raise KeyError(f"ключ {key!r} не найден")
        return self._data[key]

    def __setitem__(self, key, value):
        self._data[key] = value
```

**Применение:** безопасное получение.

```python
# список — проверка индекса:
значение = lst[i] if 0 <= i < len(lst) else None

# словарь — .get() не бросает KeyError:
значение = d.get("b", "по умолчанию")

# или с try/except:
try:
    значение = d["b"]
except KeyError:
    значение = "по умолчанию"
```

---

#### AttributeError

**Что значит:** у объекта нет такого атрибута или метода.

```python
"строка".nonexist   # AttributeError: 'str' object has no attribute 'nonexist'
None.upper()        # AttributeError: 'NoneType' object has no attribute 'upper'
[].push             # AttributeError: 'list' object has no attribute 'push'
```

Частая причина — опечатка в имени метода или работа с `None` когда ожидался объект.

**Магические методы:** `__getattr__` вызывается когда атрибут не найден обычным способом. Позволяет перехватить и вернуть что-то вместо `AttributeError`. `__getattribute__` — вызывается при **каждом** обращении к атрибуту.

```python
class ГибкийОбъект:
    def __init__(self):
        self._данные = {}

    def __getattr__(self, name):
        # вызывается только если атрибут не нашёлся обычным путём
        if name in self._данные:
            return self._данные[name]
        raise AttributeError(f"нет атрибута '{name}'")

    def __setattr__(self, name, value):
        if name.startswith("_"):
            super().__setattr__(name, value)   # обычное присвоение
        else:
            self._данные[name] = value         # сохранить в словарь

obj = ГибкийОбъект()
obj.имя = "Иван"    # через __setattr__
print(obj.имя)      # через __getattr__ → "Иван"
print(obj.нет)      # AttributeError: нет атрибута 'нет'
```

---

#### ImportError / ModuleNotFoundError

**Что значит:** модуль не найден или из него нельзя импортировать то что просишь.

```python
import nonexistent        # ModuleNotFoundError: No module named 'nonexistent'
from os import xyz        # ImportError: cannot import name 'xyz' from 'os'
from math import Pi       # ImportError (правильно: math.pi, строчными)
```

**Применение:** проверка опциональных зависимостей.

```python
try:
    import numpy as np
    NUMPY_AVAILABLE = True
except ImportError:
    NUMPY_AVAILABLE = False

def обработать(данные):
    if NUMPY_AVAILABLE:
        return np.array(данные).mean()
    return sum(данные) / len(данные)
```

---

#### OSError и файловые ошибки

**Что значит:** ошибка на уровне операционной системы — файлы, права, сеть, процессы.

```python
open("нет.txt")          # FileNotFoundError: [Errno 2] No such file or directory
open("/etc/shadow")      # PermissionError: [Errno 13] Permission denied
import os
os.mkdir("уже_есть")     # FileExistsError: [Errno 17] File exists
os.rmdir("файл.txt")     # NotADirectoryError
```

`OSError` — базовый. Атрибуты объекта исключения: `errno` (числовой код), `strerror` (текст), `filename`.

```python
import errno

try:
    open("нет.txt")
except OSError as e:
    print(e.errno)      # → 2
    print(e.strerror)   # → No such file or directory
    print(e.filename)   # → нет.txt
    if e.errno == errno.ENOENT:
        print("файл не существует")
    elif e.errno == errno.EACCES:
        print("нет прав")
```

---

#### RecursionError / NotImplementedError

```python
# RecursionError — рекурсия слишком глубокая:
def f(): f()
f()   # RecursionError: maximum recursion depth exceeded

import sys
sys.getrecursionlimit()    # → 1000 по умолчанию
sys.setrecursionlimit(5000)

# NotImplementedError — метод должен быть переопределён в подклассе:
class Фигура:
    def площадь(self):
        raise NotImplementedError("реализуй в подклассе")

class Квадрат(Фигура):
    def __init__(self, сторона):
        self.сторона = сторона

    def площадь(self):
        return self.сторона ** 2

Фигура().площадь()    # NotImplementedError: реализуй в подклассе
Квадрат(5).площадь()  # → 25
```

---

#### StopIteration

**Что значит:** итератор исчерпан. `for` перехватывает автоматически. Вручную — при `next()`.

**Магические методы:** `__next__` должен бросать `StopIteration` когда элементы кончились. `__iter__` возвращает `self`.

```python
it = iter([1, 2])
next(it)         # → 1
next(it)         # → 2
next(it)         # StopIteration

next(it, None)   # → None вместо исключения (безопасный вариант)

# свой итератор:
class Диапазон:
    def __init__(self, n):
        self.n = n
        self.i = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.i >= self.n:
            raise StopIteration
        self.i += 1
        return self.i

for x in Диапазон(3):
    print(x)   # → 1, 2, 3
```

---

#### AssertionError

**Что значит:** условие `assert` оказалось ложным. Используется для отладки, не для валидации пользовательских данных.

```python
assert 1 == 2                           # AssertionError
assert isinstance(x, int), f"ожидался int, получен {type(x).__name__}"

# ВАЖНО: assert отключается при python -O script.py — не полагайся на него в продакшене
```

---

#### UnicodeError

**Что значит:** ошибка кодирования или декодирования текста.

```python
"привет".encode("ascii")       # UnicodeEncodeError: 'ascii' codec can't encode...
b"\xff\xfe".decode("utf-8")    # UnicodeDecodeError: invalid start byte
```

**Применение:** явно указывай кодировку и используй параметр `errors`.

```python
"привет".encode("ascii", errors="ignore")         # → b""
"привет".encode("ascii", errors="replace")        # → b"??????"
"привет".encode("ascii", errors="backslashreplace") # → b"\\u043f\\u0440..."

open("файл.txt", encoding="utf-8", errors="replace")
```

---

### Магические методы, связанные с исключениями

|Метод|Где используется|
|---|---|
|`__getitem__(self, key)`|Должен бросать `KeyError`/`IndexError` если ключ не найден|
|`__next__(self)`|Должен бросать `StopIteration` когда элементы кончились|
|`__getattr__(self, name)`|Вызывается вместо `AttributeError` когда атрибут не найден|
|`__setattr__(self, name, val)`|Перехватывает любое присваивание атрибута|
|`__add__`, `__sub__` и др.|Должны возвращать `NotImplemented` (не raise!) для неподдерживаемых типов|
|`__exit__(self, exc_type, exc_val, tb)`|Получает информацию об исключении, `return True` подавляет его|

```python
# NotImplemented vs NotImplementedError — разные вещи:
class Вектор:
    def __add__(self, other):
        if not isinstance(other, Вектор):
            return NotImplemented      # ← синглтон-объект, не raise
            # Python попробует other.__radd__(self), потом автоматически → TypeError
        return Вектор(...)

    def абстрактный(self):
        raise NotImplementedError      # ← исключение, для абстрактных методов
```

---

### Просмотр иерархии любого исключения

```python
import inspect

print(inspect.getmro(FileNotFoundError))
# (<class 'FileNotFoundError'>, <class 'OSError'>,
#  <class 'Exception'>, <class 'BaseException'>, <class 'object'>)
```

---

### Свои исключения

Создаются наследованием от `Exception` или конкретного потомка. Подробнее — в [Обработка ошибок Python](../Ядро%20языка/Обработка%20ошибок%20Python.md#свои-исключения).

```python
class AppError(Exception):
    """Корень всех ошибок приложения — удобно ловить одним except"""
    pass

class ValidationError(AppError):
    def __init__(self, поле, сообщение):
        self.поле = поле
        super().__init__(f"[{поле}] {сообщение}")

class NotFoundError(AppError):
    def __init__(self, сущность, id):
        super().__init__(f"{сущность} с id={id} не найден")

# наследуй от встроенного если семантика совпадает:
class NegativeValueError(ValueError):
    """Чужой код поймает как ValueError — удобно"""
    pass
```

> Механизм обработки (`try/except/else/finally`, `raise`, `raise from`, паттерны) — в [Обработка ошибок Python](../Ядро%20языка/Обработка%20ошибок%20Python.md).
