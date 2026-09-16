# Данные

У каждого сервиса своя база. В одном кластере PostgreSQL это отдельные базы с отдельными ролями: роль problems физически не может прочитать базу identity. Ссылки между сервисами — это просто id без внешних ключей. Если сервису часто нужны чужие данные, он держит их локальную копию, которую обновляют события.

## Где что хранится

| Данные | Сервис-владелец | Хранилище | Почему там |
| --- | --- | --- | --- |
| Пользователи, сессии, OAuth-аккаунты, роли | identity | PostgreSQL | Транзакции, уникальность email и ника |
| Задачи, версии, лимиты | problems | PostgreSQL | Версионирование и связи |
| Тесты, пакеты задач | problems | S3 по sha256 | Большие файлы, дедупликация по хэшу |
| Сабмиты и статусы | submissions | PostgreSQL, партиции по месяцам | Самая большая таблица, отсечение партиций |
| Код решений | submissions | S3 по sha256 | Большие и неизменяемые данные |
| Попытки проверки, dead letter | judge-orchestrator | PostgreSQL | История попыток и причины неудач |
| Кэш тестов на ноде | judge-worker | Локальный диск, LRU | Не качать тесты на каждую проверку |
| Контесты, регистрации, вопросы | contests | PostgreSQL | Связи и транзакции |
| Таблицы результатов | standings | Redis sorted sets | Быстрая сортировка; пересобирается из Kafka |
| Рейтинг и его история | rating | PostgreSQL | Аудит изменений |
| Счётчики rate limit | pkg/ratelimit | Redis | Атомарные операции и TTL |
| Аналитика | analytics | ClickHouse | Колоночные агрегаты по миллионам строк |
| Задания ШАД и шаблоны | tasks | PostgreSQL, S3 | Спецификация в базе, файлы в S3 |
| Курсы, дедлайны, ведомость | courses | PostgreSQL | Связи и расчёт оценок |
| Git-репозитории | git-server | Диск, бэкап через `git bundle` в S3 | Git работает с файловой системой |
| Эмбеддинги разборов | tutor | PostgreSQL + pgvector | Векторный поиск рядом с данными |

## identity

```mermaid
erDiagram
  users ||--o{ sessions : "входит"
  users ||--o{ oauth_accounts : "привязывает"
  users ||--o{ user_roles : "имеет"
  users {
    uuid id PK
    text email UK
    text handle UK
    text password_hash "argon2id, может быть пустым"
    bool email_verified
    timestamptz created_at
  }
  sessions {
    bytea token_hash PK "sha256 от токена"
    uuid user_id FK
    timestamptz created_at
    timestamptz expires_at
    text user_agent
  }
  oauth_accounts {
    text provider PK
    text provider_user_id PK
    uuid user_id FK
  }
  user_roles {
    uuid user_id PK, FK
    text role PK "participant, author, admin"
  }
  signing_keys {
    text kid PK
    bytea private_key "зашифрован"
    timestamptz not_after
  }
```

## problems

```mermaid
erDiagram
  problems ||--o{ problem_versions : "версии"
  problem_versions ||--o{ tests : "содержит"
  problems {
    uuid id PK
    text slug UK
    uuid author_id "id из identity, без FK"
    int published_version
    timestamptz created_at
  }
  problem_versions {
    uuid problem_id PK, FK
    int version PK
    text statement_md
    int time_limit_ms
    int memory_limit_mb
    text checker "tokens, float, testlib"
    text package_sha256
  }
  tests {
    uuid problem_id PK, FK
    int version PK, FK
    int index PK
    text input_sha256
    text answer_sha256
    int group_no
  }
  outbox {
    bigint id PK
    text topic
    text key
    bytea payload
    timestamptz created_at
    timestamptz published_at
  }
```

## submissions

```mermaid
erDiagram
  submissions ||--o{ test_results : "результаты"
  submissions {
    uuid id PK "UUIDv7"
    timestamptz created_at PK "ключ партиции"
    uuid user_id "из токена"
    text kind "problem или task"
    uuid problem_id "или task_id"
    int problem_version
    uuid contest_id "может быть пустым"
    text lang
    text source_sha256
    int time_limit_ms "снимок"
    int memory_limit_mb "снимок"
    text status "queued, compiling, running, judged, failed"
    text verdict
    int score
    int time_ms
    int memory_kb
    int version "optimistic lock"
    text idempotency_key "уникальность — в отдельной таблице"
  }
  test_results {
    uuid submission_id PK, FK
    int test_index PK
    text verdict
    int time_ms
    int memory_kb
  }
```

`idempotency_key` уникален в паре с `user_id`. В партиционированной таблице уникальный индекс обязан включать ключ партиции, поэтому ключ идемпотентности вынесен в отдельную небольшую таблицу — это одна из ловушек, которую предстоит обнаружить в задаче про партиционирование.

## judge-orchestrator

```mermaid
erDiagram
  jobs ||--o{ attempts : "попытки"
  jobs {
    uuid submission_id PK
    text pool "algo, task, rejudge"
    int priority
    text state "pending, running, done, dead"
    int attempts_count
    timestamptz created_at
  }
  attempts {
    uuid submission_id PK, FK
    int attempt_no PK
    text worker_id
    timestamptz started_at
    timestamptz finished_at
    text outcome "ok, worker_lost, fail"
    text error
  }
```

## contests и rating

```mermaid
erDiagram
  contests ||--o{ contest_problems : "задачи"
  contests ||--o{ participants : "участники"
  contests ||--o{ clarifications : "вопросы"
  problem_copies ||--o{ contest_problems : "локальная копия"
  contests {
    uuid id PK
    text title
    text rules "icpc, ioi"
    timestamptz starts_at
    interval duration
    interval freeze_before_end
    text state
  }
  contest_problems {
    uuid contest_id PK, FK
    text label PK "A, B, C"
    uuid problem_id FK
  }
  problem_copies {
    uuid problem_id PK "из problems.v1"
    text title
    int version
  }
  participants {
    uuid contest_id PK, FK
    uuid user_id PK
    text handle "копия из identity.v1"
    timestamptz registered_at
  }
  clarifications {
    uuid id PK
    uuid contest_id FK
    uuid user_id
    text question
    text answer
    bool public
  }
```

```mermaid
erDiagram
  ratings ||--o{ rating_changes : "история"
  rating_jobs ||--o{ rating_changes : "создаёт"
  ratings {
    uuid user_id PK
    text discipline PK "algo, ctf, ml"
    float rating
    float deviation
    float volatility
  }
  rating_changes {
    uuid user_id PK, FK
    uuid contest_id PK
    float before
    float after
  }
  rating_jobs {
    uuid contest_id PK
    text state "pending, running, done"
    int checkpoint "сколько участников обработано"
  }
```

## Роли и доступ

- У каждого сервиса роль `<сервис>_app` с правами только на свою базу и роль `<сервис>_migrator` для миграций.
- Миграции лежат в `services/<сервис>/migrations` и выполняются при деплое отдельным шагом, до старта новой версии.
- Миграция должна быть совместима со старой версией кода: сначала добавить колонку, потом начать писать, потом удалить старое (expand и contract).
- В вехе 10 в таблицы с данными организаций добавляется `org_id` и политики RLS.
