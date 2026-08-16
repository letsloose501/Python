[← Оглавление](../README.md)

# Docker Compose — несколько окружений

---

Один и тот же набор сервисов нужен в разном виде. На проде база не должна светить порт наружу — к ней ходит только приложение из внутренней сети. В разработке порт нужен обязательно: иначе не подключишься из IDE и не запустишь отладку. Плюс в разработке хочется веб-интерфейсы (pgAdmin, Kafka UI), которым на проде делать нечего.

Скопировать [compose-файл](../DevOps/Docker.md#docker-compose--оркестрация-нескольких-контейнеров) и поправить копию — самый очевидный путь и самый плохой: через месяц в одном файле `postgres:16.3`, в другом `postgres:15`, и никто не помнит почему. Compose даёт четыре механизма, чтобы описать общее один раз, а различия — отдельно.

## Задача

```
        ОБЩЕЕ                    ПРОД                    РАЗРАБОТКА
   ┌─────────────┐        ┌──────────────────┐    ┌────────────────────┐
   │  postgres   │───────▶│ + приложение     │    │ + pgAdmin          │
   │  kafka      │        │ + init-скрипты   │    │ + Kafka UI         │
   └─────────────┘        │ портов наружу нет│    │ + порты наружу     │
                          └──────────────────┘    └────────────────────┘
```

`postgres` и `kafka` есть в обоих окружениях, но настроены по-разному. Нужно описать их один раз.

## Способ 1 — профили

**Профиль** — метка на сервисе, по которой его включают в запуск. Сервис без профиля стартует всегда, сервис с профилем — только когда профиль активирован.

```yaml
services:
  app:
    image: petclinic:latest
    profiles: [prod]
  postgres:
    image: postgres:16.3        # без profiles → есть в любом запуске
  pgadmin:
    image: dpage/pgadmin4:8.12.0
    profiles: [dev]
```

```shell
docker compose --profile dev up -d      # postgres + pgadmin
docker compose --profile prod up -d     # postgres + app
```

> **Потолок способа.** Профиль умеет только включать и выключать сервис целиком. Задать одному и тому же `postgres` разные порты или разные init-скрипты в зависимости от профиля нельзя.

Профили хороши для «добавить необязательный сервис», а не для «настроить сервис по-разному».

## Способ 2 — несколько файлов через `-f`

Штатный и самый старый механизм: базовый файл плюс файл-переопределение, оба скармливаются одной командой.

```shell
docker compose -f compose.yaml -f compose.dev.yaml up -d
```

Файлы применяются **слева направо**: каждый следующий накладывается на результат предыдущих.

Частный случай — **файл `compose.override.yaml`**: если он лежит рядом с `compose.yaml`, Compose подхватывает его сам, без флагов. Удобно для локальных правок разработчика, которые не должны уезжать в репозиторий.

Чтобы не набирать длинную команду каждый раз, набор файлов задают переменной `COMPOSE_FILE`.

## Правила слияния

Ключевая часть темы. Здесь ломаются интуитивные ожидания: **не всё, что написано в верхнем файле, заменяет нижний** — часть значений складывается.

| Что сливается | Как |
|---|---|
| **Мапа** (`environment`, `labels`, `deploy`) | недостающие ключи добавляются, конфликтующие — берётся значение из переопределяющего файла |
| **Список** (`dns`, `expose`) | значения **дописываются** в конец, ничего не заменяется |
| **Команды** (`command`, `entrypoint`, `healthcheck.test`) | заменяются целиком — побеждает последний файл |
| **Ресурсы с ключом уникальности** (`ports`, `volumes`, `secrets`, `configs`) | совпавшая по ключу запись сливается, несовпавшая — **добавляется** |

Из последней строки следует главное практическое правило про порты — см. [Ловушка 1 — порты не заменяются, а складываются](#ловушка-1--порты-не-заменяются-а-складываются).

Когда нужно именно заменить, а не склеить, есть два YAML-тега:

```yaml
services:
  app:
    ports: !override ["8080:8080"]   # заменить список целиком, а не дописать
    environment: !reset null         # убрать атрибут, вернув значение по умолчанию
```

## Способ 3 — `include`

**`include`** — втягивает другой compose-файл целиком как отдельное приложение и копирует все его определения в текущее.

```yaml
# app-compose.yaml
include:
  - services.yaml        # оттуда придут postgres и kafka

services:
  app:
    image: petclinic:latest
```

Отличие от `-f`, которое и есть смысл существования `include`:

> Относительные пути **внутри включённого файла** резолвятся относительно **него самого**, а не корня текущего проекта.

Это позволяет держать подпроект со своими `build: ./src` и бинд-маунтами в отдельной папке и подключать его откуда угодно, ничего не переписывая. Базу для путей можно задать явно через `project_directory`, переменные — через `env_file`.

При конфликте имён сервисов `include` не сливает их, а **предупреждает**: втянутое приложение остаётся самостоятельным. Настроить втянутый сервис под себя через `include` нельзя — для этого следующий способ.

## Способ 4 — `extends`

**`extends`** — берёт **один сервис** из другого файла как основу и позволяет дополнить его на месте.

```yaml
# services.yaml — общий знаменатель
services:
  postgres:
    image: postgres:16.3
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

```yaml
# compose.dev.yaml
services:
  postgres:
    extends:
      service: postgres        # какой сервис берём
      file: services.yaml      # из какого файла (можно опустить — тогда из текущего)
    ports:
      - "5432:5432"            # дополняем под разработку
    environment:
      POSTGRES_PASSWORD: root

volumes:
  postgres_data:               # объявить обязательно — см. ниже
```

Свойства `extends`:

- **транзитивен** — если взятый сервис сам содержит `extends`, Compose раскрутит цепочку до конца;
- **циклы запрещены** — при кольцевой ссылке Compose падает с ошибкой;
- путь в `file:` резолвится **относительно главного compose-файла** (в отличие от `include`).

## Что `extends` не переносит

Самая частая причина «а почему не работает». Compose **не импортирует автоматически** ресурсы, на которые ссылается взятый сервис:

`volumes` · `networks` · `configs` · `secrets` · `depends_on` · `links` · `volumes_from` · ссылки на сервис через `ipc`, `pid`, `network_mode`

> Всё это обязан объявить **тот файл, который расширяет**. Именно поэтому в примере выше блок `volumes: postgres_data:` продублирован — без него запуск падает, хотя в `services.yaml` том объявлен.

```yaml
# НЕПРАВИЛЬНО — том объявлен только в services.yaml
services:
  postgres:
    extends:
      service: postgres
      file: services.yaml
# → ошибка: volume postgres_data undefined

# ПРАВИЛЬНО — объявляем том и здесь
services:
  postgres:
    extends:
      service: postgres
      file: services.yaml

volumes:
  postgres_data:
```

## Сравнение способов

| | Профили | `-f` файлы | `include` | `extends` |
|---|---|---|---|---|
| **Единица переиспользования** | сервис в том же файле | весь файл | весь файл | один сервис |
| **Настроить взятое** | нет | да | нет | да |
| **Пути внутри** | — | от корня проекта | **от самого файла** | от главного файла |
| **Когда брать** | включить/выключить необязательный сервис | локальные правки, `compose.override.yaml` | подключить готовый подпроект | общая база сервисов под разные окружения |

## Ловушки

### Ловушка 1 — порты не заменяются, а складываются

Списки с ключом уникальности **дописываются**. Написал порт в базовом файле и «переопределил» в потомке — получишь оба.

```yaml
# НЕПРАВИЛЬНО
# services.yaml
services:
  postgres:
    image: postgres:16.3
    ports:
      - "5432:5432"

# compose.dev.yaml
services:
  postgres:
    extends:
      service: postgres
      file: services.yaml
    ports:
      - "15432:5432"
# → наружу открыты ОБА порта: 5432 и 15432
```

```yaml
# ПРАВИЛЬНО — в общем файле портов нет вообще
# services.yaml
services:
  postgres:
    image: postgres:16.3     # ports не объявляем

# compose.dev.yaml
services:
  postgres:
    extends:
      service: postgres
      file: services.yaml
    ports:
      - "15432:5432"         # единственный источник правды
```

> **Общее правило.** В базовый файл кладётся только то, что одинаково во всех окружениях. Различающееся туда не пишется вовсе — потому что «переопределить» его потом не всегда получится.

Для прода это ещё и вопрос безопасности: наружу должно торчать только приложение, база и брокер — во внутренней сети.

### Ловушка 2 — окружения дерутся за одни тома

Имя проекта по умолчанию = имя каталога с compose-файлом. Порядок приоритетов:

```
-p флаг  →  COMPOSE_PROJECT_NAME  →  name: в файле  →  имя каталога
```

`compose.dev.yaml` и `compose.prod.yaml` лежат в одной папке, `name:` ни в одном не задан — значит **имя проекта у них одинаковое**. Отсюда общие имена томов и контейнеров: подняли dev поверх prod — Compose пересоздаёт контейнеры, а данные оказываются общими.

```yaml
# compose.prod.yaml
name: petclinic-prod

# compose.dev.yaml
name: petclinic-dev
```

### Ловушка 3 — `.env` рядом с кодом это ещё не секреты

Вынести пароли из compose-файла в `env_file: prod.env` — правильный шаг, но сам по себе он ничего не защищает: файл лежит в том же репозитории. Минимум — `prod.env` в `.gitignore` и пример `prod.env.example` в репозитории. Дальше — механизм `secrets` в Compose, который монтирует секрет файлом в `/run/secrets/`, а не светит его в `docker inspect`.

## Сквозной пример: прод и разработка из одной базы

**Шаг 1. `services.yaml`** — только то, что одинаково везде. Портов и паролей здесь нет намеренно.

```yaml
services:
  postgres:
    image: postgres:16.3
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:                                    # проверка живости
      test: pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB
      interval: 10s
      retries: 5
  kafka:
    image: confluentinc/cp-kafka:7.6.1
    restart: always
    volumes:
      - kafka_data:/var/lib/kafka/data
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://0.0.0.0:9092

volumes:
  postgres_data:
  kafka_data:
```

`$$POSTGRES_USER` — двойной доллар экранирует переменную, чтобы её подставил не Compose при чтении файла, а shell внутри контейнера.

**Шаг 2. `compose.prod.yaml`** — приложение плюс расширенные сервисы, наружу только порт приложения.

```yaml
name: petclinic-prod                # 1. своё имя проекта → свои тома

services:
  app:
    image: petclinic:latest
    restart: always
    ports:
      - "8080:8080"                 # 2. единственный порт наружу
    env_file:
      - prod.env                    # 3. пароли не в этом файле
    depends_on:
      - postgres
  postgres:
    extends:                        # 4. берём общий postgres
      service: postgres
      file: services.yaml
    env_file:
      - prod.env
    volumes:                        # 5. дописывается к тому из services.yaml
      - ./db/init:/docker-entrypoint-initdb.d:ro
  kafka:
    extends:
      service: kafka
      file: services.yaml

volumes:                            # 6. обязательно, extends их не переносит
  postgres_data:
  kafka_data:
```

**Шаг 3. `compose.dev.yaml`** — те же сервисы, но с портами наружу и с интерфейсами.

```yaml
name: petclinic-dev

services:
  postgres:
    extends:
      service: postgres
      file: services.yaml
    ports:
      - "5432:5432"                 # 1. чтобы подключиться из IDE
    environment:
      POSTGRES_DB: petclinic
      POSTGRES_USER: root
      POSTGRES_PASSWORD: root       # 2. локально не секрет
  kafka:
    extends:
      service: kafka
      file: services.yaml
    ports:
      - "9092:9092"                 # 3. не забыть: без этого отладка не подключится
  pgadmin:
    image: dpage/pgadmin4:8.12.0
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: root
  kafkaui:
    image: provectuslabs/kafka-ui:v0.7.2
    ports:
      - "8989:8080"
    environment:
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092

volumes:
  postgres_data:
  kafka_data:
```

**Шаг 4. Запуск.**

```shell
docker compose -f compose.dev.yaml up -d      # поднять окружение разработки
docker compose -f compose.dev.yaml config     # посмотреть ИТОГОВЫЙ конфиг после слияния
docker compose -f compose.prod.yaml up -d     # прод
```

**Как это читать.** `services.yaml` сам по себе не запускается — это библиотека определений. Каждое окружение берёт оттуда сервисы через `extends` и дописывает своё: прод — init-скрипты и `env_file`, разработка — порты и интерфейсы. Версию образа теперь меняешь в одном месте.

> **`docker compose config` — главный инструмент отладки темы.** Он печатает конфиг после всех слияний. Если непонятно, почему открыт лишний порт или потерялась переменная — смотреть надо туда, а не в исходные файлы.

## Spring Boot: окружение поднимается само

Если приложение на Spring Boot, для разработки compose-файл часто вообще не нужно запускать руками. Модуль `spring-boot-docker-compose` при старте приложения сам вызывает `docker compose up`, создаёт бины подключений к поднятым сервисам, а при выходе — `docker compose stop`.

```properties
spring.docker.compose.file=compose.dev.yaml
spring.docker.compose.profiles.active=dev
spring.docker.compose.lifecycle-management=start-and-stop
```

Модуль **только для разработки**: в собранный артефакт он по умолчанию не попадает и в тестах отключён.

## Связи

- [Docker](../DevOps/Docker.md) — базовая заметка: образы, контейнеры, тома, сети и сам Compose;
- [Kubernetes](../DevOps/Kubernetes.md) — следующий уровень: Compose тянет один хост, k8s — кластер с self-healing и масштабированием;
- [Развертывание проекта](../DevOps/Развертывание%20проекта.md) — куда эти файлы попадают на сервере;
- [CI CD](../DevOps/CI%20CD.md) — в пайплайне окружения разводятся теми же файлами;
- Событийная архитектура и Kafka — брокер из примера;
- [Веб-безопасность](../Веб-безопасность.md) — про секреты и не открытые наружу порты.

## Источники

- [Merge Compose files — docs.docker.com](https://docs.docker.com/reference/compose-file/merge/)
- [Services top-level element: `extends`](https://docs.docker.com/reference/compose-file/services/#extends)
- [Include top-level element](https://docs.docker.com/reference/compose-file/include/)
- [Specify a project name](https://docs.docker.com/compose/how-tos/project-name/)
- [Spring Boot — Docker Compose support](https://docs.spring.io/spring-boot/reference/features/dev-services.html)
- [Хабр, Haulmont: Лучший способ создания нескольких окружений для Spring Boot приложения](https://habr.com/ru/companies/haulmont/articles/848696/) — исходная статья
