# Путь сабмита

Сабмит проходит через четыре сервиса: submissions принимает и хранит его, judge-orchestrator управляет очередями, judge-worker проверяет в песочнице, а по событию `submission.judged` обновляются таблица, рейтинг и аналитика. Главная гарантия — принятый сабмит всегда получит вердикт ровно один раз, даже если любой сервис упадёт посреди проверки.

## Последовательность

```mermaid
sequenceDiagram
  autonumber
  actor U as Участник
  participant G as gateway
  participant S as submissions
  participant P as problems
  participant N as NATS JetStream
  participant O as judge-orchestrator
  participant W as judge-worker
  participant K as Kafka
  participant R as realtime

  U->>G: Submit(problem, lang, code, idempotency_key)
  G->>S: + внутренний токен
  S->>P: GetLimits(problem) с дедлайном
  P-->>S: лимиты, версия задачи
  S->>S: транзакция: сабмит queued + снимок лимитов
  S-->>U: submission_id
  S->>N: judge.requests
  N->>O: запрос
  O->>O: пул и приоритет, попытка №1
  O->>N: judge.jobs.algo
  N->>W: задача
  W->>W: тесты из кэша или S3
  loop каждый тест
    W->>W: runner: компиляция и запуск
    W->>N: in-progress, продлить ack
  end
  W->>N: judge.results
  W->>N: ack задачи
  N->>O: результат
  O->>N: judge.verdicts
  N->>S: вердикт
  S->>S: транзакция: UPDATE если версия совпала + outbox
  S->>K: submission.judged через relay
  K->>R: событие
  R-->>U: WebSocket: новый статус
```

В вехе 2 шаги 21–23 устроены проще: Kafka ещё нет, статус в CLI приходит через server stream из submissions. Если сервис падает между сохранением сабмита (шаг 5) и публикацией (шаг 7), зависший `queued` перевыставляет sweeper. В вехе 4 публикации переезжают в outbox.

## Статусы сабмита

```mermaid
stateDiagram-v2
  [*] --> queued: принят
  queued --> compiling: воркер взял задачу
  compiling --> judged: CE
  compiling --> running
  running --> judged: все тесты или первый провал
  compiling --> queued: воркер упал
  running --> queued: воркер упал
  queued --> failed: повторы исчерпаны
  judged --> queued: rejudge
  failed --> queued: rejudge
  judged --> [*]
```

Статусы хранит submissions, попытки проверки — judge-orchestrator. Переход в недопустимый статус запрещён проверкой в `UPDATE ... WHERE status IN (...)`.

## Кто за что отвечает

| Сервис | Владеет | Не знает |
| --- | --- | --- |
| submissions | Сабмит, код, статус, итоговый вердикт, снимок лимитов | Как устроены очереди и пулы |
| judge-orchestrator | Попытки, пулы (`algo`, `task`, `course`), приоритеты, dead letter, rejudge | Кто автор и в каком контесте сабмит |
| judge-worker | Выполнение одной задачи от начала до конца | Базы данных любых сервисов |
| problems | Задача, лимиты, тесты в S3 | Что происходит с сабмитами |

## Гарантии

| Что может сломаться | Что происходит | Где закреплено |
| --- | --- | --- |
| Клиент повторил запрос | Тот же `idempotency_key` → тот же сабмит | Уникальный индекс в submissions |
| problems не отвечает при отправке | Быстрый отказ по дедлайну, сабмит не создаётся | Дедлайн и circuit breaker в `pkg/resilience` |
| problems упал после приёма | Проверка идёт: лимиты уже скопированы в сабмит | Снимок лимитов |
| submissions упал до публикации в NATS | Sweeper, потом outbox публикует запрос | Веха 2, потом веха 4 |
| Воркер упал посреди проверки | Нет ack — NATS передоставит задачу другому воркеру | Ack deadline + heartbeat |
| Задача валит воркеры раз за разом | После N попыток — dead letter и статус `failed` | judge-orchestrator |
| Вердикт пришёл дважды | Второй `UPDATE` не проходит по версии и статусу | Optimistic concurrency |
| Kafka недоступна | События копятся в outbox и уходят позже | `pkg/outbox` |
| Событие потерялось где-то ещё | Сверка находит расхождение и шлёт алерт | Периодическая сверка, веха 4 |

## Вердикты

| Вердикт | Значит |
| --- | --- |
| OK | Все тесты пройдены |
| WA | Неверный ответ |
| PE | Нарушен формат вывода |
| TLE | Превышено время: CPU или wall |
| MLE | Превышена память |
| OLE | Слишком большой вывод |
| RE | Ненулевой код выхода или сигнал |
| CE | Ошибка компиляции |
| SV | Запрещённый системный вызов |
| FAIL | Ошибка judge или чекера — виноваты мы, а не участник |

## Цели по времени

- p95 от сабмита до вердикта без учёта времени тестов — меньше 5 с.
- p99 — меньше 15 с.
- Потерянных вердиктов — 0: это проверяет сверка.
