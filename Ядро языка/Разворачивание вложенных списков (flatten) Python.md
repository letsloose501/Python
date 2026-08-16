[← Оглавление](../README.md)

# Разворачивание вложенных списков (flatten) Python

> **Flatten** — превратить вложенную структуру любой глубины (`[1, [2, [3]]]`) в один плоский поток значений (`1, 2, 3`). Классическая задача на понимание [генераторов](../Ядро%20языка/Циклы%20Python.md#генераторы), [протокола итератора](../Ядро%20языка/Магические%20методы%20Python.md#контейнеры-и-последовательности) и разницы между рекурсией и итерацией.

---

Данные из API, парсеров, деревьев часто приходят вложенными непредсказуемо: список списков, внутри — ещё списки. Чтобы обойти все значения одним циклом, структуру «сплющивают». Сложность не в самой идее, а в **глубине**: наивное рекурсивное решение красиво, но упирается в лимит рекурсии. Правильное решение — итеративное, и оно раскрывает, как под капотом устроен `for`.

## Уровень 1 — вложенность ровно на один слой

Если структура плоская на один уровень (`[[1, 2], [3, 4]]`), разворачивание тривиально. Два идиоматичных способа:

```python
вложенный = [[1, 2], [3, 4], [5, 6]]

# через генераторное включение (два for подряд):
[x for подсписок in вложенный for x in подсписок]   # → [1, 2, 3, 4, 5, 6]

# через itertools.chain — лениво, без промежуточного списка:
import itertools
list(itertools.chain.from_iterable(вложенный))      # → [1, 2, 3, 4, 5, 6]
```

`chain.from_iterable` берёт итерируемое из итерируемых и выдаёт элементы подряд. Но оба способа знают только про **один** уровень — на `[[1, [2]]]` они вернут `[1, [2]]`, не спустившись глубже.

## Свой flatten на один слой — генератор и итератор вручную

Прежде чем идти в глубину, полезно написать разворачивание на один слой самому — так видно, из чего `for` собран. Два способа: генератор (`yield`) и класс-итератор (`__iter__`/`__next__`).

**Генератор** — два вложенных `for`, состояние ведёт сам язык:

```python
def flat_generator(list_of_lists):
    for sublist in list_of_lists:
        for item in sublist:
            yield item

data = [["a", "b", "c"], ["d", "e", "f", "h", False], [1, 2, None]]
list(flat_generator(data))   # → ['a', 'b', 'c', 'd', 'e', 'f', 'h', False, 1, 2, None]
```

Тот же генератор, но **на ручных курсорах** (без `for`) — чтобы увидеть, что происходит под `for`: два индекса, внешний и внутренний, двигаются вручную:

```python
def flat_generator(list_of_lists):
    outer_cursor = 0
    while outer_cursor < len(list_of_lists):
        inner_cursor = 0
        while inner_cursor < len(list_of_lists[outer_cursor]):
            yield list_of_lists[outer_cursor][inner_cursor]
            inner_cursor += 1
        outer_cursor += 1
```

**Класс-итератор** — то же поведение без `yield`: состояние (курсоры) хранится в полях объекта, `__next__` вызывается на каждом шаге `for` ([протокол итератора](../Ядро%20языка/Магические%20методы%20Python.md#контейнеры-и-последовательности)):

```python
class FlatIterator:
    def __init__(self, list_of_lists: list):
        self.list_of_lists = list_of_lists

    def __iter__(self):
        self.outer_cursor = 0       # сброс состояния при старте итерации
        self.inner_cursor = 0
        return self

    def __next__(self):
        while self.outer_cursor < len(self.list_of_lists):
            if self.inner_cursor < len(self.list_of_lists[self.outer_cursor]):
                item = self.list_of_lists[self.outer_cursor][self.inner_cursor]
                self.inner_cursor += 1
                return item
            # внутренний список кончился — к следующему внешнему
            self.outer_cursor += 1
            self.inner_cursor = 0
        raise StopIteration          # внешний список кончился — конец итерации

list(FlatIterator(data)) == list(flat_generator(data))   # → True
```

> Генератор и класс-итератор здесь **эквивалентны** по поведению — но генератор в разы короче, потому что `yield` сам сохраняет состояние между вызовами. Класс нужен, когда состояние сложнее курсоров или нужен дополнительный API.

## Уровень 2 — произвольная глубина через рекурсию

Чтобы развернуть любую глубину, нужно на каждый вложенный список **зайти внутрь**. Самое короткое решение — рекурсивный генератор с `yield from`:

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)   # item — список: рекурсивно спускаемся
        else:
            yield item                 # листовое значение — отдаём наружу

list(flatten([[["a"], ["b", "c"]], ["d", [["f"], "h"]], [1, [[["!"]]]]]))
# → ['a', 'b', 'c', 'd', 'f', 'h', 1, '!']
```

**`yield from it`** — делегирование: отдаёт все значения из `it` наружу, эквивалент `for x in it: yield x`. Читается идеально. Но у решения есть скрытый потолок.

## Почему рекурсия падает на большой глубине

Каждый уровень вложенности — это **новый кадр на стеке вызовов**. CPython ограничивает глубину рекурсии, по умолчанию ~1000 кадров ([предел рекурсии](../Ядро%20языка/Функции%20Python.md#предел-рекурсии)):

```python
import sys
sys.getrecursionlimit()     # → 1000

# структура, вложенная сама в себя 5000 раз:
huge = "deep"
for _ in range(5000):
    huge = [huge]           # → [[[[...["deep"]...]]]]

list(flatten(huge))         # → RecursionError: maximum recursion depth exceeded
```

Поднять лимит (`sys.setrecursionlimit`) — не решение: упрёшься в переполнение стека C и краш интерпретатора. Нужен подход, где глубина не тратит стек вызовов.

## Уровень 3 — итеративно, через стек итераторов

Идея: заменить рекурсию **явным стеком итераторов**. Держим список открытых итераторов; берём элемент с вершины, если это список — кладём его итератор наверх и спускаемся, если итератор исчерпан — снимаем его. Стек живёт в куче, а не на стеке вызовов, поэтому лимита рекурсии нет.

Так же, как [`for` внутри работает через `iter()`/`next()`/`StopIteration`](../Ядро%20языка/Циклы%20Python.md#как-работает-внутри), здесь мы вызываем протокол вручную:

```python
class FlatIterator:
    def __init__(self, list_of_lists):
        self.list_of_lists = list_of_lists

    def __iter__(self):
        self.stack = [iter(self.list_of_lists)]   # стек итераторов
        return self

    def __next__(self):
        while self.stack:
            try:
                item = next(self.stack[-1])        # элемент с вершины
            except StopIteration:
                self.stack.pop()                   # итератор исчерпан — снять
                continue
            if isinstance(item, list):
                self.stack.append(iter(item))      # спуститься внутрь
            else:
                return item                        # листовое значение
        raise StopIteration                        # стек пуст — конец
```

То же самое, но короче — как **генератор** (сам ведёт своё состояние, не нужен класс):

```python
def flatten(nested):
    stack = [iter(nested)]
    while stack:
        try:
            item = next(stack[-1])
        except StopIteration:
            stack.pop()
            continue
        if isinstance(item, list):
            stack.append(iter(item))
        else:
            yield item

huge = "deep"
for _ in range(100_000):
    huge = [huge]
list(flatten(huge))         # → ['deep'] — глубина 100 000, без RecursionError
```

## Уровень 4 — production-версия

Чтобы функция годилась как библиотечная, добавляют два краевых случая:

```python
from collections.abc import Iterable, Iterator
from typing import Any

def flatten(nested: Iterable[Any]) -> Iterator[Any]:
    stack: list[Iterator[Any]] = [iter(nested)]
    while stack:
        try:
            item = next(stack[-1])
        except StopIteration:
            stack.pop()
            continue
        # 1. str/bytes — атомарны, не разбиваем на символы
        # 2. проверяем Iterable, а не только list → кортежи, множества, генераторы
        if isinstance(item, Iterable) and not isinstance(item, (str, bytes)):
            stack.append(iter(item))
        else:
            yield item

list(flatten(["ab", ["cd", ["ef"]]]))          # → ['ab', 'cd', 'ef'] — строки целы
list(flatten([1, (2, 3), (x for x in [4, 5])])) # → [1, 2, 3, 4, 5] — любой iterable
```

**Почему `str`/`bytes` — особый случай:** строка сама итерируема, и если её не считать атомарной, `"abc"` рассыплется на `"a", "b", "c"`. Хуже: строка из одного символа `"a"` итерируется в саму себя — бесконечный спуск.

## Сравнение подходов

| Подход | Глубина | Ленивый | Строки атомарны | Годится когда |
|---|---|---|---|---|
| `chain` / включение | только 1 уровень | да / нет | — | заранее известно: ровно один слой |
| Рекурсивный `yield from` | до ~1000 | да | нет (только list) | читаемость важнее, глубина мала |
| Итеративный стек | любая (память) | да | опционально | данные с непредсказуемой/большой глубиной |

> Рекурсивное решение короче и понятнее — бери его для мелкой известной вложенности. Как только глубина может быть большой или неизвестной — только итеративный стек.

## Ловушки

```python
# НЕПРАВИЛЬНО — проверять на что спускаться неполно:
if isinstance(item, list):          # кортеж (2, 3) не развернётся — уйдёт как есть
    ...

# НЕПРАВИЛЬНО — считать строку «просто итерируемой»:
if isinstance(item, Iterable):      # "abc" рассыплется на буквы,
    stack.append(iter(item))        # а "a" даст бесконечную рекурсию

# ПРАВИЛЬНО — Iterable, но str/bytes исключены:
if isinstance(item, Iterable) and not isinstance(item, (str, bytes)):
    stack.append(iter(item))
```

## Связи

- [Циклы Python](../Ядро%20языка/Циклы%20Python.md#генераторы) — `yield`, `yield from`, генераторные выражения; там же flatten на 1 уровень
- [Магические методы Python](../Ядро%20языка/Магические%20методы%20Python.md#контейнеры-и-последовательности) — `__iter__` / `__next__` / `StopIteration`, на которых держится стековый обход
- [Функции Python](../Ядро%20языка/Функции%20Python.md#рекурсия) — предел рекурсии и почему хвостовая рекурсия в Python не оптимизируется
- [Составные типы данных Python](../Данные/Составные%20типы%20данных%20Python.md#list--список) — что разворачиваем
- [itertools](../Библиотеки/Модули/Функциональные%20инструменты/itertools.md) — `chain`, `chain.from_iterable` для плоского случая
- [Стек](../Данные/АТД/Стек.md) — АТД, лежащий в основе итеративного решения

## Источники

- [Python Docs — Iterator Types (`__iter__`/`__next__`/`StopIteration`)](https://docs.python.org/3/library/stdtypes.html#iterator-types)
- [Python Docs — itertools.chain / chain.from_iterable](https://docs.python.org/3/library/itertools.html#itertools.chain)
- [PEP 380 — Syntax for Delegating to a Subgenerator (`yield from`)](https://peps.python.org/pep-0380/)
- [Python Docs — sys.setrecursionlimit](https://docs.python.org/3/library/sys.html#sys.setrecursionlimit)
