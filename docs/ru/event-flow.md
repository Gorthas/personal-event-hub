# Event Flow

## Статус

Документ определяет утверждённую семантику обработки v1. Он не задаёт endpoint paths, HTTP status codes или детали wire format.

## Happy path

```text
GitHub Actions или другой зарегистрированный Source
        |
        | generic Event input
        v
1. HTTP request decoding
        |
        v
2. Source identification and authentication
        |
        v
3. Event validation
        |
        v
4. Internal Event construction
        |
        v
5. Event persistence in PostgreSQL
        |                           ACCEPTANCE POINT
        v
6. Telegram message formatting
        |
        v
7. One synchronous Telegram attempt
        |
        v
8. Delivery construction and persistence
        |
        v
9. HTTP response: Event accepted
```

Ответ сообщает только, принял ли Hub Event. Результат Telegram не раскрывается Source.

## Семантика по шагам

| Шаг | Ответственность Hub | Результат | Что происходит при ошибке |
|---|---|---|---|
| HTTP decoding | Прочитать authentication material и декодировать input | Transport-neutral input | Остановка; ничего не сохраняется |
| Source authentication | Найти и проверить зарегистрированный Source | Authenticated Source | Остановка; нет Event и вызова Telegram |
| Event validation | Проверить обязательные поля и структуру timestamp | Validated input | Остановка; нет Event и вызова Telegram |
| Event construction | Объединить input, Source, identity Event и `received_at` | Internal Event | Остановка; Event не принят |
| Event persistence | Сохранить Event и связь с Source | Persisted Event | Остановка; not accepted; Telegram не вызывается |
| Telegram formatting | Сформировать человекочитаемое сообщение | Telegram representation | Event остаётся принятым; delivery failed |
| Telegram attempt | Выполнить одну синхронную best-effort попытку | Confirmed success или known failure | Event принят; retry отсутствует |
| Delivery persistence | Сохранить результат, время попытки и возможную ошибку | Delivery record | Event принят; History может показать `not recorded` |
| HTTP response | Сообщить acceptance Event | Success с identity Event или ошибка acceptance | Детали Delivery не возвращаются |

## Входной контракт

Source передаёт:

- `title`;
- `message`;
- `occurred_at`.

Все поля обязательны. `title` и `message` не должны быть пустыми, а `occurred_at` должен иметь допустимое представление timestamp.

Source не передаёт:

- доверенную identity Source внутри payload;
- identity Event;
- `received_at`;
- delivery state;
- Telegram data.

Identity Source устанавливается authentication. Identity Event и `received_at` создаёт Hub. Delivery относится к downstream processing.

## Точка acceptance

Центральный контракт v1:

```text
valid request != accepted Event
persisted Event = accepted Event
```

Успешные authentication и validation необходимы, но недостаточны. Hub сообщает acceptance только после успешного сохранения Event.

После persistence:

- Telegram failure не отменяет acceptance;
- ошибка сохранения Delivery не отменяет acceptance;
- Event остаётся доступным в History;
- отсутствие Delivery отображается как `not recorded`.

## Семантика Delivery

v1 выполняет ровно одну Telegram-попытку для каждого нормально обработанного принятого Event. Попытка синхронная и best-effort:

- подтверждённый успех Telegram становится `Delivery(success)`;
- явная ошибка Telegram, timeout или другой неподтверждённый результат становится `Delivery(failed)`, если этот результат удалось сохранить;
- Hub не выполняет автоматический retry;
- Hub не обещает eventual delivery.

Timeout или потерянный ответ трактуются консервативно: Hub не может подтвердить успех, даже если Telegram фактически получил сообщение.

## Граница ответа

Source отвечает за передачу Event в Hub. Он не отвечает за операцию Hub-to-Telegram.

Поэтому:

```text
Event не сохранён
-> Event not accepted

Event сохранён
-> Event accepted
```

Telegram success, Telegram failure и ошибка сохранения Delivery не меняют этот смысл ответа. Это диагностическая информация для владельца через History.

Поскольку доставка синхронна, Source в v1 всё равно ждёт Telegram-попытку, хотя не получает её результат. Устранение этой задержки потребует будущего asynchronous processing design.

## History view

History упорядочивается по времени `received_at`, созданному Hub, и показывает:

- `success` — сохранён успешный Delivery;
- `failed` — сохранён неуспешный Delivery;
- `not recorded` — Event есть, Delivery отсутствует.

Event details объединяет неизменяемые данные Event с известными данными Delivery, если они существуют.

## Явные non-guarantees

v1 не гарантирует:

- безопасный replay одного submission;
- deduplication;
- recovery после падения между обработкой Event и Delivery;
- retry после ошибки Telegram;
- eventual Telegram delivery;
- строгий порядок Events или notifications.

Следствия разобраны в [Модели отказов](failure-model.md).
