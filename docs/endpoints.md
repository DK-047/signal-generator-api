# Справочник эндпоинтов

[← К оглавлению](../README.md)

Базовый URL: `https://api.example.com` (пример). Все запросы требуют заголовок
`Authorization: Bearer <токен>`. Все ответы содержат заголовок `X-Correlation-Id`.
Полное машиночитаемое описание — [`openapi.yaml`](../openapi.yaml).

| Метод | Путь | Операция | Успех |
|---|---|---|---|
| `GET` | [`/v1/devices/{deviceId}`](#получить-информацию-об-устройстве) | Сведения об устройстве | `200` |
| `GET` | [`/v1/devices/{deviceId}/status`](#получить-текущий-статус-устройства) | Текущее состояние | `200` |
| `GET` | [`/v1/devices/{deviceId}/configuration`](#получить-конфигурацию-устройства) | Действующая конфигурация | `200` |
| `PUT` | [`/v1/devices/{deviceId}/configuration`](#заменить-конфигурацию-устройства) | Замена конфигурации | `200` |
| `POST` | [`/v1/devices/{deviceId}/commands`](#отправить-команду-устройству) | Отправка команды | `202` |
| `GET` | [`/v1/devices/{deviceId}/commands/{commandId}`](#получить-статус-выполнения-команды) | Статус команды | `200` |
| `GET` | [`/v1/devices/{deviceId}/telemetry`](#получить-телеметрию-устройства) | Записи телеметрии | `200` |

## Коды ответов по операциям

| Операция | 400 | 401 | 403 | 404 | 409 | 422 | 429 | 500 | 502 | 503 | 504 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `GET /devices/{id}` | | ✓ | ✓ | ✓ | | | ✓ | ✓ | | ✓ | |
| `GET /status` | | ✓ | ✓ | ✓ | | | ✓ | ✓ | ✓ | ✓ | ✓ |
| `GET /configuration` | | ✓ | ✓ | ✓ | | | ✓ | ✓ | | ✓ | |
| `PUT /configuration` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `POST /commands` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `GET /commands/{id}` | | ✓ | ✓ | ✓ | | | ✓ | ✓ | | ✓ | |
| `GET /telemetry` | ✓ | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | | ✓ | |

Значение каждого кода и поля `code` в теле — в руководстве [Ошибки](guides/errors.md).

## Общие параметры пути

| Параметр | Тип | Ограничения | Описание |
|---|---|---|---|
| `deviceId` | string | 1–64 символа | Идентификатор устройства, например `sg-001` |
| `commandId` | string | 1–64 символа | Идентификатор команды из ответа на её отправку, например `cmd-10001` |

---

## Получить информацию об устройстве

`GET /v1/devices/{deviceId}` · operationId `getDevice` · тег Devices

Что это за устройство: название, модель, версия прошивки и доступно ли оно сейчас для платформы.
Чем устройство занято прямо сейчас, показывает [статус](#получить-текущий-статус-устройства).

**Пример**

```bash
curl "$BASE_URL/v1/devices/sg-001" -H "Authorization: Bearer $TOKEN"
```

**Ответ `200` — объект `Device`**

```json
{
  "id": "sg-001",
  "name": "Signal Generator 001",
  "model": "SG-Demo",
  "firmwareVersion": "1.2.0",
  "connectivity": "ONLINE"
}
```

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `id` | string, 1–64 | да | Идентификатор устройства |
| `name` | string, 1–128 | да | Название |
| `model` | string, 1–64 | да | Модель |
| `firmwareVersion` | string, 1–32 | да | Версия прошивки |
| `connectivity` | `ONLINE` \| `OFFLINE` \| `UNKNOWN` | да | Доступность устройства для платформы (канал связи) |

**Ошибки:** 401, 403, 404 (`DEVICE_NOT_FOUND`), 429, 500, 503.

---

## Получить текущий статус устройства

`GET /v1/devices/{deviceId}/status` · operationId `getDeviceStatus` · тег Devices

Чем устройство занято прямо сейчас и включён ли выход. Удобно вызвать перед отправкой команды
или сменой конфигурации.

**Пример**

```bash
curl "$BASE_URL/v1/devices/sg-001/status" -H "Authorization: Bearer $TOKEN"
```

**Ответ `200` — объект `DeviceStatus`**

```json
{
  "deviceId": "sg-001",
  "connectivity": "ONLINE",
  "state": "READY",
  "outputEnabled": false,
  "updatedAt": "2026-01-15T10:20:30Z"
}
```

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `deviceId` | string, 1–64 | да | Идентификатор устройства |
| `connectivity` | `ONLINE` \| `OFFLINE` \| `UNKNOWN` | да | Доступность для платформы |
| `state` | `IDLE` \| `READY` \| `BUSY` \| `ERROR` | да | Рабочее состояние |
| `outputEnabled` | boolean | да | Включён ли выход |
| `updatedAt` | date-time | да | Когда получено это состояние |

`connectivity` и `state` независимы. Если устройство недоступно (`OFFLINE`/`UNKNOWN`), `state`
содержит **последнее известное** состояние, а `updatedAt` — время, когда оно было получено.

| `state` | Значение |
|---|---|
| `IDLE` | Простаивает |
| `READY` | Готово к работе |
| `BUSY` | Занято (например, конфигурацию в этом состоянии менять нельзя — `409 DEVICE_BUSY`) |
| `ERROR` | В состоянии ошибки |

**Ошибки:** 401, 403, 404, 429, 500, 502 (`DEVICE_COMMUNICATION_ERROR`), 503, 504 (`DEVICE_TIMEOUT`).

---

## Получить конфигурацию устройства

`GET /v1/devices/{deviceId}/configuration` · operationId `getDeviceConfiguration` · тег Configuration

Действующая конфигурация — та, что устройство последний раз подтвердило (её же вернул последний
успешный `PUT`). Запрос читает данные платформы и к устройству не обращается, поэтому работает,
даже когда устройство недоступно.

**Пример**

```bash
curl "$BASE_URL/v1/devices/sg-001/configuration" -H "Authorization: Bearer $TOKEN"
```

**Ответ `200` — объект `DeviceConfiguration`**

```json
{
  "profile": "STANDARD",
  "outputLevel": 10,
  "telemetryIntervalSeconds": 5
}
```

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `profile` | `STANDARD` \| `LOW_NOISE` \| `TEST` | да | Профиль работы |
| `outputLevel` | integer, 0–100 | да | Уровень выхода (абстрактная демонстрационная величина, не физические единицы) |
| `telemetryIntervalSeconds` | integer, 1–60 | да | Как часто записывается телеметрия, секунд |

**Ошибки:** 401, 403, 404, 429, 500, 503.

---

## Заменить конфигурацию устройства

`PUT /v1/devices/{deviceId}/configuration` · operationId `updateDeviceConfiguration` · тег Configuration

Заменяет конфигурацию **целиком**: отправляйте все три поля, лишние поля не допускаются.
Чтобы изменить одно поле — сначала [получите текущую конфигурацию](#получить-конфигурацию-устройства).

**Тело запроса — `DeviceConfiguration`** (поля и ограничения — как в ответе GET выше)

```bash
curl -X PUT "$BASE_URL/v1/devices/sg-001/configuration" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"profile": "STANDARD", "outputLevel": 10, "telemetryIntervalSeconds": 5}'
```

**Ответ `200`** — действующая конфигурация (то же, что отправили).

**Что означает `200`.** Запрос синхронный: `200` приходит, только когда конфигурация сохранена
**и уже применена устройством**. Если устройство не подтвердило применение (`502` или `504`),
в силе остаётся прежняя конфигурация — частично применённой не бывает, запрос можно повторить.

| Ситуация | Ответ |
|---|---|
| Тело не является корректным JSON | `400 BAD_REQUEST` |
| Поле пропущено, лишнее поле, неверный тип, значение вне диапазона | `422 VALIDATION_ERROR` |
| Конфигурацию нельзя менять в текущем состоянии (например, устройство занято) | `409 DEVICE_BUSY` — подождите и повторите |
| Нет связи с устройством / не ответило вовремя | `502` / `504` — прежняя конфигурация в силе |

**Ошибки:** 400, 401, 403, 404, 409, 422, 429, 500, 502, 503, 504.

---

## Отправить команду устройству

`POST /v1/devices/{deviceId}/commands` · operationId `submitDeviceCommand` · тег Commands

Запускает, останавливает или сбрасывает устройство. Выполнение **асинхронное**: API сразу отвечает
`202 Accepted`, а результат смотрят через [статус команды](#получить-статус-выполнения-команды).
Подробно — [Команды и идемпотентность](guides/commands-and-idempotency.md).

**Заголовки**

| Заголовок | Обяз. | Описание |
|---|---|---|
| `Idempotency-Key` | нет (рекомендуется) | 1–128 символов. Защищает от дублирования при повторной отправке. Хранится 24 часа |

**Тело запроса — `CommandRequest`**

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `command` | `START` \| `STOP` \| `RESET` | да | Команда |
| `parameters` | object | нет | Параметры команды. В этой версии всегда пустой объект `{}` (по умолчанию `{}`) |

```bash
curl -i -X POST "$BASE_URL/v1/devices/sg-001/commands" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 3f6c1a4e-8b1d-4c55-9a0e-2d7f5b1c9e30" \
  -d '{"command": "START", "parameters": {}}'
```

**Ответ `202` — объект `Command`**

```http
HTTP/1.1 202 Accepted
Location: /v1/devices/sg-001/commands/cmd-10001
X-Correlation-Id: 7f3c2b1e-0000-4000-8000-000000000010

{
  "commandId": "cmd-10001",
  "deviceId": "sg-001",
  "command": "START",
  "status": "ACCEPTED",
  "createdAt": "2026-01-15T10:21:00Z"
}
```

`Location` — относительный URL статуса команды.

| Ситуация | Ответ |
|---|---|
| Тело не является корректным JSON | `400 BAD_REQUEST` |
| Неизвестная команда, непустой `parameters`, лишнее поле | `422 VALIDATION_ERROR` |
| Устройство не может выполнить команду сейчас | `409 DEVICE_NOT_READY` |
| Тот же `Idempotency-Key` с другим телом | `409 IDEMPOTENCY_KEY_REUSED` |
| Тот же ключ, исходный запрос ещё обрабатывается | `409 IDEMPOTENCY_REQUEST_IN_PROGRESS` |

Отклонённый запрос (`4xx`) ресурс команды **не создаёт**.

**Ошибки:** 400, 401, 403, 404, 409, 422, 429, 500, 502, 503, 504.

---

## Получить статус выполнения команды

`GET /v1/devices/{deviceId}/commands/{commandId}` · operationId `getCommand` · тег Commands

**Пример**

```bash
curl "$BASE_URL/v1/devices/sg-001/commands/cmd-10001" -H "Authorization: Bearer $TOKEN"
```

**Ответ `200` — объект `Command`** (пример неудачной команды)

```json
{
  "commandId": "cmd-10002",
  "deviceId": "sg-001",
  "command": "RESET",
  "status": "FAILED",
  "createdAt": "2026-01-15T10:25:00Z",
  "completedAt": "2026-01-15T10:25:03Z",
  "failure": {
    "code": "HARDWARE_FAULT",
    "message": "The simulated device reported a fault during reset."
  }
}
```

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `commandId` | string, 1–64 | да | Идентификатор команды |
| `deviceId` | string, 1–64 | да | Устройство |
| `command` | `START` \| `STOP` \| `RESET` | да | Команда |
| `status` | `ACCEPTED` \| `IN_PROGRESS` \| `SUCCEEDED` \| `FAILED` | да | Ход выполнения |
| `createdAt` | date-time | да | Когда команда принята |
| `completedAt` | date-time | при `SUCCEEDED`, `FAILED` | Когда завершилась |
| `failure` | object `{code, message}` | при `FAILED` | Причина сбоя; `code` — открытый набор значений |

| `status` | Итоговый? | `completedAt` | `failure` |
|---|---|---|---|
| `ACCEPTED` | нет | нет | нет |
| `IN_PROGRESS` | нет | нет | нет |
| `SUCCEEDED` | да | есть | нет |
| `FAILED` | да | есть | есть |

**Ошибки:** 401, 403, 404 (`DEVICE_NOT_FOUND` или `COMMAND_NOT_FOUND`), 429, 500, 503.

---

## Получить телеметрию устройства

`GET /v1/devices/{deviceId}/telemetry` · operationId `getDeviceTelemetry` · тег Telemetry

Записи, которые устройство делало с течением времени. Записи идут по возрастанию `timestamp`,
порядок стабилен. Подробно — [Телеметрия и постраничная выдача](guides/telemetry-pagination.md).

**Параметры запроса**

| Параметр | Тип | По умолч. | Описание |
|---|---|---|---|
| `from` | date-time | — | Начало окна, **включительно** |
| `to` | date-time | — | Конец окна, **не включая**. Если заданы оба — `to` позже `from` |
| `limit` | integer, 1–500 | `100` | Записей на странице. Между страницами менять можно |
| `cursor` | string, 1–512 | — | `nextCursor` из предыдущего ответа. Привязан к `from` и `to` |

```bash
curl "$BASE_URL/v1/devices/sg-001/telemetry?from=2026-01-15T10:00:00Z&to=2026-01-15T11:00:00Z&limit=2" \
  -H "Authorization: Bearer $TOKEN"
```

**Ответ `200` — объект `TelemetryPage`**

```json
{
  "deviceId": "sg-001",
  "items": [
    { "timestamp": "2026-01-15T10:20:30Z", "state": "READY", "outputEnabled": false, "temperatureC": 36.6, "outputLevel": 10 },
    { "timestamp": "2026-01-15T10:20:35Z", "state": "READY", "outputEnabled": false, "temperatureC": 36.7, "outputLevel": 10 }
  ],
  "nextCursor": "eyJvIjoyfQ"
}
```

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `deviceId` | string | да | Устройство |
| `items` | array of `TelemetryRecord` | да | Записи по возрастанию `timestamp` |
| `nextCursor` | string \| null | да | Курсор следующей страницы; `null` — записей больше нет |

**`TelemetryRecord`**

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `timestamp` | date-time | да | Время записи |
| `state` | `IDLE` \| `READY` \| `BUSY` \| `ERROR` | да | Рабочее состояние |
| `outputEnabled` | boolean | да | Был ли включён выход |
| `temperatureC` | number | нет | Температура, °C (иллюстративное значение) |
| `outputLevel` | integer, 0–100 | нет | Уровень выхода |

| Ситуация | Ответ |
|---|---|
| `limit=abc`, `from=не-дата` | `400 BAD_REQUEST` |
| Курсор недействителен или получен с другими `from`/`to` | `400 INVALID_CURSOR` |
| `limit=0`, `limit=1000` | `422 VALIDATION_ERROR` |
| `from` не раньше `to` | `422 VALIDATION_ERROR` |

**Ошибки:** 400, 401, 403, 404, 422, 429, 500, 503.
