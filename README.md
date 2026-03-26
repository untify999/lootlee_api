# Lootlee API Documentation

Неофициальная документация по HTTP API маркетплейса Lootlee.

Собрано на основе анализа работы платформы. Для тех, кто хочет автоматизировать рутину: ботов для мониторинга, скрипты для масс-менеджмента товаров, интеграции с внешними системами или свои клиенты.

Что можно делать через API: управлять товарами пачками (создавать черновики, массово менять цены/остатки/статусы, поднимать по расписанию), работать с финансами (смотреть баланс, выводить деньги, создавать платежные сессии), обрабатывать заказы (подтверждать сделки программно, читать и отправлять сообщения в чаты, открывать споры), интегрировать тикеты поддержки и диалоги вне заказов.

Важно: Документация неофициальная. API может меняться при обновлениях платформы. Рекомендуется закладывать обработку ошибок и не завязываться жестко на структуру полей.

- Base URL: `https://lootlee.market`
- Auth: сессионная авторизация (`sessionid` cookie)
- Формат: в основном JSON (`application/json`)
- CSRF: для `POST/PUT/PATCH/DELETE` передавайте `X-CSRFToken`

---

## 1) Быстрый старт

### 1.1 Авторизация

1. Получите CSRF cookie:

```http
GET /accounts/login/ HTTP/1.1
Host: lootlee.market
```

2. Выполните логин:

```http
POST /accounts/login/ HTTP/1.1
Host: lootlee.market
Content-Type: application/x-www-form-urlencoded
Cookie: csrftoken=CSRF_VALUE

login=EMAIL_OR_USERNAME&password=PASSWORD&csrfmiddlewaretoken=CSRF_VALUE
```

3. Используйте `sessionid` в последующих запросах.

---

## 2) Общие правила

- Все методы работают только в контексте текущего пользователя.
- Для mutating запросов используйте `X-CSRFToken`.
- Для batch endpoint'ов:
  - размер батча до `100` элементов;
  - частичный успех (ошибка в одном элементе не ломает весь батч);
  - при `429` учитывайте заголовок `Retry-After`.

---

## 3) Баланс

### 3.1 Получить баланс

`GET /payments/balance/api/`

### 3.2 Создать payout

`POST /payments/balance/api/payout/`

Тело:

```json
{
  "amount": "1000.00",
  "payment_method": "card"
}
```

`payment_method`: `card | sbp | crypto`

---

## 4) Товары

### 4.1 Получить свои товары

`GET /products/api/me/products/?status=active&limit=18&cursor=...&q=...`

Поддерживаемые статусы (UI API): `active | moderation | rejected | draft`.

### 4.2 Сводка по статусам

`GET /products/api/me/products/summary/`

### 4.3 Поднять один товар

`POST /products/api/bump/{product_id}/`

### 4.4 Поднять все доступные товары

`POST /products/api/bump/all/`

### 4.5 Статус cooldown по bump

`GET /products/api/bump/{product_id}/status/`

### 4.6 Редактирование товара (HTML flow)

`POST /products/my-products/{product_id}/edit/` (multipart/form-data)

Важно:
- если товар в `DRAFT` и не передан `submit_action=publish`, остаётся `DRAFT`;
- в остальных случаях переводится в `PENDING_MODERATION`.

---

## 5) Batch API по товарам

Все endpoint’ы ниже: `POST`, JSON body.

### 5.1 Массовое обновление полей

`POST /products/api/batch/update/`

Пример:

```json
{
  "items": [
    { "id": 101, "price": "199.00", "old_price": "249.00", "stock_quantity": 15, "is_active": true, "auto_delivery": false },
    { "id": 102, "price": "149.00", "stock_quantity": 3 }
  ]
}
```

Поддерживаемые поля:
- `price` (> 0)
- `old_price` (> 0 и строго больше `price`)
- `stock_quantity` (1..99999)
- `is_active` (bool)
- `auto_delivery` (bool)

### 5.2 Массовая смена статуса

`POST /products/api/batch/status/`

Пример:

```json
{
  "product_ids": [101, 102, 103],
  "status": "PENDING_MODERATION"
}
```

Разрешённые статусы:
- `DRAFT`
- `PENDING_MODERATION`
- `HIDDEN`

Для `PENDING_MODERATION` проверяются обязательные поля товара.

