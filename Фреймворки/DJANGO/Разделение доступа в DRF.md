[← Оглавление](../../README.md)

# Разделение доступа в DRF

> _Сначала «кто ты?» (аутентификация), потом «что тебе можно?» (авторизация)_

---

API нельзя открывать всем подряд: одни данные читают все, другие меняет только владелец, третьи — только админ. DRF решает это в три слоя: **аутентификация** (узнать пользователя), **permissions** (проверить права) и **throttling** (ограничить частоту запросов). Разберём по порядку.

## Три понятия: не путать

- **Идентификация** — сопоставление предъявленного идентификатора (логина) с известными в системе. Отвечает: «такой пользователь есть?».
- **Аутентификация** — проверка подлинности. Отвечает: **«ты правда тот, за кого себя выдаёшь?»** (пароль, токен).
- **Авторизация** — проверка прав. Отвечает: **«тебе это можно?»** (обычный юзер смотрит товары, модератор их редактирует).

Аутентификация всегда идёт до авторизации: сначала система узнаёт, кто ты, потом решает, что тебе разрешено.

Проверяют подлинность по разным **факторам**: что ты *знаешь* (пароль), что *имеешь* (ключ, смарт-карта), *кто* ты (отпечаток, лицо), *где* находишься (IP). Комбинация факторов — многофакторная аутентификация.

## 5 типов аутентификации в DRF

| Тип | Как работает |
|---|---|
| **Basic** | логин и пароль в HTTP-заголовке каждого запроса. Просто, но небезопасно без HTTPS |
| **Session** | как в обычном Django: сессия на сервере + cookie у клиента |
| **Token** | у каждого пользователя уникальный токен, шлётся в заголовке |
| **JWT** | самодостаточный токен с подписью (access + refresh), состояние не хранится на сервере |
| **Custom** | свой класс от `BaseAuthentication` со своей логикой |

Тип задаётся в `settings.py` через `DEFAULT_AUTHENTICATION_CLASSES`. Разберём два рабочих — Token и JWT.

## Token Authentication

Каждый пользователь получает **токен**, который шлёт в заголовке каждого запроса. Токены хранятся в базе.

```python
# settings.py
INSTALLED_APPS = [..., 'rest_framework', 'rest_framework.authtoken']

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}
```

```bash
python manage.py migrate           # создать таблицу токенов
python manage.py createsuperuser   # создать пользователя
```

Пользователь получает токен POST-запросом на `api-token-auth` (логин + пароль) → в ответ `{"token": "..."}`. Дальше токен идёт в заголовке:

```
Authorization: Token your_token_value
```

DRF сам проверит токен и, если он валиден, пустит к ресурсу.

## JWT Authentication

**JWT** (JSON Web Token) — открытый стандарт (RFC 7519): компактный самодостаточный токен из трёх частей — **заголовок**, **полезная нагрузка** и **подпись**. Главное отличие от обычного токена: сервер **не хранит** его в базе — подлинность проверяется по подписи. Ставится отдельным пакетом:

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
```

```python
# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view()),           # получить пару токенов
    path('api/token/refresh/', TokenRefreshView.as_view()),      # обновить access
]
```

Клиент шлёт логин/пароль на `api/token/` → получает **два** токена:

```json
{ "access": "...", "refresh": "..." }
```

- **access** — короткоживущий, идёт в заголовке `Authorization: Bearer <access>`;
- **refresh** — долгоживущий: когда access протухает, на `api/token/refresh/` меняют его на новый access, не вводя пароль заново.

## Permissions — права доступа

После аутентификации DRF проверяет **разрешения**. Четыре встроенных:

| Permission | Кто получает доступ |
|---|---|
| `AllowAny` | все, даже анонимные |
| `IsAuthenticated` | только аутентифицированные |
| `IsAdminUser` | только админы (`is_staff=True`) |
| `IsAuthenticatedOrReadOnly` | чтение всем, запись — только аутентифицированным |

Назначаются через атрибут `permission_classes`:

```python
from rest_framework.permissions import IsAuthenticated

class MyView(APIView):
    permission_classes = [IsAuthenticated]     # несколько — И (все должны выполниться)
```

### Пользовательские разрешения

Своя логика — класс от `BasePermission` с одним из методов:

- **`has_permission(request, view)`** — доступ к представлению в целом;
- **`has_object_permission(request, view, obj)`** — доступ к **конкретному объекту**.

Типичный случай — «читать всем, менять только владельцу». `SAFE_METHODS` — это `GET`, `HEAD`, `OPTIONS` (не меняют данные):

```python
# permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:   # чтение — всем
            return True
        return obj.user == request.user                  # запись — только владельцу
