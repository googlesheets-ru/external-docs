# Пояснение полей в запросе экономики заказов Озон

## 📋 Основные идентификаторы заказа

| Поле | Источник | Назначение | Пример |
|------|----------|------------|--------|
| **posting_number** | `fbs_order.posting_number` | Уникальный номер отправления Озон | "0100169990-0103-1" |
| **posting_number_short** | Функция обрезки | Группировка связанных отправлений одного заказа | "0100169990-0103" |
| **order_id** | `fbs_order.order_id` | ID заказа в системе Озон | "28537586406" |
| **order_number** | `fbs_order.order_number` | Номер заказа для покупателя | Внутренний номер |

## 📊 Статус и временные данные

| Поле | Источник | Назначение | Возможные значения |
|------|----------|------------|-------------------|
| **order_status** | `fbs_order.status` | Статус отправления | delivered, cancelled, awaiting_packaging |
| **order_substatus** | `fbs_order.substatus` | Детальный статус отправления | posting_canceled, posting_received |
| **order_date** | `fbs_order.in_process_at` | Дата принятия заказа в обработку | 2024-01-11T20:43:21.000Z |
| **shop_name** | `fbs_order.shop_name` | Название магазина продавца | "GetPwr" |
| **region** | `posting_analytics_data.region` | Регион доставки | "Москва и область" |

## 🛍️ Информация о товарах

| Поле | Источник | Назначение | Пример |
|------|----------|------------|--------|
| **sku** | `posting_product.sku` | SKU товара в системе Озон | 1823748558 |
| **offer_id** | `posting_product.offer_id` | Артикул продавца | "1531" |
| **product_name** | `posting_product.name` | Название товара | "Аккумулятор Li-ion 18650" |
| **quantity** | `posting_product.quantity` | Количество товара в отправлении | 1 |
| **product_price** | `posting_product.price` | Цена за единицу товара | 590.00 |
| **unit_selfcost** | `ms_self_cost_data.selfcost` или `posting_product.selfcost` | Себестоимость единицы товара | 125.00 |

## 💰 Расчетные поля по товарам

| Поле | Формула | Назначение | Логика расчета |
|------|---------|------------|----------------|
| **total_product_revenue** | Зависит от статуса заказа | Выручка от товара с учетом отмен и возвратов | См. раздел "Логика расчета выручки" |
| **total_selfcost** | `quantity * unit_selfcost` (только для валидных заказов) | Общая себестоимость товара | 0 для отмененных заказов |

## 🏪 Финансовые данные маркетплейса

| Поле | Источник | Назначение | Пример |
|------|----------|------------|--------|
| **commission_percent** | `posting_fin_data_product.commission_percent` | Процент комиссии Озон | 22 |
| **commission_amount** | `posting_fin_data_product.commission_amount` | Сумма комиссии (отрицательная) | -147.5 |
| **payout** | `posting_fin_data_product.payout` | Выплата продавцу от Озон | 1131 |
| **total_discount_value** | `posting_fin_data_product.total_discount_value` | Размер скидки на товар | 1450 |

## 🚚 Услуги логистики (детализация)

| Поле | Источник JSON | Назначение | Тип расхода |
|------|---------------|------------|-------------|
| **service_pickup** | `item_services.marketplace_service_item_pickup` | Забор товара со склада | Логистика |
| **service_delivery** | `item_services.marketplace_service_item_deliv_to_customer` | Доставка покупателю | Логистика |
| **service_fulfillment** | `item_services.marketplace_service_item_fulfillment` | Услуги фулфилмента | Логистика |
| **service_return** | `item_services.marketplace_service_item_return_flow_trans` | Обработка возвратов | Логистика |
| **total_services_cost** | Сумма всех service_* | Общие расходы на услуги | Итого логистика |

## 💸 Транзакции по отправлению

| Поле | Источник | Назначение | Как считается |
|------|----------|------------|---------------|
| **transaction_revenue** | `ozon_fin_transaction` | Доходы от всех транзакций | SUM(amount > 0) |
| **transaction_net_revenue** | `ozon_fin_transaction` | Чистая выручка с учетом возвратов | SUM(доставки + возвраты + отмены) |
| **transaction_commission** | `ozon_fin_transaction` | Комиссионные расходы | SUM(WHERE operation_type_name LIKE '%комиссия%') |
| **transaction_logistics** | `ozon_fin_transaction` | Логистические расходы | SUM(WHERE operation_type_name LIKE '%доставка%') |
| **transaction_bonus** | `ozon_fin_transaction` | Расходы на бонусы | SUM(WHERE operation_type_name LIKE '%бонус%') |
| **transaction_promotion** | `ozon_fin_transaction` | Расходы на продвижение | SUM(WHERE operation_type_name LIKE '%продвижени%') |
| **transaction_total_expenses** | `ozon_fin_transaction` | Все расходы по транзакциям | SUM(amount < 0) |

## 📈 Экономические показатели (КПЭ)

