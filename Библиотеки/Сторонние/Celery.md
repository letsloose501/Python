[← Оглавление](../../README.md)

# Celery

> _Не заставляй клиента ждать: тяжёлую работу — в фон_

---

**Celery** — распределённая очередь задач для [Python](../../Python.md). Она выносит долгие или тяжёлые операции (отправка писем, обработка видео, распознавание лиц, обучение модели, генерация отчётов) в **фоновые процессы**, чтобы веб-приложение отвечало клиенту мгновенно, а не висело секундами. Три её роли: **распараллеливание** задач, **очереди** задач и **асинхронное** (отложенное) выполнение.

## Какую проблему решает

Иногда обработка запроса требует много времени или ресурсов CPU/RAM. Заставлять клиента ждать ответа дольше пары секунд — недопустимо. Решение:

1. приложение создаёт **задачу** и сразу возвращает клиенту её `id`;
2. задача выполняется **в фоне** отдельным процессом;
3. клиент по `id` проверяет статус, а по готовности забирает результат.

```
Клиент ──запрос──▶ приложение ──кладёт задачу──▶ [очередь] ──▶ воркер (фон)
       ◀─id сразу─                                                 │ считает
       ──спросить результат по id──────────────────────────◀──────┘ готово
```

_Пример: сервис распознавания лиц. Клиент шлёт `POST` с фото → веб-сервер кладёт задачу в очередь и отдаёт `id`; тяжёлая модель распознавания крутится в воркере; клиент `GET`-ом по `id` забирает результат, когда тот готов._

## Как устроено

Celery состоит из трёх частей:

- **Брокер (broker)** — очередь, куда складываются задачи. Приложение (producer) кладёт задачу, воркер её оттуда забирает. Обычно [Redis](../../Библиотеки/Сторонние/Redis.md) (просто, быстро) или [RabbitMQ](../../Библиотеки/Сторонние/RabbitMQ.md) (полноценный AMQP-брокер, надёжнее для критичных задач).
- **Воркер (worker)** — процесс, который берёт задачи из очереди и выполняет. Их запускают много и на разных машинах — отсюда «распределённая».
- **Backend результатов (result backend)** — хранилище, куда воркер кладёт результат и статус, чтобы клиент их потом забрал (часто [Redis](../../Библиотеки/Сторонние/Redis.md), или БД через `django-celery-results`).

```
python app ──▶ БРОКЕР (Redis/RabbitMQ) ──▶ ВОРКЕР (Celery) ──▶ RESULT BACKEND (Redis)
 producer            очередь                   считает              результат + статус
```

> Брокер **обязателен**, result backend — нет. Если результат не нужен (просто «отправь письмо и забудь») — backend можно не настраивать, это экономит ресурсы.

**Частое заблуждение: «воркер должен стоять рядом с приложением».** Не должен. Приложение при постановке задачи всего лишь **кладёт данные в брокер** — само оно воркера не знает и не вызывает. Воркер независимо подключается к брокеру по адресу и забирает задачи. Значит, все три части живут где угодно и масштабируются раздельно:

```
FastAPI (сервер A) ──кладёт──▶ Redis (сервер B) ◀──забирает── Celery worker (сервер C, 32 ядра + GPU)
```

Практический вывод: под тяжёлые задачи (видео, ML, рендер) берут отдельную машину с нужным железом, а веб-серверу хватает слабой. Единственное, что их связывает, — общий адрес брокера в конфиге и одинаковый код задач на обеих сторонах (воркеру нужен сам модуль с `@app.task`, иначе он не сможет её выполнить).

## Установка и запуск

```bash
pip install "celery[redis]"
docker run -d -p 6379:6379 redis     # брокер и backend
```

## Определение задачи

Задача — обычная функция с декоратором `@app.task`. Приложение Celery настраивают на брокер и backend:

```python
# tasks.py
import time
from celery import Celery

app = Celery('tasks',
             broker='redis://127.0.0.1:6379/0',    # откуда брать задачи
             backend='redis://127.0.0.1:6379/1')    # куда класть результаты

@app.task
def sort_iter(data):
    time.sleep(1)          # имитируем долгую работу
    return sorted(data)
```

## Запуск воркера

Воркер — отдельный процесс; в проде на отдельном сервере, но можно и локально:

```bash
celery -A tasks worker --loglevel=info
#   -A tasks — модуль, где искать приложение Celery (app)
#   --loglevel=info — подробный лог
#   -c 4 — число дочерних процессов (по умолчанию = числу ядер CPU)
```

## Вызов задачи: `delay` и `apply_async`

Ключевая разница — как вызвать функцию:

```python
sort_iter([3, 1, 2])          # обычный вызов — выполнится ЗДЕСЬ и сейчас (блокирует)
sort_iter.delay([3, 1, 2])    # .delay() — отправить в очередь, вернуть результат-объект СРАЗУ
```

`.delay(args)` — короткая форма. `.apply_async()` — полная, с параметрами доставки:

```python
sort_iter.apply_async(
    args=[[3, 1, 2]],
    countdown=10,          # выполнить через 10 секунд
    eta=datetime(...),     # или в конкретный момент
    expires=60,            # протухнет, если не выполнится за 60 с
    queue='sort_queue',    # в какую очередь положить
    priority=5,            # приоритет (если брокер поддерживает)
    retry=True,
)
```

## Результат: `AsyncResult` и состояния

`.delay()`/`.apply_async()` не ждут — возвращают **`AsyncResult`**. Забрать результат — `.get()` (он-то и подождёт, пока воркер посчитает):

```python
from tasks import sort_iter

# 4 задачи уходят в очередь почти мгновенно и считаются параллельно воркерами
results = [sort_iter.delay([i, i-1]) for i in range(4)]
print([r.get() for r in results])       # забрать результаты (блокирует до готовности)
```

Работа с объектом результата без блокировки:

```python
r = sort_iter.delay([3, 1, 2])
r.id            # id задачи — по нему клиент спросит статус позже
r.ready()       # True, если задача завершилась (не блокирует)
r.status        # 'PENDING' | 'STARTED' | 'SUCCESS' | 'FAILURE' | 'RETRY'
r.successful()  # True, если SUCCESS
r.get(timeout=5)              # ждать максимум 5 с, иначе TimeoutError
r.get(propagate=False)        # при ошибке вернуть исключение, а не бросать его
```

| Состояние | Значит |
|---|---|
| `PENDING` | задача неизвестна backend'у — ещё не начата **или** такого id нет |
| `STARTED` | воркер взял в работу (только при `task_track_started=True`) |
| `SUCCESS` | выполнена, результат доступен |
| `FAILURE` | упала с исключением |
| `RETRY` | ожидает повторной попытки |

> `PENDING` — это «ничего не знаю о задаче». Celery не отличает «ещё в очереди» от «такого id не было» — оба дают `PENDING`. Не полагайся на `PENDING` как на «в процессе».

## Повторы и надёжность

Фоновые задачи падают (сеть моргнула, сервис недоступен). Celery умеет **повторять** их автоматически.

```python
# ручной повтор: bind=True даёт доступ к self (сама задача)
@app.task(bind=True, max_retries=3, default_retry_delay=10)
def send_email(self, to):
    try:
        smtp_send(to)
    except SMTPException as exc:
        raise self.retry(exc=exc)     # положить обратно в очередь, попробовать снова

# автоповтор для указанных исключений + экспоненциальная задержка
@app.task(autoretry_for=(SMTPException,),
          retry_backoff=True,          # 1с, 2с, 4с, 8с… между попытками
          retry_kwargs={'max_retries': 5})
def send_email(to):
    smtp_send(to)
```

- **`acks_late=True`** — подтверждать задачу брокеру **после** выполнения, а не при получении. Если воркер упадёт посреди работы — задача вернётся в очередь и достанется другому. Ценой возможного **повторного** выполнения.
- **Идемпотентность.** Из-за повторов и `acks_late` задача может выполниться дважды. Делай обработчик идемпотентным (проверяй, не сделано ли уже), иначе — двойные списания и письма.

## Модели конкурентности (пулы)

Воркер может исполнять задачи по-разному, и выбор зависит от того, чем задача занята — процессором или ожиданием сети. Это прямое приложение разницы [CPU-bound и IO-bound](../../Библиотеки/Модули/Параллелизм/CPU-bound%20против%20IO-bound.md).

