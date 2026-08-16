[← Оглавление](../README.md)

# Работа с форматами данных в Python

> О самих форматах — в Форматы данных. Здесь — как читать и писать их на Python.

---



## Контекстный менеджер и `with`

**Контекстный менеджер** — объект, который берёт на себя открытие и закрытие ресурса (файла, соединения, блокировки). Гарантирует, что ресурс будет закрыт **всегда** — даже если произошла ошибка.

Используется через конструкцию `with ... as`:

```python
with open("file.txt", "r", encoding="utf-8") as f:
    data = f.read()
# файл закрыт автоматически — даже если read() упало с ошибкой
```

Без `with` — ресурс может не закрыться:

```python
f = open("file.txt", "r")
data = f.read()   # если здесь исключение — f.close() не вызовется!
f.close()
```

### Как работает внутри

Контекстный менеджер реализует два [магических метода](../Ядро%20языка/Магические%20методы%20Python.md):

|Метод|Когда вызывается|
|---|---|
|`__enter__(self)`|При входе в блок `with` — возвращает объект для `as`|
|`__exit__(self, exc_type, exc_val, tb)`|При выходе — всегда, даже при исключении|

Если `__exit__` возвращает `True` — исключение подавляется. `False` (или `None`) — исключение пробрасывается дальше.

```python
class МойМенеджер:
    def __enter__(self):
        print("открываем")
        return self          # это попадёт в переменную after as

    def __exit__(self, exc_type, exc_val, tb):
        print("закрываем")
        return False         # не подавлять исключение

with МойМенеджер() as m:
    print("работаем")
# → открываем → работаем → закрываем
```

### contextlib.contextmanager — менеджер через генератор

Удобный способ создать контекстный менеджер без класса:

```python
from contextlib import contextmanager

@contextmanager
def временный_файл(path):
    f = open(path, "w")
    try:
        yield f          # всё до yield — __enter__, после — __exit__
    finally:
        f.close()

with временный_файл("tmp.txt") as f:
    f.write("данные")
```

### Несколько ресурсов сразу

```python
with open("input.txt", encoding="utf-8") as fin, \
     open("output.txt", "w", encoding="utf-8") as fout:
    fout.write(fin.read())
```

---

## Работа с текстовыми файлами

### Режимы открытия

|Режим|Описание|
|---|---|
|`"r"`|Чтение — файл должен существовать (по умолчанию)|
|`"w"`|Запись — создаёт файл или **перезаписывает** существующий|
|`"a"`|Дозапись — добавляет в конец, не трогает существующее|
|`"x"`|Создание — ошибка если файл уже существует|
|`"r+"`|Чтение и запись — файл должен существовать|
|`"w+"`|Чтение и запись — перезаписывает|
|`"rb"` / `"wb"`|Бинарное чтение / запись|
|`"ab"`|Бинарная дозапись|

---

### Методы чтения

**`f.read(size=-1)`** — читает весь файл как одну строку. Если передать `size` — читает не более `size` символов.

```python
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()        # весь файл одной строкой
    # "строка 1\nстрока 2\nстрока 3"

with open("data.txt", "r", encoding="utf-8") as f:
    chunk = f.read(100)       # первые 100 символов
```

**`f.readline()`** — читает одну строку (включая `\n` в конце). При повторном вызове — следующую. Возвращает `""` в конце файла.

```python
with open("data.txt", "r", encoding="utf-8") as f:
    line = f.readline()       # "строка 1\n"
    line = f.readline()       # "строка 2\n"
    line = f.readline()       # "" — конец файла
```

**`f.readlines()`** — читает все строки в список. Каждая строка сохраняет `\n`.

```python
with open("data.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()
    # ["строка 1\n", "строка 2\n", "строка 3"]
```

**Итерация по файлу** — самый экономный способ: не загружает всё в память сразу, читает по одной строке.

```python
with open("data.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())   # strip() убирает \n и пробелы по краям
```

**Сравнение методов чтения:**

|Метод|Возвращает|Память|Когда использовать|
|---|---|---|---|
|`read()`|`str` (весь файл)|Весь файл|Маленькие файлы|
|`read(n)`|`str` (n символов)|Минимум|Потоковая обработка|
|`readline()`|`str` (одна строка)|Одна строка|Построчно вручную|
|`readlines()`|`list[str]`|Весь файл|Нужен список строк|
|`for line in f`|`str` построчно|Одна строка|**Большие файлы**|

---

### Методы записи

