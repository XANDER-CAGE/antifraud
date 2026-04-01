# Antifraud API

Документация по интеграции с сервисом антифрода для разработчиков платежного шлюза.

Сервис принимает данные о транзакции, прогоняет через набор правил и возвращает решение.


## Содержание

- [Общие сведения](#общие-сведения)
- [POST /check](#post-check)
- [POST /check_order](#post-check_order)
- [POST /update](#post-update)
- [POST /rules](#post-rules)
- [GET /-/stats](#get---stats)
- [Формат решений](#формат-решений)
- [Коды ошибок](#коды-ошибок)
- [Заголовки](#заголовки)


## Общие сведения

- Протокол: HTTP/1.1
- Content-Type: `application/json; charset=utf-8`
- Кодировка: UTF-8
- Суммы передаются в тийинах (1 сум = 100 тийин)
- Идемпотентность: повторный запрос с тем же `OrderId` вернет сохраненный результат без повторной проверки


## POST /check

Основной метод. Проверка платежной транзакции.

### Запрос

```json
{
  "AFMerchantId": 1,
  "key": "profile_key",
  "password": "profile_password",
  "OrderId": "ORD-20260331-001",
  "PaymentId": "PAY-001",
  "Amount": 1500000,
  "Type": "payment",
  "IP": "195.158.31.1",

  "AFCustomer": {
    "MerchantCustomerId": "customer-123",
    "Email": "user@example.com",
    "Phone": "+998901234567"
  },

  "CardData": {
    "Token": "tok_unique_card_token",
    "MaskedPan": "860006******1234",
    "ExpDate": "12/29",
    "CardHolder": "IVAN IVANOV"
  }
}
```

### Параметры запроса

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| AFMerchantId | integer | да | ID мерчанта |
| key | string | да | Ключ профиля |
| password | string | да | Пароль профиля |
| OrderId | string | да | Уникальный ID заказа (идемпотентность) |
| PaymentId | string | нет | ID платежа |
| Amount | integer | да | Сумма в тийинах |
| Type | string | да | Тип операции: `payment`, `p2p`, `refund` |
| IP | string | да | IPv4 адрес клиента |
| SettlementAccount | string | нет | Расчетный счет |
| HumoTranType | string | нет | Тип транзакции HUMO |
| LinkGuid | string | нет | GUID для связанных транзакций (рекурренты, подписки) |
| LinkType | string | нет | Тип связи (subscription, recurring) |

### Объект AFCustomer

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| MerchantCustomerId | string | да | ID клиента в системе мерчанта |
| Email | string | нет | Email клиента |
| Phone | string | да | Телефон клиента |

### Объект CardData

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| Token | string | да | Уникальный токен карты |
| MaskedPan | string | да | Маскированный PAN (формат: `860006******1234`) |
| ExpDate | string | да | Срок действия: `MM/YY` или `MMYY` |
| CardHolder | string | да | Имя держателя карты (латиница) |

### Объект RecipientCardData (только для P2P)

Структура аналогична CardData. Передается при `Type: "p2p"`.

```json
{
  "RecipientCardData": {
    "Token": "tok_recipient_token",
    "MaskedPan": "986001******5678",
    "ExpDate": "0000",
    "CardHolder": "RECIPIENT NAME"
  }
}
```

### Объект EcommerceData (опционально)

Расширенные данные для e-commerce проверок.

```json
{
  "EcommerceData": {
    "RefundAmount": 0,
    "OriginalOrderId": null,
    "ReturnReason": null,
    "ItemCategory": "electronics",
    "ItemCount": 2,
    "IsDigitalGoods": false,
    "DeliveryAddress": "ul. Amir Temur 1",
    "DeliveryCountry": "UZ",
    "DeliveryCity": "Tashkent",
    "DeliveryPostalCode": "100000",
    "BillingAddress": "ul. Amir Temur 1",
    "AccountAgeDays": 365,
    "PreviousOrderCount": 15,
    "PreviousRefundCount": 0
  }
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| RefundAmount | integer | Сумма возврата (тийины) |
| OriginalOrderId | string | ID оригинального заказа (для возвратов) |
| ReturnReason | string | Причина возврата |
| ItemCategory | string | Категория товара |
| ItemCount | integer | Количество позиций |
| IsDigitalGoods | boolean | Цифровой товар |
| DeliveryAddress | string | Адрес доставки |
| DeliveryCountry | string | Страна доставки (ISO alpha-2) |
| DeliveryCity | string | Город доставки |
| DeliveryPostalCode | string | Почтовый индекс |
| BillingAddress | string | Адрес плательщика |
| AccountAgeDays | integer | Возраст аккаунта клиента (дни) |
| PreviousOrderCount | integer | Количество предыдущих заказов |
| PreviousRefundCount | integer | Количество предыдущих возвратов |

### Ответ

```json
{
  "Decision": "Allow",
  "Authorized": false,
  "TransactionId": 42,
  "Rules": [
    {
      "Id": 1,
      "Name": "AmountTooHigh",
      "Decision": null,
      "Description": "Сумма платежа не должна быть больше 10000000."
    },
    {
      "Id": 2,
      "Name": "EmailInBlacklist",
      "Decision": null,
      "Description": "E-mail user@example.com в черном списке"
    }
  ],
  "time": "12.50 ms"
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| Decision | string | Итоговое решение: `Allow`, `Decline`, `Review`, `None` |
| Authorized | boolean | Подтверждена ли оплата (всегда `false` при /check) |
| TransactionId | integer | Внутренний ID транзакции |
| Rules | array | Список проверенных правил |
| Rules[].Id | integer | ID правила |
| Rules[].Name | string | Название правила |
| Rules[].Decision | string/null | Решение правила (`Decline`, `Review` или `null` если не сработало) |
| Rules[].Description | string | Описание результата проверки |
| time | string | Время обработки |


## POST /check_order

Проверка на уровне заказа. Аналогичен `/check`, но `CardData` опционален. Предназначен для предварительной проверки до этапа оплаты, когда карта еще не известна.

Формат запроса и ответа идентичен `/check`. Отличие: если `CardData` не передан, правила зависящие от карты пропускаются.


## POST /update

Обновление статуса транзакции после проведения оплаты через банк. Вызывается после получения ответа от эквайера.

### Запрос

```json
{
  "OrderId": "ORD-20260331-001",
  "Authorized": true,
  "ActionCode": "0"
}
```

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| OrderId | string | да | ID заказа (тот же что в /check) |
| Authorized | boolean | да | `true` -- оплата прошла, `false` -- отказ |
| ActionCode | string | нет | Код ответа от банка-эквайера |

### Логика обработки

- `Authorized: true` устанавливается только если `Decision` транзакции = `Accept`
- При `Authorized: false` + `ActionCode` система проверяет настроенные Action Code Rules
- Action Code Rules могут автоматически добавить карту/клиента в черный список после N отказов с определенным кодом

### Ответ

```json
{
  "Decision": "Allow",
  "Authorized": true,
  "TransactionId": 42,
  "time": "2.10 ms"
}
```


## POST /rules

Получение списка всех правил системы. Для отладки и мониторинга.

### Запрос

```json
{
  "AFMerchantId": 1,
  "key": "profile_key"
}
```

### Ответ

Массив правил с привязанными выражениями.


## GET /-/stats

Статистика работы сервиса.

### Ответ

```json
{
  "served": 15234,
  "active": 3,
  "processing": 1,
  "queued": 0,
  "errors": 12,
  "fetching": 0,
  "resizing": 0
}
```

| Поле | Описание |
|------|----------|
| served | Обработано запросов с момента старта |
| active | Активных соединений |
| processing | Обрабатывается сейчас |
| queued | В очереди |
| errors | Ошибок |
| fetching | Запросов к внешним сервисам |


## Формат решений

| Решение | Код | Что делать |
|---------|-----|------------|
| `Allow` | - | Разрешить платеж |
| `Decline` | - | Отклонить платеж |
| `Review` | - | Отправить на ручную проверку |
| `None` | - | Правила не сработали, применяется ForceDecision профиля |

Итоговое решение определяется так:

1. Правила применяются по порядку `sort_order`
2. Если правило сработало -- его `Decision` становится решением
3. Правило с наибольшим весом (`weight`) имеет приоритет
4. Если ни одно правило не сработало -- применяется `ForceDecision` профиля


## Коды ошибок

| HTTP код | Описание |
|----------|----------|
| 200 | Успешная обработка (даже при Decline) |
| 400 | Невалидный JSON в теле запроса |
| 500 | Внутренняя ошибка сервиса |

Ошибки бизнес-логики возвращаются с HTTP 200 и полем `error`:

```json
{
  "error": "Merchant not found id:99"
}
```


## Заголовки

| Заголовок | Направление | Описание |
|-----------|-------------|----------|
| Content-Type | запрос | `application/json` (обязательно) |
| X-Request-ID | запрос | UUID для трассировки (опционально, генерируется если не передан) |
| Content-Type | ответ | `application/json` |


## Пример полного цикла

### 1. Проверка транзакции

```bash
curl -X POST http://antifraud:80/check \
  -H 'Content-Type: application/json' \
  -H 'X-Request-ID: req-abc-123' \
  -d '{
    "AFMerchantId": 1,
    "key": "1",
    "password": "pass",
    "OrderId": "ORD-001",
    "PaymentId": "PAY-001",
    "Amount": 1500000,
    "Type": "payment",
    "IP": "195.158.31.1",
    "AFCustomer": {
      "MerchantCustomerId": "cust-123",
      "Email": "user@example.com",
      "Phone": "+998901234567"
    },
    "CardData": {
      "Token": "tok_abc",
      "MaskedPan": "860006******1234",
      "ExpDate": "12/29",
      "CardHolder": "IVAN IVANOV"
    }
  }'
```

Ответ: `{"Decision": "Allow", "TransactionId": 42, ...}`

### 2. Обработка ответа

- Если `Decision = Allow` -- проводить оплату через банк
- Если `Decision = Decline` -- отклонить платеж, не отправлять в банк
- Если `Decision = Review` -- поставить в очередь на ручную проверку

### 3. Обновление после ответа банка

```bash
curl -X POST http://antifraud:80/update \
  -H 'Content-Type: application/json' \
  -d '{
    "OrderId": "ORD-001",
    "Authorized": true,
    "ActionCode": "0"
  }'
```

### 4. Обновление при отказе банка

```bash
curl -X POST http://antifraud:80/update \
  -H 'Content-Type: application/json' \
  -d '{
    "OrderId": "ORD-001",
    "Authorized": false,
    "ActionCode": "51"
  }'
```

При накоплении определенного количества отказов с одним кодом -- карта/клиент автоматически попадает в черный список.


## P2P перевод

Для P2P передается `Type: "p2p"` и дополнительный блок `RecipientCardData`.

```bash
curl -X POST http://antifraud:80/check \
  -H 'Content-Type: application/json' \
  -d '{
    "AFMerchantId": 1,
    "key": "1",
    "password": "pass",
    "OrderId": "P2P-001",
    "Amount": 5000000,
    "Type": "p2p",
    "IP": "195.158.31.1",
    "AFCustomer": {
      "MerchantCustomerId": "sender-456",
      "Phone": "+998901234567"
    },
    "CardData": {
      "Token": "tok_sender",
      "MaskedPan": "860006******1111",
      "ExpDate": "12/29",
      "CardHolder": "SENDER NAME"
    },
    "RecipientCardData": {
      "Token": "tok_recipient",
      "MaskedPan": "986001******2222",
      "ExpDate": "0000",
      "CardHolder": "RECIPIENT NAME"
    }
  }'
```

При P2P проверяются обе карты: отправителя и получателя. BIN, банк, страна, черные списки, санкции -- для обеих сторон.


## Идемпотентность

Повторный вызов `/check` с тем же `OrderId` не создает новую транзакцию. Возвращается сохраненный результат первой проверки.

Это позволяет безопасно повторять запросы при сетевых ошибках и таймаутах.