| Поле | Формула | Экономический смысл | Применение |
|------|---------|-------------------|------------|
| **gross_profit** | `total_product_revenue - total_selfcost` | Валовая прибыль | Базовая рентабельность товара |
| **operating_profit** | `gross_profit - |commission_amount| - |total_services_cost|` | Операционная прибыль | Прибыль после расходов маркетплейса |
| **net_profit** | `operating_profit - transaction_bonus - transaction_promotion` | Чистая прибыль | Итоговая прибыльность заказа |
| **profit_margin_percent** | `(net_profit / total_product_revenue) * 100` | Рентабельность в % | Сравнение эффективности товаров |

## 🔍 Поля контроля качества данных

| Поле | Тип | Назначение | Возможные значения |
|------|-----|------------|-------------------|
| **is_valid_order** | `boolean` | Флаг валидности заказа | `false` для cancelled/posting_canceled |
| **has_return_or_cancel** | `boolean` | Наличие возвратов/отмен в транзакциях | `true` если есть операции возврата/отмены |
| **return_cancel_amount** | `numeric` | Сумма возвратов/отмен | Абсолютная сумма операций возврата |

## 🔍 Дополнительная аналитическая информация

| Поле | Источник | Назначение | Пример |
|------|----------|------------|--------|
| **operation_types_count** | `COUNT(DISTINCT operation_type)` | Количество типов операций по заказу | 5 |
| **operation_types** | `STRING_AGG(operation_type_name)` | Список всех типов операций | "Доставка; Комиссия; Бонусы" |
| **delivery_date_begin** | `posting_analytics_data.delivery_date_begin` | Начало периода доставки | 2024-01-12 |
| **delivery_date_end** | `posting_analytics_data.delivery_date_end` | Конец периода доставки | 2024-01-15 |

## 💡 Ключевые принципы расчетов

### 🎯 Приоритеты данных:
1. **Себестоимость**: МойСклад (`ms_self_cost_data`) → posting_product → 0
2. **Финансы**: posting_fin_data_product → ozon_fin_transaction
3. **Услуги**: item_services (детально) → transaction_summary (агрегированно)

### 📊 Логика расчета выручки:

```sql
CASE 
    WHEN status = 'cancelled' OR substatus = 'posting_canceled' THEN 0
    WHEN has_return_or_cancel = true THEN GREATEST(transaction_net_revenue, 0)
    ELSE quantity * product_price
END
```

**Отмененные заказы:**
- `is_valid_order = false`
- `total_product_revenue = 0`
- Все виды прибыли = 0

**Заказы с возвратами:**
- `is_valid_order = true`, `has_return_or_cancel = true`
- `total_product_revenue = GREATEST(transaction_net_revenue, 0)`
- Прибыль рассчитывается от чистой выручки

**Обычные заказы:**
- `is_valid_order = true`, `has_return_or_cancel = false`
- `total_product_revenue = quantity * product_price`
- Стандартные расчеты прибыли

### 📈 Подробный порядок расчета прибыли:

#### 🔢 Этап 1: Определение выручки (`total_product_revenue`)

```sql
total_product_revenue = CASE 
    WHEN NOT is_valid_order THEN 0  -- Отмененные заказы
    WHEN has_return_or_cancel THEN GREATEST(transaction_net_revenue, 0)  -- С возвратами
    ELSE quantity * product_price  -- Обычные заказы
END
```

**Примеры:**
- **Обычный заказ:** `2 × 590₽ = 1,180₽`
- **Отмененный заказ:** `0₽` (независимо от цены товара)
- **Заказ с возвратом:** `GREATEST(-43₽, 0) = 0₽` (если возврат превышает доходы)

#### 🔢 Этап 2: Расчет себестоимости (`total_selfcost`)

```sql
total_selfcost = CASE 
    WHEN is_valid_order THEN quantity * unit_selfcost
    ELSE 0 
END
```

**Примеры:**
- **Обычный заказ:** `2 × 125₽ = 250₽`
- **Отмененный заказ:** `0₽` (себестоимость не учитывается)

#### 🔢 Этап 3: Валовая прибыль (`gross_profit`)

```sql
gross_profit = total_product_revenue - total_selfcost
```

**Примеры:**
- **Обычный заказ:** `1,180₽ - 250₽ = 930₽`
- **Отмененный заказ:** `0₽ - 0₽ = 0₽`
- **Заказ с возвратом:** `0₽ - 250₽ = -250₽` (убыток)

#### 🔢 Этап 4: Операционная прибыль (`operating_profit`)

```sql
operating_profit = gross_profit - ABS(commission_amount) - ABS(total_services_cost)
```

**Разбивка расходов:**
- `commission_amount` = `-147.5₽` → `ABS(-147.5) = 147.5₽`
- `total_services_cost` = `service_pickup + service_delivery + service_fulfillment + service_return`
- Пример: `0₽ + 0₽ + 15₽ + 0₽ = 15₽`

