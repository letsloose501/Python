[← Оглавление](../../README.md)

# CRUD в DRF

> _От «всё руками» до «весь CRUD одним классом» — выбираешь уровень контроля_

---

**CRUD** — четыре базовые операции над данными: **Create** (создать), **Read** (прочитать), **Update** (обновить), **Delete** (удалить). В REST API каждая ложится на свой HTTP-метод. DRF даёт **лестницу** инструментов: от ручного `APIView` до `ModelViewSet`, где весь CRUD пишется в три строки. Эта заметка — про то, какой уровень выбрать.

## CRUD и HTTP-методы

| CRUD | HTTP-метод | Действие |
|---|---|---|
| **Create** | `POST` | создать новую запись |
| **Read** | `GET` | получить список или одну запись |
| **Update** | `PUT` / `PATCH` | обновить (полностью / частично) |
| **Delete** | `DELETE` | удалить запись |

Дальше — на модели `Phone` и её сериализаторе:

```python
# models.py
class Phone(models.Model):
    name = models.CharField(max_length=100)
    ram = models.PositiveSmallIntegerField()
    color = models.CharField(max_length=20)
    price = models.DecimalField(max_digits=10, decimal_places=2)

# serializers.py
class PhoneSerializer(serializers.ModelSerializer):
    class Meta:
        model = Phone
        fields = '__all__'
```

## Лестница абстракции DRF

Один и тот же CRUD можно написать на четырёх уровнях — чем выше, тем меньше кода и меньше контроля:

```
FBV / APIView   ──▶  Generic views  ──▶  ViewSet / ModelViewSet
всё вручную          готовый класс        весь CRUD одним
(полный контроль)    на действие          классом + роутер
```

## APIView — всё вручную

Максимальный контроль: сам достаёшь запись, ловишь 404, валидируешь, сохраняешь. Пример — получить, обновить и удалить один телефон:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class PhoneDetailsView(APIView):
    def get(self, request, phone_id):
        try:
            phone = Phone.objects.get(id=phone_id)
            return Response(PhoneSerializer(phone).data)
        except Phone.DoesNotExist:
            return Response({'message': 'Не найдено'}, status=status.HTTP_404_NOT_FOUND)

    def patch(self, request, phone_id):
        try:
            phone = Phone.objects.get(id=phone_id)
            serializer = PhoneSerializer(phone, data=request.data, partial=True)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
        except Phone.DoesNotExist:
            return Response({'message': 'Не найдено'}, status=status.HTTP_404_NOT_FOUND)

    def delete(self, request, phone_id):
        try:
            Phone.objects.get(id=phone_id).delete()
            return Response({'message': 'Удалено'})
        except Phone.DoesNotExist:
            return Response({'message': 'Не найдено'}, status=status.HTTP_404_NOT_FOUND)
```

Много повторяющегося кода — его и убирают уровни выше.

## Generic views — готовый класс на действие

**Generic-классы** уже содержат логику типового действия. Достаточно указать `queryset` и `serializer_class`. В основе всех — `GenericAPIView`; под каждое действие свой класс:

| Класс | Действие | HTTP |
|---|---|---|
| `ListAPIView` | список | GET |
| `RetrieveAPIView` | одна запись | GET |
| `CreateAPIView` | создание | POST |
| `UpdateAPIView` | обновление | PUT/PATCH |
| `DestroyAPIView` | удаление | DELETE |

```python
from rest_framework import generics