### 5.3 Массовый bump выбранных товаров

`POST /products/api/batch/bump/`

Пример:

```json
{
  "product_ids": [101, 102, 103]
}
```

Особенности:
- учитывается cooldown `PRODUCT_BUMP_COOLDOWN_HOURS`;
- для элементов на cooldown приходит `error_code=cooldown` + `retry_after_seconds`;
- есть anti-spam lock с `429` и `Retry-After`.

### 5.4 Массовое удаление/скрытие

`POST /products/api/batch/delete-or-hide/`

Пример:

```json
{
  "product_ids": [101, 102, 103],
  "mode": "auto"
}
```

`mode`:
- `auto` — если есть заказы: скрыть, иначе удалить
- `hide` — скрыть
- `delete` — удалить (если есть заказы, вернётся ошибка)

### 5.5 Массовое создание черновиков

`POST /products/api/batch/create-drafts/`

Пример:

```json
{
  "items": [
    {
      "name": "Fortnite V-Bucks 1000",
      "description": "Описание",
      "instructions": "Инструкция",
      "price": "299.00",
      "old_price": "349.00",
      "stock_quantity": 20,
      "category_name": "Fortnite",
      "section_name": "В-баксы",
      "auto_delivery": false
    }
  ]
}
```

Результат: товары создаются в статусе `DRAFT`.

### 5.6 Частые `error_code` в batch

- `invalid_json`
- `batch_too_large`
- `not_found_or_forbidden`
- `validation_error`
- `not_ready_for_moderation`
- `cooldown`
- `rate_limited`
- `has_orders_cannot_delete`
- `mass_bump_disabled`

---

## 6) Заказы и C2C

### 6.1 Мои заказы как покупатель

`GET /orders/api/my/buy/`

### 6.2 Мои заказы как продавец

`GET /orders/api/my/sell/`

### 6.3 Детали заказа

`GET /orders/api/{order_id}/`

### 6.4 Чат заказа: получить сообщения

`GET /orders/api/{order_id}/chat/`

Опционально: `since=ISO8601`.

### 6.5 Чат заказа: отправить сообщение

`POST /orders/api/{order_id}/chat/send/`

```json
{ "content": "Текст сообщения" }
```

### 6.6 C2C confirm-seller

`POST /orders/api/c2c/{order_id}/confirm-seller/`

### 6.7 C2C confirm-buyer

`POST /orders/api/c2c/{order_id}/confirm-buyer/`

### 6.8 C2C buyer-verified

`POST /orders/api/c2c/{order_id}/buyer-verified/`

```json
{ "verified": true }
```

### 6.9 Batch confirm-seller

`POST /orders/api/batch/confirm-seller/`

Пример:

```json
{
  "order_ids": [9001, 9002, 9003]
}
```

---

## 7) Conversations API

### 7.1 Сообщения диалога

`GET /conversations/api/{conversation_id}/messages/`

Параметры:
- `limit` (int, optional, default `50`)
- `since` (ISO-8601, optional)

Пример:

```http
GET /conversations/api/42/messages/?limit=50 HTTP/1.1
Host: lootlee.market
Cookie: sessionid=...
Accept: application/json
```

Пример ответа:

```json
{
  "messages": [
    {
      "id": 101,
      "sender_id": 7,
      "sender_name": "seller_name",
      "message_type": "USER",
      "content": "Привет",
      "image_url": null,
      "image_display_url": null,
      "timestamp": "2026-03-20T10:22:11.000000+00:00",
      "is_read": true,
      "read_at": "2026-03-20T10:23:00.000000+00:00",
      "sender_avatar_url": "https://media.lootlee.market/avatars/abc.webp"
    }
  ],
  "conversation_id": 42,
  "other_user_last_seen": "2026-03-20T10:24:30.000000+00:00"
}
```

### 7.2 Отправить сообщение в диалог

`POST /conversations/api/{conversation_id}/messages/send/`

Вариант 1 (JSON):

```http
POST /conversations/api/42/messages/send/ HTTP/1.1
Host: lootlee.market
Cookie: sessionid=...
X-CSRFToken: ...
Content-Type: application/json

{
  "content": "Добрый день!"
}
```

Вариант 2 (multipart, с картинкой):

