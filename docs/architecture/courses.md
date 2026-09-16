# Курсы, git и антиплагиат

Курс — это группа студентов, набор заданий из tasks и дедлайны. Студент сдаёт работу через git push в свой репозиторий. git-server проверяет push хуком и публикует `repo.pushed`, courses создаёт сабмит, а дальше работает та же проверка по стадиям, что и в [заданиях в формате ШАД](grading.md). plagiarism сравнивает сдачи внутри группы, преподаватель ревьюит код по строкам.

## Путь от push до оценки

```mermaid
sequenceDiagram
  actor St as Студент
  participant G as git-server
  participant K as Kafka
  participant C as courses
  participant S as submissions
  participant J as judge
  participant P as plagiarism

  St->>G: git push
  G->>G: pre-receive: размер, запрещённые файлы, время push
  G->>K: repo.pushed (repo, sha, pushed_at)
  K->>C: событие
  C->>C: какое задание, какой дедлайн
  C->>S: SubmitTask(задание, файлы из коммита sha)
  S->>J: проверка по стадиям
  J-->>S: баллы по группам
  S->>K: submission.judged
  K->>C: баллы
  C->>C: штраф за дедлайн, строка ведомости
  K->>P: submission.judged
  P->>P: отпечатки, сравнение внутри группы
  C-->>St: оценка и отчёт у коммита
```

Время сдачи — это `pushed_at` на сервере. Дату коммита легко подделать через `git commit --date`, поэтому ей не верим.

## Компоненты

| Компонент | Как сделать |
| --- | --- |
| Git-сервер | На старте Forgejo с вебхуком, потом свой на Go: smart HTTP, `git-upload-pack` и `git-receive-pack`, хуки |
| Хранение репозиториев | Диск git-server, бэкап через `git bundle` в S3 |
| Шаблон задания | Репозиторий-шаблон курса; приватные тесты хранятся в tasks и в репозиторий не попадают |
| Проверка | Сабмит типа `task`, тот же пайплайн стадий, что и при отправке файлами |
| Дедлайн | Мягкий и жёсткий; время — `pushed_at` |
| Ревью | Комментарии к строкам диффа, статус «принято / на доработку» |
| Антиплагиат | Сравнение внутри группы и с прошлыми годами, анализ истории коммитов |

## Протокол git в двух словах

```mermaid
sequenceDiagram
  participant Cl as git-клиент
  participant G as git-server

  Cl->>G: GET /repo.git/info/refs?service=git-receive-pack
  G-->>Cl: список refs и capabilities в pkt-line
  Cl->>G: POST /repo.git/git-receive-pack: команды обновления refs + packfile
  G->>G: распаковать объекты, pre-receive хук
  alt хук разрешил
    G->>G: обновить refs, post-receive хук
    G-->>Cl: ok refs/heads/main
  else хук отклонил
    G-->>Cl: ng refs/heads/main + причина
  end
```

## Дедлайны и оценка

| Когда сдано | Оценка |
| --- | --- |
| До мягкого дедлайна | Баллы проверки полностью |
| Между мягким и жёстким | Минус 10% за каждые начатые сутки, не больше 50% |
| После жёсткого | Сдача принимается, но в ведомость не идёт |

Оценка за задание — лучшая по итоговому баллу сдача с учётом штрафа. Правила можно менять на уровне курса.

## Антиплагиат

```mermaid
flowchart LR
  SRC["код сдачи"] --> TOK["токенизация<br/>имена → ID, литералы → LIT"]
  TOK --> KG["k-граммы токенов"]
  KG --> HASH["хэши k-грамм"]
  HASH --> WIN["winnowing<br/>минимум в каждом окне"]
  WIN --> IDX[("индекс отпечатков<br/>по заданию")]
  IDX --> PAIRS["пары с общими отпечатками"]
  PAIRS --> SIM["сходство по Жаккару"]
  SIM --> REP["отчёт преподавателю<br/>с подсветкой совпадений"]
  SRC -. v2 .-> AST["AST через tree-sitter<br/>нормализация порядка"]
  AST -.-> PAIRS
```

Дополнительные сигналы из истории git: один огромный коммит за минуту до дедлайна или код, который появился целиком без промежуточных состояний. Это не доказательство, а флаг для преподавателя.

## Модель данных courses

```mermaid
erDiagram
  courses ||--o{ groups : "потоки"
  groups ||--o{ members : "студенты"
  courses ||--o{ assignments : "задания"
  assignments ||--o{ grades : "оценки"
  members ||--o{ grades : "получает"
  courses {
    uuid id PK
    text title
    uuid owner_id
  }
  groups {
    uuid id PK
    uuid course_id FK
    text name
  }
  members {
    uuid group_id PK, FK
    uuid user_id PK
    text role "student, teacher, assistant"
    text repo_id
  }
  assignments {
    uuid id PK
    uuid course_id FK
    uuid task_id "из tasks"
    timestamptz soft_deadline
    timestamptz hard_deadline
  }
  grades {
    uuid assignment_id PK, FK
    uuid user_id PK
    uuid best_submission_id
    int raw_score
    int final_score
    text review_status "none, accepted, rework"
  }
```
