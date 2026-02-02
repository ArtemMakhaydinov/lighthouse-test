# Подписки и webhook платежи

Требование: обработка webhook платежей для подписок с идемпотентностью, транзакциями, восстановлением и дебагом.

## Часть 1. Схема данных и уникальности

Ниже минимальная схема с фокусом на идемпотентность и восстановление.

### `users`

Поля:

- `id` (PK)
- `email` (nullable)
- `external_customer_id` (nullable) — идентификатор клиента в платежной системе
- `created_at`

Уникальности:

- `email` UNIQUE (nullable) — гарантирует единственного пользователя по email, если он есть
- `external_customer_id` UNIQUE (nullable) — гарантирует единственного пользователя по ID платежной системы

Индексы:

- `email` — быстрый поиск пользователя при приходе webhook с email
- `external_customer_id` — быстрый поиск по ID платежной системы

### `subscriptions`

Поля:

- `id` (PK)
- `user_id` (FK -> users.id)
- `status` (`active`, `canceled`, `past_due`, `expired`)
- `plan_id`
- `current_period_start`
- `current_period_end`
- `created_at`
- `updated_at`

Уникальности:

- `user_id` UNIQUE — один активный объект подписки на пользователя (упрощение для задания)

Индексы:

- `user_id` — быстрый доступ к подписке пользователя
- `status` — выборка активных/просроченных для биллинга

### `payments`

Поля:

- `id` (PK)
- `user_id` (FK -> users.id, nullable)
- `subscription_id` (FK -> subscriptions.id, nullable)
- `external_payment_id` — ID платежа из платежной системы
- `amount`
- `currency`
- `status` (`pending`, `succeeded`, `failed`, `refunded`)
- `paid_at` (nullable)
- `raw_payload` (jsonb/text)
- `created_at`
- `updated_at`

Уникальности:

- `external_payment_id` UNIQUE — ключ идемпотентности платежа

Индексы:

- `external_payment_id` — дедупликация платежей
- `user_id` — история платежей пользователя
- `subscription_id` — связь с подпиской
- `status` — отчеты по успешным/неуспешным

### `webhook_events`

Поля:

- `id` (PK)
- `provider` — имя провайдера
- `event_id` — ID события из webhook, если есть
- `event_type`
- `external_payment_id` (nullable)
- `payload` (jsonb/text)
- `status` (`received`, `processed`, `failed`, `ignored`)
- `error_code` (nullable)
- `error_message` (nullable)
- `created_at`
- `processed_at` (nullable)

Уникальности:

- `(provider, event_id)` UNIQUE — идемпотентность событий, если `event_id` есть
- `external_payment_id` UNIQUE WHERE status = 'processed' — гарантирует, что один платеж обработан один раз

Индексы:

- `(provider, event_id)` — дедупликация
- `external_payment_id` — корреляция с платежом
- `status` — ретраи/аналитика по сбоям
- `created_at` — очередность и выборки по времени

## Часть 2. Пошаговая логика обработчика webhook (псевдокод)

