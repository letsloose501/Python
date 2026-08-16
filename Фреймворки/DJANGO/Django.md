[← Оглавление](../../README.md)

# Django

> _«The web framework for perfectionists with deadlines»_ — слоган Django

---

**Django** — «батарейки в комплекте» фреймворк для веб-приложений на [Python](../../Python.md): из коробки даёт ORM, админку, маршрутизацию, шаблоны, аутентификацию и защиту. Он сам генерирует структуру проекта по соглашениям, поэтому даже новый разработчик знает, где что искать. Задача разработчика — не собирать инфраструктуру с нуля, а заполнять готовые слоты своей логикой.

## Проект и приложение

Django различает две единицы:

- **Проект** — весь сайт целиком: настройки, база данных, список подключённых приложений.
- **Приложение** — изолированный кусок функциональности (пользователи, блог, корзина). Приложения **переиспользуются** между проектами — ближайшая аналогия из Python: модуль/пакет.

В проекте может быть одно приложение (`main`) или несколько.

```
myshop/                 ← проект
├── manage.py           запуск команд в контексте проекта
├── myshop/             пакет настроек проекта
│   ├── settings.py     настройки (БД, приложения, статика)
│   ├── urls.py         корневые маршруты
│   ├── wsgi.py/asgi.py точки входа для сервера
└── main/               ← приложение
    ├── models.py       данные (таблицы)
    ├── views.py        логика ответа на запрос
    ├── admin.py        регистрация моделей в админке
    ├── apps.py         настройки приложения
    └── tests.py        тесты
```

## Установка и создание

```bash
pip install django
django-admin startproject myshop .   # создать проект в текущей папке (точка!)
cd myshop
python manage.py startapp main       # создать приложение main
```

**`manage.py`** — утилита запуска команд в контексте проекта. Главные:

```bash
python manage.py runserver       # поднять сервер разработки (localhost:8000)
python manage.py makemigrations  # собрать изменения моделей в миграцию
python manage.py migrate         # применить миграции к базе
python manage.py createsuperuser # создать администратора
python manage.py shell           # интерактивный shell с загруженным Django
```

> Новое приложение надо **зарегистрировать** — добавить в `INSTALLED_APPS` в `settings.py`. Иначе Django его не увидит: не найдёт модели, шаблоны, миграции.

## MVT — как Django делит ответственность

Django следует паттерну, который сам называет **MVT** (Model–View–Template) — его вариант классического MVC. Идея — не мешать всё в кучу, а разнести по ролям:

| Слой Django | Отвечает за | Классический MVC |
|---|---|---|
| **Model** (`models.py`) | данные и работа с БД | Model |
| **Template** (`.html`) | как выглядит страница | View |
| **View** (`views.py`) | логику: принять запрос, собрать данные, выбрать шаблон | Controller |

> Путаница в терминах — норма: то, что в MVC зовётся «контроллером», в Django называется **View**, а «представление» (отображение) вынесено в **Template**. Держи в голове роли, а не названия.

```
Запрос ─▶ urls.py ─▶ View ─┬─▶ Model ─▶ база данных
                           └─▶ Template ─▶ HTML ─▶ Ответ
```

## Клиент и сервер

Веб-приложение реализует **клиент-серверное** взаимодействие:

- **Клиент** — программа, которая запрашивает информацию (браузер, мобильное приложение).
- **Сервер** — программа, которая её отдаёт (Django).

Клиент шлёт **запрос** по протоколу HTTP, сервер возвращает **ответ** — HTML-страницу или JSON-данные. Обычно они на разных машинах, но могут быть и на одной.

## View — обработка запроса

**View** — функция (или класс), которая принимает `request` и возвращает ответ. Простейший — через `HttpResponse`, страница с шаблоном — через `render`:

```python
from django.shortcuts import render, get_object_or_404
from django.http import HttpResponse
from .models import Product

def test_page(request):
    return HttpResponse('Всем привет')            # просто текст

def products(request):
    products = Product.objects.all()               # достаём данные из БД
    return render(request, 'products.html', {'products': products})  # рендерим шаблон

def product_details(request, product_id):
    product = get_object_or_404(Product, id=product_id)   # 404, если не найдено
    return render(request, 'details.html', {'product': product})
```

- **`render(request, шаблон, контекст)`** — собирает HTML из шаблона и словаря-контекста.
- **`get_object_or_404(Модель, ...)`** — вернуть объект или автоматически отдать 404. Избавляет от ручных `try/except`.

## URL-маршрутизация

**Маршрутизация** (роутинг) связывает URL-адрес с view-функцией. Описывается в `urls.py` списком `urlpatterns`:

```python
from django.contrib import admin
from django.urls import path
from main.views import test_page, products, product_details

urlpatterns = [
    path('admin/', admin.site.urls),
    path('index/', test_page),
    path('products/', products, name='list'),
    path('products/<int:product_id>/', product_details, name='details'),
]
```

- **`path('адрес/', view, name=...)`** — первый аргумент это путь в адресной строке, второй — вызываемая view.
- **`name`** — имя маршрута, чтобы ссылаться на URL по имени (`{% url 'details' ... %}`), а не по строке. Тогда при смене адреса ссылки не сломаются.
- **`<int:product_id>`** — **конвертер пути**: вытаскивает часть URL, приводит к типу и передаёт во view как аргумент.

### Конвертеры пути

Стандартные конвертеры: `str`, `int`, `slug`, `uuid`, `path`. Если нужного типа нет — можно **написать свой** конвертер: класс с `regex`, `to_python` (строка → объект) и `to_url` (объект → строка).

```python
from datetime import datetime
from django.urls import register_converter

class DateConverter:
    regex = r'[0-9]{4}-[0-9]{2}-[0-9]{2}'          # что ловим в URL
    format = '%Y-%m-%d'
    def to_python(self, value: str) -> datetime:    # URL → объект для view
        return datetime.strptime(value, self.format)
    def to_url(self, value: datetime) -> str:       # объект → URL (для reverse)
        return value.strftime(self.format)

register_converter(DateConverter, 'date')           # регистрируем под именем 'date'

# теперь в маршруте:  path('reports/<date:dt>/', user_report)
# во view dt придёт уже как datetime — валидация не нужна
```

## Шаблоны

**Template** — HTML с расширениями Django для подстановки данных. Файлы кладут в папку `templates/` приложения; передаёт в них данные словарь-**контекст** из `render`.

- **`{{ переменная }}`** — подстановка значения.
- **`{% тег %}`** — логика: циклы, условия, наследование. Циклы и блоки обязательно закрываются (`{% endfor %}`).
- Методы и свойства вызывают **без скобок** — Django сам поймёт (`product.image.url`).

```html
<body>
  {% for product in products %}
    <h2>{{ product.name }}</h2>
    <img src="{{ product.image.url }}" width="300">
    <a href="{% url 'details' product_id=product.id %}">Подробнее</a>
  {% endfor %}
</body>
```

Мощные теги — наследование шаблонов (`{% extends 'base.html' %}`, `{% block %}`) и включение (`{% include %}`): общий каркас в `base.html`, страницы его расширяют.

## Админка

Django даёт готовую **админ-панель** для управления данными. Модель надо в ней зарегистрировать (`admin.py`):

```python
from django.contrib import admin
from .models import Product

admin.site.register(Product)          # простой вариант
```

