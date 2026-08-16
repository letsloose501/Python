[← Оглавление](../README.md)

# Обработка ошибок Python

> `try/except` — механизм перехвата исключений во время выполнения. Позволяет программе продолжить работу вместо аварийного завершения. О видах исключений и их иерархии — в [Ошибки Python](../Ядро%20языка/Ошибки%20Python.md).

---

### Зачем обрабатывать исключения

Без обработки любая ошибка роняет программу. С обработкой — можно восстановиться, залогировать, показать пользователю понятное сообщение.

```python
# без обработки — программа падает:
число = int(input("Введи число: "))   # пользователь ввёл "abc" → ValueError, всё

# с обработкой — программа продолжает:
try:
    число = int(input("Введи число: "))
except ValueError:
    print("это не число")
    число = 0
```

---

### Базовая конструкция

```python
try:
    код_который_может_упасть
except ТипОшибки:
    что_делать_при_ошибке
```

Если исключение возникло — выполнение `try` **прерывается немедленно**, управление переходит в `except`. Остаток `try` пропускается.

```python
try:
    print("шаг 1")
    x = 1 / 0          # ← падает здесь
    print("шаг 2")     # ← никогда не выполнится
except ZeroDivisionError:
    print("поймали")
print("продолжаем")

# → шаг 1
# → поймали
# → продолжаем
```

---

### Что и где оборачивать в try

**Какой код помещать в `try` — только тот, что реально может упасть, и как можно меньше.** Остальное — за пределами `try`, чтобы исключение из «рискованной» строки не спуталось с ошибкой в коде, который ты не собирался охранять.

```python
# ПЛОХО — под try весь блок, поймаем не то
try:
    data = load(path)          # рискует упасть (файл/сеть)
    result = transform(data)   # если упадёт ЗДЕСЬ — тоже примем за "ошибку загрузки"
    save(result)
except OSError:
    print("не удалось загрузить")

# ХОРОШО — под try только рискованная строка
try:
    data = load(path)
except OSError:
    print("не удалось загрузить")
    return
result = transform(data)       # ошибки здесь не маскируются
save(result)
```

**Где обрабатывать** — там, где можешь **осмысленно** что-то сделать: подставить значение, повторить, показать пользователю. Если на этом уровне сделать нечего — **не лови**, дай исключению всплыть туда, где решение есть (обычно это граница программы: обработчик запроса, `main`, воркер). Ловишь только чтобы залогировать — **пробрось дальше** (`raise`).

> Правило: лови **конкретный** тип, **близко** к источнику — если знаешь, что делать; иначе пусть летит выше. Общий `except Exception` уместен лишь на верхней границе как последний рубеж (с логированием).

---

### Полная конструкция: try / except / else / finally

```python
try:
    код
except ТипОшибки:
    при ошибке
else:
    если ошибки не было
finally:
    выполняется всегда
```

|Блок|Когда выполняется|
|---|---|
|`try`|Всегда — это охраняемый код|
|`except`|Только если в `try` возникло совпадающее исключение|
|`else`|Только если `try` завершился **без** исключений|
|`finally`|**Всегда** — и при ошибке, и без неё, даже если был `return`|

```python
def прочитать_файл(путь):
    f = None
    try:
        f = open(путь, encoding="utf-8")
        содержимое = f.read()
    except FileNotFoundError:
        print(f"файл {путь} не найден")
        return None
    except PermissionError:
        print("нет прав на чтение")
        return None
    else:
        print("файл прочитан успешно")
        return содержимое
    finally:
        if f:
            f.close()   # выполнится в любом случае
```

**`else`** — полезен когда нужно отличить «код в try упал» от «код в try выполнился нормально, и вот дополнительная логика». Код в `else` не прикрыт блоком `try` — если упадёт там, это уже другое исключение.

**`finally`** — гарантированная очистка ресурсов: закрыть файл, соединение, отпустить блокировку. Выполняется даже при `return` внутри `try`.

```python
def f():
    try:
        return 1        # return не спасёт от finally
    finally:
        print("finally!")  # выполнится перед возвратом

f()   # печатает "finally!", возвращает 1
```