| Пул | Как исполняет | Для чего |
|---|---|---|
| **prefork** (по умолчанию) | дочерние процессы | CPU-bound и большинство задач; самый надёжный |
| **gevent**, **eventlet** | гринлеты в одном процессе | IO-bound: тысячи одновременных сетевых вызовов |
| **threads** | потоки через `concurrent.futures` | лёгкий IO-bound без гринлетов |
| **solo** | последовательно в главном потоке | отладка, тесты |

```bash
celery -A tasks worker --pool=prefork -c 8      # 8 процессов — по числу ядер
celery -A tasks worker --pool=gevent  -c 500    # 500 гринлетов — сотни HTTP-запросов
```

Официальная рекомендация — начинать с `prefork` и уходить с него, только когда есть конкретная причина.

> **Смена пула молча отключает часть возможностей.** На `gevent`/`eventlet` перестают работать `soft_timeout` и `max_tasks_per_child` — без ошибки и предупреждения. Задача, которую вы рассчитывали прервать по мягкому таймауту, просто будет висеть.

## Prefetch — сколько задач воркер забирает вперёд

**Prefetch** — сколько сообщений воркер **резервирует за собой** заранее, ещё не начав их выполнять. Считается как `worker_prefetch_multiplier` × число слотов конкурентности.

Смысл в пропускной способности: для коротких задач выгодно забрать пачку сразу и не ходить к брокеру за каждой. Но с **длинными** задачами это оборачивается бедой: воркер захватил десяток сообщений, выполняет их по очереди медленно, а другие свободные воркеры их уже не получат — очередь стоит, хотя мощности простаивают.

```python
worker_prefetch_multiplier = 1      # для долгих задач — брать по одной
task_acks_late = True               # подтверждать после выполнения
```

Эта пара — «взять ровно одну задачу за раз». Она требует идемпотентности: неподтверждённая задача при падении воркера уйдёт другому и выполнится повторно.

> **`task_acks_late` и prefetch — разные механизмы, их часто путают.** `acks_late` отвечает на вопрос «когда подтверждать» и защищает от падения воркера. Prefetch отвечает на «сколько резервировать» и определяет, как задачи распределяются между воркерами.

Есть и отдельная настройка `worker_disable_prefetch` — брать новую задачу только когда освободился слот; поддерживается пока только для брокера [Redis](../../Библиотеки/Сторонние/Redis.md).

## Безопасность

Воркер **доверяет** тому, что приходит из брокера, и выполняет это у себя. Поэтому вопрос «что можно прислать в сообщении» — это вопрос безопасности.

**Сериализатор по умолчанию — JSON**, начиная с Celery 4.0. И это защита, а не ограничение: JSON умеет только простые типы, из него нельзя «собрать» произвольный объект.

> **`pickle` небезопасен по своей природе.** Он сериализует почти любой объект Python, а значит, подготовленное сообщение может выполнить произвольный код на воркере. Включать его можно, только если брокер закрыт и все клиенты доверенные и аутентифицированы.

```python
task_serializer = 'json'
accept_content = ['json']      # принимать ТОЛЬКО json, что бы ни прислали
```

Если нужно гарантировать не только формат, но и **отправителя**, Celery умеет подписывать сообщения асимметричной криптографией: `task_serializer='auth'`, `accept_content=['auth']` плюс `security_key`, `security_certificate`, `security_cert_store`, `security_digest`. Активируется вызовом `app.setup_security()`, который заодно выключает все небезопасные сериализаторы.

Сам брокер защищают отдельно: файрвол со списком разрешённых адресов, права доступа на уровне брокера (в [RabbitMQ](../../Библиотеки/Сторонние/RabbitMQ.md) они детальные) и шифрование канала через `broker_use_ssl`.

## Конфигурация и роутинг по очередям

Настройки выносят в отдельный модуль и подключают `config_from_object`. Разные задачи можно направлять в **разные очереди**, а под каждую — держать свои воркеры (изоляция тяжёлых задач от быстрых):

```python
# celeryconfig.py
from kombu import Queue, Exchange

broker_url = 'redis://127.0.0.1:6379/0'
result_backend = 'redis://127.0.0.1:6379/1'

task_queues = (
    Queue('sort_queue', Exchange('sort_ex'), routing_key='sort_route'),
    Queue('sum_queue',  Exchange('sum_ex'),  routing_key='sum_route'),
)
task_routes = {
    'tasks.sort_iter': {'queue': 'sort_queue', 'routing_key': 'sort_route', 'priority': 10},
    'tasks.get_sum':   {'queue': 'sum_queue',  'routing_key': 'sum_route',  'priority': 5},
}
```

