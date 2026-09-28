# Границы API

## Статус и уровень детализации

API surface v1 утверждён на уровне смысловых операций. Конкретные endpoint paths, HTTP methods, status codes, headers, request/response schemas, механика pagination и формат export намеренно ещё не определены.

В дизайне есть три внешние API-зоны и две независимые caller identity:

```text
Owner
  -> Management API
  -> History API

Registered Source
  -> Ingestion API
```

Это концептуальные границы API. Они не подразумевают отдельные servers или processes.

## Management API

Management operations защищаются Owner credential.

### Create Source

Цель: зарегистрировать нового внешнего отправителя.

Вход:

- уникальное имя Source.

Успешный результат:

- Source создан;
- сгенерирован отдельный credential Source;
- plaintext credential раскрыт один раз для настройки внешнего отправителя.

Точное представление credential и response schema отложены до реализации.

### List Sources

Цель: показать зарегистрированных отправителей.

Список раскрывает стабильную identity и имя Source. Credentials в нём не показываются. Счётчики, last-event data, статистика доставки и другие dashboard fields не входят в scope v1.

### Rotate Source credential

Цель: восстановить доступ после утраты или компрометации credential, не удаляя Source и его историю.

Успешный результат:

- старый credential становится недействительным;
- один новый credential становится актуальным;
- новый plaintext credential раскрывается один раз;
- identity Source, имя, Events и Deliveries остаются прежними.

В v1 у Source один действующий credential. Параллельные ключи, срок действия и история credentials не входят в дизайн.

### Export Source history

Цель: скопировать Events Source и существующие Deliveries за пределы Hub, в частности перед hard delete.

Export — операция чтения. Она не удаляет, не архивирует, не отключает и не изменяет Source. Import и restore находятся за пределами v1. Формат export пока открыт.

### Delete Source

Цель: безвозвратно удалить подключение и все принадлежащие ему данные.

Успешный результат:

- Source удалён;
- его credential больше не работает;
- все его Events удалены;
- все Deliveries этих Events удалены.

Это hard delete, а не archive или soft delete. Пользовательский interface должен предупредить владельца до вызова операции, но после вызова API выполняет именно destructive-семантику.

## Ingestion API

Ingestion аутентифицируется credential конкретного зарегистрированного Source.

### Submit Event

Цель: принять одно универсальное событие от внешнего отправителя.

Входные данные:

| Поле | Требование | Смысл |
|---|---|---|
| `title` | Обязательно, не пусто | Кратко: что произошло |
| `message` | Обязательно, не пусто | Человекочитаемые подробности |
| `occurred_at` | Обязательный структурно корректный timestamp | Когда событие произошло по данным Source |

Успешный результат:

- authentication и validation пройдены;
- Event сохранён;
- Event принят Hub;
- ответ может идентифицировать принятый Event.

Ответ не раскрывает Telegram success, failure или diagnostics. Эти результаты принадлежат owner-facing History.

Семантика ошибок:

- authentication failure: Event не создаётся;
- validation failure: Event не создаётся;
- ошибка persistence Event: acceptance не возвращается, Telegram не вызывается;
- ошибка Telegram или persistence Delivery после сохранения Event: Event остаётся принятым.

## History API

History operations защищены Owner credential.

### List Event history

Цель: показать ограниченный набор последних принятых Events, упорядоченных по `received_at`.

Краткий вид содержит достаточно информации, чтобы определить Event, Source, title, время события и вычисленное delivery state. Точный limit или page size остаётся вопросом реализации.

### Filter by delivery state

Поддерживаемые смысловые фильтры:

- `success`;
- `failed`;
- `not recorded`;
- отсутствие фильтра — все доступные недавние Events.

`not recorded` означает отсутствие Delivery record для Event. Это не сохранённое значение `Delivery.result`.

### Get Event details

Цель: изучить один принятый Event и известный результат Telegram-попытки.

Detail view объединяет:

- identity или имя Source;
- title и message;
- `occurred_at` и `received_at`;
- результат Delivery и `attempted_at`, если они сохранены;
- человекочитаемую причину failed Delivery;
- `not recorded`, если Delivery отсутствует.

Отдельное редактирование или удаление Event отсутствует.

## Граница конфигурации приложения

Следующие значения не входят в runtime API:

- настройки PostgreSQL connection;
- Telegram bot credential;
- единственный Telegram target;
- Owner credential;
- HTTP/server settings.

Они передаются приложению извне при старте. Изменение Telegram configuration в v1 требует изменения внешней конфигурации и перезапуска приложения.

## Операции, намеренно отсутствующие в v1

- rename или общий update Source;
- archive или disable Source;
- повторное чтение существующего Source secret;
- обязательная отдельная операция Source details;
- edit или отдельный delete Event;
- edit, manual retry или attempt history Delivery;
- управление Telegram configuration во время работы;
- управление Destination или Route;
- фильтр History по Source;
- text search, date-range search или analytics;
- полностью определённая pagination;
- import или restore экспортированных данных.

Эти ограничения удерживают API в рамках утверждённых пользовательских сценариев, а не спекулятивного полного CRUD.