Создать администратора — `python manage.py createsuperuser`. Тонкую настройку отображения (какие столбцы, фильтры) делают через класс `ModelAdmin` — подробно в [ORM Django](../../Фреймворки/DJANGO/ORM%20Django.md#админка--настройка-отображения).

## Пагинация

**Пагинация** — вывод контента постранично. Встроенный `Paginator` разбивает список на страницы:

```python
from django.core.paginator import Paginator

def products(request):
    products = Product.objects.all()
    paginator = Paginator(products, 10)            # по 10 на страницу
    page = paginator.get_page(request.GET.get('page'))  # номер из ?page=2
    return render(request, 'products.html', {'page': page})
```

В шаблоне — переключатели, с проверкой, что страница существует:

```html
{% if page.has_previous %}
  <a href="?page={{ page.previous_page_number }}">Назад</a>
{% endif %}
{% if page.has_next %}
  <a href="?page={{ page.next_page_number }}">Вперёд</a>
{% endif %}
```

> **Параметры запроса** приходят через `request.GET` (словарь) — например `?page=2`. Все значения там **строки**: для числа приводи вручную (`int(...)`).

## Сквозной пример: каталог товаров

Соберём минимальный каталог — от модели до страницы. Порядок именно такой.

**Шаг 1. Модель** (`main/models.py`) — что храним:

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField()
    price = models.IntegerField()
    image = models.ImageField(upload_to='products')

    def __str__(self):                 # как объект выглядит в админке и логах
        return self.name
```

**Шаг 2. Миграции** — создать таблицу в БД:

```bash
python manage.py makemigrations       # собрать изменения модели в миграцию
python manage.py migrate              # применить к базе
```

**Шаг 3. View** (`main/views.py`) — достать и отдать:

```python
from django.shortcuts import render
from .models import Product

def products(request):
    return render(request, 'products.html', {'products': Product.objects.all()})
```

**Шаг 4. Маршрут** (`urls.py`):

```python
path('products/', products, name='list'),
```

**Шаг 5. Шаблон** (`main/templates/products.html`) — как показать (см. блок «Шаблоны» выше).

**Шаг 6. Медиа-файлы** — чтобы отдавались картинки, в `settings.py` и `urls.py`:

```python
# settings.py
MEDIA_URL = 'media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# urls.py — только для разработки (DEBUG)
from django.conf import settings
from django.conf.urls.static import static
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

**Как это читать.** Данные описаны в модели → миграции создают таблицу → view берёт объекты и передаёт в шаблон → маршрут связывает адрес с view → шаблон рисует HTML. Это и есть цикл MVT в действии.

## Связи

- [ORM Django](../../Фреймворки/DJANGO/ORM%20Django.md) — модели, связи и запросы к БД подробно;
- [Кастомная модель User](../../Фреймворки/DJANGO/Кастомная%20модель%20User.md) — как расширить встроенного пользователя и почему это решают до первой миграции;
- [Сигналы Django](../../Фреймворки/DJANGO/Сигналы%20Django.md) — реакция на события моделей (`post_save`, `pre_delete`) и когда её лучше не использовать;
- [API](../../Фреймворки/DJANGO/API.md) — когда вместо HTML-страниц нужен REST API (Django REST Framework);
- [Django с многими приложениями](../../Фреймворки/DJANGO/Django%20с%20многими%20приложениями.md) — Django как чистый backend для отдельного фронтенда (SPA);
- [Тестирование Django приложений](../../Фреймворки/DJANGO/Тестирование%20Django%20приложений.md) — как покрывать всё это тестами;
- [Развертывание проекта](../../DevOps/Развертывание%20проекта.md) — как выкатить Django-проект на боевой сервер (gunicorn, nginx);
- [FLASK](../../Фреймворки/FLASK.md), [FastAPI](../../Фреймворки/FastAPI.md) — более лёгкие альтернативы, когда «батарейки» Django избыточны.

## Источники

- [Django documentation](https://docs.djangoproject.com/en/stable/)
- [Django — Writing your first app (tutorial)](https://docs.djangoproject.com/en/stable/intro/tutorial01/)
- [URL dispatcher — path converters](https://docs.djangoproject.com/en/stable/topics/http/urls/#path-converters)
- [Pagination](https://docs.djangoproject.com/en/stable/topics/pagination/)
