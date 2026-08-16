[← Оглавление](../../../README.md)

# argparse

> _Превратить `python script.py --count 5 файл.txt` в удобные значения — с проверкой типов и автоматической справкой._

---

**argparse** — модуль разбора **аргументов командной строки**. [`sys.argv`](../../../Библиотеки/Модули/Система%20и%20окружение/sys.md) даёт лишь сырой список строк; `argparse` описывает, какие аргументы бывают, проверяет их, приводит к нужному типу и сам генерирует справку `--help`. Так скрипт превращается в удобную консольную утилиту.

Зачем: любой скрипт, которым пользуются из терминала (обработчик файлов, инструмент развёртывания), должен принимать параметры — путь, флаги, числа.

## От sys.argv к argparse

```python
# вручную через sys.argv — хрупко: нет проверок, типов, справки
import sys
имя = sys.argv[1]        # а если не передали? IndexError

# argparse — декларативно:
import argparse

parser = argparse.ArgumentParser(description="Обработчик файлов")
parser.add_argument("файл")                        # позиционный (обязательный)
parser.add_argument("--count", type=int, default=1)  # опциональный с типом и дефолтом
parser.add_argument("--verbose", action="store_true")  # флаг: есть → True

args = parser.parse_args()
args.файл, args.count, args.verbose
```

## Позиционные vs опциональные

- **Позиционные** (`add_argument("файл")`) — обязательны, задаются порядком: `script.py данные.txt`;
- **Опциональные** (`add_argument("--count")`) — по имени, необязательны: `script.py --count 5`.

## Полезные параметры add_argument

```python
parser.add_argument("--mode",
    type=str,                        # привести к типу (int, float, str)
    default="fast",                  # значение, если не передано
    choices=["fast", "slow"],        # допустимые значения (иначе ошибка)
    required=True,                   # сделать опциональный обязательным
    help="режим работы",             # текст для --help
)
parser.add_argument("--verbose", action="store_true")  # флаг без значения → True/False
```

## Автоматический --help

`argparse` сам добавляет `-h/--help` и печатает справку из `description` и `help` каждого аргумента:

```
$ python script.py --help
usage: script.py [-h] [--count COUNT] [--verbose] файл

Обработчик файлов

positional arguments:
  файл

options:
  -h, --help     show this help message and exit
  --count COUNT
  --verbose
```

При неверных аргументах argparse сам печатает ошибку и завершает программу — не нужно проверять вручную.

## Связи

- [sys](../../../Библиотеки/Модули/Система%20и%20окружение/sys.md) — `sys.argv` как сырой источник, который `argparse` разбирает;
- Терминал — как аргументы и флаги передаются команде в оболочке;
- [Развертывание проекта](../../../DevOps/Развертывание%20проекта.md) — скрипты-утилиты с параметрами для автоматизации.

## Источники

- [Python Docs — argparse](https://docs.python.org/3/library/argparse.html)
- [Argparse Tutorial](https://docs.python.org/3/howto/argparse.html)
