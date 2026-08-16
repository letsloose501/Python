[← Оглавление](../README.md)

# aiogram

> _Телеграм-бот — это просто программа, которая читает входящие события и отвечает на них_

---

**aiogram** — асинхронный фреймворк для написания Telegram-ботов на [Python](../Python.md). Он оборачивает **Telegram Bot API** (HTTP-интерфейс Телеграма для ботов) в удобные объекты и берёт на себя рутину: получает новые сообщения, разбирает их в типизированные объекты, находит нужный обработчик и вызывает его. Ты пишешь только логику «на такое событие ответь так», всё остальное фреймворк делает сам. Построен на [asyncio](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md), поэтому один процесс держит тысячи одновременных диалогов.

> Здесь описан **aiogram 3.x** (актуальная линейка, 3.29+). Версия 2.x устроена иначе и **несовместима** — половина старых туториалов вводит в заблуждение (см. [aiogram 3 против 2.x](#aiogram-3-против-2x)).

## Как работает бот: события и два способа их получать

Бот не «слушает» Телеграм напрямую. Всё, что происходит с ботом (пришло сообщение, нажали кнопку, зашли в чат), Телеграм упаковывает в **Update** — объект-событие. Задача бота — забирать эти апдейты и реагировать. Забрать их можно двумя способами:

| | **Long polling (поллинг)** | **Webhook (вебхук)** |
|---|---|---|
| Кто инициирует | бот сам дёргает `getUpdates` у Телеграма | Телеграм сам шлёт POST на твой URL |
| Что нужно | ничего, работает откуда угодно | публичный HTTPS-адрес |
| Когда брать | разработка, небольшие боты | прод, высокая нагрузка |
| В aiogram | `dp.start_polling(bot)` | сервер на [aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) |

**Поллинг** — «звоню и спрашиваю: новости есть?» в бесконечном цикле; **вебхук** — «Телеграм сам постучится, когда событие случится» (та же механика, что и у [обычных вебхуков](../Библиотеки/Сторонние/Работа%20с%20WEB/Вебхуки%20%28webhooks%29.md)). Начинают почти всегда с поллинга — он не требует ни домена, ни сертификата.

## Минимальный бот

Классический эхо-бот: на `/start` здоровается, на любое сообщение — повторяет его.

```python
import asyncio
from aiogram import Bot, Dispatcher
from aiogram.filters import CommandStart
from aiogram.types import Message

dp = Dispatcher()                         # корневой роутер — сюда вешаем обработчики

@dp.message(CommandStart())               # фильтр: сработает только на /start
async def start(message: Message):
    await message.answer(f"Привет, {message.from_user.full_name}!")

@dp.message()                             # без фильтра — ловит всё ОСТАЛЬНОЕ
async def echo(message: Message):
    await message.answer(message.text)

async def main():
    bot = Bot(token="123456:ABC-DEF...")  # токен от @BotFather
    await dp.start_polling(bot)            # запустить цикл получения апдейтов

asyncio.run(main())                       # точка входа: создать event loop
```

- **`Bot`** — клиент к Telegram Bot API: через него отправляются ответы (`bot.send_message`, `message.answer` внутри зовёт его).
- **`Dispatcher`** (`dp`) — диспетчер: получает апдейты и раздаёт их обработчикам по фильтрам.
- **`@dp.message(...)`** — регистрирует **обработчик (handler)** сообщений; аргумент — фильтр.
- **`message.answer(...)`** — ответить в тот же чат (`message.reply(...)` — с цитированием исходного).

> **Токен — это пароль бота.** Не хардкодь его в коде и не коммить в git — читай из переменной окружения (`os.getenv("BOT_TOKEN")`) или конфига. Утёк токен — чужой получит полный контроль над ботом.

## Порядок обработчиков решает всё

Диспетчер проверяет обработчики **сверху вниз** и отдаёт апдейт **первому**, чей фильтр подошёл; остальные не вызываются. Поэтому общий обработчик-«ловушка» без фильтра должен идти **последним** — иначе он перехватит всё, и специфичные до него не дойдут.

```python
# НЕПРАВИЛЬНО — эхо стоит первым и съедает всё, /help недостижим:
@dp.message()
async def echo(message): ...
@dp.message(Command("help"))              # никогда не сработает
async def help_(message): ...

# ПРАВИЛЬНО — от частного к общему, ловушка в конце:
@dp.message(Command("help"))              # сначала конкретные команды
async def help_(message): ...
@dp.message()                             # эхо-ловушка — последней
async def echo(message): ...
```

## Архитектура: Bot, Dispatcher, Router

В маленьком боте всё висит на `dp`. Но код растёт, и держать сотню обработчиков в одном файле нельзя. Здесь появляется **Router** — группа обработчиков, которую можно вынести в отдельный файл и подключить к диспетчеру.

```
        ┌──────── Dispatcher (корневой роутер) ────────┐
        │  include_router(...)                          │
        │     ├── user_router      (handlers/user.py)   │
        │     ├── admin_router     (handlers/admin.py)  │
        │     └── payments_router  (handlers/pay.py)    │
        └───────────────────────────────────────────────┘
```

- **`Dispatcher`** — корневой роутер и «точка входа» апдейтов; он один на приложение.
- **`Router`** — самостоятельный набор обработчиков; их регистрируют на роутере, а роутер подключают к диспетчеру.

```python
# handlers/user.py
from aiogram import Router
from aiogram.filters import CommandStart
from aiogram.types import Message

router = Router()                         # свой роутер модуля

@router.message(CommandStart())
async def start(message: Message):
    await message.answer("Привет!")

# main.py
from handlers import user
dp = Dispatcher()
dp.include_router(user.router)            # подключаем модуль к диспетчеру
```

> Регистрируй обработчики **на роутерах, а не прямо на `dp`** — так модули не зависят друг от друга и не тянут циклических импортов. Диспетчер лишь собирает роутеры воедино.

## Виды апдейтов

`message` — не единственное событие. У роутера есть отдельный декоратор под каждый тип апдейта:

| Декоратор | Событие | Объект в обработчике |
|---|---|---|
| `@router.message()` | входящее сообщение | `Message` |
| `@router.callback_query()` | нажатие inline-кнопки | `CallbackQuery` |
| `@router.inline_query()` | inline-режим (`@bot запрос`) | `InlineQuery` |
| `@router.edited_message()` | сообщение отредактировали | `Message` |
| `@router.chat_member()` | изменение участника чата | `ChatMemberUpdated` |
| `@router.poll_answer()` | ответ в опросе | `PollAnswer` |

Самые ходовые — `message` (пользователь пишет) и `callback_query` (пользователь жмёт inline-кнопку).

## Фильтры — кому достанется апдейт

**Фильтр** — условие, при котором обработчик срабатывает. Их два вида: готовые классы-фильтры и «магический» фильтр `F`.

### Готовые фильтры

```python
from aiogram.filters import Command, CommandStart, CommandObject

@router.message(CommandStart())           # ровно /start
@router.message(Command("help"))          # /help (и /help@botname)
@router.message(Command("weather"))
async def weather(message: Message, command: CommandObject):
    city = command.args                   # аргументы после команды: "/weather Москва" → "Москва"
```

**`CommandObject`** приезжает в обработчик автоматически, если апдейт прошёл через `Command` — в нём разобранные аргументы команды.

### Магический фильтр `F`

**`F`** (magic filter) — способ фильтровать по **любому полю** апдейта коротким выражением, без отдельного класса. `F` — это «заготовка объекта», к которой обращаешься как к будущему сообщению:

```python
from aiogram import F

@router.message(F.text == "привет")       # текст РОВНО "привет"
@router.message(F.text.lower() == "привет")  # с приведением к нижнему регистру
@router.message(F.text.startswith("/"))   # начинается с /
@router.message(F.photo)                  # сообщение содержит фото
@router.message(F.content_type == "document")  # документ
@router.callback_query(F.data == "yes")   # inline-кнопка с callback_data="yes"
```

`F.text` означает «поле `text` пришедшего сообщения», а `== "привет"`, `.startswith(...)` — условие на него. В aiogram 2.x для этого были аргументы вроде `content_types=` и `text=` — в 3.x их заменил единый `F`.

### Комбинирование фильтров

```python
# И (AND) — перечисли через запятую, нужны ВСЕ:
@router.message(Command("buy"), F.chat.type == "private")   # /buy только в личке

# ИЛИ (OR) — оператор | или or_f:
@router.message(F.text == "да" | F.text == "yes")

# НЕ (NOT) — тильда ~:
@router.message(~F.text.startswith("/"))                    # всё, что НЕ команда
```

## Клавиатуры: reply и inline

Кнопки бывают двух принципиально разных видов:

- **Reply-клавиатура** — заменяет системную клавиатуру пользователя; нажатие **отправляет обычное сообщение** с текстом кнопки. Бот ловит его как `message`.
- **Inline-клавиатура** — кнопки **под сообщением**; нажатие шлёт не текст, а скрытый `callback_data`. Бот ловит его как `callback_query`.

Собирать клавиатуры удобнее **билдерами** — они сами раскладывают кнопки по рядам:

```python
from aiogram.utils.keyboard import ReplyKeyboardBuilder, InlineKeyboardBuilder

# Reply-клавиатура
kb = ReplyKeyboardBuilder()
kb.button(text="🍏 Меню")
kb.button(text="📞 Контакты")
kb.adjust(2)                              # 2 кнопки в ряд
await message.answer("Выбери:", reply_markup=kb.as_markup(resize_keyboard=True))

# Inline-клавиатура
ikb = InlineKeyboardBuilder()
ikb.button(text="Да",  callback_data="vote:yes")   # callback_data — скрытая метка
ikb.button(text="Нет", callback_data="vote:no")
ikb.adjust(2)
await message.answer("Голосуй:", reply_markup=ikb.as_markup())
```

- **`.button(...)`** — добавляет кнопку (у inline обязателен `callback_data`).
- **`.adjust(2, 1)`** — раскладка по рядам: 2 кнопки, затем 1.
- **`.as_markup()`** — превращает билдер в готовую разметку для `reply_markup`.

Нажатие inline-кнопки ловится обработчиком `callback_query`:

```python
@router.callback_query(F.data.startswith("vote:"))
async def on_vote(callback: CallbackQuery):
    choice = callback.data.split(":")[1]        # "yes" или "no"
    await callback.message.answer(f"Ты выбрал: {choice}")
    await callback.answer()                     # ← ОБЯЗАТЕЛЬНО: убрать «часики» на кнопке
```

> **Всегда зови `callback.answer()`** в конце обработчика inline-кнопки. Иначе у пользователя крутится вечная «загрузка» на кнопке — Телеграм ждёт подтверждения, что нажатие принято. Внутрь можно передать текст — выскочит всплывающее уведомление.

## FSM — многошаговые диалоги

Часто данные нужно собрать **по шагам**: спросить имя → потом возраст → потом подтверждение. Городить это на флагах в переменных — путь к хаосу. Для этого есть **FSM (finite state machine, конечный автомат)** — механизм, который запоминает, на каком шаге сейчас находится **каждый пользователь**, и направляет его сообщение в нужный обработчик.

Шаги описывают классом-наследником `StatesGroup`, каждый шаг — `State`:

```python
from aiogram.fsm.state import StatesGroup, State
from aiogram.fsm.context import FSMContext

class Form(StatesGroup):
    name = State()                        # шаг 1: ждём имя
    age = State()                         # шаг 2: ждём возраст

@router.message(CommandStart())
async def start(message: Message, state: FSMContext):
    await state.set_state(Form.name)      # перевели пользователя в состояние «жду имя»
    await message.answer("Как тебя зовут?")

@router.message(Form.name)                # сработает ТОЛЬКО если пользователь в состоянии Form.name
async def got_name(message: Message, state: FSMContext):
    await state.update_data(name=message.text)   # сохранили введённое
    await state.set_state(Form.age)              # следующий шаг
    await message.answer("Сколько тебе лет?")

@router.message(Form.age)
async def got_age(message: Message, state: FSMContext):
    data = await state.update_data(age=message.text)
    await state.clear()                          # диалог окончен — сбросить состояние
    await message.answer(f"Готово: {data['name']}, {data['age']} лет")
```

- **`FSMContext`** (`state`) — приезжает в обработчик автоматически; через него читают и меняют состояние и данные.
- **`state.set_state(Form.age)`** — перевести пользователя на шаг; фильтр `@router.message(Form.age)` ловит именно этот шаг.
- **`state.update_data(...)` / `state.get_data()`** — сохранить/прочитать данные диалога (общий словарь на пользователя).
- **`state.clear()`** — завершить диалог, стереть состояние и данные.

### Storage — где живёт состояние

Состояние надо где-то хранить. Хранилище задают **при создании диспетчера**:

```python
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.storage.redis import RedisStorage

dp = Dispatcher(storage=MemoryStorage())          # в оперативке — для разработки
dp = Dispatcher(storage=RedisStorage.from_url("redis://localhost:6379"))  # для прода
```

- **`MemoryStorage`** — в памяти процесса. Просто, но при перезапуске бота **все диалоги теряются**, и не работает при нескольких воркерах.
- **`RedisStorage`** — состояние в [Redis](../Библиотеки/Сторонние/Redis.md): переживает перезапуск и общее для всех воркеров. Стандарт для прода.

## Middleware — общий слой вокруг обработчиков

**Middleware (промежуточный слой)** — код, который выполняется **вокруг каждого** обработчика: до и после. Через него удобно делать сквозные вещи — логирование, троттлинг, подсовывание сессии БД, проверку подписки. Идея та же, что у [middleware в FastAPI](../Фреймворки/FastAPI.md#cors-и-middleware) и [aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#сервер).

```python
from aiogram import BaseMiddleware
from aiogram.types import TelegramObject

class LoggingMiddleware(BaseMiddleware):
    async def __call__(self, handler, event: TelegramObject, data: dict):
        print(f"Апдейт: {event}")         # ДО обработчика
        result = await handler(event, data)   # вызвать сам обработчик
        print("Обработано")               # ПОСЛЕ обработчика
        return result

router.message.middleware(LoggingMiddleware())    # повесить на все message этого роутера
```

Всё, что middleware кладёт в `data`, обработчик может принять как аргумент по имени ключа — так в него прокидывают, например, открытую сессию БД или объект пользователя из базы.

## База данных: aiogram + async SQLAlchemy

Бот почти всегда должен что-то **помнить** между сообщениями: кто уже зарегистрирован, настройки, заказы. Для этого нужна база — и подключают её через связку с асинхронной [SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md) (async, потому что обработчики — корутины, а синхронный драйвер заморозит event loop).

Наивная мысль — открыть одну сессию БД и пользоваться ею везде. Так делать **нельзя**: `AsyncSession` не рассчитана на параллельное использование, а бот обрабатывает много апдейтов разом. Правильно — **своя сессия на каждый апдейт**, и раздаёт её [middleware](#middleware--общий-слой-вокруг-обработчиков) через тот самый механизм внедрения зависимостей.

**Шаг 1. Модель и фабрика сессий** — движок и `sessionmaker` создают **один раз** на всё приложение:

```python
from sqlalchemy import BigInteger, String
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase): ...

class User(Base):
    __tablename__ = "users"
    id:   Mapped[int] = mapped_column(BigInteger, primary_key=True)   # telegram user id
    name: Mapped[str] = mapped_column(String)

engine = create_async_engine("postgresql+asyncpg://app:secret@localhost/app")
Sessionmaker = async_sessionmaker(engine, expire_on_commit=False)     # фабрика сессий
```

**Шаг 2. Middleware** открывает сессию на каждый апдейт и кладёт её в `data`:

```python
from aiogram import BaseMiddleware

class DbSessionMiddleware(BaseMiddleware):
    def __init__(self, sessionmaker):
        self.sessionmaker = sessionmaker

    async def __call__(self, handler, event, data):
        async with self.sessionmaker() as session:   # СВОЯ сессия на каждый апдейт
            data["session"] = session                 # прокинуть в обработчик
            return await handler(event, data)         # async with сам закроет её после

dp.update.middleware(DbSessionMiddleware(Sessionmaker))   # на ВСЕ типы апдейтов
```

**Шаг 3. Обработчик** просто объявляет аргумент `session` — aiogram подставит его из `data`:

```python
@router.message(CommandStart())
async def start(message: Message, session: AsyncSession):   # session приезжает из middleware
    user = await session.get(User, message.from_user.id)    # уже в базе?
    if user is None:
        session.add(User(id=message.from_user.id, name=message.from_user.full_name))
        await session.commit()                              # ← без commit запись не сохранится
        await message.answer("Зарегистрировал тебя!")
    else:
        await message.answer(f"С возвращением, {user.name}!")
```

- **`dp.update.middleware(...)`** — вешает middleware на внешний уровень `update`, поэтому сессию получают обработчики **любого** типа (message, callback_query…).
- **Имя аргумента = ключ в `data`.** Положил `data["session"]` — обработчик берёт `session`. Это и есть внедрение зависимостей aiogram.
- **`async with` в middleware** гарантирует, что сессия закроется после обработчика, даже при ошибке.

> **Одна глобальная сессия на всех — гонки и битые данные.** Сессия SQLAlchemy держит состояние (identity map, транзакцию) и не переживает параллельного доступа. Middleware даёт каждому апдейту **отдельную** сессию — это и есть правильный «scope». Готовую связку async-сервера с async SQLAlchemy целиком смотри в [примере на aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#сквозной-пример-async-rest-api-на-aiohttp--sqlalchemy).

## Приём платежей: Telegram Stars

Бот умеет **брать деньги** прямо в Телеграме. За цифровые товары (подписка, доступ, донат) платят **Telegram Stars** — внутренней валютой Телеграма с кодом **`XTR`**; для неё не нужен внешний платёжный провайдер (`provider_token` пустой). Оплата идёт в три шага-события:

```
bot.send_invoice ─▶ пользователь платит ─▶ pre_checkout_query ─▶ successful_payment
   (счёт)                                   (подтвердить за 10 с)   (деньги пришли)
```

```python
from aiogram.types import LabeledPrice, PreCheckoutQuery, Message

# 1. Выставить счёт — цена в звёздах (для XTR amount = число звёзд)
@router.message(Command("buy"))
async def buy(message: Message):
    await message.answer_invoice(
        title="Премиум-доступ",
        description="Доступ на 30 дней",
        payload="premium_30d",              # своя метка заказа — вернётся в successful_payment
        currency="XTR",                      # Telegram Stars
        prices=[LabeledPrice(label="Премиум", amount=100)],   # 100 звёзд
    )

# 2. Предчек — Телеграм спрашивает «проводить?». ОБЯЗАН ответить ok=True за 10 секунд
@router.pre_checkout_query()
async def pre_checkout(query: PreCheckoutQuery):
    await query.answer(ok=True)             # тут проверяют наличие товара/промокода

# 3. Деньги пришли — выдать товар
@router.message(F.successful_payment)
async def paid(message: Message):
    payment = message.successful_payment
    payload = payment.invoice_payload       # "premium_30d" — что именно купили
    charge_id = payment.telegram_payment_charge_id   # ID платежа (нужен для возврата)
    await message.answer("Оплачено! Доступ выдан.")
```

- **`answer_invoice`** — отправить счёт в чат (то же, что `bot.send_invoice(chat_id, ...)`).
- **`pre_checkout_query`** — последний шанс отменить сделку (товар кончился, промокод не подошёл). **Не ответишь за 10 секунд — платёж сорвётся**, поэтому здесь только быстрые проверки, без тяжёлых запросов.
- **`successful_payment`** — деньги уже списаны; выдавать товар нужно **здесь**, а не в pre_checkout.
- **Возврат:** `await bot.refund_star_payment(user_id, charge_id)` вернёт звёзды по сохранённому `charge_id`.

> **Промокоды и учёт каналов** — это уже логика приложения, не Телеграма: коды и статистику покупок держат в БД (см. [sqlite3](../Библиотеки/Модули/Форматы%20данных/sqlite3.md) / [SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md)), сверяют промокод в `pre_checkout_query`, а `invoice_payload` используют, чтобы связать платёж с заказом.

## Деплой через вебхук на aiohttp

Поллинг хорош для разработки, но в проде переходят на **вебхук**: Телеграм сам шлёт апдейты POST-запросом на твой HTTPS-адрес — не нужно крутить бесконечный `getUpdates`, отклик мгновенный, легче масштабировать. aiogram поднимает под это сервер на [aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md), который принимает запросы и скармливает их **тому же** диспетчеру. Обработчики и роутеры при этом **не меняются** — меняется только «транспорт» доставки апдейтов.

Что понадобится: **публичный HTTPS-URL** (домен + TLS-сертификат) — обычно бот стоит на `localhost`, а наружу его выставляет nginx как reverse-proxy с сертификатом (см. [Развертывание проекта](../DevOps/Развертывание%20проекта.md)).

Три детали связывают aiogram с aiohttp:

- **`bot.set_webhook(url, secret_token=...)`** на старте — говорит Телеграму, куда слать апдейты;
- **`SimpleRequestHandler`** — регистрирует в aiohttp-приложении маршрут-приёмник, который передаёт апдейты диспетчеру;
- **`setup_application`** — подключает к серверу жизненный цикл диспетчера (`startup`/`shutdown`).

```python
from aiohttp import web
from aiogram import Bot, Dispatcher
from aiogram.webhook.aiohttp_server import SimpleRequestHandler, setup_application

WEBHOOK_PATH   = "/webhook"                    # путь приёмника на сервере
BASE_URL       = "https://mybot.example.com"   # твой публичный домен
WEBHOOK_SECRET = "случайная-строка-секрет"      # для проверки подлинности запросов

async def on_startup(bot: Bot):
    # сказать Телеграму слать апдейты на BASE_URL + WEBHOOK_PATH
    await bot.set_webhook(f"{BASE_URL}{WEBHOOK_PATH}", secret_token=WEBHOOK_SECRET)

def main():
    dp = Dispatcher()
    dp.include_router(router)                   # те же роутеры, что и при поллинге
    dp.startup.register(on_startup)             # выполнить on_startup при запуске

    bot = Bot(token="ТОКЕН")
    app = web.Application()                     # aiohttp-приложение

    # маршрут-приёмник: aiohttp → диспетчер; secret_token отсекает подделки
    SimpleRequestHandler(dispatcher=dp, bot=bot, secret_token=WEBHOOK_SECRET) \
        .register(app, path=WEBHOOK_PATH)
    setup_application(app, dp, bot=bot)         # прицепить lifecycle диспетчера

    web.run_app(app, host="127.0.0.1", port=8080)   # за ним — nginx с HTTPS

if __name__ == "__main__":
    main()
```

**`secret_token`** — общий секрет между тобой и Телеграмом: он кладёт его в заголовок `X-Telegram-Bot-Api-Secret-Token` каждого запроса, а `SimpleRequestHandler` сверяет и **отклоняет** запросы без него. Твой URL публичный, так что без этой проверки на приёмник может постучаться кто угодно — та же логика защиты, что у [обычных вебхуков](../Библиотеки/Сторонние/Работа%20с%20WEB/Вебхуки%20%28webhooks%29.md#безопасность--это-главное) (только здесь простой общий токен, а не HMAC-подпись).

> **Вебхук и поллинг взаимоисключающи.** Пока у бота установлен вебхук, `getUpdates` (поллинг) возвращает **409 Conflict** — Телеграм не отдаёт апдейты двумя каналами разом. Чтобы вернуться к поллингу при разработке, сбрось вебхук: `await bot.delete_webhook()`.

## aiogram 3 против 2.x

Версия 3 переписана с нуля и **ломает совместимость**. Это главная причина, почему старые статьи не работают. Ключевые отличия:

| | **aiogram 2.x** | **aiogram 3.x** |
|---|---|---|
| Bot в диспетчере | `Dispatcher(bot)` | `Dispatcher()`, `bot` → в `start_polling(bot)` |
| Запуск | `executor.start_polling(dp)` | `asyncio.run(dp.start_polling(bot))` |
| Обработчики | `@dp.message_handler(...)` | `@router.message(...)` |
| Модульность | плоско на `dp` | роутеры + `include_router` |
| Фильтры контента | `content_types=[...]`, `text=` | магический фильтр `F` |
| Состояние по умолчанию | ловилось само | фильтр состояния указывают явно |
| `parse_mode` | аргумент `Bot(parse_mode=...)` | `Bot(default=DefaultBotProperties(parse_mode=...))` |
| Под капотом | свои классы | все методы — [pydantic](../Библиотеки/Сторонние/pydantic.md)-модели с валидацией |

> Общее правило распознать версию: если в примере есть `@dp.message_handler` или `executor.start_polling` — это **устаревший 2.x**, в 3.x так писать нельзя.

Актуальный способ задать общий `parse_mode` (чтобы во всех сообщениях работал HTML/Markdown):

```python
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode

bot = Bot(token=TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
```

## aiogram против других библиотек

| | **aiogram** | **python-telegram-bot** | **pyTelegramBotAPI (telebot)** |
|---|---|---|---|
| Модель | async (asyncio) | async (с v20) | синхронная (async опционально) |
| Стиль | роутеры, фильтры, FSM, DI | похоже, свой API | простой, декораторы |
| Порог входа | средний | средний | самый низкий |
| Когда брать | серьёзный async-бот, масштаб | зрелая альтернатива | маленький бот, быстрый старт |

aiogram — выбор по умолчанию в русскоязычном сообществе: асинхронный, с продуманной архитектурой (роутеры, FSM, middleware) и активной документацией.

## Ловушка: блокирующий код в обработчике

Обработчики крутятся в **одном** [event loop](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#event-loop--сердце-asyncio). Синхронный вызов, который *ждёт* (обычный `requests.get`, тяжёлый запрос к БД, `time.sleep`), **замораживает весь бот** — пока он ждёт, ни один другой пользователь не получает ответа.

```python
# НЕПРАВИЛЬНО — блокирует всех:
@router.message()
async def handler(message):
    r = requests.get("https://slow-api.com")   # синхронный, ЖДЁТ → бот стоит
    time.sleep(5)                               # весь бот замер на 5 секунд

# ПРАВИЛЬНО — асинхронные инструменты с await:
@router.message()
async def handler(message):
    async with aiohttp.ClientSession() as s:    # не блокирует loop
        async with s.get("https://slow-api.com") as r:
            data = await r.json()
    await asyncio.sleep(5)                       # асинхронная пауза
```

> В `async`-обработчике — только неблокирующие (`await`) вызовы. Тяжёлый **счёт** (CPU-bound: обработка картинок, парсинг) выноси в отдельный процесс, иначе он тоже застопорит loop — та же логика, что в [FastAPI](../Фреймворки/FastAPI.md#cpu-задачи-io-bound-против-cpu-bound).

## Сквозной пример: бот-анкета с кнопкой

Соберём маленький, но цельный бот: `/start` показывает inline-кнопку, по нажатию запускается FSM-анкета (имя → возраст), в конце — сводка. Один файл, запускается как есть.

```python
import asyncio
from aiogram import Bot, Dispatcher, F, Router
from aiogram.filters import CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import StatesGroup, State
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import Message, CallbackQuery
from aiogram.utils.keyboard import InlineKeyboardBuilder

router = Router()

# 1. Шаги анкеты
class Form(StatesGroup):
    name = State()
    age = State()

# 2. /start → сообщение с inline-кнопкой «Заполнить»
@router.message(CommandStart())
async def start(message: Message):
    kb = InlineKeyboardBuilder()
    kb.button(text="📝 Заполнить анкету", callback_data="fill")
    await message.answer("Привет! Заполним анкету?", reply_markup=kb.as_markup())

# 3. Нажали кнопку → входим в первый шаг FSM
@router.callback_query(F.data == "fill")
async def fill(callback: CallbackQuery, state: FSMContext):
    await state.set_state(Form.name)
    await callback.message.answer("Как тебя зовут?")
    await callback.answer()                     # убрать «часики»

# 4. Шаг «имя» → сохранить, спросить возраст
@router.message(Form.name)
async def got_name(message: Message, state: FSMContext):
    await state.update_data(name=message.text)
    await state.set_state(Form.age)
    await message.answer("Сколько тебе лет?")

# 5. Шаг «возраст» → проверить число, показать сводку, завершить
@router.message(Form.age, F.text.regexp(r"^\d+$"))   # фильтр: только цифры
async def got_age(message: Message, state: FSMContext):
    data = await state.update_data(age=message.text)
    await state.clear()
    await message.answer(f"✅ Готово!\nИмя: {data['name']}\nВозраст: {data['age']}")

# 5b. Возраст не число → попросить снова (состояние НЕ меняем)
@router.message(Form.age)
async def wrong_age(message: Message):
    await message.answer("Возраст — это число. Попробуй ещё раз.")

async def main():
    bot = Bot(token="ТОКЕН_ОТ_BOTFATHER")
    dp = Dispatcher(storage=MemoryStorage())    # storage нужен для FSM
    dp.include_router(router)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
```

**Как это читать.** Апдейты идут в `Dispatcher`, тот раздаёт их роутеру. `/start` рисует кнопку; нажатие (`callback_query`) переводит пользователя в состояние `Form.name`. Дальше его сообщения ловят обработчики, отфильтрованные **по состоянию** (`Form.name`, `Form.age`), — так один и тот же пользователь проходит анкету по шагам, а его данные копятся в `state`. Фильтр `F.text.regexp(...)` на шаге возраста показывает, как валидировать ввод прямо в декораторе: подошло число — обработчик 5, не подошло — запасной 5b просит повторить. `MemoryStorage` хранит прогресс; в проде его меняют на [Redis](../Библиотеки/Сторонние/Redis.md).

## Связи

- [Asyncio Event Loop и aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — asyncio и event loop, на которых стоит aiogram; вебхук-сервер бота — это aiohttp;
- Асинхронность — почему для ботов (много ожидания сети) async идеален;
- [Вебхуки (webhooks)](../Библиотеки/Сторонние/Работа%20с%20WEB/Вебхуки%20%28webhooks%29.md) — второй способ получать апдейты; Telegram как провайдер вебхуков;
- [pydantic](../Библиотеки/Сторонние/pydantic.md) — все типы и методы aiogram 3 построены на нём (валидация апдейтов);
- [Redis](../Библиотеки/Сторонние/Redis.md) — прод-хранилище состояний FSM (`RedisStorage`);
- [FastAPI](../Фреймворки/FastAPI.md) — та же идея фильтров/middleware и ловушка блокирующего кода в async;
- [SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md) — типичная связка: aiogram + async SQLAlchemy для хранения данных бота;
- [discord.py](../Фреймворки/discord.py.md) — «сестра» для Discord: та же async-модель, но живой WebSocket-gateway и UI-компоненты вместо клавиатур;
- [Развертывание проекта](../DevOps/Развертывание%20проекта.md) — nginx + TLS перед вебхук-сервером бота в проде;
- [Docker](../DevOps/Docker.md) — бота обычно катят в контейнере рядом с Redis/БД.

## Источники

- [aiogram — Documentation (latest)](https://docs.aiogram.dev/en/latest/)
- [aiogram — Quickstart](https://docs.aiogram.dev/en/latest/index.html)
- [aiogram — Filtering events (magic filter F)](https://docs.aiogram.dev/en/latest/dispatcher/filters/index.html)
- [aiogram — Finite State Machine](https://docs.aiogram.dev/en/latest/dispatcher/finite_state_machine/index.html)
- [aiogram — Keyboard builders](https://docs.aiogram.dev/en/latest/utils/keyboard.html)
- [aiogram — Migration FAQ (2.x → 3.0)](https://docs.aiogram.dev/en/latest/migration_2_to_3.html)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Груша М. — «Пишем Telegram-ботов с aiogram 3.x»](https://mastergroosha.github.io/aiogram-3-guide/) — лучший русскоязычный гайд
- [Хабр — Telegram Боты на Aiogram 3.x: Первые Шаги](https://habr.com/ru/companies/amvera/articles/820527/)
- [Хабр — За границей Hello World: полный гайд по Aiogram 3](https://habr.com/ru/articles/732136/)