#### Ловушка: return в finally перекрывает try и глотает исключение

Обратная сторона «`finally` выполняется всегда»: если сам `finally` делает `return` (или `raise`), он задаёт **новый** выход из функции и затирает то, что шло из `try`. `return` из `try` перекрывается молча, а исключение — **проглатывается**.

```python
def f():
    try:
        return "из try"
    finally:
        return "из finally"

f()   # → "из finally"   — return из try перекрыт
```

Опаснее всего с исключением: `finally` с `return` гасит его беззвучно — баг исчезает без следа.

```python
def g():
    try:
        raise ValueError("важная ошибка")
    finally:
        return "ок"       # исключение затёрто выходом из finally

g()   # → "ок"   — ValueError проглочен, наверх не всплывает
```

Разбор: при выходе из `try` — по `return` или по исключению — Python запоминает это как **отложенное действие** и идёт выполнять `finally`. Завершился `finally` нормально — отложенное доигрывается (вернуть значение / пробросить исключение). Но `return`, `raise`, `break` или `continue` внутри `finally` — это новый выход; он заменяет отложенное действие, и вместе с ним пропадает исходное исключение.

> В `finally` — только очистка (закрыть файл, отпустить блокировку). Не ставь туда `return`/`break`/`continue` и безусловный `raise`: потеряешь и результат, и ошибку из `try`. Нужно преобразовать исключение — бросай из `except` (`raise … from e`), а не из `finally`.

---

### Несколько except

Python проверяет блоки `except` **сверху вниз** и входит в первый подходящий. Дальше не проверяет.

```python
try:
    x = int(input("число: "))
    y = 10 / x
except ValueError:
    print("не число")
except ZeroDivisionError:
    print("делить на ноль нельзя")
except Exception as e:
    print(f"что-то ещё: {e}")
```

> Конкретные исключения — выше. `Exception` — последним как резервный. Если поставить `Exception` первым — он поглотит всё, конкретные блоки никогда не сработают.

Несколько типов в одном `except` — через кортеж:

```python
except (ValueError, TypeError) as e:
    print(f"ошибка типа или значения: {e}")
```

---

### as — получить объект исключения

```python
try:
    int("abc")
except ValueError as e:
    print(e)          # → invalid literal for int() with base 10: 'abc'
    print(type(e))    # → <class 'ValueError'>
    print(e.args)     # → ('invalid literal for int() with base 10: \'abc\'',)
```

`e` — объект исключения. Атрибуты зависят от класса:

```python
try:
    open("нет.txt")
except OSError as e:
    print(e.errno)      # → 2 (числовой код ОС)
    print(e.strerror)   # → No such file or directory
    print(e.filename)   # → нет.txt
```

---

### Перехват по родительскому классу

Перехватывая родителя — ловишь и всех потомков. Удобно для групп схожих ошибок.

```python
try:
    x = 1 / 0
except ArithmeticError:    # поймает ZeroDivisionError, OverflowError, FloatingPointError
    print("арифметическая ошибка")

try:
    d = {}
    d["нет"]
except LookupError:        # поймает KeyError и IndexError
    print("нет такого ключа или индекса")
```

`except Exception` — ловит всё кроме системных (`SystemExit`, `KeyboardInterrupt`, `GeneratorExit`). Голый `except:` ловит вообще всё — почти всегда плохая идея.

---

### raise — бросить исключение

```python
raise ValueError("неверное значение")
raise TypeError               # без сообщения — тоже работает
raise                         # перебросить текущее исключение (только внутри except)
```

```python
def проверить_возраст(возраст):
    if not isinstance(возраст, int):
        raise TypeError(f"ожидался int, получен {type(возраст).__name__}")
    if возраст < 0:
        raise ValueError(f"возраст не может быть отрицательным: {возраст}")
    if возраст > 150:
        raise ValueError(f"нереалистичный возраст: {возраст}")
    return возраст
```

**Перебросить после логирования:**

