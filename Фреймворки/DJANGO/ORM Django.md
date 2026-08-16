[← Оглавление](../../README.md)

# ORM Django

> _Пишешь на Python — Django пишет SQL за тебя_

---

**ORM** (Object-Relational Mapping) — слой, который связывает классы Python с таблицами базы данных: класс — это таблица, объект — строка, атрибут — столбец. Ты работаешь с данными через привычные объекты и методы, а ORM переводит это в SQL. Django ORM встроен в фреймворк (в отличие от независимой [SQLAlchemy](../../Библиотеки/Сторонние/ORM/SQLAlchemy.md)) и тесно связан с моделями, миграциями и админкой.

## Модель — это таблица

**Модель** — класс-наследник `models.Model`. Каждый атрибут-поле становится столбцом; поле `id` (первичный ключ) Django добавляет автоматически.

```python
from django.db import models

class Car(models.Model):
    brand = models.CharField(max_length=50)
    model = models.CharField(max_length=50)
    color = models.CharField(max_length=20)

    def __str__(self):                       # как объект отображается в админке/логах
        return f'{self.brand} {self.model}: {self.color}'
```

Эта модель создаст таблицу с 4 столбцами: `id`, `brand`, `model`, `color`. Раз модель — обычный класс, в ней можно объявлять методы и свойства, например `__str__` для читаемого вывода.

## Миграции

**Миграция** — файл с описанием изменений схемы БД. Работает в два шага (аналогия с git: сначала коммит, потом пуш):

```bash
python manage.py makemigrations   # 1. собрать изменения моделей в файл-миграцию
python manage.py migrate          # 2. применить миграции к базе (создать/изменить таблицы)
```

Меняешь модель — снова `makemigrations` + `migrate`. Так схема БД всегда следует за кодом и хранится в истории.

## Типы полей

Под каждый вид данных — свой тип поля. Полный список — в документации; основные:

| Поле | Для чего |
|---|---|
| `CharField(max_length=…)` | короткая строка (обязателен `max_length`) |
| `TextField()` | длинный текст без лимита |
| `IntegerField()` / `SmallIntegerField()` | целые числа |
| `DecimalField(max_digits, decimal_places)` | точные числа — **цены, деньги** |
| `BooleanField(default=…)` | да/нет |
| `DateTimeField(auto_now_add=…, auto_now=…)` | дата и время |
| `ImageField(upload_to=…)` / `FileField` | картинки и файлы |
| `ForeignKey(...)` | связь «многие к одному» |
| `ManyToManyField(...)` | связь «многие ко многим» |

> **`auto_now_add` vs `auto_now`.** `auto_now_add=True` ставит время **один раз при создании** (когда создан) — для `created_at`. `auto_now=True` обновляет время **при каждом сохранении** (когда изменён) — для `updated_at`.

## Запросы: CRUD через ORM

Каждая модель имеет **менеджер** `objects` — через него идут запросы к таблице.

**Создание** — двумя способами:

```python
car = Car(brand='BMW', model='X5', color='black')
car.save()                                   # 1. создать объект и сохранить
# или одной строкой:
Car.objects.create(brand='BMW', model='X5', color='black')   # 2. create()
```

**Чтение** — `all`, `filter`, `get`:

```python
Car.objects.all()                            # все записи (QuerySet)
Car.objects.filter(brand='BMW')              # все с brand == 'BMW'
Car.objects.get(id=1)                        # ровно одна запись (иначе исключение)
```

**Модификаторы поиска** (field lookups) — через двойное подчёркивание `__`:

```python
Car.objects.filter(brand__contains='BM')     # содержит подстроку
Car.objects.filter(brand__startswith='B')    # начинается с
Car.objects.filter(price__gte=600)           # >= (gte, lte, gt, lt)
```

> **QuerySet ленивый** — запрос к БД уходит не в момент `filter`, а когда результат реально нужен (итерация, `list()`, срез). Поэтому цепочки `filter().filter()` дёшевы: они собирают один SQL, а не бегают в базу на каждом шаге.

## Связи между таблицами

### Многие к одному — ForeignKey

**`ForeignKey`** — у многих записей один «родитель» (у многих машин один владелец). `on_delete` задаёт, что делать при удалении родителя; `related_name` — имя для **обратного** доступа.

```python
class Person(models.Model):
    name = models.CharField(max_length=50)
    car = models.ForeignKey(Car, on_delete=models.CASCADE, related_name='owners')
    #   CASCADE — удалить машину → удалятся связанные person
    #   related_name='owners' — с машины видно её владельцев
```

```python
person.car                # прямой доступ: машина владельца
car.owners.all()          # обратный доступ (через related_name): все владельцы машины
car.owners.count()        # сколько их
```

### Многие ко многим — ManyToMany

**`ManyToManyField`** — запись связана с многими и наоборот (заказ ↔ товары). Django создаёт промежуточную таблицу автоматически:

```python
class Product(models.Model):
    name = models.CharField(max_length=100)
    price = models.IntegerField()

class Order(models.Model):
    products = models.ManyToManyField(Product, related_name='orders')
```

**Проблема:** в заказе нужно ещё и **количество** каждого товара — а в автоматической связи хранится только сам факт связи. Решение — **явная промежуточная модель** через `through`:

```python
class OrderPosition(models.Model):           # промежуточная модель со своими полями
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='positions')
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='positions')
    quantity = models.IntegerField()          # ← то, ради чего она нужна

class Order(models.Model):
    # связь идёт через нашу модель, но удобные свойства остаются:
    products = models.ManyToManyField(Product, related_name='orders', through='OrderPosition')
```

Теперь есть и количество, и прямой доступ в обе стороны:

