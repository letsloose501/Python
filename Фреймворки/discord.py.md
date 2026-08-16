[← Оглавление](../README.md)

# discord.py

> _Бот держит с Discord живой провод и слышит всё, что происходит на сервере, в реальном времени_

---

**discord.py** — асинхронный фреймворк для ботов Discord на [Python](../Python.md). Он оборачивает **Discord API** в объекты (сервер, канал, роль, участник) и держит постоянное соединение, по которому Discord сам присылает события: пришло сообщение, зашёл участник, нажали кнопку. Ты пишешь реакции на события и команды, остальное берёт на себя фреймворк. Как и [aiogram](../Фреймворки/aiogram.md) для Telegram, построен на [asyncio](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — один процесс обслуживает целый сервер (в терминах Discord — **гильдию**) с тысячами участников.

> Здесь описан **discord.py 2.x** (актуальная линейка, Python 3.8+). Библиотеку **закрывали** в 2021-м и возродили в 2022-м с версии 2.0 — за время паузы появились форки (pycord, nextcord, disnake) с чуть другим синтаксисом. Поэтому старые примеры и куски из форков часто **не совпадают** (см. [discord.py против форков](#discordpy-против-форков)).

## Как работает бот: gateway и события

Discord-бот не опрашивает сервер по таймеру. Он открывает с Discord постоянное **WebSocket-соединение (gateway)** и держит его; по этому «проводу» Discord **сам** шлёт события в реальном времени. discord.py принимает их и вызывает твои обработчики.

```
        ┌─────────── Discord ───────────┐
        │  события: сообщение, вход,     │
        │  реакция, нажатие кнопки       │
        └───────────────┬────────────────┘
                        │ gateway (постоянный WebSocket)
                        ▼
                  твой бот (discord.py)
                        │
             on_message / on_member_join / …
```

Это отличается и от поллинга, и от вебхуков [Telegram-бота](../Фреймворки/aiogram.md): там апдейты забирают по одному, здесь — живой поток событий по сокету.

### Intents — какие события бот хочет слышать

**Intents (намерения)** — набор категорий событий, которые бот получает. Discord не шлёт всё подряд: ты явно включаешь нужное. Часть категорий **привилегированные** (`members`, `presences`, `message_content`) — их надо дополнительно включить в панели разработчика Discord, иначе бот их не получит.

```python
intents = discord.Intents.default()      # базовый набор (без привилегированных)
intents.message_content = True           # читать ТЕКСТ сообщений (привилегированный!)
intents.members = True                   # события об участниках (привилегированный!)
```

> **`message_content` — грабли №1.** Без него `message.content` приходит **пустым**, и префиксные команды «молча не работают». Включи intent и в коде, и в панели разработчика.

## Минимальный бот

```python
import discord
from discord.ext import commands

intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)   # бот с префиксом команд "!"

@bot.event                                # регистрируем обработчик события
async def on_ready():
    print(f"Вошли как {bot.user}")

@bot.command()                            # префиксная команда: !ping
async def ping(ctx):
    await ctx.send("pong")                # ctx.send — ответить в тот же канал

bot.run("ТОКЕН")                          # запустить: подключиться к gateway и слушать
```

- **`commands.Bot`** — бот с расширением команд; `command_prefix` — символ перед командой (`!ping`).
- **`@bot.event`** — вешает обработчик события (`on_ready`, `on_message`, `on_member_join`…).
- **`ctx`** (Context) — контекст команды: кто, где, в каком канале; через него отвечают.
- **`bot.run(TOKEN)`** — блокирующий запуск (сам поднимает event loop).

> **Токен = пароль бота.** Храни в переменной окружения, не в коде и не в git. Утёк — чужой управляет ботом.

## Команды: префиксные против slash

Есть два вида команд, и это важно:

- **Префиксные** (`!ping`) — старый способ: бот **читает текст** каждого сообщения и ищет префикс. Требует привилегированный `message_content`.
- **Slash-команды** (`/ping`) — современный способ через **app_commands**: команды регистрируются у Discord, всплывают в меню по `/`, с подсказками и типами аргументов. Не требуют чтения всех сообщений.

Discord подталкивает к slash-командам. У `commands.Bot` уже есть «дерево команд» **`bot.tree`**, куда их вешают:

```python
from discord import app_commands

@bot.tree.command(name="ping", description="Проверка связи")
async def ping(interaction: discord.Interaction):
    await interaction.response.send_message("pong")     # ответ на взаимодействие

@bot.tree.command(name="greet")
@app_commands.describe(user="кого поприветствовать")    # подсказка к аргументу
async def greet(interaction: discord.Interaction, user: discord.Member):
    await interaction.response.send_message(f"Привет, {user.mention}!")
```

- **`interaction`** (Interaction) — объект взаимодействия (аналог `ctx` для slash); ответ идёт через `interaction.response`.
- **Тип аргумента = тип в Discord.** `user: discord.Member` — Discord сам покажет выбор участника и приведёт значение.

### Синхронизация дерева

Slash-команды **не появятся**, пока их не «синхронизировать» — отправить Discord их список:

```python
@bot.event
async def on_ready():
    await bot.tree.sync()                               # глобально (обновляется до часа)
    # await bot.tree.sync(guild=discord.Object(id=123)) # на один сервер — мгновенно, для теста
```

> **Не зови `sync()` на каждый запуск без нужды** — у Discord жёсткие лимиты. Синхронизируй, только когда список команд **изменился**. Глобальная синхронизация раскатывается до часа; на конкретный сервер (`guild=...`) — сразу, поэтому её берут для разработки.

### Cogs — модульность

Как роутеры в [aiogram](../Фреймворки/aiogram.md#архитектура-bot-dispatcher-router), в discord.py код бьют на **Cog** — класс-модуль с командами и обработчиками, который подключают отдельно:

```python
class Moderation(commands.Cog):
    def __init__(self, bot):
        self.bot = bot

    @commands.command()
    async def kick(self, ctx, member: discord.Member):
        await member.kick()

async def setup(bot):                       # обязательная точка входа модуля
    await bot.add_cog(Moderation(bot))

# в главном файле:
await bot.load_extension("cogs.moderation") # подключить модуль (путь как импорт)
```

## Объектная модель сервера

Всё управление строится вокруг четырёх объектов:

| Объект | Что это | Пример |
|---|---|---|
| **Guild** | сервер Discord целиком | `interaction.guild` |
| **Channel** | канал: текстовый, голосовой, категория | `guild.text_channels` |
| **Role** | роль: набор прав + цвет, выдаётся участникам | `guild.roles` |
| **Member** | участник **на конкретном сервере** (у него роли, ник) | `interaction.user` |
| **User** | глобальный аккаунт (без привязки к серверу) | `member._user` |

**Member против User:** `User` — это человек вообще; `Member` — тот же человек **на данном сервере**, со своими ролями, ником и правами. Управляешь почти всегда `Member`.

## Управление каналами

Создание, изменение и удаление — методы гильдии и канала (все `await`, это сетевые вызовы):

```python
guild = interaction.guild

# создать
category = await guild.create_category("Игры")
channel  = await guild.create_text_channel("общий", category=category)
voice    = await guild.create_voice_channel("голосовой")

# изменить
await channel.edit(name="переименован", topic="описание", slowmode_delay=5)  # slowmode 5 сек

# удалить
await channel.delete(reason="уборка")
```

**Приватный канал** делают через **overwrites** — точечные разрешения на канал поверх ролей. `PermissionOverwrite` задаёт, что можно/нельзя конкретной роли или участнику:

```python
overwrites = {
    guild.default_role: discord.PermissionOverwrite(view_channel=False),  # @everyone не видит
    vip_role:           discord.PermissionOverwrite(view_channel=True),   # роль VIP — видит
}
await guild.create_text_channel("секретный", overwrites=overwrites)
```

## Управление ролями

**Роль** — именованный набор прав и цвет; права участника — объединение прав всех его ролей.

```python
role = await guild.create_role(
    name="VIP",
    colour=discord.Colour.gold(),                       # цвет ника
    hoist=True,                                          # показывать отдельной группой в списке
    mentionable=True,                                    # можно упоминать @VIP
    permissions=discord.Permissions(manage_messages=True),  # что роль разрешает
)

await member.add_roles(role, reason="купил подписку")   # выдать роль
await member.remove_roles(role)                          # снять роль
await role.delete()                                      # удалить роль
```

- **`discord.Permissions(...)`** — набор прав (управлять сообщениями, банить, кидать участников…).
- **`hoist`** — выносит носителей роли отдельной группой в списке участников.

## Управление пользователями (участниками)

Модерация — методы `Member` (кик, бан, тайм-аут, смена ника):

```python
import datetime

await member.edit(nick="Новый ник")                          # сменить ник на сервере
await member.kick(reason="спам")                             # выгнать (может вернуться)
await member.ban(reason="нарушение", delete_message_days=1)  # забанить + стереть сутки сообщений
await member.timeout(datetime.timedelta(minutes=10))         # тайм-аут (временный мут)

for m in guild.members:                                      # перебрать участников (нужен intent members)
    print(m.display_name, [r.name for r in m.roles])
```

Команды модерации закрывают **проверкой прав** — чтобы кикать мог не любой:

```python
@bot.tree.command()
@app_commands.checks.has_permissions(kick_members=True)      # только у кого есть право
async def kick(interaction: discord.Interaction, member: discord.Member):
    await member.kick()
    await interaction.response.send_message(f"{member.mention} выгнан", ephemeral=True)
```

> **`ephemeral=True`** — ответ видит **только вызвавший**, остальные в канале его не видят. Удобно для модерации и приватных сообщений бота.

## UI-компоненты: кнопки, меню, модалки

Под сообщением бот вешает интерактивные элементы. Все они живут внутри **View** — контейнера, который слушает нажатия. Нажатие приходит как **Interaction** в callback.

### Кнопки и их виды

```python
class Confirm(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=60)        # через 60 сек кнопки «протухнут»

    @discord.ui.button(label="Да", style=discord.ButtonStyle.success)
    async def yes(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("Подтверждено ✅", ephemeral=True)
        self.stop()                          # прекратить слушать эту View

    @discord.ui.button(label="Нет", style=discord.ButtonStyle.danger)
    async def no(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("Отменено ❌", ephemeral=True)
        self.stop()

await channel.send("Уверен?", view=Confirm())   # прикрепить View к сообщению
```

> **Порядок аргументов callback в discord.py — `(self, interaction, button)`.** В форке **pycord он обратный** (`button, interaction`), а поле ввода зовётся `InputText`, не `TextInput`. Перепутанные примеры из форков — частая причина ошибок.

Стиль задаёт цвет и смысл кнопки:

| `ButtonStyle` | Цвет | Когда |
|---|---|---|
| `primary` | синий (blurple) | главное действие |
| `secondary` | серый | второстепенное |
| `success` | зелёный | подтверждение |
| `danger` | красный | опасное (удалить, бан) |
| `link` | как ссылка | ведёт на URL, без callback |

### Выпадающее меню (Select)

```python
class ColorMenu(discord.ui.View):
    @discord.ui.select(placeholder="Выбери цвет", options=[
        discord.SelectOption(label="Красный", value="red",  emoji="🔴"),
        discord.SelectOption(label="Синий",   value="blue", emoji="🔵"),
    ])
    async def pick(self, interaction: discord.Interaction, select: discord.ui.Select):
        await interaction.response.send_message(f"Выбрано: {select.values[0]}")
```

`select.values` — список выбранных значений (меню можно настроить на несколько выборов).

### Модальное окно (форма)

**Modal** — всплывающая форма с полями ввода; появляется **только** в ответ на взаимодействие (команду или нажатие кнопки), не отправляется сообщением.

```python
class Feedback(discord.ui.Modal, title="Анкета"):
    name  = discord.ui.TextInput(label="Имя", placeholder="Как тебя зовут?")
    about = discord.ui.TextInput(label="О себе", style=discord.TextStyle.paragraph, required=False)

    async def on_submit(self, interaction: discord.Interaction):
        await interaction.response.send_message(f"Привет, {self.name.value}!", ephemeral=True)

@bot.tree.command()
async def form(interaction: discord.Interaction):
    await interaction.response.send_modal(Feedback())   # показать форму
```

- **`discord.ui.TextInput`** — поле ввода; `TextStyle.short` (строка) или `TextStyle.paragraph` (абзац).
- Значения читают в `on_submit` через `self.<поле>.value`.

> **Постоянные (persistent) View.** Обычная View живёт до `timeout` — после перезапуска бота кнопки мертвы. Чтобы кнопки работали вечно, задай `timeout=None`, каждому элементу — `custom_id`, и зарегистрируй `bot.add_view(...)` в `setup_hook`.

## Embeds — родные карточки Discord

**Embed** — оформленная «карточка» сообщения: заголовок, описание, поля, картинки, полоса цвета. Её рисует **сам клиент Discord**, тебе не нужно ничего генерировать — просто задаёшь данные.

```python
embed = discord.Embed(
    title="Профиль",
    description="Информация об участнике",
    colour=discord.Colour.blurple(),
)
embed.set_thumbnail(url=member.display_avatar.url)   # маленькая картинка в углу
embed.add_field(name="Уровень", value="7",    inline=True)
embed.add_field(name="Сообщений", value="1420", inline=True)
embed.set_footer(text="Сервер XYZ")
await channel.send(embed=embed)
```

Embed — это «карточка из готовых блоков». Когда нужна **своя вёрстка** (аватар + полоса опыта + фон) — рисуют картинку сами (ниже).

## Генерация изображений: карточка профиля на Pillow

Для полностью своей графики (карточка ранга, welcome-баннер) изображение **рисуют** библиотекой **[Pillow](https://pillow.readthedocs.io/)** (`PIL`) и отправляют как файл. Схема из трёх шагов:

```
аватар участника (готовое фото) ──▶ Pillow: собрать карточку ──▶ discord.File ──▶ отправить
```

- **Взять готовый аватар:** `avatar_bytes = await member.display_avatar.read()` — Discord отдаёт **байты** картинки, их открывает Pillow.
- **Собрать в памяти:** рисуешь на холсте, сохраняешь в `io.BytesIO` (файл в оперативке, без записи на диск).
- **Отправить:** оборачиваешь буфер в `discord.File`.

```python
import io
from PIL import Image, ImageDraw

def render_card(avatar_bytes: bytes, name: str, level: int) -> io.BytesIO:
    card   = Image.new("RGB", (600, 200), (30, 30, 40))        # фон-холст
    draw   = ImageDraw.Draw(card)
    avatar = Image.open(io.BytesIO(avatar_bytes)).resize((160, 160))
    card.paste(avatar, (20, 20))                               # вклеить готовый аватар
    draw.text((200, 60), name, fill="white")                  # имя
    draw.text((200, 110), f"Уровень: {level}", fill="gray")   # уровень
    buffer = io.BytesIO()
    card.save(buffer, format="PNG")
    buffer.seek(0)                                             # перемотать в начало — иначе файл пустой
    return buffer
```

> **Pillow синхронный и грузит CPU** — рисование картинки *блокирует* весь бот, пока идёт (это [CPU-bound](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#зачем-асинхронность) работа, не I/O). Выноси рендер в отдельный поток через `asyncio.to_thread(render_card, ...)`, иначе на время генерации бот перестаёт отвечать всем — та же ловушка, что [блокирующий код в aiogram](../Фреймворки/aiogram.md#ловушка-блокирующий-код-в-обработчике). Готовые обёртки для карточек (`easy-pil`, `DiscordLevelingCard`) делают это поверх той же Pillow.

## discord.py против форков

Пауза 2021–2022 породила форки — API похож, но в деталях расходится:

| | **discord.py** | **pycord / nextcord / disnake** |
|---|---|---|
| Статус | оригинал, снова активен (2.x) | форки времён паузы, живут отдельно |
| callback кнопки | `(self, interaction, button)` | у pycord обратный порядок |
| поле модалки | `discord.ui.TextInput` | у pycord `InputText` |
| Совместимость | пакеты **несовместимы** между собой — нельзя мешать в одном проекте | |

> Ставится discord.py как `pip install -U discord.py`, импортируется `import discord`. Если в примере `import nextcord`/`import disnake` или у кнопки аргументы наоборот — это **форк**, а не discord.py.

## discord.py против aiogram

| | **discord.py** | **[aiogram](../Фреймворки/aiogram.md)** |
|---|---|---|
| Платформа | Discord | Telegram |
| Связь | постоянный WebSocket (gateway) | поллинг или вебхук |
| Команды | префиксные и slash (`app_commands`) | обработчики + фильтры |
| Модульность | Cogs | роутеры |
| Формы | Modal (окно) | FSM (пошагово в чате) |
| UI | кнопки/меню/модалки в View | inline/reply-клавиатуры |
| Основа | asyncio | asyncio |

Модель событий разная (у Discord — живой сокет и понятие сервера/ролей; у Telegram — чат и апдейты), но обе библиотеки async и идейно близки: объекты платформы + обработчики.

## Ловушка: успеть ответить за 3 секунды

На любое взаимодействие (slash-команда, кнопка) Discord ждёт ответ **в течение 3 секунд**, иначе показывает «приложение не отвечает». Если работа дольше (генерация картинки, запрос к API) — сначала **возьми паузу** через `defer`, потом досылай результат через `followup`:

```python
# НЕПРАВИЛЬНО — рендер дольше 3 сек, взаимодействие «протухнет»:
@bot.tree.command()
async def profile(interaction):
    buffer = render_card(...)                       # долго → Discord уже показал ошибку
    await interaction.response.send_message(...)

# ПРАВИЛЬНО — defer резервирует ответ, followup досылает позже:
@bot.tree.command()
async def profile(interaction):
    await interaction.response.defer()              # «думаю…», лимит снят
    buffer = await asyncio.to_thread(render_card, ...)
    await interaction.followup.send(file=discord.File(buffer, "card.png"))
```

## Сквозной пример: команда /profile с карточкой профиля

Соберём бот с одной slash-командой `/profile`: берёт **готовый аватар** участника, рисует на нём карточку через **Pillow** (в отдельном потоке, чтобы не морозить бот), и показывает её внутри **embed**. Один файл, запускается как есть.

```python
import io
import asyncio
import discord
from discord.ext import commands
from PIL import Image, ImageDraw

intents = discord.Intents.default()
intents.members = True                              # нужен для данных участника
bot = commands.Bot(command_prefix="!", intents=intents)

# 1. Рендер карточки — СИНХРОННЫЙ Pillow (позовём его в отдельном потоке)
def render_card(avatar_bytes: bytes, name: str, level: int) -> io.BytesIO:
    card = Image.new("RGB", (600, 200), (30, 30, 40))
    draw = ImageDraw.Draw(card)
    avatar = Image.open(io.BytesIO(avatar_bytes)).resize((160, 160))
    card.paste(avatar, (20, 20))                    # вклеить готовый аватар
    draw.text((200, 60),  name, fill="white")
    draw.text((200, 110), f"Уровень: {level}", fill="gray")
    buffer = io.BytesIO()
    card.save(buffer, format="PNG")
    buffer.seek(0)
    return buffer

# 2. Slash-команда
@bot.tree.command(name="profile", description="Карточка профиля")
async def profile(interaction: discord.Interaction):
    await interaction.response.defer()              # рендер долгий → берём паузу (правило 3 сек)
    member = interaction.user

    avatar_bytes = await member.display_avatar.read()               # готовое изображение аватара
    buffer = await asyncio.to_thread(                               # CPU-рендер в отдельном потоке
        render_card, avatar_bytes, member.display_name, 7
    )

    file  = discord.File(buffer, filename="profile.png")
    embed = discord.Embed(title=f"Профиль {member.display_name}",
                          colour=discord.Colour.blurple())
    embed.set_image(url="attachment://profile.png")                # показать файл внутри embed
    await interaction.followup.send(embed=embed, file=file)        # followup после defer

# 3. Синхронизировать команды при старте
@bot.event
async def on_ready():
    await bot.tree.sync()
    print(f"Готов: {bot.user}")

bot.run("ТОКЕН")
```

**Как это читать.** `/profile` вызывает взаимодействие; `defer()` снимает лимит в 3 секунды, потому что рендер небыстрый. `display_avatar.read()` отдаёт **байты готового аватара** — их открывает Pillow. `asyncio.to_thread` уносит **синхронный** рендер в отдельный поток, чтобы event loop не встал и бот отвечал другим. Готовая картинка уходит `discord.File`, а `attachment://profile.png` в embed показывает этот файл **внутри** карточки. `followup.send` — потому что после `defer` обычный `response` уже использован. Так соединяются slash-команды, работа с готовым изображением, генерация своей графики и embed.

## Практические системы

Дальше — типовые «движки» серверных ботов. Все они соединяют примитивы выше (события, команды, картинки) с **базой данных** — async [SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md), потому что обработчики асинхронные. В discord.py **нет** middleware-инъекции сессии, как в [aiogram](../Фреймворки/aiogram.md#база-данных-aiogram--async-sqlalchemy): фабрику сессий создают один раз и открывают `async with Sessionmaker()` прямо в обработчике.

```python
from sqlalchemy import BigInteger, Integer
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase): ...

engine = create_async_engine("postgresql+asyncpg://app:secret@localhost/app")
Sessionmaker = async_sessionmaker(engine, expire_on_commit=False)

async def setup_hook():                              # создать таблицы один раз при старте
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
bot.setup_hook = setup_hook
```

### Система уровней (leveling)

Идея: за сообщения начисляем **опыт (XP)**, из него считаем уровень. Два обязательных нюанса: **кулдаун** (не чаще раза в минуту, иначе флудом фармят) и **растущая кривая** уровней. Каноничная формула (её популяризовал бот MEE6) — сколько XP нужно с уровня `L` на `L+1`:

```
xp(L → L+1) = 5·L² + 50·L + 100
```

```python
class Level(Base):
    __tablename__ = "levels"
    guild_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    user_id:  Mapped[int] = mapped_column(BigInteger, primary_key=True)   # составной ключ
    xp:       Mapped[int] = mapped_column(Integer, default=0)

def xp_for_next(level: int) -> int:
    return 5 * level**2 + 50 * level + 100

def level_from_xp(total: int) -> tuple[int, int, int]:
    level, need = 0, 100
    while total >= need:                    # «съедаем» пороги, пока хватает XP
        total -= need
        level += 1
        need = xp_for_next(level)
    return level, total, need               # уровень, XP в текущем уровне, порог до следующего
```

Начисление — в `on_message`, с кулдауном по каждому (сервер, пользователь):

```python
import time, random
_cooldown: dict[tuple[int, int], float] = {}

@bot.event
async def on_message(message: discord.Message):
    if message.author.bot or message.guild is None:
        return
    key = (message.guild.id, message.author.id)
    if time.monotonic() - _cooldown.get(key, 0) < 60:      # не чаще раза в минуту
        await bot.process_commands(message)                # ← не проглотить команды!
        return
    _cooldown[key] = time.monotonic()

    async with Sessionmaker() as session:
        row = await session.get(Level, key)
        before, _, _ = level_from_xp(row.xp if row else 0)
        gained = random.randint(15, 25)                    # 15–25 XP за сообщение
        if row is None:
            session.add(Level(guild_id=key[0], user_id=key[1], xp=gained))
            after = 0
        else:
            row.xp += gained
            after, _, _ = level_from_xp(row.xp)
        await session.commit()

    if after > before:
        await message.channel.send(f"🎉 {message.author.mention} — уровень {after}!")
    await bot.process_commands(message)                    # обработать команды после
```

> **Переопределил `on_message` — сам вызови `bot.process_commands(message)`.** По умолчанию `commands.Bot` в `on_message` запускает префиксные команды. Свой `on_message` **заменяет** это поведение — без явного `process_commands` все команды `!...` перестанут работать.

### Карточка ранга

Расширение [карточки профиля](#генерация-изображений-карточка-профиля-на-pillow): добавляем **полосу прогресса** до следующего уровня. Рисуем той же Pillow (и так же уносим рендер в поток — он [CPU-bound](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#зачем-асинхронность)):

```python
from PIL import Image, ImageDraw
import io

def render_rank(avatar: bytes, name: str, level: int, cur: int, need: int) -> io.BytesIO:
    card = Image.new("RGB", (600, 200), (30, 30, 40))
    draw = ImageDraw.Draw(card)
    card.paste(Image.open(io.BytesIO(avatar)).resize((160, 160)), (20, 20))
    draw.text((200, 30), name, fill="white")
    draw.text((200, 70), f"Уровень {level}", fill="gray")
    x0, x1, y0, y1 = 200, 560, 130, 160
    draw.rectangle((x0, y0, x1, y1), fill=(60, 60, 70))              # фон полосы
    filled = x0 + int((x1 - x0) * cur / need)                        # доля до уровня
    draw.rectangle((x0, y0, filled, y1), fill=(90, 160, 255))       # заполнение
    draw.text((x0, y1 + 6), f"{cur}/{need} XP", fill="white")
    buf = io.BytesIO(); card.save(buf, "PNG"); buf.seek(0)
    return buf

@bot.tree.command(name="rank", description="Моя карточка ранга")
async def rank(interaction: discord.Interaction):
    await interaction.response.defer()                              # рендер долгий → пауза (правило 3 сек)
    async with Sessionmaker() as session:
        row = await session.get(Level, (interaction.guild.id, interaction.user.id))
    level, cur, need = level_from_xp(row.xp if row else 0)
    avatar = await interaction.user.display_avatar.read()
    buf = await asyncio.to_thread(render_rank, avatar, interaction.user.display_name, level, cur, need)
    await interaction.followup.send(file=discord.File(buf, "rank.png"))
```

### Экономика

Валюта на сервере: баланс, ежедневная награда, переводы, магазин. Таблица `Balance(guild_id, user_id, coins)`, команды `/balance`, `/daily`, `/pay`.

Главная тонкость — **атомарность списания**. Наивный «прочитал баланс → проверил → записал» ловит [гонку](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md#ловушка-гонка-данных-через-await): два одновременных `/pay` прочитают по 100 монет и оба спишут — уйдёт в минус. Правильно — списывать **одним** SQL-запросом с условием «хватает средств» и смотреть, сколько строк изменилось:

```python
from sqlalchemy import update

async def transfer(session, gid, sender, receiver, amount) -> bool:
    res = await session.execute(                          # СПИСАТЬ атомарно
        update(Balance)
        .where(Balance.guild_id == gid,
               Balance.user_id == sender,
               Balance.coins >= amount)                    # ← условие в самом UPDATE
        .values(coins=Balance.coins - amount)
    )
    if res.rowcount == 0:                                  # ничего не списалось → денег не хватило
        return False
    await session.execute(                                 # ЗАЧИСЛИТЬ получателю
        update(Balance).where(Balance.guild_id == gid, Balance.user_id == receiver)
        .values(coins=Balance.coins + amount)
    )
    await session.commit()
    return True
```

> **Никогда не пиши баланс как `coins = прочитанное − сумма`** из Python: между чтением и записью успевает влезть другая операция. Проверку «хватает ли» переноси **в сам `UPDATE ... WHERE coins >= amount`** — база выполнит его атомарно, а `rowcount == 0` честно скажет «недостаточно средств».

Ежедневную награду `/daily` ограничивают, храня время последней выдачи (поле `last_daily`) и сравнивая с текущим — прошло ли 24 часа. Для команд с фиксированным окном подходит и встроенный кулдаун `@app_commands.checks.cooldown(1, 60)`.

### Учёт онлайна: время в голосовых каналах

«Онлайн на сервере» надёжнее всего мерить как **время в голосовых каналах** — оно фиксируется событием `on_voice_state_update` (вход/выход/переход). Засекаем момент входа, на выходе считаем длительность и копим в БД:

```python
class VoiceTime(Base):
    __tablename__ = "voice_time"
    guild_id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    user_id:  Mapped[int] = mapped_column(BigInteger, primary_key=True)
    seconds:  Mapped[int] = mapped_column(Integer, default=0)

_since: dict[tuple[int, int], float] = {}

@bot.event
async def on_voice_state_update(member, before: discord.VoiceState, after: discord.VoiceState):
    if member.bot:
        return
    key = (member.guild.id, member.id)
    if before.channel is None and after.channel is not None:      # вошёл в голосовой
        _since[key] = time.monotonic()
    elif before.channel is not None and after.channel is None:    # вышел совсем
        start = _since.pop(key, None)
        if start is None:
            return
        secs = int(time.monotonic() - start)
        async with Sessionmaker() as session:
            row = await session.get(VoiceTime, key)
            if row is None:
                session.add(VoiceTime(guild_id=key[0], user_id=key[1], seconds=secs))
            else:
                row.seconds += secs
            await session.commit()
```

- Переход между каналами (`before.channel != after.channel`, оба не `None`) не сбрасывает счёт — человек всё это время в голосе, отсечка только на полном выходе.
- Хочешь не считать AFK/замьюченных — проверяй `after.self_mute`, `after.afk`, `member.guild.afk_channel`.

> **Счётчик входов живёт в памяти — рестарт его теряет.** Кто сидел в голосе во время перезапуска бота, «повиснет» без отсечки. Для точности при старте (`on_ready`) пройди `guild.voice_channels` → `channel.members` и заново засеки время для тех, кто уже в голосе. Отдельная тема — «онлайн» по статусу (в сети/не беспокоить): нужен привилегированный intent `presences` и `on_presence_update`, но статус ненадёжен (его прячут), поэтому меряют именно голос.

### События сервера (Scheduled Events)

**Запланированное событие** — карточка мероприятия с датой, на которую участники подписываются и получают напоминание. Создаёт `guild.create_scheduled_event`. Тип (`entity_type`) диктует обязательные поля:

```python
import datetime
start = discord.utils.utcnow() + datetime.timedelta(days=1)      # время ДОЛЖНО быть aware

# 1) Внешнее событие (по времени и месту): нужны location и end_time
event = await guild.create_scheduled_event(
    name="Игровой вечер",
    description="Собираемся катать",
    start_time=start,
    end_time=start + datetime.timedelta(hours=2),                # обязателен для external
    entity_type=discord.EntityType.external,
    location="ссылка на созвон или адрес",                        # обязателен для external
    privacy_level=discord.PrivacyLevel.guild_only,
)

# 2) Событие в голосовом/стейдж-канале: нужен channel (end_time не обязателен)
event = await guild.create_scheduled_event(
    name="Созвон",
    start_time=start,
    entity_type=discord.EntityType.voice,
    channel=some_voice_channel,                                  # обязателен для voice/stage
)

await event.start()                       # начать (.end() завершить, .cancel() отменить, .edit(...) изменить)
async for user in event.users():          # кто подписался
    print(user)
print(event.url)                          # ссылка-приглашение на событие
```

> **`start_time` — только «aware» datetime** (с таймзоной). Бери `discord.utils.utcnow()` или `datetime.now().astimezone()`; «наивное» время без зоны Discord не примет.

### Трибуны (Stage Channels)

**Трибуна (stage channel)** — особый голосовой канал для докладов и AMA: есть **спикеры** и **слушатели**, слушатели «поднимают руку», чтобы получить слово. Сам эфир — это **stage instance** (у него своя тема):

```python
# создать канал-трибуну
stage = await guild.create_stage_channel("Трибуна")

# запустить эфир с темой
instance = await stage.create_instance(topic="AMA с разработчиком")
await instance.edit(topic="Новая тема эфира")     # сменить тему на ходу
await instance.delete()                           # завершить эфир

# роли на трибуне: вошедший — слушатель (suppress=True). Управление голосом:
await member.edit(suppress=False)                 # поднять участника в спикеры
await member.request_to_speak()                   # запросить слово («поднять руку»)
```

Если создать [событие](#события-сервера-scheduled-events) с `entity_type=discord.EntityType.stage_instance` и указать трибуну, Discord **сам** запустит и завершит эфир вместе с событием.

## Голос и музыка: музыкальный бот

Музыкальный бот делает три вещи: **подключается** к голосовому каналу, **декодирует** звук через FFmpeg и **отдаёт** его в канал, беря аудиопоток (обычно с YouTube через yt-dlp). Плюс очередь треков. Здесь появляются **внешние зависимости** — не чистый Python:

```bash
pip install -U "discord.py[voice]" yt-dlp   # [voice] тянет PyNaCl (шифрование голоса)
# FFmpeg ставится ОТДЕЛЬНО как системная программа и должен быть в PATH
```

> **Голос не заработает без PyNaCl и FFmpeg.** `discord.py[voice]` ставит PyNaCl (шифрование голосового трафика), а **FFmpeg** — это внешняя программа (не pip-пакет), её ставят в систему и добавляют в `PATH`. Нет FFmpeg — бот подключится к каналу, но звука не будет. PCM-звук в кодек **Opus** discord.py кодирует сам (libopus обычно идёт в комплекте).

### Подключение к голосу

Голосовое соединение — **одно на сервер**; объект — `VoiceClient`, доступен как `guild.voice_client` / `ctx.voice_client`:

```python
channel = ctx.author.voice.channel        # канал, где сидит вызвавший
vc = await channel.connect()              # подключиться → VoiceClient
await vc.move_to(other_channel)           # перейти в другой канал
await vc.disconnect()                     # отключиться
```

### Источник звука: FFmpeg + yt-dlp

Discord играет **`AudioSource`**. Локальный файл — это `discord.FFmpegPCMAudio("song.mp3")`. Для YouTube нужен промежуток: **yt-dlp** достаёт из ссылки/запроса прямой URL аудиопотока, а его уже открывает FFmpeg.

```python
import asyncio, discord, yt_dlp
from collections import deque

YTDL = yt_dlp.YoutubeDL({
    "format": "bestaudio/best",
    "noplaylist": True,
    "quiet": True,
    "default_search": "ytsearch",         # текст (не URL) → поиск по YouTube
    "source_address": "0.0.0.0",
})
FFMPEG = {
    "before_options": "-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5",  # не рвать стрим
    "options": "-vn",                     # -vn: только аудио, без видео
}

async def fetch(query: str) -> tuple[str, str]:
    loop = asyncio.get_running_loop()
    # extract_info БЛОКИРУЮЩИЙ (сеть + парсинг) → в отдельный поток, иначе морозит бот
    data = await loop.run_in_executor(None, lambda: YTDL.extract_info(query, download=False))
    if "entries" in data:                 # поиск/плейлист → берём первый результат
        data = data["entries"][0]
    return data["url"], data["title"]     # прямой стрим-URL и название
```

> **`yt_dlp.extract_info` — блокирующий вызов** (лезет в сеть, парсит страницу). Прямо в `async`-коде он заморозит весь бот на секунду-две. Уноси его в поток через `loop.run_in_executor(...)` — та же [ловушка блокирующего кода](../Фреймворки/aiogram.md#ловушка-блокирующий-код-в-обработчике), что везде в async.

### Очередь и главная ловушка — `after` в чужом потоке

Играют одним вызовом `vc.play(source, after=callback)`. Тонкость: **`after` вызывается в отдельном потоке**, когда трек доиграл, — там **нельзя** `await` и нельзя напрямую трогать event loop. Чтобы запустить следующий трек, надо «перепрыгнуть» обратно в loop через `asyncio.run_coroutine_threadsafe`:

```
/play → fetch (yt-dlp в executor) → vc.play(source, after)
                                         │ трек доиграл
                                         ▼
                         after(err)  ←  ОТДЕЛЬНЫЙ ПОТОК (нет await)
                                         │ run_coroutine_threadsafe(play_next, bot.loop)
                                         ▼
                                   play_next()  ←  снова в event loop → следующий трек
```

```python
queues: dict[int, deque[str]] = {}        # очередь ЗАПРОСОВ на каждый сервер (не URL!)

async def play_next(guild: discord.Guild):
    vc = guild.voice_client
    q = queues.setdefault(guild.id, deque())
    if vc is None or not q:
        return
    query = q.popleft()
    stream_url, title = await fetch(query)               # URL достаём ПЕРЕД игрой — не протухнет
    source = discord.PCMVolumeTransformer(               # обёртка ради регулировки громкости
        discord.FFmpegPCMAudio(stream_url, **FFMPEG), volume=0.5)

    def after(err):                                      # ← ЧУЖОЙ ПОТОК: без await!
        if err:
            print("player error:", err)
        asyncio.run_coroutine_threadsafe(play_next(guild), bot.loop)   # назад в event loop

    vc.play(source, after=after)
    # здесь уместно отправить в текстовый канал «▶ Сейчас играет: {title}»
```

Команда `!play`: подключиться, добавить в очередь и, если тишина, запустить:

```python
@bot.command()
async def play(ctx, *, query: str):
    if ctx.author.voice is None:
        return await ctx.send("Сначала зайди в голосовой канал.")
    if ctx.voice_client is None:
        await ctx.author.voice.channel.connect()
    queues.setdefault(ctx.guild.id, deque()).append(query)
    await ctx.send(f"➕ В очередь: {query}")
    if not ctx.voice_client.is_playing():                # ничего не играет → стартуем
        await play_next(ctx.guild)
```

> **В очереди храни запрос/название, а не готовый URL.** Прямая стрим-ссылка от YouTube **протухает** через несколько часов — трек в конце длинной очереди не проиграется. Извлекай URL заново в `play_next`, прямо перед воспроизведением.

### Управление воспроизведением

```python
@bot.command()
async def skip(ctx):
    ctx.voice_client.stop()          # stop → срабатывает after → следующий трек

@bot.command()
async def pause(ctx):
    ctx.voice_client.pause()

@bot.command()
async def resume(ctx):
    ctx.voice_client.resume()

@bot.command()
async def volume(ctx, percent: int):
    ctx.voice_client.source.volume = percent / 100   # PCMVolumeTransformer.volume: 0.0–1.0

@bot.command()
async def stop(ctx):
    queues.get(ctx.guild.id, deque()).clear()
    await ctx.voice_client.disconnect()
```

- **`is_playing()` / `is_paused()`** — состояние проигрывателя;
- **`skip` = `stop()`**: остановка текущего трека штатно вызывает `after`, а тот берёт следующий из очереди.

> **Стриминг с YouTube нарушает его условия использования.** Для учебного/личного бота это обычная практика, но у публичного бота это правовой риск — «взрослые» проекты берут аудио из легальных источников (свои файлы, лицензированные API).

## Модерация и логи

Модерацию собирают из трёх слоёв: **своя система наказаний** (варны с БД и эскалацией), **нативная автомодерация** Discord (фильтры на его серверах) и **логирование** — что происходит на сервере, в отдельный канал. Плюс **audit log** — журнал самого Discord, ловящий даже действия, сделанные вручную мимо бота.

> **Ловушка №1 всей модерации — иерархия ролей.** Бот **не может** кикнуть/забанить того, чья высшая роль **выше или равна** высшей роли бота — Discord вернёт `Forbidden (403)`. Роль бота должна стоять в списке **выше** целей, и у него должны быть сами права (`ban_members`, `kick_members`, `moderate_members`). Это причина №1 «почему бан падает».

### Warn-система с эскалацией

Предупреждения копят в БД, а по их числу **автоматически** ужесточают наказание. Таблица хранит каждый варн (кто, кому, за что, когда):

```python
import datetime
from sqlalchemy import BigInteger, Integer, String, DateTime, select, func, delete

class Warn(Base):
    __tablename__ = "warns"
    id:         Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    guild_id:   Mapped[int] = mapped_column(BigInteger, index=True)
    user_id:    Mapped[int] = mapped_column(BigInteger, index=True)
    moderator:  Mapped[int] = mapped_column(BigInteger)
    reason:     Mapped[str] = mapped_column(String)
    created_at: Mapped[datetime.datetime] = mapped_column(DateTime, default=datetime.datetime.utcnow)

@bot.tree.command(name="warn", description="Выдать предупреждение")
@app_commands.checks.has_permissions(moderate_members=True)     # команда только для модераторов
async def warn(interaction: discord.Interaction, member: discord.Member, reason: str):
    async with Sessionmaker() as session:
        session.add(Warn(guild_id=interaction.guild.id, user_id=member.id,
                         moderator=interaction.user.id, reason=reason))
        await session.commit()
        count = await session.scalar(                           # сколько всего варнов
            select(func.count()).select_from(Warn)
            .where(Warn.guild_id == interaction.guild.id, Warn.user_id == member.id))

    await interaction.response.send_message(f"⚠ {member.mention} предупреждён ({count}). Причина: {reason}")

    if count >= 5:                                               # эскалация по числу
        await member.ban(reason="5 предупреждений")
    elif count >= 3:
        await member.timeout(datetime.timedelta(hours=1), reason="3 предупреждения")
```

Просмотр и сброс — `/warnings` (список) и `/clearwarns` (`delete(Warn).where(...)`):

```python
@bot.tree.command(name="warnings")
async def warnings(interaction: discord.Interaction, member: discord.Member):
    async with Sessionmaker() as session:
        rows = (await session.scalars(select(Warn)
            .where(Warn.guild_id == interaction.guild.id, Warn.user_id == member.id))).all()
    if not rows:
        return await interaction.response.send_message("Предупреждений нет.", ephemeral=True)
    text = "\n".join(f"{i+1}. {w.reason} — <@{w.moderator}>" for i, w in enumerate(rows))
    await interaction.response.send_message(f"Предупреждения {member.mention}:\n{text}", ephemeral=True)
```

### Лог событий в отдельный канал

Удаления, правки, входы и выходы логируют, слушая события и отправляя [embed](#embeds--родные-карточки-discord) в мод-канал:

```python
LOG_CHANNEL_ID = 123456789012345678

async def mod_log(guild: discord.Guild, embed: discord.Embed):
    if channel := guild.get_channel(LOG_CHANNEL_ID):
        await channel.send(embed=embed)

@bot.event
async def on_message_delete(message: discord.Message):
    if message.author.bot:
        return
    embed = discord.Embed(title="🗑 Сообщение удалено", colour=discord.Colour.red(),
                          description=message.content or "(без текста)")
    embed.add_field(name="Автор", value=message.author.mention)
    embed.add_field(name="Канал", value=message.channel.mention)
    await mod_log(message.guild, embed)

@bot.event
async def on_member_remove(member: discord.Member):             # вышел/кикнут/забанен
    embed = discord.Embed(title="📤 Участник покинул сервер", colour=discord.Colour.orange(),
                          description=f"{member} ({member.id})")
    await mod_log(member.guild, embed)
```

> **`on_message_delete` ловит не всё.** Событие срабатывает только для сообщений в **кэше** бота (нужен intent `message_content`); старое сообщение, которого нет в памяти, удалится «тихо». И само событие **не говорит, кто удалил** — эту информацию берут из audit log (ниже), но для удалений Discord её отдаёт неточно (батчит записи). Для входов/выходов нужен привилегированный intent `members`.

### Audit log — журнал самого Discord

**Audit log** — встроенный журнал сервера: кто кого забанил, кто менял роли и каналы. Он фиксирует действия **даже если их сделали вручную** в клиенте, мимо твоего бота. Два способа с ним работать:

```python
# 1) ПРОЧИТАТЬ историю — асинхронный итератор с фильтром по действию
async for entry in guild.audit_logs(limit=10, action=discord.AuditLogAction.ban):
    print(entry.user, "забанил", entry.target, "—", entry.reason)   # кто, кого, почему

# 2) РЕАГИРОВАТЬ в реальном времени — событие на новую запись журнала
@bot.event
async def on_audit_log_entry_create(entry: discord.AuditLogEntry):
    if entry.action is discord.AuditLogAction.kick:
        embed = discord.Embed(title="👢 Кик", colour=discord.Colour.dark_orange())
        embed.add_field(name="Кого", value=str(entry.target))       # цель действия
        embed.add_field(name="Модератор", value=str(entry.user))    # кто сделал
        embed.add_field(name="Причина", value=entry.reason or "—", inline=False)
        await mod_log(entry.guild, embed)
```

- **`AuditLogEntry`**: `.action` (что), `.user` (кто сделал), `.target` (над кем/чем), `.reason`, `.created_at`, `.changes` (что именно поменялось).
- **`on_audit_log_entry_create`** — ловит **ручную** модерацию тоже: банните вы через бота или правой кнопкой в клиенте — событие придёт. Боту нужно право **«Просмотр журнала аудита»**.

### Нативная автомодерация (AutoMod API)

Discord умеет фильтровать сообщения **на своей стороне** — правила создаёт `guild.create_automod_rule`, и они срабатывают **до** того, как сообщение дойдёт до бота, без intent `message_content`:

```python
rule = await guild.create_automod_rule(
    name="Фильтр слов",
    event_type=discord.AutoModRuleEventType.message_send,
    trigger=discord.AutoModTrigger(
        type=discord.AutoModRuleTriggerType.keyword,
        keyword_filter=["*запретное*", "спам-фраза"],           # шаблоны (* — любой остаток)
    ),
    actions=[                                                    # у каждого действия — свой параметр
        discord.AutoModRuleAction(type=discord.AutoModRuleActionType.block_message),
        discord.AutoModRuleAction(type=discord.AutoModRuleActionType.send_alert_message,
                                  channel_id=LOG_CHANNEL_ID),    # уведомить модераторов
        discord.AutoModRuleAction(type=discord.AutoModRuleActionType.timeout,
                                  duration=datetime.timedelta(minutes=10)),  # мут нарушителю
    ],
    enabled=True,
)

@bot.event
async def on_automod_action(execution: discord.AutoModAction):   # лог срабатываний
    print(f"AutoMod поймал {execution.member}: {execution.matched_keyword}")
```

- **Типы триггеров** (`AutoModRuleTriggerType`): `keyword` (слова/regex), `spam`, `keyword_preset` (готовые списки Discord), `mention_spam` (массовые упоминания).
- **Действия** (`AutoModRuleActionType`): `block_message`, `send_alert_message` (нужен `channel_id`), `timeout` (нужен `duration`). У одного действия — **только один** из `channel_id`/`duration`/`custom_message`.

Свой фильтр в `on_message` берут, когда нужна логика, которой AutoMod не умеет (сверка с БД, сложные условия) — но он работает в твоём боте и требует `message_content`:

| | **Нативный AutoMod** | **Свой `on_message`** |
|---|---|---|
| Где работает | на серверах Discord | в твоём боте |
| Когда срабатывает | **до** отправки сообщения | после получения |
| Intent `message_content` | не нужен | нужен |
| Гибкость | keyword / spam / mention / regex | любая логика |
| Когда брать | базовые фильтры «из коробки» | сложные правила, интеграция с БД |

```python
import re
INVITE = re.compile(r"discord\.gg/\w+")

@bot.event
async def on_message(message: discord.Message):
    if message.author.bot or message.guild is None:
        return
    if INVITE.search(message.content):                          # чужое приглашение
        await message.delete()
        await message.channel.send(f"{message.author.mention}, реклама запрещена.", delete_after=5)
        return
    await bot.process_commands(message)                         # ← иначе команды перестанут работать
```

## Связи

- [SQLAlchemy](../Библиотеки/Сторонние/ORM/SQLAlchemy.md) — async-ORM под всеми системами выше (уровни, экономика, голосовое время, варны); атомарные `UPDATE ... WHERE`;
- [aiogram](../Фреймворки/aiogram.md) — «сестра» для Telegram: та же async-модель, объекты платформы + обработчики; сравнение выше;
- [Asyncio Event Loop и aiohttp](../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — asyncio, на котором стоит discord.py; почему блокирующий рендер Pillow надо уносить в поток;
- Асинхронность — I/O-bound (сеть) против CPU-bound (рисование картинки): почему для второго нужен отдельный поток/процесс;
- [concurrent.futures](../Библиотеки/Модули/Параллелизм/concurrent.futures.md) — `asyncio.to_thread` под капотом; пул потоков для синхронного кода;
- [Работа с файлами Python](../Ядро%20языка/Работа%20с%20файлами%20Python.md) — `io.BytesIO` как файл в памяти для картинки без записи на диск;
- [Redis](../Библиотеки/Сторонние/Redis.md) — хранилище данных бота (уровни, настройки) между перезапусками;
- [Развертывание проекта](../DevOps/Развертывание%20проекта.md) — бота катят как сервис/контейнер, gateway-соединение живёт постоянно.

## Источники

- [discord.py — Documentation](https://discordpy.readthedocs.io/en/stable/)
- [discord.py — Quickstart](https://discordpy.readthedocs.io/en/stable/quickstart.html)
- [discord.py — Interactions / UI API](https://discordpy.readthedocs.io/en/stable/interactions/api.html)
- [discord.py — примеры (GitHub: examples/views)](https://github.com/Rapptz/discord.py/tree/master/examples)
- [discord.py 2.0+ slash commands — гайд (AbstractUmbra gist)](https://gist.github.com/AbstractUmbra/a9c188797ae194e592efe05fa129c57f)
- [Discord Developer Portal — Intents](https://discord.com/developers/docs/topics/gateway#gateway-intents)
- [Pillow (PIL) — Documentation](https://pillow.readthedocs.io/en/stable/)