```python
try:
    рискованная_операция()
except Exception as e:
    logging.error(f"ошибка: {e}")
    raise    # ← перебросить дальше, не глотать
```

---

### raise from — цепочка исключений

Когда одна ошибка вызвала другую — сохраняет оба в Traceback.

```python
try:
    int("abc")
except ValueError as e:
    raise RuntimeError("не удалось обработать данные") from e
```

Traceback покажет оба с пометкой `The above exception was the direct cause of the following exception`. Полезно при обёртке низкоуровневых ошибок в высокоуровневые.

```python
# скрыть исходное исключение (raise from None):
try:
    int("abc")
except ValueError:
    raise AppError("ошибка обработки") from None  # исходный ValueError не показывается
```

---

### Свои исключения

```python
class AppError(Exception):
    """Базовый класс ошибок приложения"""
    pass

class ValidationError(AppError):
    def __init__(self, поле, сообщение):
        self.поле = поле
        super().__init__(f"[{поле}] {сообщение}")

class NotFoundError(AppError):
    def __init__(self, сущность, id):
        super().__init__(f"{сущность} с id={id} не найден")
        self.сущность = сущность
        self.id = id


def найти_пользователя(id: int):
    if not isinstance(id, int):
        raise TypeError("id должен быть int")
    if id <= 0:
        raise ValidationError("id", "должен быть положительным")
    users = {1: "Иван", 2: "Аня"}
    if id not in users:
        raise NotFoundError("Пользователь", id)
    return users[id]


try:
    print(найти_пользователя(99))
except ValidationError as e:
    print(f"валидация провалилась: {e.поле} — {e}")
except NotFoundError as e:
    print(f"не найдено: {e}")
except AppError as e:
    print(f"общая ошибка приложения: {e}")   # резерв для всей группы

# → не найдено: Пользователь с id=99 не найден
```

Иерархия своих исключений: ловишь как конкретный тип, так и всю группу через базовый класс.

---

### Магические методы в обработке ошибок

#### `__exit__` — контекстный менеджер

`__exit__` вызывается при выходе из `with`-блока, в том числе при исключении. Получает три аргумента: тип, объект и traceback исключения. `return True` подавляет исключение.

```python
class СупрессорОшибки:
    def __init__(self, *типы):
        self.типы = типы

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, tb):
        if exc_type and issubclass(exc_type, self.типы):
            print(f"подавляем {exc_type.__name__}: {exc_val}")
            return True   # исключение подавлено
        return False      # пробросить дальше

with СупрессорОшибки(ZeroDivisionError, ValueError):
    x = 1 / 0    # → "подавляем ZeroDivisionError: division by zero"
    # выполнение продолжается после with

# то же самое готовое:
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove("нет.txt")   # не падает если файла нет
```

#### `__init__` и `__str__` в своих исключениях

```python
class HTTPError(Exception):
    def __init__(self, код, сообщение, url=None):
        self.код = код
        self.url = url
        super().__init__(сообщение)

    def __str__(self):
        base = f"HTTP {self.код}: {self.args[0]}"
        if self.url:
            base += f" (url: {self.url})"
        return base

    def __repr__(self):
        return f"HTTPError(код={self.код}, сообщение={self.args[0]!r})"

    @property
    def клиентская(self):
        return 400 <= self.код < 500

    @property
    def серверная(self):
        return 500 <= self.код < 600


try:
    raise HTTPError(404, "страница не найдена", url="/api/users")
except HTTPError as e:
    print(e)               # → HTTP 404: страница не найдена (url: /api/users)
    print(e.клиентская)    # → True
    print(repr(e))         # → HTTPError(код=404, сообщение='страница не найдена')
```

---

### Игнорирование ошибки

```python
# через pass:
try:
    опасный_код()
except SomeError:
    pass

# через contextlib.suppress — чище:
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("файл.txt")   # если файла нет — молча пропустит
```

---

### Паттерны применения

#### EAFP — проще просить прощения, чем разрешения

