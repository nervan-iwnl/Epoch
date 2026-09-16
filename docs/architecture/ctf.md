# CTF

CTF — это три сервиса. ctf хранит задачи, флаги и скоринг. ctf-operator — контроллер Kubernetes, который поднимает инстанс задачи для каждой команды и удаляет его по TTL. ctf-proxy пускает команду к её инстансу через один вход. Инстансы — самая опасная часть платформы: внутри намеренно уязвимые программы, а атакуют их намеренно.

## Как команда получает инстанс

```mermaid
sequenceDiagram
  actor T as Команда
  participant C as ctf
  participant API as Kubernetes API
  participant OP as ctf-operator
  participant PX as ctf-proxy

  T->>C: StartInstance(задача)
  C->>C: лимит: один инстанс на задачу у команды
  C->>API: создать ChallengeInstance (команда, задача, TTL)
  API-->>OP: событие watch
  OP->>API: Namespace, Pod под gVisor, Service, NetworkPolicy
  OP->>API: status: ready, адрес, expiresAt
  C-->>T: токен доступа и адрес ctf-proxy
  T->>PX: подключение с токеном
  PX->>PX: проверить токен, найти инстанс команды
  PX->>API: трафик до Service инстанса
```

## Цикл reconcile оператора

```mermaid
flowchart TD
  EV["изменение ChallengeInstance<br/>или таймер requeue"] --> GET{объект существует?}
  GET -- нет --> DONE([ничего не делать])
  GET -- да --> DEL{удаляется или TTL истёк?}
  DEL -- да --> CLEAN["удалить Pod, Service, NetworkPolicy"]
  CLEAN --> FIN["снять finalizer"]
  FIN --> DONE
  DEL -- нет --> ENS["привести к нужному виду:<br/>Namespace · Pod · Service · NetworkPolicy"]
  ENS --> ST["обновить status: адрес, expiresAt"]
  ST --> RQ["requeue к моменту expiresAt"]
```

Контроллер не запоминает, что делал раньше: на каждом цикле он сравнивает желаемое с тем, что есть, и исправляет разницу. Поэтому его можно перезапустить в любой момент.

## Жизнь инстанса

```mermaid
stateDiagram-v2
  [*] --> requested
  requested --> provisioning: оператор создаёт ресурсы
  provisioning --> running: pod готов
  provisioning --> failed: не поднялся за 2 минуты
  running --> expired: TTL истёк
  running --> deleted: команда остановила
  expired --> deleted: ресурсы удалены
  failed --> deleted
  deleted --> [*]
```

## Сетевая изоляция

```mermaid
flowchart LR
  subgraph teamA["namespace ctf-team-a"]
    A1["web-100"]
    A2["pwn-300"]
  end
  subgraph teamB["namespace ctf-team-b"]
    B1["web-100"]
  end
  PX["ctf-proxy"] --> A1
  PX --> A2
  PX --> B1
  A1 -. запрещено .-> B1
  A1 -. запрещено .-> NET(("интернет"))
  A2 -. запрещено .-> APP["сервисы платформы"]
```

- Namespace на команду, default-deny: разрешён только входящий трафик от ctf-proxy.
- Egress запрещён: инстанс не может скачать эксплойт или атаковать внешние адреса.
- Отдельный node pool с taint, инстансы не попадают на ноды с сервисами и judge.
- RuntimeClass `gvisor`: побег из контейнера упирается в ядро в user space, а не в ядро хоста.
- Лимиты CPU и памяти на pod, TTL 1–2 часа.

## Флаги

- **Статические:** в базе хранится `HMAC(pepper, флаг)`, сравнение за постоянное время.
- **Динамические:** `флаг = "epoch{" + hex(HMAC(секрет_задачи, team_id))[:32] + "}"`. Оператор кладёт флаг в инстанс при создании.
- **Флаг чужой команды** узнаётся сразу: вычисляем HMAC для всех команд и находим владельца. Это сигнал модератору.
- **Лимит попыток:** 10 в минуту на команду и задачу.

## Dynamic scoring

Стоимость задачи падает по мере того, как её решают:

```text
value = max(minimum, initial + (minimum − initial) × solves² / decay²)
```

`initial` — стоимость без решений, `minimum` — нижняя граница, `decay` — после скольких решений стоимость доходит до минимума. При каждом новом решении пересчитываются очки всех команд, которые решили эту задачу.

## Задача «сломай judge»

Копия epoch-runner в отдельном инстансе с тем же набором защит. Флаг лежит вне песочницы. Всё, что найдут участники, — готовые баг-репорты для настоящего runner.