```
function handleWebhook(request):
  // 1) Валидация входа
  if request.body is empty:
    return 400 (bad_request: empty_body)

  if required fields missing (event_id or external_payment_id or amount or currency):
    // допускаем частично пустые, но минимум нужен для идемпотентности
    return 422 (unprocessable_entity: missing_min_fields)

  // 2) Проверка подписи/секрета
  if !verifySignature(request.headers, request.body, webhook_secret):
    return 401 (unauthorized: invalid_signature)

  // 3) Дедупликация события (ранняя, но с учетом статуса)
  if event_id exists:
    existing = find webhook_events where (provider, event_id)
    if existing exists and existing.status in ['processed', 'ignored']:
      return 200 (already_processed)
    // если status 'received' или 'failed' — продолжаем (ретрай/дозавершение)

  // 4) Запись webhook_events в статусе received (идемпотентно)
  insert into webhook_events (provider, event_id, event_type, external_payment_id, payload, status)
  on conflict (provider, event_id) do update set
    status = 'received',
    payload = excluded.payload

  // 5) Транзакция
  begin transaction

    // 5.1) Лочим/находим пользователя и подписку
    user = find user by external_customer_id or email (for update if found)
    subscription = find subscription by user_id (for update if found)

    // 5.2) Создание/обновление payment (идемпотентно)
    payment = insert into payments (...)
      values (external_payment_id, amount, currency, status='succeeded', paid_at=now, raw_payload)
      on conflict (external_payment_id) do update set
        status=excluded.status,
        paid_at=coalesce(payments.paid_at, excluded.paid_at),
        raw_payload=excluded.raw_payload
      returning *

    // 5.3) Проверка соответствия суммы/плана
    if amount != expected_amount_for_plan:
      mark webhook_events status='failed' with error_code='amount_mismatch'
      rollback
      return 409 (conflict: amount_mismatch)

    // 5.4) Активация/продление подписки
    if subscription not exists:
      if user exists:
        create subscription with current_period_start=paid_at, end=paid_at+plan_duration, status='active'
        update payments set subscription_id=subscription.id where id=payment.id
      else:
        // user нет, создаем payment, событие остается failed для ретрая
        mark webhook_events status='failed' with error_code='user_missing'
        commit
        return 202 (accepted: will_retry)
    else:
      if payment already applied to subscription (guard by external_payment_id in payments):
        // no-op, идемпотентно
      else:
        extend subscription.current_period_end by plan_duration
        set status='active'
        update payments set subscription_id=subscription.id where id=payment.id

    // 5.5) Обновляем webhook_events
    update webhook_events set status='processed', processed_at=now where (provider, event_id)

  commit transaction

  return 200 (ok)

// Ошибки БД/неожиданное
on error:
  rollback
  update webhook_events set status='failed', error_code='internal_error'
  return 500 (internal_error)
```

Возвраты:

- `200` — успешно обработано или дубль
- `202` — обработка отложена (например, пользователя еще нет)
- `401` — подпись неверна
- `4xx` — валидация/согласованность данных
- `5xx` — внутренние ошибки, нужны ретраи провайдера

Гарантия восстановления:

- Идемпотентность через `external_payment_id` и `event_id`
- Транзакция на запись `payment` + изменение `subscription`
- Повторный webhook безопасен, дубли не создаются

## Часть 3. Edge cases

- Webhook пришел дважды: если уже `processed`/`ignored` — `200`, иначе продолжаем обработку (ретрай/дозавершение).
- Webhook пришел раньше создания user: сохраняем `payment`, отмечаем `webhook_event` как `failed` с `user_missing`, возвращаем `202` и ждем повтора.
- Webhook пришел без email, но есть `externalPaymentId`: ищем по `external_customer_id`, иначе создаем `payment` и оставляем событие в `failed` для ретрая.
- Webhook пришел с другой суммой, чем план: `409`, событие `failed` с `amount_mismatch`, подписку не трогаем.
- Webhook пришел через неделю: если `payment` новый, продлеваем подписку по `paid_at`, иначе no-op.
- Сервер упал после записи payment, но до subscription: повторный webhook увидит `payment` по `external_payment_id` и корректно завершит продление.

## Часть 4. Debuggability и наблюдаемость

Минимальные логи (structured):

- `event_id`, `external_payment_id`, `provider`, `event_type`
- `user_id`, `subscription_id` (если есть)
- `amount`, `currency`
- `payment_status`, `webhook_event_status`
- `error_code`, `error_message`
- `request_id`, `trace_id`

Минимальные метрики:

- счетчик `webhook_events_total` с label `status`
- счетчик `payments_created_total` и `payments_dedup_total`
- latency обработки webhook (p95/p99)
- счетчик `amount_mismatch_total`
- счетчик `user_missing_total`
- счетчик `signature_invalid_total`

Алерты:

- резкий рост `failed` в `webhook_events`
- рост `signature_invalid_total`
- доля `payments_dedup_total` > baseline (возможные ретраи/проблемы)
- p99 latency выше порога

Храним `payload` в `webhook_events.payload` для полного дебага инцидентов.