```python
# LBYL (Look Before You Leap) — проверить до:
if "ключ" in d:
    значение = d["ключ"]
else:
    значение = None

# EAFP (Easier to Ask Forgiveness than Permission) — попробовать и поймать:
try:
    значение = d["ключ"]
except KeyError:
    значение = None
```

Python-стиль — EAFP. Быстрее когда ошибка редкая, читабельнее при сложных проверках.

#### Логирование без остановки

```python
import logging

for элемент in данные:
    try:
        обработать(элемент)
    except Exception as e:
        logging.error(f"ошибка при обработке {элемент!r}: {e}", exc_info=True)
        continue   # продолжить цикл
```

`exc_info=True` — логирует полный Traceback.

#### Повтор при временной ошибке

```python
import time

def с_повтором(fn, попыток=3, задержка=1.0):
    последняя_ошибка = None
    for попытка in range(попыток):
        try:
            return fn()
        except (ConnectionError, TimeoutError) as e:
            последняя_ошибка = e
            print(f"попытка {попытка + 1}/{попыток} не удалась: {e}")
            time.sleep(задержка)
    raise последняя_ошибка   # пробросить после всех попыток

с_повтором(lambda: запрос_к_api(), попыток=3, задержка=2.0)
```

#### Валидация с накоплением ошибок

```python
class ValidationError(Exception):
    def __init__(self, ошибки: list):
        self.ошибки = ошибки
        super().__init__("; ".join(ошибки))

def валидировать_пользователя(данные: dict):
    ошибки = []
    if not данные.get("имя"):
        ошибки.append("имя обязательно")
    if not данные.get("email") or "@" not in данные["email"]:
        ошибки.append("некорректный email")
    возраст = данные.get("возраст")
    if not isinstance(возраст, int) or возраст < 0:
        ошибки.append("возраст должен быть неотрицательным числом")
    if ошибки:
        raise ValidationError(ошибки)

try:
    валидировать_пользователя({"имя": "", "email": "bad", "возраст": -1})
except ValidationError as e:
    for err in e.ошибки:
        print(f"• {err}")
# → • имя обязательно
# → • некорректный email
# → • возраст должен быть неотрицательным числом
```

#### Безопасное преобразование типов

```python
def в_int(значение, по_умолчанию=None):
    try:
        return int(значение)
    except (ValueError, TypeError):
        return по_умолчанию

в_int("42")      # → 42
в_int("abc")     # → None
в_int(None, 0)   # → 0
```

---

### Что не делать

```python
# ПЛОХО — проглотить всё молча:
try:
    код()
except:
    pass

# ПЛОХО — поймать Exception без логирования:
try:
    код()
except Exception:
    pass   # ошибка пропала, отладка невозможна

# ПЛОХО — конкретный тип после общего:
except Exception:
    ...
except ValueError:    # ← никогда не сработает, Exception выше
    ...

# ПЛОХО — возвращать None при ошибке без документации:
def получить(id):
    try:
        return данные[id]
    except KeyError:
        return None   # вызывающий код не знает: None = нет данных или ошибка?

# ХОРОШО — конкретное выше, логировать, пробрасывать если не умеешь обработать:
try:
    код()
except ValueError as e:
    обработать_конкретно(e)
except Exception as e:
    logging.error(e, exc_info=True)
    raise
```

---

### Сводная таблица конструкций

|Конструкция|Что делает|
|---|---|
|`try / except`|Перехватить исключение|
|`except (A, B)`|Перехватить несколько типов|
|`except A as e`|Получить объект исключения|
|`except Exception`|Поймать всё кроме системных|
|`else`|Выполнить если `try` прошёл без ошибок|
|`finally`|Выполнить всегда (очистка)|
|`raise Exc(...)`|Бросить исключение|
|`raise`|Перебросить текущее|
|`raise A from B`|Цепочка исключений с контекстом|
|`raise A from None`|Скрыть исходное исключение|
|`suppress(Exc)`|Игнорировать конкретный тип|

> О видах исключений, иерархии и магических методах — в [Ошибки Python](../Ядро%20языка/Ошибки%20Python.md).