```python
app = Celery('tasks')
app.config_from_object('celeryconfig')
```

```bash
# каждый воркер слушает свою очередь (-Q) — тяжёлые задачи не тормозят лёгкие
celery -A tasks worker -E -l INFO -n sort_worker -Q sort_queue
celery -A tasks worker -E -l INFO -n sum_worker  -Q sum_queue
```

## Периодические задачи — Celery Beat

**Celery Beat** — планировщик: отдельный процесс, который по расписанию сам кладёт задачи в очередь (аналог cron). Воркеры их выполняют как обычно.

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'report-every-morning': {
        'task': 'tasks.daily_report',
        'schedule': crontab(hour=9, minute=0),    # каждый день в 9:00
    },
    'ping-every-30s': {
        'task': 'tasks.ping',
        'schedule': 30.0,                          # каждые 30 секунд
    },
}
```

```bash
celery -A tasks beat -l INFO        # запустить планировщик (отдельно от воркера)
```

## Композиция задач (Canvas)

Задачи связывают в конвейеры через **сигнатуры** (`.s()`):

- **`chain`** — цепочка: результат одной идёт на вход следующей.
- **`group`** — параллельно запустить пачку задач.
- **`chord`** — group + финальная задача-callback над всеми результатами.

```python
from celery import chain, group, chord

chain(add.s(2, 2), mul.s(4))()          # mul((2+2), 4) = 16 — последовательно
group(add.s(i, i) for i in range(10))() # 10 задач параллельно
chord(group(add.s(i, i) for i in range(10)))(summarize.s())  # собрать результаты
```

## Интеграция с Django

Стандартная раскладка проекта:

```python
# proj/celery.py — создать приложение Celery
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')
app = Celery('proj')
app.config_from_object('django.conf:settings', namespace='CELERY')  # брать CELERY_* из settings
app.autodiscover_tasks()          # найти tasks.py во всех приложениях
```

```python
# proj/__init__.py — чтобы Celery поднимался вместе с Django
from .celery import app as celery_app
__all__ = ('celery_app',)
```

```python
# settings.py
CELERY_BROKER_URL = 'redis://127.0.0.1:6379/0'
CELERY_RESULT_BACKEND = 'django-db'      # хранить результаты в БД Django
INSTALLED_APPS = [..., 'django_celery_results']
```

```bash
pip install django-celery-results
python manage.py migrate django_celery_results
```

Задачи пишут в `<app>/tasks.py` с `@shared_task` (не привязан к конкретному `app`):

```python
from celery import shared_task

@shared_task
def send_welcome_email(user_id):
    ...
```

## Интеграция с Flask

У Flask нет глобального `app`, поэтому задачам нужно вручную входить в контекст приложения (`app_context`) — иначе не будет доступа к конфигу, БД, расширениям:

```python
# celeryapp.py
from celery import Celery

def make_celery(app):
    celery = Celery('celeryapp')
    celery.conf.update(app.config)
    class ContextTask(celery.Task):
        def __call__(self, *args, **kwargs):
            with app.app_context():          # каждый вызов — внутри контекста Flask
                return self.run(*args, **kwargs)
    celery.Task = ContextTask
    return celery
```

```python
# flaskapp.py
from flask import Flask
from celeryapp import make_celery

flask_app = Flask(__name__)
flask_app.config.update(CELERY_BROKER_URL='redis://127.0.0.1:6379/0',
                        CELERY_RESULT_BACKEND='redis://127.0.0.1:6379/1')
celery = make_celery(flask_app)

@celery.task
def get_sum(data):
    return sum(data)

@flask_app.route('/sum/<data>')
def sum_view(data):
    nums = [int(x) for x in data.split('!')]
    task = get_sum.delay(nums)       # отдать в фон
    return {'task_id': task.id}      # вернуть id клиенту (не ждать .get()!)
