[← Оглавление](../../README.md)

# Паттерны в практике — Lexi

> Lexi — WYSIWYG-редактор документов («что видишь, то и получаешь»). Восемь паттернов GoF применяются здесь к реальным задачам проектирования. Каждая задача — конкретный повод применить паттерн.

Связанные файлы: [Паттерны проектирования](../../ООП/Паттерны/Паттерны%20проектирования.md) · [Структурные паттерны](../../ООП/Паттерны/Структурные%20паттерны.md) · [Паттерны поведения](../../ООП/Паттерны/Паттерны%20поведения.md)

### Карта паттернов в Lexi

|Задача|Паттерн|
|---|---|
|Представление структуры документа|[Компоновщик — структура документа](#компоновщик--структура-документа)|
|Алгоритм форматирования строк|[Стратегия — алгоритм форматирования](#стратегия--алгоритм-форматирования)|
|Рамки и полосы прокрутки|[Декоратор — оформление UI](#декоратор--оформление-ui)|
|Стиль оформления (Motif / PM)|[Абстрактная фабрика — стиль оформления](#абстрактная-фабрика--стиль-оформления)|
|Оконные системы (X / Windows)|[Мост — оконная система](#мост--оконная-система)|
|Отмена и повтор операций|[Команда — операции и отмена](#команда--операции-и-отмена)|
|Обход структуры документа|[Итератор — обход структуры](#итератор--обход-структуры)|
|Проверка правописания|[Посетитель — анализ текста](#посетитель--анализ-текста)|

### Компоновщик — структура документа

Документ состоит из символов, строк, колонок, таблиц, рисунков. У всех — общие свойства: умеют рисовать себя и сообщать размеры. При этом группа элементов должна вести себя так же, как одиночный элемент.

**Решение — рекурсивная композиция:**

```
Страница
└── Колонка
    └── Строка
        ├── Символ 'H'
        ├── Символ 'i'
        └── Изображение logo.png
```

Все элементы — глифы (`Glyph`). Клиент работает с любым одинаково.

```python
from abc import ABC, abstractmethod

class Glyph(ABC):
    @abstractmethod
    def draw(self, window): ...

    @abstractmethod
    def bounds(self) -> tuple: ...

    # Операции с потомками — переопределяются в составных
    def add(self, glyph: "Glyph"): raise NotImplementedError
    def remove(self, glyph: "Glyph"): raise NotImplementedError
    def child(self, i: int) -> "Glyph": raise NotImplementedError

class Character(Glyph):
    """Лист — символ"""
    def __init__(self, char: str, font: str = "Arial"):
        self._char = char
        self._font = font

    def draw(self, window):
        window.draw_char(self._char, self._font)

    def bounds(self):
        return (8, 12)   # ширина и высота символа

class Image(Glyph):
    """Лист — изображение"""
    def __init__(self, path: str):
        self._path = path
        self._data = f"[данные {path}]"

    def draw(self, window):
        window.draw_image(self._data)

    def bounds(self):
        return (200, 150)

class Row(Glyph):
    """Составной — строка"""
    def __init__(self):
        self._children: list[Glyph] = []

    def add(self, glyph: Glyph):
        self._children.append(glyph)

    def remove(self, glyph: Glyph):
        self._children.remove(glyph)

    def child(self, i: int) -> Glyph:
        return self._children[i]

    def draw(self, window):
        for child in self._children:
            child.draw(window)   # рекурсивно — без проверки типа

    def bounds(self):
        ширина = sum(c.bounds()[0] for c in self._children)
        высота = max((c.bounds()[1] for c in self._children), default=0)
        return (ширина, высота)

class Column(Glyph):
    """Составной — колонка"""
    def __init__(self):
        self._rows: list[Glyph] = []

    def add(self, glyph: Glyph): self._rows.append(glyph)
    def remove(self, glyph: Glyph): self._rows.remove(glyph)
    def child(self, i): return self._rows[i]

    def draw(self, window):
        for row in self._rows:
            row.draw(window)

    def bounds(self):
        ширина = max((r.bounds()[0] for r in self._rows), default=0)
        высота = sum(r.bounds()[1] for r in self._rows)
        return (ширина, высота)

# Строим документ:
class FakeWindow:
    def draw_char(self, ch, font): print(f"[{font}] '{ch}'", end=" ")
    def draw_image(self, data): print(f"[IMG:{data}]", end=" ")

строка = Row()
for ch in "Hello":
    строка.add(Character(ch))
строка.add(Image("logo.png"))

колонка = Column()
колонка.add(строка)
колонка.draw(FakeWindow())
# → [Arial] 'H' [Arial] 'e' [Arial] 'l' [Arial] 'l' [Arial] 'o' [IMG:[данные logo.png]]
```

> **Паттерн:** [Структурные паттерны](../../ООП/Паттерны/Структурные%20паттерны.md#компоновщик) — единый интерфейс для листьев и составных объектов.

### Стратегия — алгоритм форматирования

Форматирование (разбиение текста на строки) — сложная задача с разными алгоритмами. Быстрый алгоритм даёт грубый результат, качественный — медленный. Алгоритм должен меняться без изменения документа.

```python
from abc import ABC, abstractmethod

class Compositor(ABC):
    """Абстрактная стратегия форматирования"""
    @abstractmethod
    def compose(self, glyphs: list, line_width: int) -> list[list]:
        """Возвращает список строк (каждая строка = список глифов)"""
        ...

class SimpleCompositor(Compositor):
    """Быстрый: просто заполняем строку до конца ширины"""
    def compose(self, glyphs, line_width):
        строки = []
        текущая = []
        ширина = 0
        for g in glyphs:
            w, _ = g.bounds()
            if ширина + w > line_width and текущая:
                строки.append(текущая)
                текущая = []
                ширина = 0
            текущая.append(g)
            ширина += w
        if текущая:
            строки.append(текущая)
        return строки

class TeXCompositor(Compositor):
    """Качественный: оптимизация по всему абзацу (минимум разрывов слов)"""
    def compose(self, glyphs, line_width):
        # Упрощённо — в реальном TeX алгоритм Кнута
        print(f"TeXCompositor: оптимальное разбиение {len(glyphs)} глифов")
        return SimpleCompositor().compose(glyphs, line_width)

class Composition(Glyph):
    """Контекст — использует стратегию форматирования"""
    def __init__(self, compositor: Compositor, line_width: int = 400):
        self._compositor = compositor
        self._line_width = line_width
        self._glyphs: list[Glyph] = []
        self._lines: list[list] = []

    def add_glyph(self, glyph: Glyph):
        self._glyphs.append(glyph)

    def repair(self):
        """Переформатировать — вызывает стратегию"""
        self._lines = self._compositor.compose(self._glyphs, self._line_width)

    def draw(self, window):
        for line in self._lines:
            for g in line:
                g.draw(window)

    def bounds(self):
        return (self._line_width, len(self._lines) * 14)

# Смена алгоритма без изменения Composition:
doc = Composition(SimpleCompositor(), line_width=100)
doc.add_glyph(Character("H"))
doc.repair()

# Переключить на качественный:
doc._compositor = TeXCompositor()
doc.repair()
```

> **Паттерн:** [Паттерны поведения](../../ООП/Паттерны/Паттерны%20поведения.md#стратегия) — алгоритм выносится в объект, взаимозаменяем в рантайме.

### Декоратор — оформление UI

Нужно добавить к области редактирования рамку и полосы прокрутки. Создавать `BorderedComposition`, `ScrollableComposition`, `BorderedScrollableComposition` — взрыв подклассов.

**Решение — прозрачное окружение.** Декоратор сам является глифом и содержит внутри другой глиф:

```python
class MonoGlyph(Glyph):
    """Абстрактный декоратор"""
    def __init__(self, component: Glyph):
        self._component = component

    def draw(self, window):
        self._component.draw(window)   # делегируем вложенному

    def bounds(self):
        return self._component.bounds()

class Border(MonoGlyph):
    def __init__(self, component: Glyph, ширина: int = 2):
        super().__init__(component)
        self._ширина = ширина

    def draw(self, window):
        super().draw(window)          # сначала содержимое
        self._draw_border(window)     # потом рамка

    def _draw_border(self, window):
        print(f"[рамка {self._ширина}px]")

    def bounds(self):
        w, h = self._component.bounds()
        return (w + self._ширина * 2, h + self._ширина * 2)

class Scroller(MonoGlyph):
    def draw(self, window):
        self._clip_to_visible(window)
        super().draw(window)

    def _clip_to_visible(self, window):
        print("[обрезка по видимой области]")

# Порядок обёртки меняет поведение:
doc = Composition(SimpleCompositor())

# Документ → прокрутка → рамка (рамка снаружи полосы прокрутки):
ui1 = Border(Scroller(doc), ширина=1)
ui1.draw(FakeWindow())
# → [обрезка по видимой области] → содержимое → [рамка 1px]

# Документ → рамка → прокрутка (рамка прокручивается вместе с текстом):
ui2 = Scroller(Border(doc, ширина=2))
ui2.draw(FakeWindow())
```

> **Паттерн:** [Структурные паттерны](../../ООП/Паттерны/Структурные%20паттерны.md#декоратор) — динамически добавляет обязанности без подклассов.

### Абстрактная фабрика — стиль оформления

Lexi должен работать в стиле Motif или Presentation Manager. Виджеты разные. Писать `MotifScrollBar()` напрямую — жёсткая привязка.

```python
class UIФабрика(ABC):
    """Абстрактная фабрика виджетов"""
    @abstractmethod
    def создать_скроллбар(self) -> Glyph: ...

    @abstractmethod
    def создать_кнопку(self) -> Glyph: ...

    @abstractmethod
    def создать_меню(self) -> Glyph: ...

class MotifФабрика(UIФабрика):
    def создать_скроллбар(self): return MotifScrollBar()
    def создать_кнопку(self): return MotifButton()
    def создать_меню(self): return MotifMenu()

class PMФабрика(UIФабрика):
    def создать_скроллбар(self): return PMScrollBar()
    def создать_кнопку(self): return PMButton()
    def создать_меню(self): return PMMenu()

class ДиалогЛеки:
    def __init__(self, фабрика: UIФабрика):
        # Создаём элементы через фабрику — имена конкретных классов нигде нет
        self._скроллбар = фабрика.создать_скроллбар()
        self._кнопка = фабрика.создать_кнопку()
        self._меню = фабрика.создать_меню()

# Смена стиля — одна строка:
диалог = ДиалогЛеки(MotifФабрика())
# или:
диалог = ДиалогЛеки(PMФабрика())
# Никакой другой код не меняется
```

> **Паттерн:** [Порождающие паттерны](../../ООП/Паттерны/Порождающие%20паттерны.md#абстрактная-фабрика) — создаёт семейства совместимых объектов.

### Мост — оконная система

Lexi должен работать в X Window, Windows, PM. Интерфейсы несовместимы. Решение — две независимые иерархии, связанные ссылкой.

```python
class WindowImp(ABC):
    """Реализация — платформенный API"""
    @abstractmethod
    def device_rect(self, x0, y0, x1, y1): ...

    @abstractmethod
    def device_text(self, text, x, y): ...

class XWindowImp(WindowImp):
    def device_rect(self, x0, y0, x1, y1):
        print(f"XDrawRectangle({x0},{y0},{x1},{y1})")

    def device_text(self, text, x, y):
        print(f"XDrawString('{text}',{x},{y})")

class WinAPIImp(WindowImp):
    def device_rect(self, x0, y0, x1, y1):
        print(f"DrawRect({x0},{y0},{x1-x0},{y1-y0})")   # WinAPI использует ширину/высоту

    def device_text(self, text, x, y):
        print(f"TextOut('{text}',{x},{y})")

class Window(ABC):
    """Абстракция — прикладной интерфейс"""
    def __init__(self, imp: WindowImp):
        self._imp = imp   # ← мост

    def draw_rect(self, x, y, w, h):
        self._imp.device_rect(x, y, x + w, y + h)   # транслируем вызов

    def draw_text(self, text, x, y):
        self._imp.device_text(text, x, y)

class ApplicationWindow(Window):
    def draw_contents(self):
        self.draw_rect(10, 10, 300, 200)
        self.draw_text("Документ", 20, 20)

class IconWindow(Window):
    def __init__(self, imp, bitmap_name: str):
        super().__init__(imp)
        self._bitmap = bitmap_name

    def draw_contents(self):
        # IconWindow не требует новой WindowImp
        self.draw_text(self._bitmap, 0, 0)

# Любое окно × любая платформа:
app_x   = ApplicationWindow(XWindowImp())
app_win = ApplicationWindow(WinAPIImp())
icon_x  = IconWindow(XWindowImp(), "folder.bmp")

app_x.draw_contents()   # → XDrawRectangle / XDrawString
app_win.draw_contents() # → DrawRect / TextOut
```

> **Паттерн:** [Структурные паттерны](../../ООП/Паттерны/Структурные%20паттерны.md#мост) — абстракция и реализация развиваются независимо.

### Команда — операции и отмена

Операции вызываются из меню, кнопок, горячих клавиш. Нужна поддержка отмены и повтора.

```python
from abc import ABC, abstractmethod

class Команда(ABC):
    @abstractmethod
    def execute(self): ...

    @abstractmethod
    def unexecute(self): ...

    def reversible(self) -> bool:
        return True   # большинство команд обратимы

class КомандаШрифт(Команда):
    def __init__(self, глифы: list[Glyph], новый_шрифт: str):
        self._глифы = глифы
        self._новый_шрифт = новый_шрифт
        self._старые_шрифты: list[str] = []

    def execute(self):
        for g in self._глифы:
            if isinstance(g, Character):
                self._старые_шрифты.append(g._font)
                g._font = self._новый_шрифт
            else:
                self._старые_шрифты.append(None)

    def unexecute(self):
        for g, шрифт in zip(self._глифы, self._старые_шрифты):
            if isinstance(g, Character) and шрифт:
                g._font = шрифт

class КомандаВставить(Команда):
    def __init__(self, состав: Composition, глиф: Glyph, позиция: int):
        self._состав = состав
        self._глиф = глиф
        self._позиция = позиция

    def execute(self):
        # вставляем глиф в позицию
        self._состав._glyphs.insert(self._позиция, self._глиф)
        self._состав.repair()

    def unexecute(self):
        self._состав._glyphs.pop(self._позиция)
        self._состав.repair()

class МенеджерКоманд:
    """Invoker с поддержкой undo/redo"""
    def __init__(self, уровней: int = 50):
        self._история: list[Команда] = []
        self._отменённые: list[Команда] = []
        self._уровней = уровней

    def выполнить(self, команда: Команда):
        команда.execute()
        self._история.append(команда)
        if len(self._история) > self._уровней:
            self._история.pop(0)
        self._отменённые.clear()

    def отменить(self):
        if not self._история:
            return
        команда = self._история.pop()
        if команда.reversible():
            команда.unexecute()
            self._отменённые.append(команда)

    def повторить(self):
        if not self._отменённые:
            return
        команда = self._отменённые.pop()
        команда.execute()
        self._история.append(команда)

# MenuItem не знает что делает команда — просто вызывает execute():
class MenuItem:
    def __init__(self, команда: Команда, менеджер: МенеджерКоманд):
        self._команда = команда
        self._менеджер = менеджер

    def кликнуть(self):
        self._менеджер.выполнить(self._команда)
```

> **Паттерн:** [Паттерны поведения](../../ООП/Паттерны/Паттерны%20поведения.md#команда) — запрос как объект, поддержка отмены.

### Итератор — обход структуры

Глифы хранят потомков по-разному: одни в списке, другие в дереве. Алгоритмы не должны зависеть от внутреннего устройства.

```python
from typing import Iterator as PyIterator

class PreorderIterator:
    """Обход в прямом порядке (корень → потомки)"""
    def __init__(self, корень: Glyph):
        self._стек: list[tuple[Glyph, int]] = [(корень, 0)]

    def __iter__(self):
        return self

    def __next__(self) -> Glyph:
        while self._стек:
            глиф, индекс = self._стек[-1]
            if индекс == 0:
                # Первый визит — возвращаем сам глиф
                self._стек[-1] = (глиф, 1)
                return глиф
            try:
                потомок = глиф.child(индекс - 1)
                self._стек[-1] = (глиф, индекс + 1)
                self._стек.append((потомок, 0))
            except (NotImplementedError, IndexError):
                self._стек.pop()
        raise StopIteration

# Использование — клиент не знает структуру:
def подсчитать_символы(корень: Glyph) -> int:
    return sum(1 for г in PreorderIterator(корень) if isinstance(г, Character))

строка = Row()
for ch in "Hello":
    строка.add(Character(ch))
строка.add(Image("photo.jpg"))

print(подсчитать_символы(строка))   # → 5

# Python-идиоматично — через генератор:
def все_глифы(корень: Glyph):
    yield корень
    try:
        i = 0
        while True:
            yield from все_глифы(корень.child(i))
            i += 1
    except (NotImplementedError, IndexError):
        pass
```

> **Паттерн:** [Паттерны поведения](../../ООП/Паттерны/Паттерны%20поведения.md#итератор) — доступ к элементам без раскрытия структуры.

### Посетитель — анализ текста

Нужна проверка правописания, подсчёт слов, расстановка переносов. Добавлять метод `анализировать()` в каждый подкласс Glyph — плохо.

```python
class GlyphVisitor(ABC):
    @abstractmethod
    def visit_character(self, char: Character): ...

    @abstractmethod
    def visit_image(self, image: Image): ...

    def visit_row(self, row: Row): pass      # необязательно для большинства

class SpellingVisitor(GlyphVisitor):
    def __init__(self):
        self._слово = ""
        self.ошибки: list[str] = []
        self._словарь = {"hello", "world", "python"}   # упрощённо

    def visit_character(self, char: Character):
        if char._char.isalpha():
            self._слово += char._char.lower()
        else:
            self._проверить()

    def visit_image(self, image: Image):
        self._проверить()   # изображение прерывает слово

    def _проверить(self):
        if self._слово and self._слово not in self._словарь:
            self.ошибки.append(self._слово)
        self._слово = ""

class WordCountVisitor(GlyphVisitor):
    def __init__(self):
        self.слов = 0
        self._в_слове = False

    def visit_character(self, char: Character):
        if char._char.isalpha():
            if not self._в_слове:
                self.слов += 1
                self._в_слове = True
        else:
            self._в_слове = False

    def visit_image(self, image: Image):
        self._в_слове = False

# Добавляем accept() в глифы:
class Character(Glyph):
    def __init__(self, char: str, font: str = "Arial"):
        self._char = char
        self._font = font

    def draw(self, window): window.draw_char(self._char, self._font)
    def bounds(self): return (8, 12)

    def accept(self, visitor: GlyphVisitor):
        visitor.visit_character(self)   # вызываем нужный метод посетителя

class Row(Glyph):
    def __init__(self): self._children = []
    def add(self, g): self._children.append(g)
    def remove(self, g): self._children.remove(g)
    def child(self, i): return self._children[i]
    def draw(self, w):
        for c in self._children: c.draw(w)
    def bounds(self): return (0, 0)

    def accept(self, visitor: GlyphVisitor):
        for child in self._children:
            if hasattr(child, "accept"):
                child.accept(visitor)   # рекурсивно
        visitor.visit_row(self)

# Новый анализ — новый посетитель, глифы не трогаем:
строка = Row()
for ch in "Hello World":
    строка.add(Character(ch))

счётчик = WordCountVisitor()
строка.accept(счётчик)
print(f"Слов: {счётчик.слов}")   # → 2

проверка = SpellingVisitor()
строка.accept(проверка)
print(f"Ошибок: {len(проверка.ошибки)}")
```

> **Паттерн:** [Паттерны поведения](../../ООП/Паттерны/Паттерны%20поведения.md#посетитель) — новые операции без изменения классов. Добавить анализ = добавить класс посетителя.

### Итог — восемь паттернов в одном приложении

```
Lexi
├── Документ (Glyph-дерево) ←─────── Компоновщик
│   ├── Row
│   │   ├── Character
│   │   └── Image
│   └── Column
│       └── Row...
├── Форматирование ←───────────────── Стратегия
│   ├── SimpleCompositor
│   └── TeXCompositor
├── UI-оформление ←────────────────── Декоратор
│   ├── Border(Scroller(Composition))
│   └── Scroller(Composition)
├── Виджеты стиля ←────────────────── Абстрактная фабрика
│   ├── MotifФабрика
│   └── PMФабрика
├── Платформа ←────────────────────── Мост
│   ├── Window ──→ XWindowImp
│   └── Window ──→ WinAPIImp
├── Операции ←─────────────────────── Команда
│   ├── КомандаШрифт
│   └── КомандаВставить
├── Обход ←────────────────────────── Итератор
│   └── PreorderIterator
└── Анализ ←───────────────────────── Посетитель
    ├── SpellingVisitor
    └── WordCountVisitor
```

Ни один паттерн не был «вставлен ради паттерна» — каждый решает конкретную задачу проектирования, которую иначе пришлось бы решать хуже.
