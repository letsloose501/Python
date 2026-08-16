[← Оглавление](../../README.md)

# Кастомная модель User

> _Решение, которое надо принять до первой миграции — потом оно стоит в разы дороже_

---

Встроенная модель `django.contrib.auth.models.User` даёт логин, пароль, имя, фамилию, почту и флаги прав. В реальном проекте почти всегда нужно больше: телефон, аватар, дата рождения, роль, вход по email вместо username. [Django](../../Фреймворки/DJANGO/Django.md) даёт четыре способа это сделать, и выбор между ними — частый вопрос на собеседовании: *«какую связь использовал бы для расширения стандартного юзера?»*

> **Главное правило:** способ выбирают **в начале проекта**. Замена модели пользователя после того, как таблицы созданы, официально описана как крайне сложная — Django не умеет мигрировать её автоматически.

## Четыре способа

| Способ | Когда брать | Что меняется в БД |
|---|---|---|
| **Профиль через `OneToOneField`** | проект уже живёт, менять `User` поздно | отдельная таблица + JOIN на каждое обращение |
| **`AbstractUser`** | новый проект, стандартных полей хватает, нужно добавить свои | одна таблица вместо `auth_user` |
| **`AbstractBaseUser`** | нужен полный контроль: вход по email, свой набор полей | своя таблица с нуля |
| **Proxy-модель** | менять только поведение (методы, менеджер, сортировка) | ничего |

### 1. Профиль — OneToOne

Отдельная модель, связанная с пользователем «один к одному». Единственный вариант, который работает на **существующем** проекте.

```python
# accounts/models.py
from django.conf import settings
from django.db import models

class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,          # ← не User напрямую!
        on_delete=models.CASCADE,
        related_name="profile",
    )
    phone = models.CharField(max_length=20, blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True)
    birth_date = models.DateField(null=True, blank=True)
```

Обращение: `user.profile.phone`. Профиль создают [сигналом](../../Фреймворки/DJANGO/Сигналы%20Django.md) `post_save` на пользователя — иначе у части пользователей его не будет, и `user.profile` бросит `Profile.DoesNotExist`.

> **Минус — лишний запрос.** Каждое `user.profile` — отдельный поход в базу. В списках это классический [N+1](../../Фреймворки/DJANGO/ORM%20Django.md#ловушка-n1-запросов): лечится `select_related("profile")`.

### 2. AbstractUser — рекомендуемый по умолчанию

`AbstractUser` — это полная реализация стандартного пользователя (username, пароль, email, `is_staff`, `is_active`, права, группы), но как **абстрактный** класс: своей таблицы не создаёт. Наследуешься — получаешь всё то же самое плюс свои поля, в одной таблице.

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    phone = models.CharField(max_length=20, blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True)
```

```python
# settings.py
AUTH_USER_MODEL = "accounts.User"        # ← до первой миграции
```

Это дефолтный выбор для нового проекта: JOIN'ов нет, ломать нечего, а добавить поле — одна строка.

### 3. AbstractBaseUser — когда нужен вход по email

`AbstractBaseUser` даёт только хеширование пароля и работу с токенами. Все поля, включая идентификатор входа, описываешь сам. Берут ради самого частого требования — **логин по email, без username**.

```python
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models

class UserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra):
        if not email:
            raise ValueError("email обязателен")
        user = self.model(email=self.normalize_email(email), **extra)
        user.set_password(password)        # ← хеширует, не пишет открытым текстом
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra):
        extra.setdefault("is_staff", True)
        extra.setdefault("is_superuser", True)
        return self.create_user(email, password, **extra)

class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = UserManager()

    USERNAME_FIELD = "email"       # чем пользователь входит
    EMAIL_FIELD = "email"
    REQUIRED_FIELDS = []           # что спросит createsuperuser сверх USERNAME_FIELD
```

Три обязательные детали:

- **`USERNAME_FIELD`** — поле-идентификатор. Должно быть `unique=True`.
- **`REQUIRED_FIELDS`** — список полей, которые `createsuperuser` спросит дополнительно. `USERNAME_FIELD` и пароль туда **не** включают.
- **`PermissionsMixin`** — добавляет `is_superuser`, группы, права и методы `has_perm()` / `has_module_perms()`. Без него не заработает админка и стандартные проверки доступа.

### 4. Proxy-модель — только поведение

Если структура полей устраивает, а нужны свои методы, менеджер или сортировка по умолчанию:

```python
class Manager(User):
    class Meta:
        proxy = True
        ordering = ["last_name"]

    def отправить_отчёт(self): ...
```

Таблица та же, миграции схемы нет.

## Как ссылаться на пользователя

Прямой импорт `from django.contrib.auth.models import User` — ошибка, из-за которой код ломается при замене модели. Правильных способов два, и они не взаимозаменяемы:

```python
# В моделях (ForeignKey, ManyToMany, OneToOne) — строка из настроек
from django.conf import settings

class Article(models.Model):
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)

# Во view, сервисах, тестах — функция
from django.contrib.auth import get_user_model
User = get_user_model()
User.objects.filter(is_active=True)
```

> Почему по-разному: `settings.AUTH_USER_MODEL` — просто строка `"accounts.User"`, её можно использовать до загрузки моделей. `get_user_model()` возвращает сам класс и требует, чтобы реестр приложений уже был готов — поэтому в `models.py` на верхнем уровне его вызывать нельзя.

## Что делать, если проект уже в проде

Смена `AUTH_USER_MODEL` на живой базе требует ручной правки схемы, переноса данных из старой таблицы пользователей и, возможно, переприменения миграций — Django сам циклические зависимости не разрулит. Поэтому:

- **проект уже работает** → способ 1, профиль через `OneToOneField`;
- **проект новый** → способ 2 или 3, и `AUTH_USER_MODEL` в `settings.py` **до** первой `migrate`.

Даже если кастомная модель сейчас не нужна, дешёвая страховка — с первого дня создать пустого наследника `AbstractUser` и прописать `AUTH_USER_MODEL`. Стоит одну строку, а право добавить поле остаётся навсегда.

## Связи

- [Django](../../Фреймворки/DJANGO/Django.md) — структура проекта, `settings.py`, `INSTALLED_APPS`;
- [Сигналы Django](../../Фреймворки/DJANGO/Сигналы%20Django.md) — как автосоздавать профиль на `post_save`;
- [ORM Django](../../Фреймворки/DJANGO/ORM%20Django.md) — `OneToOneField`, `select_related` и проблема N+1;
- [Разделение доступа в DRF](../../Фреймворки/DJANGO/Разделение%20доступа%20в%20DRF.md) — аутентификация и права поверх этой модели;
- Миграции и zero-downtime — почему структурные изменения на живой базе так дороги.

## Источники

- [Django documentation — Customizing authentication](https://docs.djangoproject.com/en/stable/topics/auth/customizing/)
- [Django documentation — Substituting a custom User model](https://docs.djangoproject.com/en/stable/topics/auth/customizing/#substituting-a-custom-user-model)
