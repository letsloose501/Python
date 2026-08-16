[← Оглавление](../../../README.md)

# email

> _Письмо — это не просто текст: у него есть заголовки, тело и вложения по стандарту MIME. `email` собирает и разбирает эту структуру._

---

**email** — модуль для **построения и разбора** сообщений электронной почты по стандарту **MIME**. Письмо состоит из заголовков (`From`, `To`, `Subject`), тела (текст и/или HTML) и вложений — `email` даёт объектную модель этой структуры.

> Важно: `email` **готовит и разбирает** сообщение, но **не отправляет** его. Отправка — модуль `smtplib` (SMTP), приём — `imaplib`/`poplib`. `email` — про содержимое письма.

Зачем на бэкенде: приложения шлют уведомления, чеки, письма для сброса пароля. Сначала `email` собирает корректное сообщение, затем `smtplib` его отправляет.

## Собрать письмо

Современный интерфейс — класс **`EmailMessage`**:

```python
from email.message import EmailMessage

msg = EmailMessage()
msg["From"] = "app@example.com"
msg["To"] = "user@example.com"
msg["Subject"] = "Подтверждение"
msg.set_content("Здравствуйте! Ваш заказ принят.")     # текстовое тело
```

## Вложения

```python
with open("чек.pdf", "rb") as f:
    msg.add_attachment(
        f.read(),
        maintype="application", subtype="pdf",
        filename="чек.pdf",
    )
```

Под капотом вложение кодируется в [base64](../../../Библиотеки/Модули/Форматы%20данных/base64.md) — потому что почта исторически передаёт только текст, а PDF двоичный.

## Структура MIME

Письмо с текстом, HTML и вложением — это дерево «частей» (multipart). MIME описывает, где что:

```
multipart/mixed
├── multipart/alternative
│   ├── text/plain   ← текстовая версия
│   └── text/html    ← HTML-версия (покажется, если клиент умеет)
└── application/pdf  ← вложение
```

`EmailMessage` строит это дерево за тебя: `set_content` задаёт основное тело, `add_alternative` — HTML-версию, `add_attachment` — вложения.

## Разобрать входящее письмо

```python
from email import message_from_bytes

msg = message_from_bytes(сырые_байты)
msg["Subject"]                       # заголовок
msg.get_body(preferencelist=("plain",)).get_content()   # текст письма
for часть in msg.iter_attachments(): # перебрать вложения
    часть.get_filename()
```

## Связи

- [base64](../../../Библиотеки/Модули/Форматы%20данных/base64.md) — кодирование вложений (MIME-часть → текст);
- Прикладной уровень — где живут протоколы почты (SMTP/IMAP/POP3);
- [Работа с сервером](../../../DevOps/Работа%20с%20сервером.md) — отправка писем из приложения как типовая задача.

## Источники

- [Python Docs — email](https://docs.python.org/3/library/email.html)
