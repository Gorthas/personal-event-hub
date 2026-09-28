# Безопасность

## Статус

Документ определяет утверждённую trust- и credential-модель v1. Конкретные криптографические алгоритмы, библиотеки, формат headers и представление данных в storage остаются решениями Security-реализации.

## Trust boundaries

В v1 есть две независимые роли аутентифицированных callers:

| Caller | Credential | Разрешённая зона |
|---|---|---|
| Registered Source | Собственный Source credential | Submit Events от имени этого Source |
| Owner | Отдельный Owner credential | Управление Sources, просмотр и export History |

Source credential не должен давать доступ к Management или History. Owner credential не используется как identity отправителя Events.

## Source credentials

У каждого Source собственный credential. Authentication устанавливает identity Source; Hub не доверяет идентификатору Source внутри Event payload.

Утверждённый жизненный цикл:

```text
Create Source
  -> generate credential
  -> store only the material needed to verify it
  -> disclose plaintext once

Rotate credential
  -> invalidate old credential
  -> generate and store new verification material
  -> disclose new plaintext once

Delete Source
  -> remove Source and credential verification data
```

Финальная реализация должна использовать безопасно сгенерированные credentials, безопасную verification/comparison и представление, не требующее recoverable plaintext Source secret. Алгоритм и библиотека дизайном пока не выбраны.

### Гарантии ротации

- В v1 одновременно действует ровно один credential Source.
- Rotation сохраняет identity Source и всю историю.
- После успешной rotation старый credential больше не проходит authentication.
- Новый plaintext credential нельзя прочитать повторно.

Несколько активных ключей, overlap windows, expiration dates, labels и key history не входят в v1.

## Owner credential

Management и History используют credential, отдельный от credentials всех Sources. Он поступает через внешнюю конфигурацию приложения, а не хранится как Source.

Owner credential защищает операции, которые могут раскрывать историю, менять доступ, экспортировать данные или безвозвратно удалить Source. Точная HTTP authentication form и представление verifier отложены.

v1 не определяет нескольких owners, user accounts, roles, sessions или fine-grained authorization.

## Telegram credential и target

Telegram bot credential и единственный target — deployment configuration, а не domain entities и не данные, управляемые через API v1.

Они загружаются при старте и доступны только Telegram boundary. Для изменения любого из значений необходимо обновить внешнюю конфигурацию и перезапустить приложение.

Hub проверяет наличие и структурную корректность обязательной конфигурации до объявления готовности. Дизайн не требует live network probe Telegram при каждом старте.

## Требования к конфигурации и секретам

- Реальные credentials должны находиться вне version control.
- Будущий example configuration file может документировать названия параметров, но не должен содержать рабочих секретов.
- Секреты не должны появляться в обычных application logs или owner-facing History.
- List Sources и History responses не должны раскрывать Source credentials.
- Delivery errors должны быть понятными и sanitized, а не raw external response или internal stack trace.
- Публичное развёртывание требует HTTPS для защиты credentials и Event data in transit.
- GitHub Actions хранит соответствующий Source credential через собственный secrets mechanism, а не в исходном workflow.

Конкретный configuration loader и deployment secret store ещё не выбраны.

## Authentication и validation — разные проверки

Authentication отвечает на вопрос:

> Какой зарегистрированный Source выполняет запрос и действителен ли его credential?

Validation отвечает на вопрос:

> Соответствуют ли данные Event от аутентифицированного Source контракту Hub?

Обе проверки должны пройти до формирования и сохранения Event. Действительный credential не делает невалидный payload допустимым, а валидный payload не обходит authentication Source.

## Удаление данных и confidentiality

Hard delete Source уничтожает его credential verification data, Events и Deliveries. Hub не хранит soft-deleted или скрытую архивную копию в своей domain model.

Владелец может экспортировать историю перед удалением. После export эта копия находится вне жизненного цикла Hub и должна защищаться средой, в которой она сохранена.

## Security-решения, отложенные до реализации

Следующие пункты обязательны для реализации, но не выдаются за существующий код:

- формат и entropy secure random credential;
- verifier или hashing approach;
- безопасное сравнение credentials;
- механизм verification Owner credential;
- HTTP-конвенция передачи credentials;
- secret injection и local-development configuration workflow;
- проверки log redaction и error sanitization;
- тесты invalid, old, missing и cross-boundary credentials.
