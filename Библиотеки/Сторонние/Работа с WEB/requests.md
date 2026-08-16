[← Оглавление](../../../README.md)

# requests

> _«HTTP for Humans» — запрос к сайту в одну строку_

---

**requests** — самая популярная библиотека для HTTP-запросов на [Python](../../../Python.md): получить страницу, дёрнуть API, отправить форму. Она синхронная (ждёт каждый ответ) и предельно простая — за это её и любят. Для тысяч одновременных запросов берут асинхронный аналог [aiohttp](../../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md).

```bash
pip install requests
```

## Запросы

Под каждый HTTP-метод — своя функция:

```python
import requests

r = requests.get('https://api.example.com/users')          # получить
r = requests.post('https://api.example.com/users', json={'name': 'Alex'})  # создать
r = requests.put(url, json=data)                            # обновить
r = requests.delete(url)                                    # удалить
```

## Параметры, заголовки, тело

```python
requests.get(url, params={'q': 'python', 'page': 2})   # ?q=python&page=2
requests.get(url, headers={'Authorization': 'Bearer token'})   # заголовки
requests.post(url, json={'key': 'value'})              # тело как JSON
requests.post(url, data={'field': 'value'})            # тело как форма
```

## Ответ

Объект ответа несёт статус, тело и заголовки:

```python
r = requests.get('https://api.example.com/users')
r.status_code      # 200 — код ответа
r.json()           # тело как объект Python (если JSON)
r.text             # тело как строка (HTML, текст)
r.headers          # заголовки ответа
r.ok               # True, если статус < 400
r.raise_for_status()   # бросить исключение при коде ошибки (4xx/5xx)
```

## Сессии

**`Session`** переиспользует TCP-соединение и хранит cookies между запросами — быстрее и удобнее для серии обращений к одному сервису:

```python
with requests.Session() as s:
    s.headers.update({'Authorization': 'Bearer token'})   # общие заголовки
    s.get('https://api.example.com/profile')              # cookie и заголовки сохраняются
    s.get('https://api.example.com/orders')
```

## Сквозной пример: забрать данные из API

```python
import requests

def get_user(user_id: int) -> dict:
    r = requests.get(f'https://api.example.com/users/{user_id}',
                     headers={'Authorization': 'Bearer token'},
                     timeout=5)          # 1. запрос с таймаутом
    r.raise_for_status()                 # 2. упасть, если код ошибки
    return r.json()                      # 3. вернуть тело как dict

user = get_user(1)
print(user['name'])
```

**Как это читать.** `get` шлёт запрос, `timeout` не даёт зависнуть навечно, `raise_for_status` превращает коды 4xx/5xx в исключение (иначе легко проглядеть ошибку), `json()` разбирает ответ. Для парсинга **HTML**-страниц ответ `r.text` отдают в [BeautifulSoup](../../../Библиотеки/Сторонние/Работа%20с%20WEB/BeautifulSoup.md).

## Скачать файл (бинарные данные)

Для картинок, архивов и прочих бинарников берут не `r.text`, а **`r.content`** (байты) и пишут файл в режиме `'wb'`:

```python
import requests

breed = "husky"
# публичный API dog.ceo — вернёт ссылку на случайное фото породы
r = requests.get(f"https://dog.ceo/api/breed/{breed}/images/random")
if r.status_code == 400:
    print("Такая порода не найдена")
else:
    image_url = r.json()["message"]
    filename = image_url.split("/")[-1]           # имя файла из URL

    img = requests.get(image_url)                 # скачать саму картинку
    with open(f"images/{filename}", "wb") as f:   # 'wb' — бинарная запись
        f.write(img.content)                      # .content — байты, не .text
```

## Класс-клиент к API — инкапсуляция токена и запросов

Когда к одному сервису много обращений, логику оборачивают в класс: токен и базовый URL — в полях, общий код запроса — в приватном методе `_make_request`. `requests.request(method, ...)` — универсальная форма вместо отдельных `get`/`put`.

```python
import requests

class YandexDiskClient:
    """Инкапсулирует токен и HTTP-логику работы с Яндекс.Диском."""

    def __init__(self, token: str):
        self.base_url = "https://cloud-api.yandex.net/v1/disk"
        self.headers = {"Authorization": f"OAuth {token}"}

    def _make_request(self, method: str, endpoint: str, **kwargs):
        url = f"{self.base_url}/{endpoint}"
        try:
            r = requests.request(method, url, headers=self.headers, **kwargs)
            r.raise_for_status()                       # 4xx/5xx → исключение
            return r.json() if r.content else {}
        except requests.exceptions.RequestException as e:
            print(f"Ошибка {method} {endpoint}: {e}")
            return None

    def create_folder(self, path: str) -> bool:
        return self._make_request("PUT", "resources", params={"path": path}) is not None

    def get_folder_info(self, path: str) -> dict:
        return self._make_request("GET", "resources", params={"path": path})

# токен НЕ хардкодят в коде — читают из конфига вне репозитория:
import json
with open("config.json") as f:
    disk = YandexDiskClient(json.load(f)["token"])
disk.create_folder("backup")
```

> **Безопасность — не хардкодь токен.** OAuth-токен даёт полный доступ к диску; если он попадёт в код и уедет на GitHub — утечка. Держи секреты в `config.json` / переменных окружения и добавь их в `.gitignore`. `_make_request` тут делает и второе доброе дело — единая обработка ошибок вместо `try/except` в каждом методе.

## Связи

- [Asyncio Event Loop и aiohttp](../../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — асинхронный аналог для тысяч параллельных запросов;
- [BeautifulSoup](../../../Библиотеки/Сторонние/Работа%20с%20WEB/BeautifulSoup.md) — разбор HTML, полученного через `requests.get(...).text`;
- [API](../../../Фреймворки/DJANGO/API.md) — REST API, которые requests и дёргает;
- [Oauth 2.0](../../../Библиотеки/Сторонние/Работа%20с%20WEB/Oauth%202.0.md#как-это-делают-на-python) — `requests-oauthlib`: обмен кода на токен и сессия, которая сама шлёт `Authorization`;
- [Selenium](../../../Библиотеки/Сторонние/Работа%20с%20WEB/Selenium.md) — когда страница отрисовывается JavaScript и `requests` видит пустой каркас.

## Источники

- [Requests documentation](https://requests.readthedocs.io/en/latest/)
- [Quickstart](https://requests.readthedocs.io/en/latest/user/quickstart/)
- [Advanced usage — Sessions](https://requests.readthedocs.io/en/latest/user/advanced/)