```

### Динамические права — get_permissions

Когда права зависят от действия (список — всем, создание — только админу), переопределяют `get_permissions`:

```python
class MyViewSet(ModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MySerializer

    def get_permissions(self):
        if self.action == 'list':
            return [permissions.AllowAny()]            # список — всем
        if self.action == 'create':
            return [permissions.IsAdminUser()]         # создание — только админ
        return [permissions.IsAuthenticated()]         # остальное — аутентифицированным
```

## Ловушка: has_object_permission не защищает список

`has_object_permission` вызывается, только когда DRF достаёт **один** объект (`retrieve`/`update`/`delete` через `get_object()`). Для `list` он **не вызывается** — значит `IsOwnerOrReadOnly` не спрячет чужие записи из списка: на `GET /items/` каждый увидит **все** объекты.

```python
# НЕПРАВИЛЬНО — думаем, что список тоже защищён:
class ItemViewSet(ModelViewSet):
    queryset = Item.objects.all()                 # ← /items/ отдаёт ВСЁ, всем
    permission_classes = [IsOwnerOrReadOnly]      # работает только для /items/{id}/

# ПРАВИЛЬНО — фильтровать список по владельцу через queryset:
class ItemViewSet(ModelViewSet):
    permission_classes = [IsAuthenticated]
    def get_queryset(self):
        return Item.objects.filter(user=self.request.user)   # видно только свои
```

> **Почему так.** Права на объект (`has_object_permission`) — про доступ к *одной* записи; видимость в *списке* — это `get_queryset`. Разные механизмы: одно не заменяет другое. Полагаться на объектные права для сокрытия чужих данных в списке — типичная дыра в безопасности.

## Throttling — ограничение частоты

**Throttling** ограничивает число запросов за период — защита от перебора и перегрузки. Настраивается глобально:

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.UserRateThrottle',
        'rest_framework.throttling.AnonRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'user': '10/minute',      # аутентифицированный — 10 запросов в минуту
        'anon': '2/minute',       # анонимный — 2 в минуту
    },
}
```

Или на конкретном view через `throttle_classes`.

## Сквозной пример: защищённый ресурс

Объявления, где создавать может любой вошедший, а редактировать/удалять — только автор.

```python
# permissions.py — своё правило (см. выше)
class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.user == request.user

# serializers.py — поле user только для чтения, ставится сервером
class AdvSerializer(serializers.ModelSerializer):
    class Meta:
        model = Adv
        fields = ['id', 'user', 'text', 'created_at', 'open']
        read_only_fields = ['user']

# views.py
from rest_framework.permissions import IsAuthenticated
from rest_framework.throttling import AnonRateThrottle

class AdvViewSet(ModelViewSet):
    queryset = Adv.objects.all()
    serializer_class = AdvSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]   # вошёл + владелец
    throttle_classes = [AnonRateThrottle]

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)      # автора берём из запроса, не из тела
```

**Как это читать.** Аутентификация определяет `request.user`; `IsAuthenticated` пускает только вошедших; `IsOwnerOrReadOnly` на уровне объекта разрешает правки лишь автору; `perform_create` подставляет автора сам — клиент не может выдать чужой `user`. Throttling ограничивает поток запросов.

## Связи

- [API](../../Фреймворки/DJANGO/API.md) — основы DRF, поверх которых работает разграничение доступа;
- [CRUD В DRF](../../Фреймворки/DJANGO/CRUD%20В%20DRF.md) — ViewSet, на который навешиваются `permission_classes` и `throttle_classes`;
- [Oauth 2.0](../../Библиотеки/Сторонние/Работа%20с%20WEB/Oauth%202.0.md) — родственный механизм авторизации через сторонние сервисы;
- [JWT — подпись и проверка токенов](../../JWT%20—%20подпись%20и%20проверка%20токенов.md) — как устроена подпись под `Bearer`-токеном; simplejwt по умолчанию подписывает общим секретом (HS256), а в микросервисах нужна асимметричная подпись;
- [HTTPS и обмен ключами Диффи — Хеллмана](../../Алгоритмы/Иные/HTTPS%20и%20обмен%20ключами%20Диффи%20—%20Хеллмана.md) — почему токены и пароли шлют только по HTTPS.

## Источники

- [DRF — Authentication](https://www.django-rest-framework.org/api-guide/authentication/)
- [DRF — Permissions](https://www.django-rest-framework.org/api-guide/permissions/)
- [DRF — Throttling](https://www.django-rest-framework.org/api-guide/throttling/)
- [Simple JWT documentation](https://django-rest-framework-simplejwt.readthedocs.io/)
- [JWT — RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)