class PhonesListView(generics.ListAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
```

**Комбинированные** generic-классы объединяют несколько действий:

- **`ListCreateAPIView`** — список (`GET`) + создание (`POST`);
- **`RetrieveUpdateDestroyAPIView`** — получить + обновить + удалить одну запись.

```python
class PhoneListCreateView(generics.ListCreateAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer

class PhoneDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
```

Двумя классами закрыт весь CRUD. Но можно ещё короче.

## ViewSet — весь CRUD одним классом

**ViewSet** объединяет все действия в одном классе. Чаще всего берут **`ModelViewSet`** — он сам реализует `list`, `retrieve`, `create`, `update`, `destroy`:

```python
from rest_framework.viewsets import ModelViewSet

class PhoneViewSet(ModelViewSet):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer
```

Маршруты для ViewSet строит **роутер** — он сам создаёт все URL CRUD:

```python
# urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register('phones', PhoneViewSet, basename='phones')

urlpatterns = [
    path('api/v1/', include(router.urls)),
]
#   /api/v1/phones/       GET (список), POST (создать)
#   /api/v1/phones/{id}/  GET, PUT, PATCH, DELETE
```

## Какой вариант выбрать

| Нужно | Выбор |
|---|---|
| полный контроль, нестандартная логика | **FBV** или **APIView** |
| одно-два конкретных действия | **generic-классы** |
| весь CRUD сразу, типовая модель | **ViewSet / ModelViewSet** |

> Правило: бери **самый высокий** уровень, которого хватает. Нестандартное поведение всегда можно переопределить, оставив остальное автоматическим.

## Валидация

Кастомные проверки пишут в сериализаторе. Метод `validate_<поле>` проверяет одно поле, `validate` — сразу несколько:

```python
class CommentSerializer(serializers.ModelSerializer):
    def validate_text(self, value):                 # проверка одного поля
        if 'спам' in value:
            raise serializers.ValidationError('Запрещённое слово')
        return value

    def validate(self, attrs):                      # проверка нескольких полей вместе
        if attrs['user'].id == 1:
            raise serializers.ValidationError('Что-то пошло не так')
        return attrs
```

## Фильтрация, поиск, сортировка

Устанавливается пакет `django-filter`, бэкенды задаются глобально в `settings.py` или на конкретном view:

```bash
pip install django-filter
```

```python
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework.filters import SearchFilter, OrderingFilter

class CommentViewSet(ModelViewSet):
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ['user']            # точная фильтрация: ?user=1
    search_fields = ['text']               # поиск по тексту: ?search=привет
    ordering_fields = ['id', 'created_at'] # сортировка: ?ordering=-created_at
```

- **`DjangoFilterBackend`** — точная фильтрация по значению поля;
- **`SearchFilter`** — поиск по подстроке (параметр `?search=`, переименовать через `SEARCH_PARAM`);
- **`OrderingFilter`** — сортировка (`?ordering=`, минус = по убыванию).

## Пагинация

Разбивка большого списка на страницы — глобально в `settings.py`:

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
}
```

Или на конкретном view — `pagination_class = LimitOffsetPagination` (постранично по `?limit=&offset=`).

## Сквозной пример: полный CRUD API за минуту

```python
# 1. models.py — данные
class Phone(models.Model):
    name = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=10, decimal_places=2)

# 2. serializers.py — JSON
class PhoneSerializer(serializers.ModelSerializer):
    class Meta:
        model = Phone
        fields = '__all__'

# 3. views.py — весь CRUD одним классом
class PhoneViewSet(ModelViewSet):
    queryset = Phone.objects.all()
    serializer_class = PhoneSerializer

# 4. urls.py — роутер строит все маршруты
router = DefaultRouter()
router.register('phones', PhoneViewSet)
urlpatterns = [path('api/v1/', include(router.urls))]
```

**Как это читать.** `ModelViewSet` + роутер дают полноценный REST-ресурс (`GET/POST/PUT/PATCH/DELETE`) без ручного кода. Дальше на этот же ViewSet навешивают права и throttling — это [Разделение доступа в DRF](../../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md).

## Связи

- [API](../../Фреймворки/DJANGO/API.md) — REST, сериализаторы, APIView (основа этой заметки);
- [Разделение доступа в DRF](../../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md) — `permission_classes`, аутентификация, throttling поверх ViewSet;
- [Тестирование Django приложений](../../Фреймворки/DJANGO/Тестирование%20Django%20приложений.md) — как тестировать CRUD-эндпоинты (APIClient);
- [ORM Django](../../Фреймворки/DJANGO/ORM%20Django.md) — `queryset` и модели, над которыми работает CRUD.

## Источники

- [DRF — Generic views](https://www.django-rest-framework.org/api-guide/generic-views/)
- [DRF — ViewSets](https://www.django-rest-framework.org/api-guide/viewsets/)
- [DRF — Routers](https://www.django-rest-framework.org/api-guide/routers/)
- [DRF — Filtering](https://www.django-rest-framework.org/api-guide/filtering/)
- [DRF — Pagination](https://www.django-rest-framework.org/api-guide/pagination/)
