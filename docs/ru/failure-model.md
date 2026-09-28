# Модель отказов

## Статус

Документ фиксирует гарантии и известные пробелы утверждённого синхронного дизайна v1. Он не заявляет поведение retry, recovery, idempotency или eventual delivery.

## Основные гарантии

v1 строится вокруг четырёх правил:

1. Ошибка authentication или validation не создаёт Event и не вызывает Telegram.
2. Telegram вызывается только после успешного persistence Event.
3. Event считается принятым только после успешного persistence.
4. Ошибки после persistence Event не отменяют его acceptance.

Эти правила определяют, что Hub может утверждать уверенно. Они не устраняют неоднозначность, возникающую из-за сетевых ошибок и падения процесса.

## Матрица отказов

| Точка отказа | Состояние Event | Telegram attempt | Состояние Delivery | Смысл для Source | Смысл для Owner |
|---|---|---|---|---|---|
| Некорректный HTTP input | Отсутствует | Нет | Отсутствует | Event not accepted | В History ничего нет |
| Source authentication failed | Отсутствует | Нет | Отсутствует | Event not accepted | В History ничего нет |
| Event validation failed | Отсутствует | Нет | Отсутствует | Event not accepted | В History ничего нет |
| Явная ошибка persistence Event | Не принят | Нет | Отсутствует | Event not accepted | Принятого Event нет |
| Telegram вернул явную ошибку | Сохранён и принят | Одна failed attempt | `failed`, если сохранён | Event accepted | Failed Delivery с sanitized reason |
| Telegram timeout или неоднозначный результат | Сохранён и принят | Одна attempt с неподтверждённым результатом | `failed`, если сохранён | Event accepted | Failed Delivery; фактическая доставка неизвестна |
| Ошибка persistence Delivery | Сохранён и принят | Завершена | Записи нет | Event accepted | `not recorded` |
| Process crash после Event и до сохранения Delivery | Сохранён и принят | Нет, не завершена или результат неизвестен | Записи нет | Response может быть потерян | `not recorded` после restart |

## Значения состояний History

### `success`

Hub сохранил Delivery, для которого Telegram outcome подтверждён как успешный.

### `failed`

Hub сохранил Delivery, для которого не удалось подтвердить успешную доставку в Telegram. Сюда входят явные ошибки Telegram и неоднозначные результаты вроде timeout.

`failed` не всегда доказывает, что Telegram ничего не получил. Например, Telegram мог принять request, а response мог потеряться.

### `not recorded`

Event существует, но Delivery record отсутствует. Возможные причины:

- падение процесса после persistence Event;
- падение во время или после Telegram call;
- ошибка persistence уже известного результата Delivery.

`not recorded` не означает pending, scheduled или автоматически recoverable.

## Неоднозначный persistence Event

Ошибка базы данных или сети может сделать результат persistence неоднозначным для caller. Например, PostgreSQL мог commit-нуть запись, но ответ Hub мог потеряться.

Поскольку в v1 нет idempotency и deduplication:

```text
Source видит failure или не получает response
        -> Source отправляет событие повторно
        -> исходный Event уже может существовать
        -> может появиться duplicate Event
```

Hub не должен заявлять, что retry безопасен в такой ситуации.

## Потерянный HTTP response после acceptance

Hub может сохранить Event и выполнить часть или весь downstream processing, но Source может не получить итоговый HTTP response.

С точки зрения Hub, Event принят. С точки зрения Source, результат неизвестен. Повторная отправка может создать duplicate, поскольку idempotency contract в v1 отсутствует.

## Неоднозначность Telegram

Timeout, потеря соединения или response могут создать следующее состояние:

```text
Telegram мог принять сообщение
Hub не получил confirmation
Hub сохраняет failed, если вообще может сохранить результат
```

Дизайн предпочитает честно обозначить неподтверждённый успех, а не заявлять delivery. Retry не выполняется, потому что он может создать duplicate notification, а delivery idempotency в v1 отсутствует.

## Crash gap между Event и Delivery

Event и Delivery не представлены одной atomic operation. Persistence Event завершается до внешнего Telegram call, а Delivery сохраняется после него.

Поэтому crash может оставить:

```text
Event: persisted
Delivery: absent
```

Именно поэтому domain relationship имеет вид `Event 1 -> 0..1 Delivery`, а History содержит `not recorded`.

## Ошибка persistence Delivery

Если результат Telegram известен, но Storage не смог сохранить Delivery:

- Event остаётся принятым;
- ingestion остаётся успешным на уровне Event contract;
- известный результат может быть потерян из persistent history;
- позже History показывает `not recorded`.

В v1 нет recovery log или replay mechanism для восстановления такого Delivery.

## Явные non-guarantees

v1 не предоставляет:

- автоматический Telegram retry;
- recovery отсутствующих Delivery records;
- idempotent Event submission;
- Event deduplication;
- delivery idempotency;
- eventual delivery;
- strict ordering;
- at-least-once или exactly-once delivery semantics;
- manual Delivery retry;
- историю нескольких delivery attempts.

Это осознанные границы, а не возможности, подразумеваемые текущим дизайном.

## Операционные следствия

- Producers интерпретируют success как acceptance Event, а не Telegram success.
- Owner использует History, чтобы различать записанный успех, записанную ошибку и отсутствующий результат.
- Logs могут содержать developer-oriented diagnostics, но History содержит sanitized user-facing errors.
- Reliability pass должен проверить cancellation, timeouts, graceful shutdown, concurrent requests и каждую описанную границу, не улучшая молча заявленные гарантии.