```python
order.products.all()          # товары заказа
product.orders.all()          # заказы, где есть товар
order.positions.all()         # позиции с количеством
```

## Ловушка: N+1 запросов

Самая частая ошибка производительности в Django ORM. Обращение к связанному объекту **в цикле** бьёт по базе на каждой итерации: 1 запрос за список + N запросов за связи = **N+1**.

```python
# НЕПРАВИЛЬНО — N+1 запросов:
for person in Person.objects.all():   # 1 запрос: SELECT * FROM person
    print(person.car.brand)           # +1 запрос на КАЖДОГО person
#   100 человек  →  101 запрос к базе

# ПРАВИЛЬНО — подтянуть связь заранее:
for person in Person.objects.select_related('car'):   # 1 запрос c JOIN
    print(person.car.brand)                            # из памяти, без похода в базу
#   100 человек  →  1 запрос
```

- **`select_related('поле')`** — для `ForeignKey` / `OneToOne`: тянет связь одним **JOIN**.
- **`prefetch_related('поле')`** — для `ManyToMany` и обратных связей: делает **отдельный** запрос и сшивает данные в Python (там JOIN не годится).

> **Почему так.** `person.car` — ленивый доступ: при первом обращении он идёт в базу. В цикле это N походов. `select_related` забирает всё заранее одним запросом. Ловить N+1 глазами помогает [Django Debug Toolbar](#django-debug-toolbar) — он показывает число SQL-запросов на страницу.

## Настройка БД (PostgreSQL)

По умолчанию Django использует SQLite. Для боевого проекта переключают на PostgreSQL. В `settings.py`, секция `DATABASES`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'demoorm',
        'USER': 'myuser',
        'PASSWORD': 'secret',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

Нужен драйвер и сама база:

```bash
pip install psycopg2-binary          # драйвер PostgreSQL для Python
createdb -U postgres demoorm         # создать базу
python manage.py migrate             # заново применить миграции — это другая БД
```

## Админка — настройка отображения

Базовая регистрация модели — в [Django](../../Фреймворки/DJANGO/Django.md#админка). Тонкая настройка — через класс `ModelAdmin`:

```python
from django.contrib import admin
from .models import Car, Order, OrderPosition

@admin.register(Car)
class CarAdmin(admin.ModelAdmin):
    list_display = ['id', 'brand', 'model', 'color']   # какие столбцы показать в списке
    list_filter = ['brand', 'model']                    # боковые фильтры
```

**Inline** — редактировать связанные записи прямо внутри родителя (позиции заказа — внутри заказа):

```python
class OrderPositionInline(admin.TabularInline):   # TabularInline — таблицей, StackedInline — блоками
    model = OrderPosition
    extra = 0                                      # сколько пустых строк показывать

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id']
    inlines = [OrderPositionInline]                # позиции редактируются внутри заказа
```

## Django Debug Toolbar

**Django Debug Toolbar** — панель отладки: показывает, какие SQL-запросы сделала страница, сколько времени заняла, какие шаблоны и переменные использованы. Незаменима, чтобы ловить лишние запросы к БД.

```bash
pip install django-debug-toolbar
```

Кратко по настройке в `settings.py`: добавить `debug_toolbar` в `INSTALLED_APPS` (после `django.contrib.staticfiles`), `DebugToolbarMiddleware` — в начало `MIDDLEWARE`, задать `INTERNAL_IPS = ['127.0.0.1']`; и маршрут `path('__debug__/', include('debug_toolbar.urls'))` в конец `urlpatterns`.

## Сквозной пример: заказы и позиции

Модель «заказ ↔ товары с количеством» через промежуточную модель — и вывод в шаблон.

**Шаг 1. Модели** — `Product`, `Order`, `OrderPosition` (см. блок «Многие ко многим» выше).

**Шаг 2. View** — собрать заказы:

```python
from django.shortcuts import render
from .models import Order

def list_orders(request):
    return render(request, 'orders.html', {'orders': Order.objects.all()})
```

**Шаг 3. Шаблон** `orders.html` — заказ и его позиции (методы в шаблоне — без скобок):

```html
{% for order in orders %}
  <p>Заказ #{{ order.id }}</p>
  {% for position in order.positions.all %}
    <p>{{ position.product.name }}: {{ position.quantity }} шт.</p>
  {% endfor %}
{% endfor %}
```

**Как это читать.** Промежуточная модель `OrderPosition` хранит количество, а `through` сохраняет удобные свойства `order.products` / `product.orders`. View достаёт заказы, шаблон разворачивает у каждого его позиции через обратную связь `positions`.

## Связи

- [Django](../../Фреймворки/DJANGO/Django.md) — как модели встраиваются в общий цикл MVT (view → шаблон);
- [SQLAlchemy](../../Библиотеки/Сторонние/ORM/SQLAlchemy.md) — независимый ORM для не-Django проектов (сравни подход);
- [API](../../Фреймворки/DJANGO/API.md) — DRF-сериализаторы строятся поверх этих же моделей;
- PostgreSQL — боевая база под Django-проект;
- [Развертывание проекта](../../DevOps/Развертывание%20проекта.md#шаг-3-миграции-и-статика) — как миграции применяются на сервере.

## Источники

- [Django — Models](https://docs.djangoproject.com/en/stable/topics/db/models/)
- [Making queries (QuerySet)](https://docs.djangoproject.com/en/stable/topics/db/queries/)
- [Model field reference](https://docs.djangoproject.com/en/stable/ref/models/fields/)
- [Many-to-many with through](https://docs.djangoproject.com/en/stable/topics/db/models/#extra-fields-on-many-to-many-relationships)
- [Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/en/latest/)
