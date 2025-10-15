# 🎯 Antifraud System - Полное описание возможностей

## 📋 Оглавление

1. [Основные функции](#основные-функции)
2. [Система правил](#система-правил)
3. [Типы проверок](#типы-проверок)
4. [Черные и белые списки](#черные-и-белые-списки)
5. [Action Codes](#action-codes)
6. [REST API](#rest-api)
7. [Интеграции](#интеграции)
8. [Примеры использования](#примеры-использования)

---

## 🎯 Основные функции

### 1. Проверка транзакций в реальном времени

**Endpoint:** `POST /check`

**Что проверяется:**
- ✅ Сумма транзакции
- ✅ Частота операций (velocity checks)
- ✅ IP адрес (GeoIP, прокси, черные списки)
- ✅ Карта (BIN, банк-эмитент, черные списки)
- ✅ Карта получателя (для P2P переводов)
- ✅ Клиент (история, черные списки)
- ✅ Телефон (черные списки)
- ✅ Страна клиента
- ✅ Страна банка
- ✅ Держатель карты (санкционные списки)
- ✅ Тип транзакции

**Решения:**
- **Accept** - транзакция одобрена
- **Reject** - транзакция отклонена
- **Review** - требуется ручная проверка
- **None** - нет решения (используется ForceDecision)

**Время обработки:** 0.06 - 200 ms (зависит от количества правил и внешних проверок)

### 2. Обновление статуса транзакции

**Endpoint:** `POST /update`

**Возможности:**
- Установка флага `Authorized` (подтверждение оплаты)
- Установка `ActionCode` (код ответа от банка)
- Автоматическая реакция на коды банка
- Добавление в черные списки по кодам

### 3. Управление через REST API

**Полностью управляемая система:**
- Мерчанты (создание, получение, список)
- Профили (создание, обновление, удаление)
- Правила (привязка к профилям, настройка)
- Выражения в правилах (переопределение параметров)
- Черные/белые списки (добавление, удаление значений)
- Action Codes (настройка реакций на коды банка)

---

## 🔧 Система правил

### Структура правила

**Правило (Rule)** состоит из:
- **Name** - название
- **Decision** - решение при срабатывании (Accept/Reject/Review)
- **Weight** - вес (приоритет)
- **SortOrder** - порядок применения
- **Expressions** - набор условий (все должны выполниться)

**Выражение (Expression)** содержит:
- **Field** - поле для проверки (amount, ip, card_id, etc)
- **TypeOperator** - оператор (>, <, IN, NOT IN, ==, !=)
- **OperandType** - тип операнда (значение, список, поле, счётчик, интервал)
- **Operand** - значение или параметр
- **Threshold** - порог срабатывания
- **PeriodMinutes** - период времени для проверки
- **StartFrom** - начало периода (hour, day, week, month, None)
- **IsGlobal** - проверять по всем профилям или только текущему

### Примеры правил

#### Правило 1: Контроль суммы
```json
{
  "name": "Amount Greater Than 100000",
  "decision": "Review",
  "expressions": [
    {
      "field": "amount",
      "operator": ">",
      "operandType": 1,  // Value
      "operand": "10000000",  // 100,000 руб в копейках
      "threshold": 0
    }
  ]
}
```

**Логика:** Если сумма > 100,000 руб → отправить на ревью

#### Правило 2: IP в черном списке
```json
{
  "name": "IP in Blacklist",
  "decision": "Reject",
  "expressions": [
    {
      "field": "ip",
      "operator": "IN",
      "operandType": 2,  // Set (список)
      "operand": "IpBL",  // Черный список IP
      "threshold": 0
    }
  ]
}
```

**Логика:** Если IP в черном списке → отклонить

#### Правило 3: Много карт с одного IP
```json
{
  "name": "Multiple Cards Per IP Per Hour",
  "decision": "Review",
  "expressions": [
    {
      "field": "payment",  // Успешные платежи
      "operator": ">",
      "operandType": 4,  // Counter (счётчик)
      "operand": "card_id",  // Считать разные карты
      "threshold": 3,  // Порог: 3 карты
      "periodMinutes": 60,  // За час
      "isGlobal": false
    }
  ]
}
```

**Логика:** Если с одного IP использовали >3 разных карт за час → ревью

#### Правило 4: Слишком много попыток
```json
{
  "name": "Too Many Failed Attempts",
  "decision": "Reject",
  "expressions": [
    {
      "field": "failed_attempt",
      "operator": ">",
      "operandType": 4,
      "operand": "card_id",
      "threshold": 5,
      "periodMinutes": 1440,  // За сутки
      "startFrom": "day"  // С начала дня
    }
  ]
}
```

**Логика:** Если >5 неудачных попыток с одной карты за день → отклонить

---

## 🔍 Типы проверок

### 1. Простое сравнение (OperandType = 1)

**Поля:**
- amount (сумма)
- customer_id
- card_id
- recipient_card_id

**Операторы:** `==`, `!=`, `>`, `>=`, `<`, `<=`

**Пример:**
```sql
amount > 10000000  -- Сумма больше 100К
```

### 2. Проверка по спискам (OperandType = 2)

**Списки:**
- **CustomerBL/WL** - черный/белый список клиентов
- **IpBL/WL** - черный/белый список IP
- **CardBL/WL** - черный/белый список карт
- **PhoneBL/WL** - черный/белый список телефонов
- **BankCountryBL/WL** - черный/белый список стран банков
- **CustomerCountryBL/WL** - черный/белый список стран клиентов
- **ProxyCountryIsNotEmpty** - IP является прокси

**Операторы:** `IN`, `NOT IN`

**Пример:**
```sql
ip IN IpBL  -- IP в черном списке
bank_country NOT IN BankCountryWL  -- Страна банка не в белом списке
```

**Особенности:**
- Списки могут быть **глобальными** (merchant_id=0)
- Или **специфичными** для мерчанта
- Поддержка **временных банов** (expire timestamp)

### 3. Сравнение полей (OperandType = 3)

**Пример:**
```sql
customer_country <> bank_country  -- Страна клиента != страна банка
```

### 4. Счётчики за период (OperandType = 4)

**Самые мощные проверки!**

#### 4.1. Количество попыток (attempt)
```python
{
  "field": "attempt",
  "operand": "card_id",  // По какому полю группировать
  "threshold": 10,  // Максимум попыток
  "periodMinutes": 60,  // За час
  "startFrom": "None"  // Или "hour", "day", "week", "month"
}
```

**Логика:** Не более 10 попыток оплаты одной картой за час

#### 4.2. Неудачные попытки (failed_attempt)
```python
{
  "field": "failed_attempt",
  "operand": "phone",
  "threshold": 3,
  "periodMinutes": 1440,
  "startFrom": "day"
}
```

**Логика:** Не более 3 неудачных попыток с одного телефона с начала дня

#### 4.3. Последовательные неудачи (failed_sequential_attempt)
```python
{
  "field": "failed_sequential_attempt",
  "operand": "card_id",
  "threshold": 3,
  "periodMinutes": 60
}
```

**Логика:** Не более 3 неудачных попыток **подряд** с одной карты за час

**Особенность:** Учитывает только последовательные неудачи без успешных между ними

#### 4.4. Сумма платежей (amount)
```python
{
  "field": "amount",
  "operand": "customer_id",
  "threshold": 50000000,  // 500К руб
  "periodMinutes": 1440,
  "startFrom": "day"
}
```

**Логика:** Не более 500К руб успешных платежей от одного клиента с начала дня

#### 4.5. Количество успешных платежей (payment)
```python
{
  "field": "payment",
  "operand": "card_id",
  "threshold": 5,
  "periodMinutes": 60
}
```

**Логика:** Не более 5 успешных платежей одной картой за час

#### 4.6. Счётчики с дополнительными фильтрами
```python
{
  "field": "attempt",
  "operand": "card_id|CardLimitList",  // Только для карт из списка
  "threshold": 2,
  "periodMinutes": 1440
}
```

**Логика:** Для карт из CardLimitList - не более 2 попыток в день

### 5. Временные интервалы (OperandType = 5)

**Проверка времени транзакции:**
```python
{
  "field": "created",
  "operand": "0|120|dow|1,2,3,4,5",  // 0:00-2:00 в рабочие дни
  "threshold": 0
}
```

**Параметры:**
- `0|120` - интервал в минутах от начала периода (0:00 - 2:00)
- `dow` - day of week
- `1,2,3,4,5` - понедельник-пятница

**Логика:** Блокировать транзакции с 0:00 до 2:00 в рабочие дни

---

## 🎨 Черные и белые списки

### Доступные списки

| Список | Поле | Назначение |
|--------|------|------------|
| **CustomerBL** | customer_id | Заблокированные клиенты |
| **CustomerWL** | customer_id | Доверенные клиенты |
| **IpBL** | ip | Заблокированные IP |
| **IpWL** | ip | Доверенные IP |
| **CardBL** | card_id | Заблокированные карты |
| **CardWL** | card_id | Доверенные карты |
| **PhoneBL** | phone | Заблокированные телефоны |
| **PhoneWL** | phone | Доверенные телефоны |
| **BankCountryBL** | bank_country | Запрещенные страны банков |
| **BankCountryWL** | bank_country | Разрешенные страны банков |
| **CustomerCountryBL** | customer_country | Запрещенные страны клиентов |
| **CustomerCountryWL** | customer_country | Разрешенные страны клиентов |

### Управление списками

#### Добавить в список
```bash
curl -X PUT http://localhost:8888/api/merchant/1/profile/1/rule/{rule-id}/expression/{expr-id}/set \
  -H "Content-Type: application/json" \
  -d '{"val": "8.8.8.8"}'  # Добавить IP в список
```

#### Удалить из списка
```bash
curl -X DELETE http://localhost:8888/api/merchant/1/profile/1/rule/{rule-id}/expression/{expr-id}/set \
  -H "Content-Type: application/json" \
  -d '{"val": "8.8.8.8"}'
```

#### Получить список
```bash
curl http://localhost:8888/api/merchant/1/profile/1/rule/{rule-id}/expression/{expr-id}/set
```

### Временные баны

**Списки поддерживают expire:**
```sql
INSERT INTO af_set (set_descriptor_type, val, merchant_id, expire)
VALUES (3, '8.8.8.8', 1, CURRENT_TIMESTAMP + 24 * INTERVAL '1 hour');
```

**Логика:**
- Значение автоматически удалится после истечения срока
- Используется для временных блокировок
- Настраивается через Action Codes

---

## ⚡ Action Codes (Коды ответа банка)

### Назначение

Когда банк возвращает код ответа (например, "недостаточно средств", "неверный CVV"), система может автоматически реагировать.

### Структура Action Code Rule

```json
{
  "action_code": "51",  // Код от банка
  "tries": 3,  // Сколько раз можно получить этот код
  "ban_time": 1440,  // На сколько минут банить (0 = навсегда)
  "black_list": 5,  // В какой список добавлять (CardBL)
  "field": "card_id"  // Какое поле банить
}
```

### Пример сценария

**Ситуация:** Карта получила код "51" (insufficient funds) 3 раза

**Действие системы:**
1. Считает сколько раз был код "51" с этой картой
2. Если ≥ 3 раз → добавляет card_id в CardBL
3. Устанавливает expire = CURRENT_TIMESTAMP + 1440 минут
4. Следующая транзакция с этой картой → автоматически Reject

### API

#### Получить правила для кодов
```bash
curl http://localhost:8888/api/action_codes/1  # для мерчанта 1
```

#### Добавить правило
```bash
curl -X PUT http://localhost:8888/api/action_codes/1 \
  -d '{
    "action_code": "51",
    "tries": 3,
    "ban_time": 1440,
    "black_list": 5
  }'
```

#### Удалить правило
```bash
curl -X DELETE http://localhost:8888/api/action_codes/1 \
  -d '{"code": "51"}'
```

---

## 🌐 REST API - Полный справочник

### Core Endpoints

#### GET `/`
**Назначение:** Статус сервера

**Ответ:**
```json
{
  "status": "ok",
  "time": "0.00 ms"
}
```

#### GET `/-/stats`
**Назначение:** Статистика работы сервера

**Ответ:**
```json
{
  "served": 1234,
  "active": 5,
  "processing": 2,
  "queued": 0,
  "errors": 12,
  "fetching": 1,
  "resizing": 0
}
```

### Transaction Endpoints

#### POST `/check`
**Назначение:** Проверка транзакции

**Запрос:**
```json
{
  "AFMerchantId": 1,
  "key": "profile_key",
  "OrderId": "unique_order_id",
  "PaymentId": "payment_id",  // Опционально
  "Amount": 50000,  // В копейках
  "Type": "payment",  // payment, refund, p2p
  "IP": "192.168.1.1",
  
  "AFCustomer": {
    "MerchantCustomerId": "customer_123",
    "Email": "user@example.com",  // Опционально
    "Phone": "+79991234567"  // Опционально
  },
  
  "CardData": {
    "Token": "card_token_unique",
    "MaskedPan": "411111******1111",
    "ExpDate": "1225",  // MMYY или MM/YY
    "CardHolder": "JOHN DOE"
  },
  
  "RecipientCardData": {  // Опционально, для P2P
    "Token": "recipient_token",
    "MaskedPan": "555555******5555",
    "ExpDate": "0126",
    "CardHolder": "JANE DOE"
  },
  
  "LinkGuid": "guid_for_linked_transactions",  // Опционально
  "LinkType": "subscription"  // Опционально
}
```

**Ответ:**
```json
{
  "Decision": "Accept",
  "Authorized": false,
  "TransactionId": 123,
  "Rules": [
    {
      "Id": 1,
      "Name": "Amount Greater Than 100000",
      "Decision": null,
      "Description": "Сумма платежа не должна быть больше 10000000."
    }
  ],
  "time": "0.23 ms"
}
```

**Идемпотентность:** Повторный вызов с тем же OrderId вернёт сохранённый результат

#### POST `/update`
**Назначение:** Обновление статуса транзакции после проведения

**Запрос:**
```json
{
  "OrderId": "order_id_from_check",
  "Authorized": true,  // true = успешно проведена
  "ActionCode": "0"  // Код ответа от банка
}
```

**Ответ:**
```json
{
  "Decision": "Accept",
  "Authorized": true,
  "TransactionId": 123
}
```

**Логика:**
- Если `Authorized=true` → можно установить только для Decision="Accept"
- Если `ActionCode` указан → система проверит Action Code Rules
- Может автоматически добавить в черные списки

#### POST `/rules`
**Назначение:** Получить список всех правил (для отладки)

**Ответ:** Массив всех правил системы с выражениями

### Merchant Management API

#### GET `/api/merchant/`
**Список всех мерчантов**

**Ответ:**
```json
{
  "list": [
    {"id": 0, "name": "Default Merchant"},
    {"id": 1, "name": "Test Merchant"}
  ],
  "time": "0.01 ms"
}
```

#### GET `/api/merchant/{id}`
**Получить конкретного мерчанта**

#### PUT `/api/merchant/`
**Создать нового мерчанта**

**Запрос:**
```json
{
  "id": 5,  // Опционально
  "name": "New Merchant"
}
```

### Profile Management API

#### GET `/api/merchant/{id}/profile/`
**Список профилей мерчанта**

**Ответ:**
```json
{
  "list": [
    {
      "id": 1,
      "forceDecision": "Accept",
      "defaultProfile": null
    }
  ]
}
```

#### GET `/api/merchant/{id}/profile/{profile-id}`
**Получить конкретный профиль**

#### PUT `/api/merchant/{id}/profile/`
**Создать новый профиль**

**Запрос:**
```json
{
  "key": "unique_profile_key",
  "password": "secure_password",
  "forceDecision": "Accept",  // Accept, Reject, Review
  "defaultProfile": 1  // ID базового профиля (опционально)
}
```

**ForceDecision:** Решение по умолчанию, если ни одно правило не сработало

**DefaultProfile:** Наследование правил от другого профиля

#### POST `/api/merchant/{id}/profile/{profile-id}`
**Обновить профиль**

**Запрос:**
```json
{
  "forceDecision": "Review",
  "defaultProfile": 2
}
```

#### DELETE `/api/merchant/{id}/profile/`
**Удалить профиль**

**Запрос:**
```json
{
  "id": 5
}
```

### Rule Management API

#### GET `/api/rule`
**Список всех доступных правил**

**Ответ:**
```json
{
  "list": [
    {
      "id": 1,
      "name": "Amount Greater Than 100000",
      "decision": "Review",
      "weight": 10,
      "expressions": [
        {
          "id": 1,
          "field": "Сумма транзакции",
          "typeOperator": ">",
          "operand": "10000000",
          "threshold": 0
        }
      ]
    }
  ]
}
```

#### GET `/api/merchant/{id}/profile/{profile-id}/rule/`
**Правила привязанные к профилю**

Возвращает правила с учётом наследования от defaultProfile

#### PUT `/api/merchant/{id}/profile/{profile-id}/rule/`
**Привязать правило к профилю**

**Запрос:**
```json
{
  "ruleId": 5
}
```

**Логика:**
- Если правило уже есть в defaultProfile → отключает его
- Если правило новое → включает для профиля

#### DELETE `/api/merchant/{id}/profile/{profile-id}/rule/`
**Отвязать правило от профиля**

**Запрос:**
```json
{
  "ruleId": 5
}
```

### Expression Configuration API

#### POST `/api/merchant/{id}/profile/{profile-id}/rule/{rule-id}/expression/{expr-id}`
**Переопределить параметры выражения для конкретного профиля**

**Запрос:**
```json
{
  "expressionId": 1,
  "threshold": 5,  // Новый порог
  "operand": "3",  // Новое значение
  "periodMinutes": 120  // Новый период
}
```

**Применение:**
- Правило общее для всех
- Но для конкретного профиля параметры другие
- Например: для VIP клиентов порог выше

### Action Codes API

#### GET `/api/action_codes/{merchant-id}`
**Получить все правила реакции на коды**

**Ответ:**
```json
{
  "list": [
    {
      "action_code": "51",
      "tries": 3,
      "ban_time": 1440,
      "black_list": 5,
      "field": "card_id"
    }
  ]
}
```

#### PUT `/api/action_codes/{merchant-id}`
**Добавить/обновить правило**

#### DELETE `/api/action_codes/{merchant-id}`
**Удалить правило**

---

## 🔗 Интеграции

### 1. BIN Lookup

**Назначение:** Получение информации о банке-эмитенте по BIN (первые 6 цифр карты)

**Endpoint:** `GET {INFO_URL}/bins/{bin}`

**Получаем:**
- Страна банка (country_code)
- Название банка (evaluated_bank_name)
- Тип карты (card_type: DEBIT, CREDIT, PREPAID)
- Бренд (VISA, MasterCard, MIR)

**Использование:**
```python
# Автоматически при проверке транзакции
bin = card.masked_pan[:6]  # "411111"
bin_info = await get_info_by_bin(bin)
transaction.bank_country = bin_info['country_code']
transaction.card_bank_name = bin_info['evaluated_bank_name']
```

### 2. GeoIP

**Назначение:** Определение страны по IP адресу

**Endpoint:** `GET {INFO_URL}/countries/{ip}`

**Получаем:**
- Код страны (countryshort: "RU", "US", "DE")
- Название страны (country: "Russia")

**Использование:**
```python
customer_country = await get_country_code_by_ip("192.168.1.1")
# Проверка в правилах: customer_country IN CustomerCountryWL
```

### 3. Proxy Detection

**Назначение:** Определение proxy/VPN

**Endpoint:** `GET {INFO_URL}/proxies/{ip}`

**Получаем:**
- Код страны прокси (countryshort)
- Флаг is_proxy

**Использование:**
```python
proxy_country = await get_proxy_country_by_ip("1.2.3.4")
if proxy_country:  # Не пустая строка = прокси
    # Правило: ProxyCountryIsNotEmpty → Reject
```

### 4. Sanctions Check

**Назначение:** Проверка держателя карты в санкционных списках

**Endpoint:** `POST {INFO_URL}/fin_queries`

**Запрос:**
```json
{
  "merchant_id": 1,
  "person": {
    "first_name": "JOHN",
    "last_name": "DOE"
  },
  "country_code": "UA"  // Опционально
}
```

**Ответ:**
```json
{
  "data": {
    "instance": {
      "responses": [
        {"rc": 401}  // 401 = в списке, 0 = нет
      ]
    }
  }
}
```

**Использование:**
```python
is_restricted = await is_restricted_holder("JOHN DOE", merchant_id, "UA")
if is_restricted:
    transaction.is_card_holder_restricted = True
    # Можно создать правило проверки этого флага
```

---

## 🎬 Система действий (Actions)

### Доступные действия

#### 1. Email уведомление

**Тип:** `email`

**Конфигурация:**
```sql
INSERT INTO af_rule_action (rule_id, action_type, parameters)
VALUES (5, 'email', 'admin@company.com, security@company.com');
```

**Логика:**
- При срабатывании правила отправляется email
- Содержит детали транзакции и правила
- Можно указать несколько адресов через запятую

**Пример письма:**
```
Subject: Antifraud notification for transaction PMT123

profile id: 1
amount: 1500.00
merchant transaction id: PMT123
rule: Amount Greater Than 100000
decision: Review
expressions: min=1440, threshold=10000000, start_from=day
```

#### 2. Добавление в черный список

**Тип:** `add_field_to_bl`

**Конфигурация:**
```sql
INSERT INTO af_rule_action (rule_id, action_type, parameters)
VALUES (10, 'add_field_to_bl', '{
  "field_name": "card_id",
  "black_list_id": 5,
  "ban_time": 1440,
  "for_merchant": true
}');
```

**Параметры:**
- `field_name` - какое поле добавлять (card_id, phone, customer_id)
- `black_list_id` - ID черного списка
- `ban_time` - на сколько минут (0 = навсегда, null = навсегда)
- `for_merchant` - только для текущего мерчанта или глобально

**Логика:**
```
1. Правило срабатывает
2. Decision = Reject
3. Action: Добавить card_id в CardBL на 24 часа
4. Следующие транзакции с этой картой → автоматически Reject
```

---

## 🔢 Типы данных и форматы

### Поля транзакции

| Поле | Тип | Формат | Пример |
|------|-----|--------|--------|
| AFMerchantId | integer | ID | 1 |
| key | string | Ключ профиля | "test_key" |
| OrderId | string | Уникальный ID | "ORD12345" |
| PaymentId | string | ID платежа | "PMT67890" |
| Amount | integer | Копейки | 50000 = 500 руб |
| Type | string | Тип операции | payment, refund, p2p |
| IP | string | IPv4 | "192.168.1.1" |
| MerchantCustomerId | string | ID в системе мерчанта | "USER123" |
| Email | string | Email | "user@example.com" |
| Phone | string | Телефон | "+79991234567" |
| Token | string | Токен карты | Уникальный |
| MaskedPan | string | Маска PAN | "411111******1111" |
| ExpDate | string | Срок действия | "1225" или "12/25" |
| CardHolder | string | Держатель | "JOHN DOE" |
| LinkGuid | string | GUID связанных транзакций | UUID |
| LinkType | string | Тип связи | subscription, recurring |

### Поля для правил

| Field | Type | Description | Применение |
|-------|------|-------------|------------|
| amount | integer | Сумма (копейки) | Контроль суммы |
| customer_id | integer | ID клиента | Лимиты по клиенту |
| card_id | integer | ID карты | Лимиты по карте |
| recipient_card_id | integer | ID карты получателя | P2P контроль |
| ip | bigint | IP адрес | GeoIP, черные списки |
| phone | string | Телефон | Контроль по телефону |
| customer_country | string | Страна клиента (GeoIP) | Географические ограничения |
| bank_country | string | Страна банка (BIN) | Контроль стран |
| proxy_country | string | Страна прокси | Выявление VPN |
| created | timestamp | Время транзакции | Временные окна |
| attempt | counter | Попытки оплаты | Velocity checks |
| failed_attempt | counter | Неудачные попытки | Fraud detection |
| failed_sequential_attempt | counter | Последовательные неудачи | Brute force |
| payment | counter | Успешные платежи | Velocity checks |
| card_bin | integer | BIN карты | Контроль по банкам |
| recipient_card_bin | integer | BIN получателя | P2P контроль |
| card_type | string | Тип карты | DEBIT/CREDIT фильтры |
| card_bank_name | string | Название банка | Контроль банков |
| card_holder | string | Держатель | Sanctions check |
| is_card_holder_restricted | boolean | В санкциях | Автоотклонение |

### Операторы

| Operator | SQL | Применение | Типы |
|----------|-----|------------|------|
| == | = | Равно | int, string |
| != | <> | Не равно | int, string |
| > | > | Больше | int |
| >= | >= | Больше или равно | int |
| < | < | Меньше | int |
| <= | <= | Меньше или равно | int |
| IN | IN | В списке | set, array |
| NOT IN | NOT IN | Не в списке | set, array |
| BETWEEN | BETWEEN | В диапазоне | int, datetime |

---

## 🏗️ Архитектура проверки

### Процесс обработки транзакции

```
1. Получение запроса POST /check
   ↓
2. Проверка идемпотентности
   └─► Если OrderId существует → вернуть сохранённый результат
   ↓
3. Получение профиля
   └─► Поиск по (AFMerchantId, key)
   └─► Если не найден → error
   ↓
4. Параллельное получение данных
   ├─► Получить/создать customer
   ├─► Получить/создать card
   ├─► Получить/создать recipient_card
   └─► Получить merchant
   ↓
5. Обогащение данных (внешние сервисы)
   ├─► BIN lookup → bank_country, card_bank_name, card_type
   ├─► GeoIP → customer_country
   ├─► Proxy check → proxy_country
   ├─► Sanctions check → is_card_holder_restricted
   └─► (То же для recipient_card)
   ↓
6. Создание записи транзакции в БД
   └─► Сохранение всех полученных данных
   ↓
7. Получение правил профиля
   └─► С учётом наследования от defaultProfile
   ↓
8. Загрузка выражений для каждого правила
   └─► С учётом переопределений для профиля
   ↓
9. Применение правил (по порядку sort_order)
   Для каждого правила:
   ├─► Генерация SQL запроса из выражений
   ├─► Выполнение проверки
   ├─► Если порог превышен:
   │   ├─► Установить Decision из правила
   │   ├─► Выполнить Actions (email, blacklist)
   │   └─► Записать в debug_rules
   └─► Продолжить следующее правило
   ↓
10. Если ни одно правило не сработало
    └─► Установить ForceDecision из профиля
    ↓
11. Сохранить итоговый Decision и debug_rules
    ↓
12. Вернуть результат
    {Decision, Authorized, TransactionId, Rules, time}
```

### Время выполнения

| Этап | Время | Оптимизация |
|------|-------|-------------|
| Проверка идемпотентности | 1-2 ms | Индекс по order_id |
| Получение профиля/мерчанта | 2-5 ms | Параллельно |
| Создание customer/card | 5-10 ms | Параллельно |
| BIN lookup (внешний) | 20-50 ms | Кеширование |
| GeoIP (внешний) | 10-30 ms | Кеширование |
| Sanctions (внешний) | 50-100 ms | Кеширование |
| Создание транзакции | 5-10 ms | Prepared statements |
| Загрузка правил | 5-10 ms | Индексы |
| Проверка правил | 1-10 ms/правило | Оптимизированный SQL |

**Итого:** 0.1 - 200 ms (типично 50-100 ms)

---

## 💡 Примеры использования

### Сценарий 1: Базовая проверка платежа

**Задача:** Проверить онлайн-платеж 5000 руб

**Запрос:**
```bash
curl -X POST http://localhost:8888/check \
  -H "Content-Type: application/json" \
  -d '{
    "AFMerchantId": 1,
    "key": "shop_key",
    "OrderId": "ORDER_12345",
    "Amount": 500000,
    "Type": "payment",
    "IP": "95.161.123.45",
    "AFCustomer": {
      "MerchantCustomerId": "user_12345",
      "Email": "user@gmail.com",
      "Phone": "+79991234567"
    },
    "CardData": {
      "Token": "tok_3xK9mP1wQ7",
      "MaskedPan": "411111******1111",
      "ExpDate": "12/25",
      "CardHolder": "IVAN PETROV"
    }
  }'
```

**Что проверяет система:**
1. ✅ Проверяет сумму (5000 руб < 100К) → OK
2. ✅ Проверяет IP в IpBL → Нет
3. ✅ Проверяет карту в CardBL → Нет
4. ✅ Определяет страну клиента (RU) → В CustomerCountryWL
5. ✅ Получает инфо о банке (BIN 411111 = VISA Russia)
6. ✅ Проверяет holder в sanctions → Нет
7. ✅ Считает попытки с этой карты → 1 (норма)

**Результат:**
```json
{
  "Decision": "Accept",
  "TransactionId": 123
}
```

### Сценарий 2: P2P перевод

**Задача:** Проверить перевод с карты на карту

**Запрос:**
```bash
curl -X POST http://localhost:8888/check \
  -d '{
    "AFMerchantId": 1,
    "key": "p2p_key",
    "OrderId": "P2P_67890",
    "Amount": 1000000,
    "Type": "p2p",
    "IP": "185.45.67.89",
    "AFCustomer": {
      "MerchantCustomerId": "sender_123",
      "Phone": "+79161234567"
    },
    "CardData": {
      "Token": "sender_card_token",
      "MaskedPan": "220220******2202",
      "ExpDate": "0326",
      "CardHolder": "SENDER NAME"
    },
    "RecipientCardData": {
      "Token": "recipient_card_token",
      "MaskedPan": "554433******4433",
      "ExpDate": "0000",
      "CardHolder": "RECIPIENT NAME"
    }
  }'
```

**Дополнительные проверки для P2P:**
- ✅ Recipient card BIN lookup
- ✅ Recipient bank country
- ✅ Recipient в sanctions lists
- ✅ Cross-border P2P detection
- ✅ Правила для recipient_card_id

### Сценарий 3: Настройка правила для VIP клиентов

**Задача:** Для VIP профиля установить порог 500К вместо 100К

```bash
# 1. Создать VIP профиль
curl -X PUT http://localhost:8888/api/merchant/1/profile/ \
  -d '{
    "key": "vip_profile",
    "forceDecision": "Accept",
    "defaultProfile": 1
  }'
# Ответ: {"id": 2}

# 2. Переопределить порог в правиле "Amount Greater Than 100000"
curl -X POST http://localhost:8888/api/merchant/1/profile/2/rule/1/expression/1 \
  -d '{
    "expressionId": 1,
    "threshold": 0,
    "operand": "50000000"
  }'
```

**Результат:**
- Для обычного профиля: >100К → Review
- Для VIP профиля: >500К → Review

### Сценарий 4: Автоматический бан по кодам

**Задача:** Банить карту на 24 часа после 3-х отказов с кодом "05" (Do not honor)

```bash
# 1. Настроить правило для кода "05"
curl -X PUT http://localhost:8888/api/action_codes/1 \
  -d '{
    "action_code": "05",
    "tries": 3,
    "ban_time": 1440,
    "black_list": 5
  }'

# 2. Транзакции с картой получают код "05"
curl -X POST http://localhost:8888/update \
  -d '{"OrderId": "ORD1", "Authorized": false, "ActionCode": "05"}'
curl -X POST http://localhost:8888/update \
  -d '{"OrderId": "ORD2", "Authorized": false, "ActionCode": "05"}'
curl -X POST http://localhost:8888/update \
  -d '{"OrderId": "ORD3", "Authorized": false, "ActionCode": "05"}'

# 3. После 3-й транзакции карта автоматически попадает в CardBL на 24 часа

# 4. Следующая транзакция с этой картой
curl -X POST http://localhost:8888/check -d '{...}'
# Ответ: {"Decision": "Reject", "Rules": [{"Name": "Card in Blacklist", "Decision": "Reject"}]}
```

### Сценарий 5: Velocity checks (контроль частоты)

**Задача:** Не более 5 платежей с одной карты в час

**Создание правила:**
```sql
-- 1. Создать правило
INSERT INTO af_rule (name, decision, sort_order, weight)
VALUES ('Max 5 Payments Per Card Per Hour', 'Review', 100, 50);

-- 2. Создать выражение
INSERT INTO af_expression (
  rule_id, field_id, type_operator_id, operand_type, 
  operand, threshold, period_minutes
) VALUES (
  currval('af_rule_id_seq'),
  10,  -- field: payment
  2,   -- operator: >
  4,   -- type: Counter
  'card_id',  -- группировка
  5,   -- порог
  60   -- период: 1 час
);

-- 3. Привязать к профилю
INSERT INTO af_profile_rule (profile_id, rule_id)
VALUES (1, currval('af_rule_id_seq'));
```

**Проверка:**
```bash
# 1-5 транзакции → Accept
for i in {1..5}; do
  curl -X POST http://localhost:8888/check -d '{...OrderId: "ORD_$i"}'
done

# 6-я транзакция → Review (правило сработало!)
curl -X POST http://localhost:8888/check -d '{...OrderId: "ORD_6"}'
# Ответ: {"Decision": "Review"}
```

---

## 🎨 Продвинутые возможности

### 1. Наследование профилей (DefaultProfile)

**Применение:** Создать базовый набор правил и переиспользовать

**Пример:**
```
Default Profile (ID=0)
├─► 10 базовых правил
├─► ForceDecision: Accept
│
├─► Shop Profile (ID=1, defaultProfile=0)
│   ├─► Наследует 10 базовых правил
│   ├─► + 3 дополнительных правила
│   └─► Переопределяет пороги в 2 правилах
│
└─► VIP Profile (ID=2, defaultProfile=0)
    ├─► Наследует 10 базовых правил
    ├─► Отключает 2 строгих правила
    └─► Увеличивает пороги в 5 правилах
```

**Логика:**
1. Правила из defaultProfile автоматически применяются
2. Можно **отключить** правило для конкретного профиля
3. Можно **переопределить** параметры (threshold, operand, period)
4. Можно **добавить** новые правила

### 2. Глобальные vs Профильные проверки

**IsGlobal флаг** в выражениях:

```python
# IsGlobal = true
# Считает транзакции по ВСЕМ профилям мерчанта
SELECT COUNT(*) FROM af_transaction 
WHERE card_id = 123 AND (profile_id = 1 OR true)

# IsGlobal = false  
# Считает только по текущему профилю
SELECT COUNT(*) FROM af_transaction
WHERE card_id = 123 AND (profile_id = 1 OR false)
```

**Применение:**
- IsGlobal=true: Общие лимиты для всего мерчанта
- IsGlobal=false: Раздельные лимиты для каждого канала

### 3. Периоды проверки (StartFrom)

**Варианты:**
- `None` - скользящее окно (последние N минут)
- `hour` - с начала текущего часа
- `day` - с начала текущего дня
- `week` - с начала текущей недели
- `month` - с начала текущего месяца

**Примеры:**

```python
# Скользящее окно: последние 60 минут
{
  "periodMinutes": 60,
  "startFrom": "None"
}

# С начала дня (сбрасывается в 00:00)
{
  "periodMinutes": 1440,
  "startFrom": "day"
}

# С начала месяца
{
  "periodMinutes": 43200,  // ~30 дней
  "startFrom": "month"
}
```

**Применение:**
- Дневные лимиты (startFrom="day")
- Месячные лимиты (startFrom="month")
- Почасовые лимиты (startFrom="hour")
- Rolling windows (startFrom="None")

### 4. Связанные транзакции (LinkGuid, LinkType)

**Назначение:** Группировка связанных транзакций

**Примеры:**
```json
// Подписка - первый платёж
{
  "OrderId": "SUB_001_FIRST",
  "LinkGuid": "subscription_abc123",
  "LinkType": "subscription_first",
  ...
}

// Подписка - recurring платежи
{
  "OrderId": "SUB_001_REC_001",
  "LinkGuid": "subscription_abc123",
  "LinkType": "subscription_recurring",
  ...
}
```

**Применение:**
- Разные правила для первого и recurring платежей
- Отслеживание fraud по подпискам
- Статистика по типам операций

### 5. Весовая система правил (Weight)

**Назначение:** Приоритизация правил

**Логика:**
- Weight = 100 → критическое (черные списки)
- Weight = 50 → важное (velocity checks)
- Weight = 10 → информационное (мониторинг)

**Применение:**
```python
# В будущем можно добавить:
if rule.weight >= 100 and rule.decision == "Reject":
    return immediately  # Критическое правило
else:
    continue checking  # Собрать все срабатывания
```

---

## 📊 Мониторинг и отчётность

### Статистика сервера

**Endpoint:** `GET /-/stats`

**Метрики:**
- `served` - обработано запросов
- `active` - активных соединений
- `processing` - обрабатывается сейчас
- `queued` - в очереди
- `errors` - ошибок
- `fetching` - внешних запросов
- `resizing` - операций изменения размера

### Debug информация

Каждая транзакция сохраняет в `debug_rules` полную информацию:

```json
{
  "Rules": [
    {
      "Id": 1,
      "Name": "Amount Greater Than 100000",
      "Decision": "Review",
      "Description": "Сумма платежа не должна быть больше 10000000. Текущая: 15000000"
    },
    {
      "Id": 2,
      "Name": "IP in Blacklist",
      "Decision": null,
      "Description": "IP 192.168.1.1 в черном списке. Найдено: 0"
    }
  ]
}
```

**Применение:**
- Анализ почему транзакция отклонена
- Отчёты для клиентов
- Оптимизация правил
- Аудит решений

### Запросы к БД для аналитики

```sql
-- Статистика решений за день
SELECT decision, COUNT(*), SUM(amount)/100 as total_rub
FROM af_transaction 
WHERE created > CURRENT_DATE
GROUP BY decision;

-- Топ срабатывающих правил
SELECT 
  jsonb_array_elements(debug_rules)->>'Name' as rule,
  COUNT(*) as triggered
FROM af_transaction
WHERE jsonb_array_length(debug_rules) > 0
GROUP BY rule
ORDER BY triggered DESC;

-- Транзакции по странам
SELECT customer_country, COUNT(*), AVG(amount)/100 as avg_rub
FROM af_transaction
GROUP BY customer_country;

-- Fraud rate по Action Codes
SELECT action_code, COUNT(*), 
  COUNT(*) FILTER (WHERE authorized=false) * 100.0 / COUNT(*) as fraud_rate
FROM af_transaction
WHERE action_code IS NOT NULL
GROUP BY action_code;
```

---

## 🔐 Безопасность

### Аутентификация профилей

**Ключ + пароль:**
```python
profile = {
  "key": "unique_profile_key",  // Передаётся в каждом запросе
  "password": "secure_password"  // Хранится в БД (TODO: hashing!)
}
```

**Применение:**
- Каждый мерчант имеет свои профили
- Профили изолированы (нельзя проверить транзакцию чужого мерчанта)
- Можно создавать разные профили для разных каналов

### Request ID

**Для трейсинга:**
```bash
curl -H "X-Request-ID: abc123" http://localhost:8888/check ...
```

**Логирование:**
```
2025-10-15 12:00:00 INFO [abc123] Processing transaction ORDER_123
2025-10-15 12:00:00 DEBUG [abc123] Checking rule "Amount Greater"
2025-10-15 12:00:00 INFO [abc123] Decision: Accept
```

---

## 📈 Масштабирование

### Connection Pool

```python
POOL_SIZE = 5  # Размер пула соединений с БД
MAX_CLIENTS = 100  # Максимум HTTP клиентов для внешних запросов
```

**Производительность:**
- Пул asyncpg: до 10К транзакций/сек на 1 инстанс
- Tornado: асинхронная обработка
- PostgreSQL: индексы на всех ключевых полях

### Горизонтальное масштабирование

**Возможности:**
- Запустить несколько инстансов Antifraud
- Использовать load balancer (nginx, HAProxy)
- Shared PostgreSQL
- Read replicas для аналитики

**Пример:**
```
                  ┌──► Antifraud Instance 1 ──┐
Load Balancer ────┼──► Antifraud Instance 2 ──┼──► PostgreSQL (Master)
                  └──► Antifraud Instance 3 ──┘        ↓
                                                   Replica (Read-only)
                                                   для аналитики
```

---

## 🎯 Реальные кейсы применения

### 1. Интернет-магазин

**Правила:**
- ✅ Не более 100К руб за платёж без 3DS
- ✅ Не более 3 карт с одного IP в час
- ✅ Не более 10 платежей с одной карты в день
- ✅ Страна клиента = страна банка (или в whitelist)
- ✅ IP не прокси
- ✅ Карта не в BL

**ForceDecision:** Accept (по умолчанию разрешаем)

### 2. Букмекерская контора (высокий риск)

**Правила:**
- ✅ Только страны из whitelist
- ✅ Только карты RU/BY банков
- ✅ Не более 50К руб за депозит
- ✅ Не более 200К руб в день на клиента
- ✅ Holder проверка в sanctions
- ✅ Максимум 3 неудачных попытки подряд
- ✅ IP не прокси (строго)

**ForceDecision:** Review (по умолчанию на проверку)

### 3. P2P сервис

**Правила:**
- ✅ Не более 15К руб за перевод (регуляторные требования)
- ✅ Не более 40К руб в месяц на клиента
- ✅ Обе карты не в BL
- ✅ Оба holders не в sanctions
- ✅ Не более 10 переводов в день
- ✅ Не более 3 разных карт-получателей в день

**Специфика:**
- Проверка и sender, и recipient
- Лимиты по сумме (регулятор)
- Контроль структурирования

### 4. Подписочный сервис

**Правила:**

**Для первого платежа (subscription_first):**
- ✅ 3DS обязателен
- ✅ Не более 10К руб
- ✅ Holder sanctions check
- ✅ IP GeoIP = банк страна

**Для recurring (subscription_recurring):**
- ✅ Менее строгие проверки
- ✅ Без 3DS (сохранённая карта)
- ✅ Лимиты по истории
- ✅ Автоматический Accept если была успешная

**Связь:** LinkGuid для группировки платежей одной подписки

---

## 🔬 Детекция мошенничества

### Паттерны fraud

#### 1. Card testing
**Признаки:**
- Много мелких транзакций подряд
- Разные суммы
- Быстрая последовательность

**Правило:**
```python
{
  "field": "attempt",
  "operand": "card_id",
  "threshold": 10,
  "periodMinutes": 10  // 10 попыток за 10 минут
}
```

#### 2. Структурирование
**Признаки:**
- Много транзакций чуть ниже лимита
- Один клиент, разные карты
- Регулярный pattern

**Правило:**
```python
# Правило 1: Сумма близка к лимиту
{
  "field": "amount",
  "operator": "BETWEEN",
  "operand": "9500000|10000000"  // 95К-100К
}

# Правило 2: Много таких транзакций
{
  "field": "payment",
  "operand": "customer_id",
  "threshold": 5,  // 5 платежей
  "periodMinutes": 1440  // За день
}
```

#### 3. Stolen card
**Признаки:**
- Новая карта (первая транзакция)
- Большая сумма сразу
- Необычная страна

**Правило:**
```python
{
  "field": "payment",
  "operand": "card_id",
  "threshold": 0  // Первый платёж
}
AND
{
  "field": "amount",
  "operator": ">",
  "operand": "5000000"  // Сразу 50К
}
```

#### 4. Аккаунт takeover
**Признаки:**
- Смена телефона/email
- Новая карта
- Новый IP/страна

**Правило:**
```python
{
  "field": "payment",
  "operand": "customer_id|phone",  // Новый телефон для customer
  "threshold": 1,  // Первая транзакция с новым телефоном
  "periodMinutes": 10
}
```

---

## 🌍 Мультитенантность

### Уровни изоляции

```
Мерчант (Merchant)
└─► Профиль 1 (Shop)
│   ├─► Правила
│   ├─► Транзакции
│   └─► Черные списки (опционально)
│
└─► Профиль 2 (Mobile App)
    ├─► Другие правила
    ├─► Транзакции
    └─► Черные списки (опционально)
```

**Изоляция:**
- Транзакции разных мерчантов изолированы
- Транзакции разных профилей могут быть изолированы (IsGlobal=false)
- Черные списки могут быть глобальными или специфичными

**Применение:**
- Один Antifraud для нескольких компаний
- Разные профили для разных каналов одной компании
- Централизованное управление

---

## 📝 Конфигурация

### Настройка профиля

```bash
# Создать профиль
curl -X PUT http://localhost:8888/api/merchant/1/profile/ \
  -d '{
    "key": "my_shop_key",
    "password": "secure_password",
    "forceDecision": "Review",  // По умолчанию на ревью
    "defaultProfile": null  // Без наследования
  }'
# Ответ: {"id": 5}

# Привязать правила
curl -X PUT http://localhost:8888/api/merchant/1/profile/5/rule/ \
  -d '{"ruleId": 1}'  # Amount check

curl -X PUT http://localhost:8888/api/merchant/1/profile/5/rule/ \
  -d '{"ruleId": 2}'  # IP blacklist

# Настроить параметры правила
curl -X POST http://localhost:8888/api/merchant/1/profile/5/rule/1/expression/1 \
  -d '{
    "expressionId": 1,
    "threshold": 0,
    "operand": "20000000"  // Порог 200К для этого профиля
  }'
```

### Настройка черных списков

```bash
# Добавить IP в черный список
curl -X PUT http://localhost:8888/api/merchant/1/profile/1/rule/2/expression/2/set \
  -d '{"val": "8.8.8.8"}'

# Добавить карту
curl -X PUT http://localhost:8888/api/merchant/1/profile/1/rule/3/expression/3/set \
  -d '{"val": "123456"}'  # card_id

# Посмотреть список
curl http://localhost:8888/api/merchant/1/profile/1/rule/2/expression/2/set

# Удалить из списка
curl -X DELETE http://localhost:8888/api/merchant/1/profile/1/rule/2/expression/2/set \
  -d '{"val": "8.8.8.8"}'
```

---

## 🎓 Best Practices

### 1. Порядок правил (SortOrder)

**Рекомендуется:**
```
1-10:   Белые списки (Accept, fast exit)
11-50:  Черные списки (Reject, safety)
51-100: Velocity checks (Review, analysis)
101+:   Сложные проверки (Review, detailed)
```

**Пример:**
```
SortOrder 1:  CustomerWL → Accept (trusted users)
SortOrder 2:  CardWL → Accept (trusted cards)
SortOrder 10: IpBL → Reject (blocked IPs)
SortOrder 11: CardBL → Reject (blocked cards)
SortOrder 50: Amount > 100K → Review
SortOrder 51: Multiple cards → Review
SortOrder 100: Complex ML model → Review
```

### 2. Профили для разных каналов

```
Мерчант: Online Shop
├─► Web Profile (key: "web_shop")
│   └─► Строгие проверки, 3DS required
├─► Mobile Profile (key: "mobile_app")
│   └─► Средние проверки, trusted devices
├─► Recurring Profile (key: "subscriptions")
│   └─► Мягкие проверки, saved cards
└─► API Profile (key: "api_partners")
    └─► Кастомные правила для партнёров
```

### 3. Тестирование правил

```bash
# 1. Создать тестовый профиль
curl -X PUT http://localhost:8888/api/merchant/1/profile/ \
  -d '{"key": "test_profile", "forceDecision": "Accept"}'

# 2. Привязать правила
curl -X PUT http://localhost:8888/api/merchant/1/profile/X/rule/ \
  -d '{"ruleId": 1}'

# 3. Тестировать с разными данными
curl -X POST http://localhost:8888/check \
  -d '{"key": "test_profile", "Amount": 50000, ...}'  # Should Accept
  
curl -X POST http://localhost:8888/check \
  -d '{"key": "test_profile", "Amount": 20000000, ...}'  # Should Review

# 4. Проверить логи
psql -c "SELECT * FROM af_transaction WHERE profile_id = X ORDER BY id DESC LIMIT 10;"
```

---

## 📋 Полный список возможностей

### ✅ Проверки

- [x] Сумма транзакции (лимиты)
- [x] Частота операций (velocity)
- [x] IP адреса (GeoIP, blacklist)
- [x] Карты (BIN, банк, blacklist)
- [x] P2P переводы (обе карты)
- [x] Клиенты (история, blacklist)
- [x] Телефоны (blacklist)
- [x] Страны (whitelist/blacklist)
- [x] Банки (whitelist/blacklist)
- [x] Держатели карт (sanctions)
- [x] Proxy/VPN detection
- [x] Временные окна (ночные блокировки)
- [x] Последовательные неудачи
- [x] Связанные транзакции

### ✅ Управление

- [x] REST API для всего
- [x] Мультимерчантность
- [x] Мультипрофильность
- [x] Наследование профилей
- [x] Переопределение параметров
- [x] Черные/белые списки
- [x] Временные баны
- [x] Action codes автоматизация

### ✅ Интеграции

- [x] BIN lookup
- [x] GeoIP
- [x] Proxy detection
- [x] Sanctions lists
- [x] Email notifications
- [x] Webhook actions (расширяемо)

### ✅ Мониторинг

- [x] Request ID tracing
- [x] Debug logs для каждой транзакции
- [x] Server stats
- [x] Аналитика в БД
- [x] Audit trail

### ✅ Производительность

- [x] Асинхронная обработка
- [x] Connection pooling
- [x] Параллельные запросы
- [x] Индексы в БД
- [x] Prepared statements

---