**`f.write(s)`** — записывает строку `s`. Возвращает количество записанных символов. Перенос строки **не добавляется** автоматически.

```python
with open("out.txt", "w", encoding="utf-8") as f:
    f.write("строка 1\n")
    f.write("строка 2\n")
```

**`f.writelines(lines)`** — записывает список строк подряд. Разделители **не добавляются** — нужно включить `\n` в каждую строку самому.

```python
lines = ["строка 1\n", "строка 2\n", "строка 3\n"]

with open("out.txt", "w", encoding="utf-8") as f:
    f.writelines(lines)
```

**`print(..., file=f)`** — удобно записывать с автоматическим `\n`:

```python
with open("out.txt", "w", encoding="utf-8") as f:
    print("строка 1", file=f)
    print("строка 2", file=f)
```

**Дозапись** — режим `"a"` добавляет в конец, не трогая существующий контент:

```python
with open("log.txt", "a", encoding="utf-8") as f:
    f.write("новая запись лога\n")
```

---

### Позиция в файле

Файловый объект хранит курсор — текущую позицию чтения/записи.

**`f.tell()`** — возвращает текущую позицию (в байтах).

**`f.seek(pos, whence=0)`** — перемещает курсор. `whence`: `0` — от начала, `1` — от текущей, `2` — от конца.

```python
with open("data.txt", "r", encoding="utf-8") as f:
    f.read(10)
    print(f.tell())      # → 10

    f.seek(0)            # вернулись в начало
    print(f.read(5))     # читаем снова с начала
```

---

## CSV

Модуль `csv` из стандартной библиотеки.

### Чтение

**`csv.reader`** — читает строки как списки:

```python
import csv

with open("data.csv", "r", encoding="utf-8") as f:
    reader = csv.reader(f)
    header = next(reader)        # первая строка — заголовки
    for row in reader:
        print(row)               # ['Иван', '25', 'Москва']
```

**`csv.DictReader`** — каждая строка как словарь, ключи — из первой строки:

```python
with open("data.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])
```

Если заголовков нет — передать список имён вручную:

```python
reader = csv.DictReader(f, fieldnames=["name", "age", "city"])
```

Загрузить всё в список сразу:

```python
with open("data.csv", "r", encoding="utf-8") as f:
    rows = list(csv.reader(f))
```

### Запись

**`csv.writer`** — запись строк из списков:

```python
with open("out.csv", "w", encoding="utf-8", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "age", "city"])       # одна строка
    writer.writerow(["Иван", 25, "Москва"])
    writer.writerows([                             # сразу несколько
        ["Мария", 31, "Питер"],
        ["Алексей", 19, "Казань"],
    ])
```

**`csv.DictWriter`** — запись из словарей:

```python
поля = ["name", "age", "city"]

with open("out.csv", "w", encoding="utf-8", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=поля)
    writer.writeheader()                           # строка заголовков
    writer.writerow({"name": "Иван", "age": 25, "city": "Москва"})
    writer.writerows([
        {"name": "Мария", "age": 31, "city": "Питер"},
    ])
```

> `newline=""` обязателен при записи CSV на Windows — иначе появятся лишние пустые строки.

С нестандартным разделителем и кавычками:

```python
csv.reader(f, delimiter=";")
csv.writer(f, delimiter="\t", quotechar="'", quoting=csv.QUOTE_MINIMAL)
```

---

## JSON

Модуль `json` из стандартной библиотеки.

### Чтение

**`json.load(f)`** — читает JSON из файлового объекта:

```python
import json

with open("data.json", "r", encoding="utf-8") as f:
    data = json.load(f)
```

**`json.loads(s)`** — читает JSON из строки:

```python
text = '{"name": "Иван", "age": 25, "active": true}'
data = json.loads(text)
print(data["name"])    # → Иван
print(type(data))      # → <class 'dict'>
```

### Запись

**`json.dump(obj, f)`** — записывает объект Python в файл:

```python
data = {"name": "Иван", "scores": [88, 92, 75], "active": True}

with open("out.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

**`json.dumps(obj)`** — сериализует объект в строку:

```python
text = json.dumps(data, ensure_ascii=False, indent=2)
print(text)
```

### Параметры сериализации

|Параметр|Описание|
|---|---|
|`ensure_ascii=False`|Кириллица как есть (`И`, не `\u0418`)|
|`indent=2`|Красивый вывод с отступами в 2 пробела|
|`sort_keys=True`|Ключи словаря по алфавиту|
|`separators=(',', ':')`|Компактный вывод без пробелов|
|`default=func`|Функция для нестандартных типов (см. ниже)|

### Нестандартные типы

`datetime`, `set`, `Decimal` — JSON не поддерживает. Нужен `default`:

```python
from datetime import datetime

