# Доменная модель

## Статус

Это утверждённая концептуальная модель v1. Она не является SQL-схемой и не задаёт типы идентификаторов, типы колонок, названия таблиц, indexes или детали migrations.

## Обзор модели

```text
Source
  |
  | 1 : N
  v
Event
  |
  | 1 : 0..1
  v
Delivery
```

В модели всего три domain entity: Source, Event и Delivery.

## Source

Source — зарегистрированный внешний отправитель, которому разрешено присылать Events в Hub.

### Утверждённые концептуальные данные

- стабильная внутренняя identity;
- уникальное человекочитаемое имя;
- данные для проверки credential конкретного Source.

Точный вид идентификатора и credential остаётся решением реализации. В частности, концептуальное наличие credentials не означает хранение plaintext secret.

### Ответственности и связи

- Один Source может владеть множеством Events.
- Каждый Event принадлежит ровно одному Source.
- Source определяется успешной authentication, а не доверенным полем из Event payload.
- Имя Source обозначает подключение для владельца и показывается в History и Telegram message.

### Жизненный цикл

```text
create
  -> active and usable
  -> credential may be rotated
  -> export may be requested
  -> hard delete
```

Identity Source и его история сохраняются при ротации credential. Старый credential перестаёт работать, новый раскрывается один раз.

Общей операции изменения или переименования Source в v1 нет. Также нет состояний disable, archive или soft delete.

### Удаление

Удаление Source уничтожает:

- сам Source;
- данные проверки его credential;
- все принадлежащие ему Events;
- все Deliveries этих Events.

Исторический snapshot Source внутри Hub не остаётся. Перед удалением владелец может экспортировать историю, но export необязателен и не изменяет данные Hub.

## Event

Event — неизменяемая внутренняя запись о факте, сообщённом аутентифицированным Source и успешно сохранённом Hub.

### Входные данные от Source

- `title` — обязателен и не пуст;
- `message` — обязателен и не пуст;
- `occurred_at` — обязателен и структурно корректен.

v1 проверяет представление timestamp, но не оценивает правдоподобность указанного времени события.

### Контекст, добавляемый Hub

- стабильная identity Event;
- связь с аутентифицированным Source;
- `received_at`, созданный Hub.

Получается следующая концептуальная модель:

```text
Event
-----
identity
Source
title
message
occurred_at
received_at
```

### Жизненный цикл

Event становится принятым Event только после успешного persistence. После создания он не редактируется владельцем и не изменяется из-за последующих результатов Telegram.

Отдельной операции удаления Event в v1 нет. Event удаляется только вместе со своим Source.

### Намеренно исключённые данные

Event не содержит:

- delivery status или error;
- Telegram configuration;
- retry count;
- severity;
- произвольные metadata;
- исходный входящий request.

Эти значения либо относятся к другим зонам, либо не имеют утверждённого сценария v1.

## Delivery

Delivery фиксирует известный результат единственной синхронной попытки Hub отправить Event через Telegram.

### Утверждённые концептуальные данные

- связь с одним Event;
- `result`: `success` или `failed`;
- `attempted_at`;
- человекочитаемый `error` при `failed`.

При `success` ошибка отсутствует. При `failed` она описывает только то, что действительно известно Hub, например timeout или явную ошибку Telegram. Delivery не предназначен для хранения raw external responses, headers, stack traces и другой низкоуровневой диагностики.

### Связь и жизненный цикл

- У Event может быть ноль или один Delivery в v1.
- Delivery принадлежит ровно одному Event.
- Delivery создаётся только после получения известного результата попытки.
- Состояния `pending` нет.
- Сохранённый Delivery не изменяется и не retry-ится.
- Delivery удаляется вместе с Event.

Ноль в связи `0..1` значим: падение процесса или ошибка сохранения результата могут оставить сохранённый Event без Delivery.

## Проекция History

History вычисляет состояние, не меняя entity Delivery:

```text
Delivery(result = success) -> success
Delivery(result = failed)  -> failed
no Delivery                -> not recorded
```

`not recorded` не означает pending, scheduled или recoverable. Оно означает лишь, что у Hub нет сохранённого результата Delivery для Event.

## Почему в v1 нет Destination и Route

Весь Hub использует один Telegram target из deployment configuration. Владелец не управляет набором назначений, а Hub не выбирает между маршрутами.

Будущая эволюция намеренно отложена:

```text
несколько targets
        -> появляется смысл Destination
        -> нужен выбор target
        -> появляется смысл routing
        -> может стать оправдан Route
```

До появления такой необходимости Destination и Route лишь добавляли бы модель, которая не выполняет никакой работы в v1.

## Открытые детали реализации

Дизайн намеренно не определяет:

- типы идентификаторов;
- названия SQL tables и columns;
- точное представление timestamp;
- максимальную длину title и message;
- конкретный механизм hashing/verification credentials;
- формат export-файла.

Эти решения относятся к соответствующим вертикалям реализации.
