[← Оглавление](../../../README.md)

# BeautifulSoup

> _HTML-каша → дерево, из которого достаёшь что нужно_

---

**BeautifulSoup** — библиотека для разбора HTML и XML: превращает текст страницы в дерево объектов, по которому удобно искать и вытаскивать данные. Основной инструмент **веб-скрейпинга** (сбора данных с сайтов). Сама страниц не скачивает — HTML ей обычно приносит [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md).

```bash
pip install beautifulsoup4 lxml
```

## Создание «супа»

Текст HTML отдают конструктору с указанием **парсера**:

```python
from bs4 import BeautifulSoup

html = '<html><body><h1>Заголовок</h1><a href="/link">Ссылка</a></body></html>'
soup = BeautifulSoup(html, 'html.parser')     # или 'lxml' — быстрее
```

Парсеры: `html.parser` (встроенный, без зависимостей), `lxml` (быстрый, нужен пакет `lxml`).

## Поиск элементов

Два главных метода — `find` (первый совпавший) и `find_all` (все):

```python
soup.find('h1')                       # первый <h1>
soup.find_all('a')                    # все ссылки <a> (список)
soup.find('div', class_='price')      # <div class="price"> (class_ с подчёркиванием!)
soup.find('a', id='main')             # по id
```

Или через **CSS-селекторы** (как в вёрстке):

```python
soup.select('div.price')              # все div с классом price
soup.select_one('#main a')            # первая ссылка внутри #main
```

## Извлечение данных

Из найденного тега берут текст и атрибуты:

```python
tag = soup.find('a')
tag.text              # текст внутри тега: 'Ссылка'
tag['href']           # значение атрибута: '/link'
tag.get('href')       # то же, но None вместо ошибки, если атрибута нет
tag.attrs             # все атрибуты словарём
```

## Сквозной пример: собрать заголовки со страницы

Связка [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md) (скачать) + BeautifulSoup (разобрать) — классика скрейпинга.

```python
import requests
from bs4 import BeautifulSoup

# 1. скачать страницу
html = requests.get('https://example.com/blog').text

# 2. разобрать в дерево
soup = BeautifulSoup(html, 'html.parser')

# 3. найти все заголовки статей и вытащить текст + ссылку
for article in soup.select('article'):
    title = article.find('h2').text
    link = article.find('a')['href']
    print(title, '→', link)
```

**Как это читать.** requests приносит HTML строкой, BeautifulSoup превращает его в дерево, `select`/`find` находят нужные узлы, `.text` и `['href']` достают данные. Если страница пустая, а контент подгружается JavaScript — requests его не увидит, тогда нужен [Selenium](../../../Библиотеки/Сторонние/Работа%20с%20WEB/Selenium.md).

## Пример посложнее — обход Хабра с фильтром по ключевым словам

Реальный скрейпер двухуровневый: со **страницы списка** собирает ссылки на статьи, затем заходит в **каждую статью** за деталями, и оставляет только те, где в тексте есть нужные слова (через [регулярку](../../../Библиотеки/Модули/Текст%20и%20строки/re.md)). Плюс подменяет `User-Agent` через `fake_headers`, чтобы сайт не отдал заглушку боту.

```python
import requests, bs4, re
from fake_headers import Headers

keywords = ["дизайн", "фото", "web", "python"]
# \b...\b — слово целиком, re.escape — обезопасить спецсимволы в словах
pattern = re.compile(r"\b(" + "|".join(map(re.escape, keywords)) + r")\b", re.IGNORECASE)

head = Headers(browser="chrome", os="windows").generate()   # случайный правдоподобный UA
response = requests.get("https://habr.com/ru/articles/", headers=head)
soup = bs4.BeautifulSoup(response.text, features="lxml")

# 1. со страницы списка — все карточки статей
articles_block = soup.select_one("div.tm-articles-list")
articles_list = articles_block.select("article.tm-articles-list__item")

parsed_data = []
for article in articles_list:
    # 2. вытащить ссылку на статью из карточки
    a = article.select_one("h2.tm-title.tm-title_h2 a")
    link = "https://habr.com" + a["href"]

    # 3. зайти в саму статью за заголовком, датой и текстом
    art = bs4.BeautifulSoup(requests.get(link, headers=head).text, features="lxml")
    header = art.select_one("h1").text.strip()
    date = art.select_one("time")["title"]
    text = art.select_one("div.article-formatted-body").text

    # 4. фильтр: оставить статью, только если в тексте есть ключевое слово
    if pattern.search(text):
        parsed_data.append({"date": date, "header": header, "link": link})

for a in parsed_data:
    print(f"{a['date']} – {a['header']} – {a['link']}")
```

Ключевые приёмы: `select_one(...)` для цепочки «карточка → заголовок → `<a>`», переход по `href` на второй уровень, и предварительно скомпилированный `pattern` для быстрого поиска слов в тексте каждой статьи.

> **Хрупкость селекторов.** Классы вроде `tm-articles-list__item` — это текущая вёрстка Хабра; при редизайне сайта селекторы придётся обновить. Скрейпер по чужой разметке всегда временный.

## Связи

- [requests](../../../Библиотеки/Сторонние/Работа%20с%20WEB/requests.md) — скачивает HTML, который BeautifulSoup разбирает;
- [Selenium](../../../Библиотеки/Сторонние/Работа%20с%20WEB/Selenium.md) — для страниц с JS-контентом (BeautifulSoup видит только статический HTML);
- [Grab](../../../Библиотеки/Сторонние/Работа%20с%20WEB/Grab.md) — фреймворк, объединяющий загрузку и парсинг в одном инструменте;
- [re](../../../Библиотеки/Модули/Текст%20и%20строки/re.md) — регулярные выражения; иногда парсинг догоняют ими, но по HTML лучше BeautifulSoup.

## Источники

- [Beautiful Soup documentation](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
- [Searching the tree (find / find_all)](https://www.crummy.com/software/BeautifulSoup/bs4/doc/#searching-the-tree)
- [CSS selectors (select)](https://www.crummy.com/software/BeautifulSoup/bs4/doc/#css-selectors)