**Примеры:**
- **Обычный заказ:** `930₽ - 147.5₽ - 15₽ = 767.5₽`
- **Отмененный заказ:** `0₽ - 0₽ - 0₽ = 0₽`
- **Заказ с возвратом:** `-250₽ - 147.5₽ - 15₽ = -412.5₽`

#### 🔢 Этап 5: Чистая прибыль (`net_profit`)

```sql
net_profit = operating_profit - transaction_bonus - transaction_promotion
```

**Разбивка маркетинговых расходов:**
- `transaction_bonus` = сумма бонусов покупателям (например, `7.29₽`)
- `transaction_promotion` = расходы на продвижение (например, `0₽`)

**Примеры:**
- **Обычный заказ:** `767.5₽ - 7.29₽ - 0₽ = 760.21₽`
- **Отмененный заказ:** `0₽ - 0₽ - 0₽ = 0₽`
- **Заказ с возвратом:** `-412.5₽ - 7.29₽ - 0₽ = -419.79₽`

#### 🔢 Этап 6: Рентабельность (`profit_margin_percent`)

```sql
profit_margin_percent = CASE
    WHEN total_product_revenue > 0 THEN 
        ROUND((net_profit / total_product_revenue * 100)::numeric, 2)
    ELSE 0
END
```

**Примеры:**
- **Обычный заказ:** `(760.21₽ / 1,180₽) × 100 = 64.42%`
- **Отмененный заказ:** `0%` (нет выручки)
- **Заказ с возвратом:** `0%` (выручка = 0)

#### 📊 Полный пример расчета:

**Обычный заказ (2 товара по 590₽):**
```
1. total_product_revenue = 2 × 590₽ = 1,180₽
2. total_selfcost = 2 × 125₽ = 250₽
3. gross_profit = 1,180₽ - 250₽ = 930₽
4. operating_profit = 930₽ - 147.5₽ - 15₽ = 767.5₽
5. net_profit = 767.5₽ - 7.29₽ - 0₽ = 760.21₽
6. profit_margin_percent = 760.21₽ / 1,180₽ × 100 = 64.42%
```

**Отмененный заказ:**
```
1. total_product_revenue = 0₽ (is_valid_order = false)
2. total_selfcost = 0₽
3. gross_profit = 0₽
4. operating_profit = 0₽
5. net_profit = 0₽
6. profit_margin_percent = 0%
```

**Заказ с возвратом (transaction_net_revenue = -43₽):**
```
1. total_product_revenue = GREATEST(-43₽, 0) = 0₽
2. total_selfcost = 1 × 125₽ = 125₽ (все равно учитывается)
3. gross_profit = 0₽ - 125₽ = -125₽
4. operating_profit = -125₽ - 147.5₽ - 15₽ = -287.5₽
5. net_profit = -287.5₽ - 7.29₽ - 0₽ = -294.79₽
6. profit_margin_percent = 0% (так как выручка = 0)
```

### 🔄 Группировка отправлений:
- **posting_number**: `"0100169990-0103-1"` - конкретное отправление
- **posting_number_short**: `"0100169990-0103"` - группа связанных отправлений
- Используется для агрегации транзакций по логическому заказу

### ⚠️ Важные особенности:
1. **Отрицательные значения** в комиссиях и услугах берутся по модулю (`ABS()`)
2. **Множественные транзакции** агрегируются по posting_number
3. **COALESCE** обеспечивает отсутствие NULL в расчетах
4. **Приведение типов** для корректной работы `ROUND()`

## 📝 Примеры использования

### Анализ прибыльности товаров:
```sql
SELECT offer_id, product_name,
       AVG(profit_margin_percent) as avg_margin,
       SUM(net_profit) as total_profit
FROM ozon_orders_economics_view 
WHERE is_valid_order = true
GROUP BY offer_id, product_name
ORDER BY total_profit DESC;
```

### Мониторинг возвратов:
```sql
SELECT DATE_TRUNC('month', order_date) as month,
       COUNT(*) FILTER (WHERE has_return_or_cancel) as returns_count,
       SUM(return_cancel_amount) as total_return_amount
FROM ozon_orders_economics_view 
GROUP BY month
ORDER BY month;
```

### Анализ операционной эффективности:
```sql
SELECT shop_name, region,
       AVG(operating_profit) as avg_operating_profit,
       AVG(total_services_cost) as avg_logistics_cost
FROM ozon_orders_economics_view 
WHERE is_valid_order = true
GROUP BY shop_name, region;
```

### Контроль качества данных:
```sql
-- Заказы без транзакций
SELECT COUNT(*) as orders_without_transactions
FROM ozon_orders_economics_view 
WHERE is_valid_order = true 
  AND operation_types_count IS NULL;

-- Проверка соответствия выручки
SELECT posting_number, 
       total_product_revenue,
       transaction_net_revenue,
       ABS(total_product_revenue - transaction_net_revenue) as difference
FROM ozon_orders_economics_view 
WHERE has_return_or_cancel = true
  AND ABS(total_product_revenue - transaction_net_revenue) > 1;
```