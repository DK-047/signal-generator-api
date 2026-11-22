# Диаграммы

[← К оглавлению](../README.md)

Диаграммы построены по контракту [`openapi.yaml`](../openapi.yaml) (версия 2.1.0). Они написаны
на [Mermaid](https://mermaid.js.org) и отображаются в GitHub, GitLab и VS Code; те же диаграммы
в виде картинок лежат в папке [`diagrams/`](diagrams/).

| # | Диаграмма | Тип | Для кого |
|---|---|---|---|
| 1 | [Контекст системы](#1-контекст-системы) | Схема компонентов | Все |
| 2 | [Модель ресурсов](#2-модель-ресурсов) | Диаграмма классов | Разработчики, аналитики |
| 3 | [Жизненный цикл команды](#3-жизненный-цикл-команды) | Диаграмма состояний | Разработчики клиента |
| 4 | [Замена конфигурации](#4-замена-конфигурации) | Последовательность | Разработчики клиента, тестировщики |
| 5 | [Отправка команды и опрос статуса](#5-отправка-команды-и-опрос-статуса) | Последовательность | Разработчики клиента |
| 6 | [Обработка Idempotency-Key](#6-обработка-idempotency-key) | Блок-схема | Разработчики сервера и клиента |
| 7 | [Чтение телеметрии по страницам](#7-чтение-телеметрии-по-страницам) | Блок-схема | Разработчики клиента |
| 8 | [Как клиенту реагировать на ответ](#8-как-клиенту-реагировать-на-ответ) | Дерево решений | Разработчики клиента |

---

## 1. Контекст системы

Кто с кем взаимодействует. Схема **логическая** и выведена из контракта: ответы `502` и `504`
говорят о промежуточном интеграционном слое между API и устройством, а `GET /configuration`
читает данные платформы, не обращаясь к устройству. Реальное устройство развёртывания может
отличаться.

```mermaid
flowchart LR
    client["Клиентское приложение<br/>или Postman"]
    subgraph platform["Платформа управления"]
        api["Signal Generator<br/>Management API, /v1"]
        store[("Данные платформы:<br/>конфигурация, команды,<br/>телеметрия")]
        integ["Интеграционный слой"]
    end
    device["Генератор сигналов<br/>(имитируемый)"]

    client -- "HTTPS, Bearer JWT" --> api
    api -- "чтение и запись" --> store
    api -- "команды, конфигурация,<br/>опрос состояния" --> integ
    integ -- "обмен данными" --> device
```

---

## 2. Модель ресурсов

Ресурсы API, их поля и связи. Все ресурсы вложены в устройство (`/v1/devices/{deviceId}/...`).
`?` — необязательное поле.


```mermaid
classDiagram
    direction LR
    class Device {
        +string id
        +string name
        +string model
        +string firmwareVersion
        +Connectivity connectivity
    }
    class DeviceStatus {
        +string deviceId
        +Connectivity connectivity
        +DeviceState state
        +boolean outputEnabled
        +datetime updatedAt
    }
    class DeviceConfiguration {
        +Profile profile
        +int outputLevel 0..100
        +int telemetryIntervalSeconds 1..60
    }
    class Command {
        +string commandId
        +string deviceId
        +CommandType command
        +CommandStatus status
        +datetime createdAt
        +datetime? completedAt
        +CommandFailure? failure
    }
    class CommandFailure {
        +string code
        +string message
    }
    class TelemetryPage {
        +string deviceId
        +TelemetryRecord[] items
        +string? nextCursor
    }
    class TelemetryRecord {
        +datetime timestamp
        +DeviceState state
        +boolean outputEnabled
        +number? temperatureC
        +int? outputLevel
    }
    class Connectivity {
        <<enumeration>>
        ONLINE
        OFFLINE
        UNKNOWN
    }
    class DeviceState {
        <<enumeration>>
        IDLE
        READY
        BUSY
        ERROR
    }
    class CommandStatus {
        <<enumeration>>
        ACCEPTED
        IN_PROGRESS
        SUCCEEDED
        FAILED
    }
    class CommandType {
        <<enumeration>>
        START
        STOP
        RESET
    }
    class Profile {
        <<enumeration>>
        STANDARD
        LOW_NOISE
        TEST
    }

    Device "1" -- "1" DeviceStatus : GET /status
    Device "1" -- "1" DeviceConfiguration : GET, PUT /configuration
    Device "1" -- "0..*" Command : POST /commands
    Command "1" -- "0..1" CommandFailure : при FAILED
    Device "1" -- "0..*" TelemetryRecord : GET /telemetry
    TelemetryPage "1" o-- "0..500" TelemetryRecord : items
    Device ..> Connectivity
    DeviceStatus ..> DeviceState
    Command ..> CommandStatus
    Command ..> CommandType
    DeviceConfiguration ..> Profile
```

Перечисления в ответах **открытые**: могут появиться новые значения
(см. [Версии и совместимость](guides/versioning.md)).

---

## 3. Жизненный цикл команды

Отклонённый запрос (`4xx`) ресурс команды не создаёт. Принятая команда (`202`) проходит
`ACCEPTED → IN_PROGRESS` и завершается `SUCCEEDED` или `FAILED`.


```mermaid
stateDiagram-v2
    direction LR
    state "Отклонена: ответ 4xx,<br/>ресурс не создаётся" as rejected
    state "ACCEPTED" as accepted
    state "IN_PROGRESS" as inprogress
    state "SUCCEEDED<br/>есть completedAt" as succeeded
    state "FAILED<br/>есть completedAt и failure" as failed

    [*] --> rejected: POST /commands отклонён
    [*] --> accepted: POST /commands → 202
    accepted --> inprogress: устройство начало выполнение
    inprogress --> succeeded
    inprogress --> failed
    rejected --> [*]
    succeeded --> [*]
    failed --> [*]
```

---

## 4. Замена конфигурации

`PUT` синхронный: `200` приходит только после того, как устройство применило конфигурацию.
Если подтверждения нет, в силе остаётся прежняя конфигурация.


```mermaid
sequenceDiagram
    autonumber
    participant C as Клиент
    participant A as API
    participant I as Интеграционный слой
    participant D as Устройство

    C->>A: GET /configuration
    A-->>C: 200 действующая конфигурация
    Note over C: меняет нужное поле,<br/>остальные оставляет как были
    C->>A: PUT /configuration (все три поля)
    alt тело не является корректным JSON
        A-->>C: 400 BAD_REQUEST
    else значения не прошли проверку
        A-->>C: 422 VALIDATION_ERROR + details
    else конфигурацию нельзя менять сейчас
        A-->>C: 409 DEVICE_BUSY
    else запрос корректен
        A->>I: применить конфигурацию
        I->>D: новая конфигурация
        alt устройство подтвердило
            D-->>I: применено
            I-->>A: подтверждение
            A-->>C: 200 действующая конфигурация
        else ошибка связи
            I-->>A: сбой обмена
            A-->>C: 502 DEVICE_COMMUNICATION_ERROR
        else нет ответа вовремя
            A-->>C: 504 DEVICE_TIMEOUT
        end
    end
    Note over C,A: при 502 и 504 в силе прежняя конфигурация,<br/>запрос можно безопасно повторить
```

---

## 5. Отправка команды и опрос статуса

API принимает команду и сразу отвечает `202`; результат клиент узнаёт опросом ресурса
из заголовка `Location`. Интервалы опроса — рекомендация, контракт их не задаёт.


```mermaid
sequenceDiagram
    autonumber
    participant C as Клиент
    participant A as API
    participant D as Устройство

    C->>A: POST /commands, Idempotency-Key: K<br/>{"command": "START"}
    A-->>C: 202 ACCEPTED<br/>Location: /commands/cmd-10001
    A->>D: выполнить START
    loop пока статус не итоговый (пауза 1, 2, 4 ... с)
        C->>A: GET /commands/cmd-10001
        A-->>C: 200 ACCEPTED или IN_PROGRESS
    end
    D-->>A: результат выполнения
    C->>A: GET /commands/cmd-10001
    alt команда выполнена
        A-->>C: 200 SUCCEEDED + completedAt
    else выполнить не удалось
        A-->>C: 200 FAILED + completedAt + failure
    end

    Note over C,A: потерян ответ на шаге 1? Повторите POST с тем же ключом K:<br/>вернётся тот же 202 и тот же commandId
```

---

## 6. Обработка Idempotency-Key

Как сервер решает, что ответить на `POST /commands` с ключом. Ключ действует в пределах
владельца токена, устройства и эндпоинта и хранится 24 часа.


```mermaid
flowchart TD
    start(["POST /commands<br/>с Idempotency-Key"]) --> known{"Ключ уже встречался<br/>у этого клиента для этого<br/>устройства за 24 часа?"}

    known -- нет --> process["Обработать как новый запрос"]
    process --> result{"Результат"}
    result -- "принят" --> save["Закрепить ключ за командой"]
    save --> r202["202 ACCEPTED<br/>новая команда"]
    result -- "4xx или 5xx" --> nosave["Ключ не сохраняется:<br/>повтор будет новым запросом"]
    nosave --> rerr["Ответ 4xx или 5xx"]

    known -- да --> inflight{"Исходный запрос<br/>ещё обрабатывается?"}
    inflight -- да --> r409a["409<br/>IDEMPOTENCY_REQUEST_IN_PROGRESS"]
    inflight -- нет --> same{"Метод, путь и тело<br/>совпадают с исходными?"}
    same -- да --> replay["Исходный ответ 202<br/>тот же commandId,<br/>новая команда не создаётся"]
    same -- нет --> r409b["409<br/>IDEMPOTENCY_KEY_REUSED"]
```

Тело сравнивается по значениям JSON: порядок полей и пробелы не важны.

---

## 7. Чтение телеметрии по страницам

Курсор привязан к `from` и `to`: между страницами их менять нельзя, `limit` — можно.


```mermaid
flowchart TD
    begin(["Задать окно from, to и limit"]) --> first["GET /telemetry<br/>без cursor"]
    first --> resp{"Ответ"}
    resp -- "200" --> handle["Обработать items"]
    handle --> more{"nextCursor = null?"}
    more -- нет --> next["GET /telemetry<br/>те же from и to,<br/>cursor = nextCursor"]
    next --> resp
    more -- да --> done(["Все записи прочитаны"])
    resp -- "400 INVALID_CURSOR" --> restart["Начать выборку заново"]
    restart --> first
    resp -- "429 или 503" --> wait["Подождать Retry-After<br/>и повторить тот же запрос"]
    wait --> resp
    resp -- "400 BAD_REQUEST<br/>или 422" --> fix(["Исправить параметры"])
```

---

## 8. Как клиенту реагировать на ответ

Сводка правил из руководств [Ошибки](guides/errors.md) и
[Лимиты и повторные попытки](guides/rate-limits-and-retries.md). Повторы после `5xx`
и таймаутов допустимы только для безопасных запросов: `GET`, `PUT /configuration`
и `POST /commands` с `Idempotency-Key`.


```mermaid
flowchart LR
    resp{"HTTP-статус"}
    resp -- "2xx" --> ok(["Успех"])
    resp -- "401" --> token["Обновить токен<br/>и повторить один раз"]
    resp -- "400, 403, 404, 422" --> nofix(["Не повторять:<br/>исправить запрос или получить доступ"])
    resp -- "409" --> code{"code"}
    code -- "DEVICE_BUSY,<br/>IDEMPOTENCY_REQUEST_IN_PROGRESS" --> later["Повторить с паузой"]
    code -- "DEVICE_NOT_READY" --> status["Проверить /status,<br/>повторить, когда устройство готово"]
    code -- "IDEMPOTENCY_KEY_REUSED" --> bug(["Ошибка клиента:<br/>для новой команды нужен новый ключ"])
    code -- "неизвестный code" --> nofix
    resp -- "429, 503" --> retryafter["Подождать Retry-After<br/>и повторить"]
    resp -- "502, 504" --> backoff["Повторить с экспоненциальной паузой<br/>(только безопасные запросы)"]
    resp -- "500" --> limited["1–2 повтора с паузой<br/>(только безопасные запросы)"]
```
