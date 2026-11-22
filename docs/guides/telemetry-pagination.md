# Телеметрия и постраничная выдача

[← К оглавлению](../../README.md)

`GET /v1/devices/{deviceId}/telemetry` возвращает записи телеметрии страницами. Используется
**курсорная** постраничная выдача: она не теряет и не повторяет записи, даже если во время чтения
появляются новые.

## Параметры

| Параметр | По умолч. | Описание |
|---|---|---|
| `from` | — | Начало окна, **включительно** (`2026-01-15T10:00:00Z`) |
| `to` | — | Конец окна, **не включая**. Если заданы оба — `to` позже `from`, иначе `422` |
| `limit` | `100` | Записей на странице, 1–500 |
| `cursor` | — | Значение `nextCursor` из предыдущего ответа |

Полуоткрытое окно `[from, to)` позволяет без пересечений читать соседние интервалы:
`[10:00, 11:00)`, затем `[11:00, 12:00)`.

## Порядок записей

Записи идут по возрастанию `timestamp`. Записи с одинаковым временем упорядочены стабильно
(по внутреннему идентификатору), поэтому при чтении по страницам ничего не теряется
и не повторяется.

## Как читать все страницы

1. Запросите первую страницу **без** `cursor`.
2. Если `nextCursor` — строка, запросите следующую страницу с `cursor=<nextCursor>`
   и **теми же** `from` и `to`.
3. Повторяйте, пока `nextCursor` не станет `null`.

```bash
# Страница 1
curl "$BASE_URL/v1/devices/sg-001/telemetry?from=2026-01-15T10:00:00Z&to=2026-01-15T11:00:00Z&limit=2" \
  -H "Authorization: Bearer $TOKEN"
# → { "items": [...2 записи...], "nextCursor": "eyJvIjoyfQ" }

# Страница 2: тот же from/to, курсор из ответа
curl "$BASE_URL/v1/devices/sg-001/telemetry?from=2026-01-15T10:00:00Z&to=2026-01-15T11:00:00Z&limit=2&cursor=eyJvIjoyfQ" \
  -H "Authorization: Bearer $TOKEN"
# → { "items": [...1 запись...], "nextCursor": null }   ← данные закончились
```

```javascript
async function* readTelemetry(deviceId, { from, to, limit = 500 } = {}) {
  let cursor = null;
  do {
    const q = new URLSearchParams({ limit: String(limit) });
    if (from) q.set("from", from);
    if (to) q.set("to", to);
    if (cursor) q.set("cursor", cursor);
    const page = await call(`${BASE_URL}/v1/devices/${deviceId}/telemetry?${q}`, { headers: authHeaders });
    yield* page.items;
    cursor = page.nextCursor;
  } while (cursor);
}
```

## Правила курсора

| Можно | Нельзя |
|---|---|
| Менять `limit` между страницами | Менять `from` и `to` между страницами |
| Передавать курсор как есть | Разбирать, изменять или составлять курсор самостоятельно |
| | Передавать пустой `cursor=` (минимум 1 символ) |

Если курсор недействителен или получен с другими `from`/`to`, вернётся `400 INVALID_CURSOR`.
Начните выборку заново без `cursor`.

## Ошибки

| Ситуация | Ответ |
|---|---|
| `limit=abc`, `from=не-дата` | `400 BAD_REQUEST` |
| Недействительный курсор | `400 INVALID_CURSOR` |
| `limit` вне 1–500 | `422 VALIDATION_ERROR` |
| `from` не раньше `to` | `422 VALIDATION_ERROR` |

## Поля записи

Обязательны `timestamp`, `state`, `outputEnabled`. Поля `temperatureC` и `outputLevel` могут
отсутствовать — проверяйте их наличие. Будьте готовы и к новым полям
(см. [Версии и совместимость](versioning.md)).
