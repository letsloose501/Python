[← Оглавление](../../README.md)

# REST API и Django REST Framework

> _Один сервер с данными — сколько угодно клиентов: сайт, мобильное, десктоп_

---

**API** (Application Programming Interface) — программный интерфейс, через который приложения обмениваются данными. Веб-**API** делает это по HTTP. Вместо готовых HTML-страниц сервер отдаёт **данные** (обычно JSON), а как их показать — решает клиент. **Django REST Framework (DRF)** — надстройка над [Django](../../Фреймворки/DJANGO/Django.md) для быстрого построения таких API.

## Почему одного Django недостаточно

Обычный [Django](../../Фреймворки/DJANGO/Django.md) формирует **готовые страницы** на сервере и отдаёт их клиенту. Удобно, пока клиент один — браузер. Но если нужны сайт, мобильные приложения под разные платформы и десктоп, придётся собирать на сервере 4 разных набора страниц под каждую особенность. Получается неповоротливый монолит.

Решение — **разделить клиент и сервер**: сервер предоставляет данные и операции над ними через API, а любые клиенты этими данными пользуются, каждый рисует свой интерфейс сам.

```
             ┌─▶ сайт (JS)
API (данные) ─┼─▶ мобильное приложение
   JSON      └─▶ десктоп
```

## Что такое REST API

**REST** (Representational State Transfer) — архитектурный стиль для веб-сервисов: работа с **ресурсами** через стандартные HTTP-методы. Он прост и единообразен, поэтому стал де-факто стандартом. Ключевые принципы:

- **Ресурсы и URI.** Каждый ресурс (данные) имеет уникальный адрес — **URI**: `https://api.example.com/users`, где `users` — ресурс.
- **HTTP-методы** задают операцию над ресурсом (таблица ниже).
- **Представление.** Ресурс передаётся в удобном формате — чаще всего **JSON**.
- **Stateless (без состояния).** Каждый запрос самодостаточен: несёт всё нужное для обработки. Сервер не хранит состояние между запросами клиента.
- **Единообразный интерфейс** — ограниченный, заранее известный набор операций. Это упрощает использование.

| HTTP-метод | Операция |
|---|---|
| `GET` | получить данные |
| `POST` | создать новый ресурс |
| `PUT` | полностью обновить ресурс |
| `PATCH` | частично обновить (только изменённые поля) |
| `DELETE` | удалить ресурс |

> **HATEOAS** — высший уровень REST: сервер отдаёт не только данные, но и ссылки на доступные действия. На практике встречается редко, но термин полезно знать.

## Django REST Framework

**DRF** — фреймворк для создания веб-API на Django. Что даёт:

- **Сериализацию** — превращение объектов моделей в JSON и обратно;
- **Аутентификацию и авторизацию** — гибкое управление доступом (см. [Разделение доступа в DRF](../../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md));
- **Готовые представления** — от простых до полного CRUD одним классом (см. [CRUD В DRF](../../Фреймворки/DJANGO/CRUD%20В%20DRF.md));
- **Валидацию** входных данных;
- **Маршрутизацию** через роутеры.

Установка:

```bash
pip install djangorestframework
```

```python
# settings.py
INSTALLED_APPS = [
    ...,
    'rest_framework',
]
```

## Сериализаторы

**Сериализатор** — механизм преобразования сложных объектов (моделей Django) в JSON и обратно. Прямое направление — **сериализация** (объект → JSON, для ответа клиенту), обратное — **десериализация** (JSON → объект, с валидацией входных данных). Два класса на выбор.

### Serializer — поля вручную

Наследуешься от `Serializer` и объявляешь поля явно, как в модели:

```python
# serializers.py
from rest_framework import serializers

class ProductSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=255)
    description = serializers.CharField()
    price = serializers.DecimalField(max_digits=10, decimal_places=2)
    created_at = serializers.DateTimeField()
```

В JSON попадут только объявленные поля (`id` и прочие, что не указал, — не попадут).

### ModelSerializer — поля из модели

Когда сериализатор почти повторяет модель, берут `ModelSerializer` — он **сам** строит поля по модели:

```python
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['name', 'description', 'price', 'created_at']
        # fields = '__all__'   — все поля модели
```

`ModelSerializer` автоматически создаёт поля нужных типов, подключает валидацию и даже обрабатывает связи (`ForeignKey`, `ManyToMany`). Короче и поддерживаемее — используется чаще.

```python
product = Product.objects.create(...)
serializer = ProductSerializer(product)
serializer.data      # {'name': '...', 'price': '150000.00', ...} — готово к отдаче в JSON
```

## Ловушка: many=True для списка

