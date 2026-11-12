# План реализации

## Статус

Архитектура и дизайн v1 завершены. Ни одна вертикаль реализации ещё не начата.

Проект будет строиться функциональными вертикалями. Каждая вертикаль должна оставлять связную пользовательскую возможность, а не просто заканчивать один технический слой.

## Порядок реализации

| Шаг | Вертикаль | Результат |
|---|---|---|
| 0 | Foundation | Запускаемый Go-сервис с configuration, composition, HTTP server, PostgreSQL connection и migrations |
| 1 | Source onboarding | Owner создаёт Source, один раз получает credential и видит список Sources |
| 2 | Event acceptance | Аутентифицированный Source отправляет валидный Event; persistence определяет acceptance |
| 3 | Telegram delivery | Принятый Event вызывает одну Telegram-попытку; результат по возможности записывается как Delivery |
| 4 | History | Owner видит список и детали Events с семантикой `success`, `failed`, `not recorded` |
| 5 | Credential lifecycle | Owner ротирует credential Source без потери identity и истории |
| 6 | Export и hard delete | Owner экспортирует историю и безвозвратно удаляет Source, Events и Deliveries |
| 7 | Reliability pass | Context propagation, timeouts, cancellation, graceful shutdown, concurrency и failure paths проверены целиком |
| 8 | Testing и completion | Boundaries и критические failure cases системно покрыты; end-to-end flow v1 подтверждён |

## Шаг 0 — Foundation

Цель: создать минимальный запускаемый фундамент для реальных вертикалей.

Планируемая работа:

- инициализировать Go application;
- определить обязательную application configuration;
- создать Startup / Composition;
- подключить PostgreSQL;
- ввести migrations и initial schema, нужную первой вертикали;
- запустить HTTP server;
- добавить минимальную readiness/startup check.

Критерий готовности: приложение стартует из configuration, проверяет обязательные настройки, подключается к PostgreSQL, применяет migrations и обслуживает HTTP, а зависимости собираются снаружи компонентов.

На этом шаге ещё не заявляются Source, Event, Telegram delivery или History.

## Шаг 1 — Source onboarding

Цель: зарегистрировать первого реального отправителя.

Планируемые возможности:

- создать Source с уникальным именем;
- сгенерировать и сохранить credential verification material;
- один раз раскрыть plaintext credential;
- защитить операцию Owner authentication;
- вывести Sources без раскрытия credentials.

Критерий готовности: Owner создаёт `github-ci`, сохраняет его credential в GitHub Actions secrets и затем видит зарегистрированный Source без возможности повторно прочитать secret.

## Шаг 2 — Event acceptance

Цель: реализовать половину Source-to-Hub до добавления Telegram.

Планируемый flow:

```text
GitHub Actions
  -> Source authentication
  -> Event validation
  -> internal Event construction
  -> PostgreSQL persistence
  -> accepted response
```

Критерий готовности: успешный response доказывает, что Event уже существует в PostgreSQL. Bad credentials, invalid input и failure persistence Event не создают принятый Event и не вызывают Telegram.

## Шаг 3 — Telegram delivery

Цель: закончить первый end-to-end product flow.

Планируемая работа:

- добавить Delivery persistence;
- добавить Telegram boundary и форматирование сообщения;
- задать timeout внешнего call;
- вызывать Telegram только после persistence Event;
- переводить confirmed success и known failures в Delivery;
- пытаться сохранить результат, не меняя acceptance Event.

Критерий готовности: реальный Event из GitHub Actions принят, сохранён, один раз отправлен в Telegram и имеет `Delivery(success|failed)`, если persistence Delivery успешен.

## Шаг 4 — History

Цель: сделать Event и Delivery flow наблюдаемым для владельца.

Планируемые возможности:

- показать ограниченный набор последних Events по `received_at`;
- фильтровать `success`, `failed` и `not recorded`;
- показывать детали Event и известную Delivery information;
- защищать History через Owner authentication.

Критерий готовности: Owner диагностирует принятые Events, включая Telegram failure и отсутствие Delivery record.

## Шаг 5 — Credential lifecycle

Цель: заменить доступ Source, не уничтожая его identity и данные.

Планируемые возможности:

- ротировать credential одного Source;
- сделать старый credential недействительным;
- один раз раскрыть новый credential;
- сохранить Source, Events и Deliveries.

Критерий готовности: старый доступ не работает, новый работает, история не меняется.

## Шаг 6 — Export и hard delete

Цель: завершить data lifecycle Source.

Планируемые возможности:

- экспортировать Events Source и существующие Deliveries;
- hard-delete Source и все принадлежащие ему данные;
- гарантировать, что его credential больше не проходит authentication.

Критерий готовности: export не изменяет данные, а после удаления внутри Hub не остаётся Source, Event, Delivery или действующего Source credential.

Формат export остаётся решением реализации.

## Шаг 7 — Reliability pass

Цель: проверить работающую систему против её документированных синхронных гарантий.

Планируемый фокус:

- context propagation;
- timeouts базы и external call;
- request cancellation;
- graceful shutdown;
- concurrent requests и поведение credential rotation;
- failures на каждой границе Event и Delivery;
- configuration validation и safe logging.

Этот проход не должен случайно заявить retry, recovery, idempotency, strict ordering или eventual delivery.

## Шаг 8 — Testing и завершение v1

Цель: системно проверить полный контракт v1.

Планируемое покрытие:

- Coordinator tests правил остановки и продолжения flow;
- Storage integration tests для Source, Event, Delivery, relations, rotation, export и delete;
- HTTP tests для Management, Ingestion, History, authentication, validation и error mapping;
- Telegram boundary tests для confirmed success, explicit error, timeout, ambiguous failure и formatting;
- end-to-end smoke test от создания Source до History;
- запуск в чистой среде, migrations, configuration documentation и проверка секретов.

Focused tests должны появляться и в предыдущих вертикалях. Финальный проход завершает и систематизирует стратегию, а не откладывает всё тестирование на конец.

## Критерии завершения v1

v1 завершена, когда:

- работает основной flow GitHub Actions-to-Hub-to-Telegram;
- принятые Events durable до доставки;
- критические failure paths протестированы;
- create, use, rotate, export и hard delete Source работают end to end;
- History точно различает `success`, `failed` и `not recorded`;
- security boundaries и правила работы с секретами проверены;
- подтверждены startup, migrations, configuration documentation и запуск в чистой среде;
- документированные ограничения совпадают с реальным поведением.