```http
POST /conversations/api/42/messages/send/ HTTP/1.1
Host: lootlee.market
Cookie: sessionid=...
X-CSRFToken: ...
Content-Type: multipart/form-data

content=Скриншот проблемы
image=<binary>
```

Пример ответа:

```json
{
  "id": 202,
  "sender_id": 15,
  "sender_name": "buyer_name",
  "message_type": "USER",
  "sender_avatar_url": null,
  "content": "Добрый день!",
  "image_url": null,
  "image_display_url": null,
  "timestamp": "2026-03-20T10:30:15.000000+00:00"
}
```

Ошибки:
- `400` — пустое сообщение / неверный формат / превышен размер
- `403` — нет доступа
- `429` — лимит отправки сообщений (rate limit)

### 7.3 Отметить сообщения прочитанными

`POST /conversations/api/{conversation_id}/messages/read/`

Пример:

```http
POST /conversations/api/42/messages/read/ HTTP/1.1
Host: lootlee.market
Cookie: sessionid=...
X-CSRFToken: ...
Content-Type: application/json

{
  "message_ids": [101, 102, 103]
}
```

Ответ:

```json
{ "updated": 3 }
```

### 7.4 Данные заказа для диалога

`GET /conversations/api/{conversation_id}/order/`

Пример ответа:

```json
{
  "order": {
    "id": 555,
    "status": "PAID_HOLD",
    "status_display": "Ожидает подтверждения"
  },
  "can_confirm_buyer": false,
  "can_confirm_seller": true,
  "can_open_dispute": false,
  "hold_timer": {
    "hold_end": "2026-03-21T12:00:00+00:00",
    "is_expired": false
  }
}
```

### 7.5 Общее число непрочитанных

`GET /conversations/api/unread/total/`

Ответ:

```json
{ "unread_total": 4 }
```

### 7.6 Отметить inbox прочитанным

`POST /conversations/api/inbox/messages/read/`

Ответ:

```json
{
  "updated": 7,
  "unread_total": 0
}
```

---

## 8) Споры (Disputes)

### 8.1 Список споров пользователя

`GET /orders/api/disputes/`

Пример ответа:

```json
{
  "disputes": [
    {
      "id": 11,
      "order_id": 555,
      "status": "OPEN",
      "created_at": "2026-03-20T10:40:00+00:00"
    }
  ]
}
```

### 8.2 Создать спор

`POST /orders/api/disputes/{order_id}/create/`

```json
{
  "reason": "Продавец не выдал доступ"
}
```

Ответ:

```json
{
  "id": 12,
  "status": "OPEN"
}
```

### 8.3 Получить детали спора

`GET /orders/api/disputes/{dispute_id}/`

Ответ:

```json
{
  "id": 12,
  "order_id": 555,
  "status": "OPEN",
  "reason": "Продавец не выдал доступ",
  "created_at": "2026-03-20T10:45:00+00:00"
}
```

---

## 9) Платежи (orders/payment)

### 9.1 Создать checkout session

`POST /orders/api/payment/create-session/`

`Content-Type: application/x-www-form-urlencoded`

Пример:

```http
product_id=777&quantity=1&user_data=Мой логин в игре: demo_user
```

Ответ:

```json
{
  "success": true,
  "session_id": "d13f2e93-7f12-4f87-b30b-0a1f9d7f2cab",
  "amount": 299.0
}
```

### 9.2 Получить доступные методы оплаты

`GET /orders/api/payment/methods/?session_id={session_id}`

Ответ:

```json
{
  "success": true,
  "methods": [
    { "id": "crypto", "name": "USDT TRC20", "icon": "₿", "available": true },
    { "id": "sbp", "name": "СБП (QR)", "icon": "📱", "available": true },
    { "id": "card", "name": "Банковская карта", "icon": "💳", "available": true }
  ],
  "amount": 299.0
}
```

### 9.3 Оплатить с баланса

`POST /orders/api/payment/pay-balance/`

```http
session_id=d13f2e93-7f12-4f87-b30b-0a1f9d7f2cab
```

Ответ:

```json
{
  "success": true,
  "order_id": 555,
  "message": "Заказ успешно создан"
}
```

### 9.4 Создать crypto invoice

`POST /orders/api/payment/create-crypto/`

JSON или form-data:

```json
{ "session_id": "d13f2e93-7f12-4f87-b30b-0a1f9d7f2cab" }
```