def конвертер(obj):
    if isinstance(obj, datetime):
        return obj.isoformat()
    if isinstance(obj, set):
        return list(obj)
    raise TypeError(f"Неизвестный тип: {type(obj)}")

data = {"дата": datetime.now(), "теги": {"python", "json"}}
print(json.dumps(data, default=конвертер, ensure_ascii=False))
# → {"дата": "2025-03-25T12:00:00", "теги": ["python", "json"]}
```

### Соответствие типов

| Python           | JSON             | Python (обратно) |
| ---------------- | ---------------- | ---------------- |
| `dict`           | `{}`             | `dict`           |
| `list`, `tuple`  | `[]`             | `list`           |
| `str`            | `"string"`       | `str`            |
| `int`            | `number`         | `int`            |
| `float`          | `number`         | `float`          |
| `True` / `False` | `true` / `false` | `bool`           |
| `None`           | `null`           | `None`           |

> `tuple` сериализуется в `[]`, но при чтении обратно становится `list`.

---

## YAML

Нужна сторонняя библиотека: `pip install pyyaml`.

### Чтение

**`yaml.safe_load(f)`** — читает один документ YAML из файла или строки:

```python
import yaml

with open("config.yml", "r", encoding="utf-8") as f:
    config = yaml.safe_load(f)

print(config["database"]["host"])    # → localhost
```

Из строки:

```python
text = "host: localhost\nport: 5432"
config = yaml.safe_load(text)
```

**`yaml.safe_load_all(f)`** — читает файл с несколькими документами (разделены `---`):

```python
with open("multi.yml", "r", encoding="utf-8") as f:
    docs = list(yaml.safe_load_all(f))
```

### Запись

**`yaml.dump(obj, f)`** — записывает объект в файл:

```python
config = {
    "database": {"host": "localhost", "port": 5432},
    "debug": True,
}

with open("out.yml", "w", encoding="utf-8") as f:
    yaml.dump(config, f, allow_unicode=True, default_flow_style=False)
```

В строку:

```python
text = yaml.dump(config, allow_unicode=True, default_flow_style=False)
```

|Параметр|Описание|
|---|---|
|`allow_unicode=True`|Кириллица как есть, не экранировать|
|`default_flow_style=False`|Блочный стиль (многострочный), а не `{key: val}`|
|`indent=2`|Размер отступа|
|`sort_keys=False`|Не сортировать ключи (сохранить порядок вставки)|

> **Всегда используй `safe_load`** вместо `yaml.load()` — `load()` может выполнить произвольный Python-код из файла.

---

## XML

Модуль `xml.etree.ElementTree` из стандартной библиотеки.

### Чтение

**`ET.parse(file)`** — читает XML из файла, возвращает объект дерева:

```python
import xml.etree.ElementTree as ET

tree = ET.parse("data.xml")
root = tree.getroot()

print(root.tag)          # имя тега: "users"
print(root.attrib)       # атрибуты: {"version": "1.0"}
print(root.text)         # текст внутри тега
```

**`ET.fromstring(s)`** — читает XML из строки:

```python
xml_str = "<user id='1'><name>Иван</name></user>"
root = ET.fromstring(xml_str)
```

Обход и поиск:

```python
for child in root:                      # прямые потомки
    print(child.tag, child.attrib)

root.find("user")                       # первый элемент с тегом "user"
root.findall("user")                    # все элементы с тегом "user"
root.findall(".//name")                 # все "name" на любой глубине (XPath)

elem = root.find("user")
print(elem.get("id"))                   # значение атрибута → "1"
print(elem.find("name").text)           # текст внутри тега → "Иван"
```

### Запись

```python
root = ET.Element("users")

user = ET.SubElement(root, "user")
user.set("id", "1")

name = ET.SubElement(user, "name")
name.text = "Иван"

tree = ET.ElementTree(root)
ET.indent(tree, space="  ")             # отступы (Python 3.9+)
tree.write("out.xml", encoding="utf-8", xml_declaration=True)
```

В строку:

```python
xml_str = ET.tostring(root, encoding="unicode")
```

> Для пространств имён, XPath и валидации — `lxml` (`pip install lxml`) удобнее стандартной библиотеки.

---

## pathlib — работа с путями

Современная замена `os.path`. `Path` — объект, а не строка.

```python
from pathlib import Path