Сериализуешь **несколько** объектов (queryset) — обязателен `many=True`. Забудешь — DRF примет queryset за один объект и упадёт, пытаясь достать у него поля модели.

```python
products = Product.objects.all()

# НЕПРАВИЛЬНО — many=True забыт:
ProductSerializer(products).data              # ошибка: у queryset нет полей модели

# ПРАВИЛЬНО:
ProductSerializer(products, many=True).data   # → [ {...}, {...} ]  — список
ProductSerializer(one_product).data           # один объект — БЕЗ many
```

> Один объект — без `many`; список или queryset — с `many=True`. Одна из самых частых ошибок новичка в DRF.

## Представления: FBV и CBV

### FBV — на функциях

View-функция как в обычном Django, но обёрнута декоратором **`@api_view`** (задаёт методы), а ответ — через **`Response`** вместо `HttpResponse`:

```python
# views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Product
from .serializers import ProductSerializer

@api_view(['GET'])
def products_list(request):
    products = Product.objects.all()
    serializer = ProductSerializer(products, many=True)   # many=True — список объектов
    return Response(serializer.data)
```

```python
# urls.py
path('api/v1/products/', products_list),
```

Логика та же, что в Django: получить данные → сериализовать → вернуть в ответе.

### CBV — на классах

**`APIView`** — базовый класс DRF (наследник джанговского `View`). Наследуешься и реализуешь метод под каждый HTTP-глагол:

```python
from rest_framework.views import APIView

class ProductsListView(APIView):
    def get(self, request):
        products = Product.objects.all()
        serializer = ProductSerializer(products, many=True)
        return Response(serializer.data)

    def post(self, request):
        return Response({'status': 'ok'})
```

```python
# urls.py — класс привязывается через as_view()
path('api/v1/products/', ProductsListView.as_view()),
```

> Названия методов класса совпадают с HTTP-методами: `get`, `post`, `patch`, `delete`. DRF сам вызовет нужный по типу запроса.

Помимо `APIView` DRF даёт ещё более краткие способы — **generic-классы** и **ViewSet**, где типовой CRUD пишется почти без кода. Им посвящена [CRUD В DRF](../../Фреймворки/DJANGO/CRUD%20В%20DRF.md).

## Сквозной пример: первый эндпоинт

От модели до JSON-ответа.

```python
# 1. models.py — данные (см. ORM Django)
class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    def __str__(self):
        return self.name

# 2. serializers.py — превращение в JSON
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = '__all__'

# 3. views.py — обработчик
@api_view(['GET'])
def products_list(request):
    products = Product.objects.all()
    return Response(ProductSerializer(products, many=True).data)

# 4. urls.py — адрес
path('api/v1/products/', products_list)
#    GET http://localhost:8000/api/v1/products/  →  [{"id":1,"name":"...","price":"..."}]
```

**Как это читать.** Модель хранит данные, сериализатор превращает объекты в JSON, view достаёт их из БД и отдаёт через `Response`, маршрут связывает адрес с view. Дальше добавляют методы `POST/PATCH/DELETE` — это уже [CRUD В DRF](../../Фреймворки/DJANGO/CRUD%20В%20DRF.md).

## Связи

- [Django](../../Фреймворки/DJANGO/Django.md) — база, поверх которой работает DRF (модели, urls, view);
- [ORM Django](../../Фреймворки/DJANGO/ORM%20Django.md) — модели и связи, которые сериализуются в API;
- [CRUD В DRF](../../Фреймворки/DJANGO/CRUD%20В%20DRF.md) — generic-классы, ViewSet, роутеры, валидация, фильтрация, пагинация;
- [Разделение доступа в DRF](../../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md) — аутентификация, права и throttling для API;
- [Django с многими приложениями](../../Фреймворки/DJANGO/Django%20с%20многими%20приложениями.md) — API как backend для отдельного фронтенда (SPA);
- [FastAPI](../../Фреймворки/FastAPI.md) — альтернатива DRF для API с async и авто-документацией;
- [REST API на FastAPI](../../Фреймворки/REST%20API%20на%20FastAPI.md) — построение REST-сервиса на async-фреймворке: коды ответов, роутеры, `Depends`, async-БД, тесты;
- [GraphQL](../../Библиотеки/Сторонние/Работа%20с%20WEB/GraphQL.md) — другой подход к API: один эндпоинт, форму ответа задаёт клиент.

## Источники

- [Django REST Framework — официальный сайт](https://www.django-rest-framework.org/)
- [DRF — Serializers](https://www.django-rest-framework.org/api-guide/serializers/)
- [DRF — Views (APIView)](https://www.django-rest-framework.org/api-guide/views/)
- [RESTful API — restapitutorial.ru](https://restapitutorial.ru/)
