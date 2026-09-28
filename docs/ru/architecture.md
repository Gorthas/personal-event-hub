# Архитектура

## Статус

Документ описывает утверждённую архитектуру v1. Дизайн завершён, реализация ещё не начата.

## Контекст системы

Personal Event Hub находится между внешними источниками событий и единственным персональным каналом уведомлений.

```text
External Source                    Owner
      |                              |
      | Submit Event                 | Manage Sources / inspect History
      v                              v
                HTTP boundary
                      |
                      v
                 Coordinator
                  /        \
                 v          v
              Storage    Telegram boundary
                 |          |
                 v          v
             PostgreSQL  Telegram Bot API
```

GitHub Actions — первый запланированный клиент-Source. Он отправляет универсальный контракт Hub, а не исходный GitHub-специфичный payload.

## Архитектурные цели

Дизайн v1 ставит в приоритет:

- явную границу приёма Event;
- durable-историю Event до внешней доставки;
- отделение правил Hub от деталей HTTP, PostgreSQL и Telegram;
- небольшие тестируемые зависимости;
- честное отражение отказов;
- намеренно ограниченный первый релиз.

Дизайн не пытается создать универсальную event-платформу, многоканальный роутер или распределённую систему сообщений.

## Ответственности во время работы приложения

### HTTP boundary

HTTP boundary умеет:

- принимать и декодировать HTTP-запросы;
- извлекать authentication material;
- отклонять некорректные транспортные данные;
- передавать transport-neutral данные application logic;
- преобразовывать результаты и ошибки приложения в HTTP-ответы.

Он не отвечает за порядок операций, доступ к PostgreSQL, вызов Telegram или правило, по которому persistence определяет acceptance.

### Coordinator

Coordinator владеет правилами операций Personal Event Hub. Для Submit Event он координирует:

```text
authenticate
-> validate
-> construct Event
-> persist Event
-> attempt Telegram delivery
-> construct Delivery
-> persist Delivery if possible
```

Он решает, где flow должен остановиться и какие downstream-ошибки влияют или не влияют на acceptance Event. Coordinator зависит от возможностей Storage и Telegram, но не владеет их техническими деталями.

### Storage boundary

Storage умеет читать и сохранять Source, Event и Delivery в PostgreSQL. Он предоставляет операции над данными, необходимые Management, Ingestion и History flow.

Storage не аутентифицирует caller, не решает, допустим ли payload как Event, не вызывает Telegram и не определяет смысл пользовательской операции. Application-решениями владеет Coordinator, а Storage отвечает за их persistent-исполнение.

### Telegram boundary

Telegram boundary:

- получает принятый Event;
- формирует понятное сообщение из имени Source, title, message и времени события;
- использует настроенные bot credential и единственный target;
- выполняет одну попытку доставки;
- переводит результат Telegram или сети в подтверждённый успех либо известную ошибку.

Он не сохраняет Delivery, не получает Source credentials, не формирует HTTP-ответы и не решает, принят ли Event.

### Startup / Composition

Startup / Composition разрешено знать конкретные компоненты приложения. Он:

1. читает и валидирует внешнюю конфигурацию;
2. подготавливает PostgreSQL connection и Storage;
3. подготавливает Telegram component;
4. создаёт Coordinator с его зависимостями;
5. создаёт HTTP boundary с Coordinator;
6. регистрирует transport routes и запускает server.

Конкретная структура Go packages и механизм абстракции зависимостей относятся к реализации, а не к дизайну B00.

## Startup flow и runtime flow

Это два разных процесса.

### Startup flow

```text
configuration
    -> PostgreSQL / Storage
    -> Telegram boundary
    -> Coordinator
    -> HTTP server
    -> ready
```

Необходимая конфигурация включает настройки PostgreSQL connection, Telegram bot credential, Telegram target, HTTP/server и Owner credential. Приложение проверяет обязательную конфигурацию до объявления готовности. Обязательный сетевой запрос в Telegram при каждом запуске не является частью утверждённого дизайна.

### Runtime flow

```text
request
   -> HTTP
   -> Coordinator
      -> Storage
      -> Telegram
```

Storage и Telegram не вызывают друг друга. HTTP не координирует application flow. Startup соединяет компоненты, но не обрабатывает Events.

## Владение данными

Hub владеет тремя domain records:

```text
Source 1 -> N Event
Event  1 -> 0..1 Delivery
```

PostgreSQL — durable system of record для этих сущностей. Telegram — внешняя зависимость доставки, а не источник истины об acceptance Event.

## Ключевые архитектурные решения

### Persistence до доставки

Telegram вызывается только после успешного сохранения Event. Поэтому Hub не отправляет уведомление о событии, которое он не смог принять.

### Acceptance не зависит от Delivery

После сохранения Event ошибка Telegram или сохранения Delivery не отменяет acceptance.

### Одна синхронная best-effort попытка

v1 выполняет одну Telegram-попытку внутри request flow. Source ждёт её завершения, хотя ответ сообщает только об acceptance Event. Это осознанное ограничение v1, а не гарантия eventual delivery.

### Один глобальный Telegram target

Данные подключения Telegram — конфигурация приложения. В v1 нет Destination, Route, routing policy или API для изменения Telegram-конфигурации во время работы.

### Hard delete

Удаление Source уничтожает Source, все его Events и их Deliveries. Скрытый архив или soft-deleted копия не сохраняются. Export — отдельная операция чтения, доступная до удаления.

## Обоснование разделения границ

Разделение следует из разных видов знаний, уже необходимых flow:

| Граница | Каким знанием владеет |
|---|---|
| HTTP | HTTP-запросы, ответы и transport decoding |
| Coordinator | Правила Hub и порядок операций |
| Storage | Persistent data и взаимодействие с PostgreSQL |
| Telegram | Форматирование и Telegram API |
| Startup / Composition | Создание конкретных компонентов и их связывание |

Для обоснования этих границ не требуется конкретный архитектурный framework. Они напрямую следуют из ответственностей утверждённого flow v1.

## Осознанные ограничения

В v1 нет очереди, retry worker, recovery process, idempotency key, deduplication, гарантии строгого порядка или нескольких targets. Последствия описаны в [Модели отказов](failure-model.md).
