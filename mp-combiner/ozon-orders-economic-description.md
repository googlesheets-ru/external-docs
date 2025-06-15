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

| Поле | Формула | Назначение | Пример |
|------|---------|------------|--------|
| **total_product_revenue** | `quantity * product_price` | Общая выручка от товара | 590.00 |
| **total_selfcost** | `quantity * unit_selfcost` | Общая себестоимость товара | 125.00 |

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
| **transaction_commission** | `ozon_fin_transaction` | Комиссионные расходы | SUM(WHERE operation_type_name LIKE '%комиссия%') |
| **transaction_logistics** | `ozon_fin_transaction` | Логистические расходы | SUM(WHERE operation_type_name LIKE '%доставка%') |
| **transaction_bonus** | `ozon_fin_transaction` | Расходы на бонусы | SUM(WHERE operation_type_name LIKE '%бонус%') |
| **transaction_promotion** | `ozon_fin_transaction` | Расходы на продвижение | SUM(WHERE operation_type_name LIKE '%продвижени%') |
| **transaction_total_expenses** | `ozon_fin_transaction` | Все расходы по транзакциям | SUM(amount < 0) |

## 📈 Экономические показатели (КПЭ)

| Поле | Формула | Экономический смысл | Применение |
|------|---------|-------------------|------------|
| **gross_profit** | `выручка - себестоимость` | Валовая прибыль | Базовая рентабельность товара |
| **operating_profit** | `валовая_прибыль - комиссии - услуги` | Операционная прибыль | Прибыль после расходов маркетплейса |
| **net_profit** | `операционная_прибыль - бонусы - продвижение` | Чистая прибыль | Итоговая прибыльность заказа |
| **profit_margin_percent** | `(чистая_прибыль / выручка) * 100` | Рентабельность в % | Сравнение эффективности товаров |

## 🔍 Дополнительная аналитическая информация

| Поле | Источник | Назначение | Пример |
|------|----------|------------|--------|
| **operation_types_count** | `COUNT(DISTINCT operation_type)` | Количество типов операций по заказу | 5 |
| **operation_types** | `STRING_AGG(operation_type_name)` | Список всех типов операций | "Доставка; Комиссия; Бонусы" |
| **delivery_date_begin** | `posting_analytics_data.delivery_date_begin` | Начало периода доставки | 2024-01-12 |
| **delivery_date_end** | `posting_analytics_data.delivery_date_end` | Конец периода доставки | 2024-01-15 |

## 💡 Ключевые принципы расчетов

### 🎯 Приоритеты данных:
1. **Себестоимость**: МойСклад → posting_product → 0
2. **Финансы**: posting_fin_data_product → ozon_fin_transaction
3. **Услуги**: item_services (детально) → transaction_summary (агрегированно)

### 📊 Логика расчета прибыли:
```
Выручка (590₽)
├─ Себестоимость (-125₽)
└─ Валовая прибыль (465₽)
   ├─ Комиссия Озон (-147.5₽)
   ├─ Услуги логистики (-0₽)
   └─ Операционная прибыль (317.5₽)
      ├─ Бонусы (-3.15₽)
      ├─ Продвижение (-0₽)
      └─ Чистая прибыль (314.35₽)
         └─ Рентабельность: 53.3%
```

### 🔄 Группировка отправлений:
- **posting_number**: `"0100169990-0103-1"` - конкретное отправление
- **posting_number_short**: `"0100169990-0103"` - группа связанных отправлений
- Используется для агрегации данных по логическому заказу

### ⚠️ Важные особенности:
1. **Отрицательные значения** в комиссиях и услугах берутся по модулю (`ABS()`)
2. **Множественные транзакции** агрегируются по posting_number
3. **COALESCE** обеспечивает отсутствие NULL в расчетах
4. **Приведение типов** `::NUMERIC` для корректной работы `ROUND()`