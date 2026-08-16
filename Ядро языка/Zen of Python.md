[← Оглавление](../README.md)

# Zen of Python

> **Zen of Python** (PEP 20) — 19 афоризмов о философии дизайна Python, написанных Тимом Питерсом. Это не правила языка, а **ценности сообщества**: что считать «хорошим» питоническим кодом. Спрятаны прямо в интерпретаторе — их печатает `import this`.

---

## import this

```python
>>> import this
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
```

## Ключевые принципы — что они значат на практике

**Explicit is better than implicit** (явное лучше неявного). Пусть из кода видно, что происходит, без скрытой магии. `from module import *` прячет, откуда имена, — избегай.

**Simple is better than complex** (простое лучше сложного) + **Readability counts** (читаемость важна). Код читают чаще, чем пишут. Понятный цикл лучше хитрого однострочника — это и есть [KISS](../ООП/Принципы/Принципы%20разработки%20—%20DRY,%20KISS,%20DI.md#kiss--keep-it-simple-stupid).

**There should be one obvious way to do it** (должен быть один очевидный способ). В отличие от Perl («много способов»), Python поощряет единый идиоматичный путь: перебор — `for x in items`, а не по индексам.

**Errors should never pass silently** (ошибки не должны замалчиваться). Не глотай исключения пустым `except: pass` — либо обработай, либо дай упасть. Подробнее — [Обработка ошибок Python](../Ядро%20языка/Обработка%20ошибок%20Python.md).

**Flat is better than nested** (плоское лучше вложенного). Глубокая вложенность `if`/циклов трудно читается — используй ранний `return`, `continue`, [функции](../Ядро%20языка/Функции%20Python.md).

**In the face of ambiguity, refuse the temptation to guess** (не угадывай в неоднозначности). Явные типы, явное поведение — не полагайся на «наверное сработает».

**Special cases aren't special enough to break the rules. Although practicality beats purity.** Держи единообразие, но не доводи до абсурда: практичность важнее чистоты, когда правило мешает делу.

## Как применять

Zen — это линза для code review своего кода: «а тут явно? просто? читаемо? нет ли проглоченной ошибки?». Не догма (`import antigravity` и «unless you're Dutch» — юмор Гвидо), но хороший ориентир, что в Python считается красивым.

> Связка с принципами разработки: Zen — про **дух** (что ценим), [SOLID](../ООП/Принципы/SOLID%20Python.md)/[DRY-KISS](../ООП/Принципы/Принципы%20разработки%20—%20DRY,%20KISS,%20DI.md) — про **конкретные приёмы**. PEP 20 задаёт настроение, [PEP 8](../Ядро%20языка/Стиль%20кода%20Python%20—%20PEP%208%20и%20Google%20Style%20Guide.md) — конкретные правила оформления.

## Связи

- [Принципы разработки — DRY, KISS, DI](../ООП/Принципы/Принципы%20разработки%20—%20DRY,%20KISS,%20DI.md) — KISS/DRY/YAGNI как конкретные приёмы того же духа
- [Стиль кода Python — PEP 8 и Google Style Guide](../Ядро%20языка/Стиль%20кода%20Python%20—%20PEP%208%20и%20Google%20Style%20Guide.md) — PEP 8: правила оформления (Zen — ценности, PEP 8 — техника)
- [Обработка ошибок Python](../Ядро%20языка/Обработка%20ошибок%20Python.md) — «errors should never pass silently» на практике
- [Python](../Python.md) — язык, чью философию описывает Zen

## Источники

- [PEP 20 — The Zen of Python](https://peps.python.org/pep-0020/)
- Печатается командой `import this` в любом интерпретаторе Python