```

## Мониторинг — Flower

**Flower** — веб-панель для Celery: очереди, воркеры, задачи, их статусы и время выполнения в реальном времени.

```bash
pip install flower
celery -A tasks flower        # → http://localhost:5555
```

## Частые ловушки

**Ждать `.get()` прямо во view.** Смысл Celery — не блокировать; если во view вызвать `.delay()` и тут же `.get()`, ты снова заставляешь клиента ждать. Правильно — вернуть `task.id`, а результат клиент заберёт отдельным запросом.

```python
# НЕПРАВИЛЬНО — снова блокируем клиента
task = process.delay(data)
return task.get()            # ждём завершения прямо в запросе — зачем тогда Celery?

# ПРАВИЛЬНО — вернуть id, забрать позже
task = process.delay(data)
return {'task_id': task.id}
```

**Передавать в задачу «тяжёлые» объекты.** Аргументы сериализуются (по умолчанию JSON) и едут через брокер. Нельзя слать ORM-объекты, соединения, файлы — передавай **id**, а объект доставай внутри задачи.

```python
send_email.delay(user)        # ✗ — объект не сериализуется / устареет
send_email.delay(user.id)     # ✓ — id, а user загрузить внутри задачи
```

**`.get()` внутри другой задачи** — риск взаимной блокировки воркеров (все заняты ожиданием). Вместо этого — Canvas (`chain`/`chord`).

**Забыть про идемпотентность** — при `acks_late`/повторах задача выполнится дважды (см. выше).

## Celery против asyncio

Оба про «не блокировать», но для разного:

| | Celery | [asyncio](../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) |
|---|---|---|
| **Для чего** | тяжёлые/долгие фоновые задачи, CPU-bound | конкурентный I/O в рамках запроса |
| **Где выполняется** | отдельные процессы-воркеры | тот же процесс, event loop |
| **Инфраструктура** | нужен брокер (Redis/RabbitMQ) | ничего лишнего |
| **Типичный кейс** | обработка видео, рассылки, отчёты, ретраи | много сетевых запросов одновременно |

Для **лёгких** фоновых задач прямо в рамках ответа (отправить одно письмо) в [FastAPI](../../Фреймворки/FastAPI.md) есть `BackgroundTasks` — без брокера. Celery берут, когда задач много, они тяжёлые или нужны ретраи/расписание.

## Связи

- [Redis](../../Библиотеки/Сторонние/Redis.md) — самый частый брокер и backend результатов; TTL, атомарность — оттуда;
- [RabbitMQ](../../Библиотеки/Сторонние/RabbitMQ.md) — надёжный AMQP-брокер, альтернатива Redis для критичных задач и сложной маршрутизации;
- [Django](../../Фреймворки/DJANGO/Django.md) · [FLASK](../../Фреймворки/FLASK.md) · [FastAPI](../../Фреймворки/FastAPI.md) — фреймворки, к которым подключают Celery для фоновых задач;
- Идемпотентность — обязательное свойство задачи при повторах и `acks_late`;
- [CPU-bound против IO-bound](../../Библиотеки/Модули/Параллелизм/CPU-bound%20против%20IO-bound.md) — от чего зависит выбор пула воркера: prefork или gevent;
- [multiprocessing](../../Библиотеки/Модули/Параллелизм/multiprocessing.md) — параллелизм в пределах одной машины; Celery масштабируется на многие;
- [Asyncio Event Loop и aiohttp](../../Библиотеки/Модули/Параллелизм/Asyncio%20Event%20Loop%20и%20aiohttp.md) — альтернатива для I/O-bound задач (сравнение выше);
- [Airflow](../../Библиотеки/Сторонние/Airflow.md) — соседний инструмент: не фоновая задача из приложения, а граф задач по расписанию.

## Источники

- [Celery documentation](https://docs.celeryq.dev/en/stable/)
- [Celery — First steps](https://docs.celeryq.dev/en/stable/getting-started/first-steps-with-celery.html)
- [Celery — Canvas: Designing Workflows](https://docs.celeryq.dev/en/stable/userguide/canvas.html)
- [Celery — Periodic Tasks (Beat)](https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html)
- [First steps with Django (Celery)](https://docs.celeryq.dev/en/stable/django/first-steps-with-django.html)
- [Celery — Concurrency](https://docs.celeryq.dev/en/stable/userguide/concurrency/index.html)
- [Celery — Optimizing](https://docs.celeryq.dev/en/stable/userguide/optimizing.html)
- [Celery — Security](https://docs.celeryq.dev/en/stable/userguide/security.html)
- Лекция «Celery» — Кирилл Табельский (Lightmap)