Ответ:

```json
{
  "success": true,
  "invoice_id": 91,
  "address": "THVG9prZ3HN2o3po2qyPs8wSfj4cdfeYjt",
  "amount": "3.891245",
  "expires_at": "2026-03-20T11:15:00+00:00"
}
```

### 9.5 Создать SBP платеж

`POST /orders/api/payment/create-sbp/`

```http
session_id=d13f2e93-7f12-4f87-b30b-0a1f9d7f2cab
```

Ответ:

```json
{
  "success": true,
  "payment_url": "https://provider.example/pay/..."
}
```

### 9.6 Создать card платеж

`POST /orders/api/payment/create-card/`

```http
session_id=d13f2e93-7f12-4f87-b30b-0a1f9d7f2cab
```

Ответ:

```json
{
  "success": true,
  "payment_url": "https://provider.example/pay/..."
}
```

### 9.7 Клиентский debug endpoint

`POST /orders/api/payment/client-debug/`

```json
{
  "event": "pay_modal_opened",
  "sessionId": "d13f2e93-7f12-4f87-b30b-0a1f9d7f2cab",
  "method": "card",
  "message": "Button clicked",
  "extra": { "screen": "checkout" }
}
```

Ответ:

```json
{ "success": true }
```

---

## 10) Тикеты (HelpDesk)

Ниже описан API-формат, используемый в интеграциях HelpDesk.

### 10.1 Список тикетов

`GET /ticket/api/my/`

Пример ответа:

```json
{
  "items": [
    {
      "id": 100,
      "subject": "Проблема с заказом",
      "status": "OPEN",
      "created_at": "2026-03-20T09:00:00Z",
      "last_update": "2026-03-20T10:00:00Z"
    }
  ]
}
```

### 10.2 Создать тикет

`POST /ticket/api/create/`

```json
{
  "subject": "Краткое описание",
  "message": "Подробное описание",
  "order_id": 555
}
```

### 10.3 Комментарии тикета

`GET /ticket/api/{ticket_id}/comments/`

Пример ответа:

```json
[
  {
    "id": 1,
    "author_role": "user",
    "content": "Текст сообщения",
    "created_at": "2026-03-20T10:05:00Z"
  }
]
```

### 10.4 Добавить комментарий

`POST /ticket/api/{ticket_id}/comment/`

```json
{
  "content": "Мой ответ по тикету"
}
```

---

## 11) Реферальный API

- `POST /referrals/api/transfer-balance/`

---

## 12) Стандарт ответа batch endpoint

У всех batch endpoint рекомендуется ориентироваться на формат:

```json
{
  "success": true,
  "updated_count": 2,
  "failed_count": 1,
  "results": [
    { "id": 101, "success": true },
    { "id": 102, "success": false, "error_code": "validation_error", "retryable": false }
  ],
  "hints": {
    "max_batch_size": 100,
    "retry_strategy": "retry only retryable=true items with exponential backoff",
    "retryable_error_codes": ["internal_error", "rate_limited", "cooldown"]
  }
}
```

---

## 13) Минимальный Python пример

```python
import requests

BASE_URL = "https://lootlee.market"

s = requests.Session()

# 1) get csrf
r = s.get(f"{BASE_URL}/accounts/login/")
r.raise_for_status()
csrf = s.cookies.get("csrftoken")

# 2) login
payload = {
    "login": "YOUR_LOGIN",
    "password": "YOUR_PASSWORD",
    "csrfmiddlewaretoken": csrf,
}
headers = {"X-CSRFToken": csrf, "Referer": f"{BASE_URL}/accounts/login/"}
r = s.post(f"{BASE_URL}/accounts/login/", data=payload, headers=headers)
r.raise_for_status()

# 3) call API
r = s.get(f"{BASE_URL}/products/api/me/products/", params={"status": "active", "limit": 20})
r.raise_for_status()
print(r.json())
```

---

## 14) Рекомендации по интеграции

- Делайте ретраи только для `retryable=true` (или `5xx/429`).
- Для `429` всегда соблюдайте `Retry-After`.
- Не хардкодьте пароль/куки в репозиторий.
- Логируйте `error_code` поэлементно в batch.
- Проверяйте изменения API при деплоях (поля могут расширяться).