p = Path("data/reports/file.csv")
```

### Атрибуты пути

```python
p.name           # → "file.csv"
p.stem           # → "file"
p.suffix         # → ".csv"
p.suffixes       # → [".tar", ".gz"]  — для "archive.tar.gz"
p.parent         # → Path("data/reports")
p.parts          # → ("data", "reports", "file.csv")
p.resolve()      # → абсолютный путь
p.is_absolute()  # → False
```

### Проверки

```python
p.exists()       # существует ли путь
p.is_file()      # это файл?
p.is_dir()       # это директория?
```

### Построение путей

```python
base = Path("data")
full = base / "reports" / "2025.csv"   # / объединяет части пути
# → Path("data/reports/2025.csv")
```

### Чтение и запись

```python
text  = p.read_text(encoding="utf-8")
p.write_text("содержимое", encoding="utf-8")   # перезаписывает целиком!

data  = p.read_bytes()
p.write_bytes(b"\x00\x01\x02")
```

> `write_text` и `write_bytes` — перезаписывают файл целиком. Для дозаписи нужен `open(p, "a")`.

### Работа с директориями

```python
Path("output/subdir").mkdir(parents=True, exist_ok=True)
# parents=True  — создать все промежуточные папки
# exist_ok=True — не падать если уже существует

p.rename(Path("data/new_name.csv"))    # переименовать / переместить
p.unlink()                             # удалить файл
p.unlink(missing_ok=True)             # удалить, не ругаться если нет
```

### Перебор файлов

```python
for f in Path("data").glob("*.csv"):       # csv в директории
    print(f)

for f in Path("data").rglob("*.json"):     # json рекурсивно
    print(f)

files = list(Path("data").iterdir())       # все файлы и папки
```

---

## Бинарные форматы

### pickle — сериализация Python-объектов

Сохраняет любой Python-объект в байты. Быстро, но только для Python.

```python
import pickle

obj = {"key": [1, 2, 3], "теги": {1, 2, 3}}

with open("data.pkl", "wb") as f:
    pickle.dump(obj, f)

with open("data.pkl", "rb") as f:
    obj = pickle.load(f)

# без файла — в байты и обратно
data = pickle.dumps(obj)
obj  = pickle.loads(data)
```

> **Никогда не загружай pickle из недоверенных источников** — при десериализации может выполниться произвольный код.

### struct — бинарная упаковка чисел

Упаковывает числа в байты по заданному формату. Нужно при работе с бинарными протоколами или файлами.

```python
import struct

data = struct.pack("if", 42, 3.14)   # 1 int + 1 float = 8 байт
n, x = struct.unpack("if", data)
print(n, x)    # → 42  3.14...
```

Основные коды формата:

|Код|Тип|Размер|
|---|---|---|
|`b` / `B`|signed / unsigned byte|1 байт|
|`h` / `H`|short|2 байта|
|`i` / `I`|int|4 байта|
|`q` / `Q`|long long|8 байт|
|`f` / `d`|float / double|4 / 8 байт|

Префикс задаёт порядок байт: `>` — big-endian, `<` — little-endian.

```python
struct.pack(">I", 256)   # → b'\x00\x00\x01\x00'
struct.pack("<I", 256)   # → b'\x00\x01\x00\x00'
```

---

## Сводная таблица методов

|Формат|Модуль|Читать из файла|Читать из строки|Записать в файл|Записать в строку|
|---|---|---|---|---|---|
|**TXT**|встроенный|`read()` / `readline()` / `readlines()`|—|`write()` / `writelines()`|—|
|**CSV**|`csv`|`csv.reader` / `DictReader`|—|`csv.writer` / `DictWriter`|—|
|**JSON**|`json`|`json.load(f)`|`json.loads(s)`|`json.dump(obj, f)`|`json.dumps(obj)`|
|**YAML**|`pyyaml`|`yaml.safe_load(f)`|`yaml.safe_load(s)`|`yaml.dump(obj, f)`|`yaml.dump(obj)`|
|**XML**|`xml.etree`|`ET.parse(file)`|`ET.fromstring(s)`|`tree.write(file)`|`ET.tostring(elem)`|
|**Pickle**|`pickle`|`pickle.load(f)`|`pickle.loads(b)`|`pickle.dump(obj, f)`|`pickle.dumps(obj)`|
