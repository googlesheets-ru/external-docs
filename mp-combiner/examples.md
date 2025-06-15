# Примеры использования представления ozon_orders_economics_view

## 📖 Общие принципы работы с представлением

### 🎯 Ключевые поля для анализа:
- **posting_number_short** - группировка связанных отправлений
- **net_profit** - итоговая прибыльность заказа
- **profit_margin_percent** - рентабельность в процентах
- **order_status = 'delivered'** - фильтр для анализа доставленных заказов

### ⚠️ Важные особенности:
- Один заказ может содержать несколько товаров (строк)
- Данные по себестоимости могут быть неполными (NULL)
- Отрицательная прибыль указывает на убыточность

---

## 1️⃣ БАЗОВАЯ АНАЛИТИКА

### 1.1 Общая статистика по продажам
```sql
-- Получаем базовые показатели эффективности продаж Озон
SELECT 
    COUNT(DISTINCT posting_number_short) AS total_orders,        -- Общее количество заказов
    COUNT(*) AS total_items,                                     -- Общее количество товарных позиций
    COUNT(DISTINCT sku) AS unique_products,                      -- Количество уникальных товаров
    SUM(total_product_revenue) AS total_revenue,                 -- Общая выручка
    SUM(total_selfcost) AS total_selfcost,                       -- Общая себестоимость
    SUM(net_profit) AS total_net_profit,                         -- Общая чистая прибыль
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin, -- Средняя рентабельность
    -- Процент прибыльных заказов
    ROUND(
        (COUNT(*) FILTER (WHERE net_profit > 0) * 100.0 / COUNT(*))::NUMERIC, 2
    ) AS profitable_orders_percent
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'  -- Только доставленные заказы
  AND order_date >= CURRENT_DATE - INTERVAL '30 days';  -- За последние 30 дней

-- 💡 Применение: Дашборд для топ-менеджмента, KPI отчетность
```

### 1.2 Быстрый обзор проблемных заказов
```sql
-- Выявляем заказы требующие внимания (убыточные или с низкой маржой)
SELECT 
    posting_number,
    product_name,
    total_product_revenue,
    net_profit,
    profit_margin_percent,
    CASE 
        WHEN net_profit < 0 THEN 'УБЫТОК'
        WHEN profit_margin_percent < 10 THEN 'НИЗКАЯ МАРЖА'
        ELSE 'НОРМА'
    END AS profit_status
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND (net_profit < 0 OR profit_margin_percent < 10)  -- Фильтр проблемных заказов
ORDER BY net_profit ASC  -- Сначала самые убыточные
LIMIT 20;

-- 💡 Применение: Ежедневный мониторинг, оперативные решения по ценообразованию
```

---

## 2️⃣ АНАЛИЗ ТОВАРОВ

### 2.1 ТОП-20 самых прибыльных товаров
```sql
-- Рейтинг товаров по абсолютной прибыли для принятия решений по ассортименту
SELECT 
    sku,
    product_name,
    COUNT(*) AS sales_count,                                     -- Количество продаж
    SUM(quantity) AS total_quantity,                             -- Общее количество проданных единиц
    ROUND(AVG(product_price)::NUMERIC, 2) AS avg_price,         -- Средняя цена
    ROUND(AVG(unit_selfcost)::NUMERIC, 2) AS avg_selfcost,      -- Средняя себестоимость
    SUM(total_product_revenue) AS total_revenue,                 -- Общая выручка
    SUM(net_profit) AS total_net_profit,                         -- Общая прибыль
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin, -- Средняя рентабельность
    -- Прибыль на единицу товара
    ROUND((SUM(net_profit) / SUM(quantity))::NUMERIC, 2) AS profit_per_unit
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '90 days'  -- За последние 3 месяца
GROUP BY sku, product_name
HAVING COUNT(*) >= 3  -- Только товары с минимум 3 продажами для статистической значимости
ORDER BY total_net_profit DESC
LIMIT 20;

-- 💡 Применение: Планирование закупок, фокус на прибыльном ассортименте
```

### 2.2 Анализ товаров с отрицательной рентабельностью
```sql
-- Выявляем товары которые стабильно приносят убытки
SELECT 
    sku,
    product_name,
    COUNT(*) AS sales_count,
    SUM(quantity) AS total_quantity,
    ROUND(AVG(product_price)::NUMERIC, 2) AS avg_price,
    ROUND(AVG(unit_selfcost)::NUMERIC, 2) AS avg_selfcost,
    SUM(net_profit) AS total_loss,                               -- Общий убыток
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin,
    -- Основные причины убыточности
    ROUND(AVG(ABS(commission_amount))::NUMERIC, 2) AS avg_commission,
    ROUND(AVG(total_services_cost)::NUMERIC, 2) AS avg_services_cost,
    -- Предлагаемая минимальная цена для безубыточности
    ROUND(
        (AVG(unit_selfcost) + AVG(ABS(commission_amount)) + AVG(total_services_cost) + AVG(transaction_bonus)) * 1.1
    ::NUMERIC, 2) AS suggested_min_price  -- +10% к безубыточности
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND net_profit < 0
  AND order_date >= CURRENT_DATE - INTERVAL '60 days'
GROUP BY sku, product_name
HAVING COUNT(*) >= 2  -- Товары с повторяющимися убытками
ORDER BY total_loss ASC  -- Самые убыточные первыми
LIMIT 15;

-- 💡 Применение: Коррекция цен, исключение из ассортимента, оптимизация логистики
```

### 2.3 ABC-анализ товаров по выручке
```sql
-- Классификация товаров по принципу Парето (80/20) для стратегического планирования
WITH product_revenue AS (
    SELECT 
        sku,
        product_name,
        SUM(total_product_revenue) AS total_revenue,
        COUNT(*) AS sales_count
    FROM ozon_orders_economics_view
    WHERE order_status = 'delivered'
      AND order_date >= CURRENT_DATE - INTERVAL '180 days'  -- За полгода
    GROUP BY sku, product_name
),
ranked_products AS (
    SELECT 
        *,
        SUM(total_revenue) OVER() AS total_company_revenue,
        SUM(total_revenue) OVER(ORDER BY total_revenue DESC) AS cumulative_revenue,
        ROW_NUMBER() OVER(ORDER BY total_revenue DESC) AS revenue_rank
    FROM product_revenue
)
SELECT 
    sku,
    product_name,
    revenue_rank,
    total_revenue,
    sales_count,
    ROUND((total_revenue * 100.0 / total_company_revenue)::NUMERIC, 2) AS revenue_percent,
    ROUND((cumulative_revenue * 100.0 / total_company_revenue)::NUMERIC, 2) AS cumulative_percent,
    -- ABC классификация
    CASE 
        WHEN cumulative_revenue <= total_company_revenue * 0.8 THEN 'A - ТОП (80% выручки)'
        WHEN cumulative_revenue <= total_company_revenue * 0.95 THEN 'B - Средние (15% выручки)'
        ELSE 'C - Остальные (5% выручки)'
    END AS abc_category
FROM ranked_products
ORDER BY revenue_rank
LIMIT 50;

-- 💡 Применение: Стратегия ассортимента, распределение ресурсов на маркетинг
```

---

## 3️⃣ АНАЛИЗ ПО МАГАЗИНАМ

### 3.1 Сравнительная эффективность магазинов
```sql
-- Сравнение показателей эффективности между магазинами
SELECT 
    shop_name,
    COUNT(DISTINCT posting_number_short) AS orders_count,
    COUNT(*) AS items_count,
    COUNT(DISTINCT sku) AS unique_products,
    SUM(total_product_revenue) AS total_revenue,
    SUM(total_selfcost) AS total_selfcost,
    SUM(net_profit) AS total_net_profit,
    -- Средние показатели
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin,
    ROUND(AVG(total_product_revenue)::NUMERIC, 2) AS avg_order_value,
    ROUND((SUM(net_profit) / COUNT(DISTINCT posting_number_short))::NUMERIC, 2) AS profit_per_order,
    -- Процент прибыльных позиций
    ROUND(
        (COUNT(*) FILTER (WHERE net_profit > 0) * 100.0 / COUNT(*))::NUMERIC, 2
    ) AS profitable_items_percent,
    -- Эффективность работы
    CASE 
        WHEN AVG(profit_margin_percent) >= 25 THEN 'ОТЛИЧНО'
        WHEN AVG(profit_margin_percent) >= 15 THEN 'ХОРОШО'
        WHEN AVG(profit_margin_percent) >= 5 THEN 'УДОВЛЕТВОРИТЕЛЬНО'
        ELSE 'ТРЕБУЕТ ВНИМАНИЯ'
    END AS performance_rating
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY shop_name
ORDER BY total_net_profit DESC;

-- 💡 Применение: Оценка эффективности торговых точек, бенчмаркинг
```

---

## 4️⃣ ВРЕМЕННАЯ АНАЛИТИКА

### 4.1 Динамика продаж по месяцам
```sql
-- Анализ сезонности и трендов продаж
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    COUNT(DISTINCT posting_number_short) AS orders_count,
    COUNT(*) AS items_count,
    SUM(total_product_revenue) AS revenue,
    SUM(net_profit) AS profit,
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin,
    -- Сравнение с предыдущим месяцем
    LAG(SUM(total_product_revenue)) OVER(ORDER BY DATE_TRUNC('month', order_date)) AS prev_month_revenue,
    ROUND(
        ((SUM(total_product_revenue) - LAG(SUM(total_product_revenue)) OVER(ORDER BY DATE_TRUNC('month', order_date))) 
         * 100.0 / NULLIF(LAG(SUM(total_product_revenue)) OVER(ORDER BY DATE_TRUNC('month', order_date)), 0))::NUMERIC, 2
    ) AS revenue_growth_percent
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '12 months'  -- За последний год
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month DESC;

-- 💡 Применение: Планирование бюджета, прогнозирование, анализ сезонности
```

### 4.2 Анализ выходных vs будние дни
```sql
-- Сравнение эффективности продаж в разные дни недели
SELECT 
    CASE 
        WHEN EXTRACT(dow FROM order_date) IN (0, 6) THEN 'Выходные'
        ELSE 'Будние дни'
    END AS day_type,
    TO_CHAR(order_date, 'Day') AS day_name,
    EXTRACT(dow FROM order_date) AS day_number,
    COUNT(DISTINCT posting_number_short) AS orders_count,
    ROUND(AVG(total_product_revenue)::NUMERIC, 2) AS avg_order_value,
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin,
    SUM(net_profit) AS total_profit
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '60 days'
GROUP BY 
    CASE WHEN EXTRACT(dow FROM order_date) IN (0, 6) THEN 'Выходные' ELSE 'Будние дни' END,
    TO_CHAR(order_date, 'Day'),
    EXTRACT(dow FROM order_date)
ORDER BY day_number;

-- 💡 Применение: Оптимизация рекламных кампаний, планирование акций
```

---

## 5️⃣ ГЕОГРАФИЧЕСКИЙ АНАЛИЗ

### 5.1 Прибыльность по регионам
```sql
-- Анализ эффективности продаж по географическим регионам
SELECT 
    COALESCE(region, 'Не указан') AS region,
    COUNT(DISTINCT posting_number_short) AS orders_count,
    COUNT(*) AS items_count,
    SUM(total_product_revenue) AS revenue,
    SUM(net_profit) AS profit,
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin,
    -- Средний чек по региону
    ROUND((SUM(total_product_revenue) / COUNT(DISTINCT posting_number_short))::NUMERIC, 2) AS avg_order_value,
    -- Логистические затраты по региону
    ROUND(AVG(total_services_cost)::NUMERIC, 2) AS avg_logistics_cost,
    -- Процент от общих продаж
    ROUND(
        (COUNT(*) * 100.0 / SUM(COUNT(*)) OVER())::NUMERIC, 2
    ) AS sales_share_percent
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY region
HAVING COUNT(*) >= 5  -- Регионы с минимум 5 заказами
ORDER BY profit DESC
LIMIT 20;

-- 💡 Применение: Региональная стратегия, оптимизация логистики, таргетинг рекламы
```

---

## 6️⃣ ФИНАНСОВАЯ АНАЛИТИКА

### 6.1 Детализация расходов и доходов
```sql
-- Подробный анализ структуры доходов и расходов
SELECT 
    'ДОХОДЫ' AS category,
    'Выручка от продаж' AS item,
    SUM(total_product_revenue) AS amount,
    ROUND((SUM(total_product_revenue) * 100.0 / SUM(total_product_revenue))::NUMERIC, 2) AS percent_of_revenue
FROM ozon_orders_economics_view
WHERE order_status = 'delivered' AND order_date >= CURRENT_DATE - INTERVAL '30 days'
UNION ALL
SELECT 
    'РАСХОДЫ' AS category,
    'Себестоимость товаров' AS item,
    SUM(total_selfcost) AS amount,
    ROUND((SUM(total_selfcost) * 100.0 / SUM(total_product_revenue))::NUMERIC, 2) AS percent_of_revenue
FROM ozon_orders_economics_view
WHERE order_status = 'delivered' AND order_date >= CURRENT_DATE - INTERVAL '30 days'
UNION ALL
SELECT 
    'РАСХОДЫ' AS category,
    'Комиссии Озон' AS item,
    SUM(ABS(commission_amount)) AS amount,
    ROUND((SUM(ABS(commission_amount)) * 100.0 / SUM(total_product_revenue))::NUMERIC, 2) AS percent_of_revenue
FROM ozon_orders_economics_view
WHERE order_status = 'delivered' AND order_date >= CURRENT_DATE - INTERVAL '30 days'
UNION ALL
SELECT 
    'РАСХОДЫ' AS category,
    'Логистические услуги' AS item,
    SUM(total_services_cost) AS amount,
    ROUND((SUM(total_services_cost) * 100.0 / SUM(total_product_revenue))::NUMERIC, 2) AS percent_of_revenue
FROM ozon_orders_economics_view
WHERE order_status = 'delivered' AND order_date >= CURRENT_DATE - INTERVAL '30 days'
UNION ALL
SELECT 
    'РАСХОДЫ' AS category,
    'Бонусы и продвижение' AS item,
    SUM(transaction_bonus + transaction_promotion) AS amount,
    ROUND((SUM(transaction_bonus + transaction_promotion) * 100.0 / SUM(total_product_revenue))::NUMERIC, 2) AS percent_of_revenue
FROM ozon_orders_economics_view
WHERE order_status = 'delivered' AND order_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY category DESC, amount DESC;

-- 💡 Применение: Управленческая отчетность, оптимизация структуры расходов
```

### 6.2 Анализ влияния комиссий на прибыльность
```sql
-- Исследование зависимости прибыльности от размера комиссии Озон
SELECT 
    CASE 
        WHEN commission_percent <= 10 THEN '≤ 10%'
        WHEN commission_percent <= 15 THEN '11-15%'
        WHEN commission_percent <= 20 THEN '16-20%'
        WHEN commission_percent <= 25 THEN '21-25%'
        ELSE '> 25%'
    END AS commission_range,
    COUNT(*) AS items_count,
    ROUND(AVG(commission_percent)::NUMERIC, 2) AS avg_commission_percent,
    ROUND(AVG(product_price)::NUMERIC, 2) AS avg_price,
    ROUND(AVG(net_profit)::NUMERIC, 2) AS avg_net_profit,
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin,
    -- Процент прибыльных позиций в каждой группе комиссий
    ROUND(
        (COUNT(*) FILTER (WHERE net_profit > 0) * 100.0 / COUNT(*))::NUMERIC, 2
    ) AS profitable_items_percent
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND commission_percent IS NOT NULL
  AND order_date >= CURRENT_DATE - INTERVAL '60 days'
GROUP BY 
    CASE 
        WHEN commission_percent <= 10 THEN '≤ 10%'
        WHEN commission_percent <= 15 THEN '11-15%'
        WHEN commission_percent <= 20 THEN '16-20%'
        WHEN commission_percent <= 25 THEN '21-25%'
        ELSE '> 25%'
    END
ORDER BY AVG(commission_percent);

-- 💡 Применение: Стратегия выбора категорий товаров, переговоры с маркетплейсом
```

---

## 7️⃣ ОПЕРАТИВНАЯ АНАЛИТИКА

### 7.1 Ежедневный мониторинг ключевых показателей
```sql
-- Оперативная сводка для ежедневного контроля бизнеса
SELECT 
    order_date::date AS date,
    COUNT(DISTINCT posting_number_short) AS orders_count,
    COUNT(*) AS items_count,
    ROUND(SUM(total_product_revenue)::NUMERIC, 2) AS daily_revenue,
    ROUND(SUM(net_profit)::NUMERIC, 2) AS daily_profit,
    ROUND(AVG(profit_margin_percent)::NUMERIC, 2) AS avg_margin,
    -- Сравнение с предыдущим днем
    ROUND(
        (SUM(total_product_revenue) - LAG(SUM(total_product_revenue)) OVER(ORDER BY order_date::date))::NUMERIC, 2
    ) AS revenue_change,
    -- Предупреждения
    CASE 
        WHEN AVG(profit_margin_percent) < 10 THEN '⚠️ НИЗКАЯ МАРЖА'
        WHEN COUNT(*) = 0 THEN '🔴 НЕТ ПРОДАЖ'
        WHEN SUM(net_profit) < 0 THEN '📉 УБЫТКИ'
        ELSE '✅ НОРМА'
    END AS status_alert
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '14 days'
GROUP BY order_date::date
ORDER BY date DESC;

-- 💡 Применение: Ежедневные планерки, раннее выявление проблем
```

### 7.2 Поиск аномалий в транзакциях
```sql
-- Выявление заказов с нестандартными финансовыми операциями
SELECT 
    posting_number,
    product_name,
    total_product_revenue,
    net_profit,
    operation_types_count,
    operation_types,
    -- Флаги аномалий
    CASE 
        WHEN operation_types_count > 8 THEN 'Много операций'
        WHEN ABS(commission_amount) > total_product_revenue * 0.5 THEN 'Высокая комиссия'
        WHEN total_services_cost > total_product_revenue * 0.3 THEN 'Высокие услуги'
        WHEN net_profit < -total_product_revenue * 0.2 THEN 'Критический убыток'
        ELSE 'Стандартный'
    END AS anomaly_type
FROM ozon_orders_economics_view
WHERE order_status = 'delivered'
  AND order_date >= CURRENT_DATE - INTERVAL '7 days'
  AND (
    operation_types_count > 8 OR
    ABS(commission_amount) > total_product_revenue * 0.5 OR
    total_services_cost > total_product_revenue * 0.3 OR
    net_profit < -total_product_revenue * 0.2
  )
ORDER BY order_date DESC
LIMIT 30;

-- 💡 Применение: Выявление ошибок учета, анализ нестандартных ситуаций
```

---

## 8️⃣ СВОДНЫЕ ОТЧЕТЫ ДЛЯ МЕНЕДЖМЕНТА

### 8.1 Исполнительная сводка (Executive Summary)
```sql
-- Краткий отчет с ключевыми метриками для руководства
WITH current_period AS (
    SELECT 
        COUNT(DISTINCT posting_number_short) AS orders,
        SUM(total_product_revenue) AS revenue,
        SUM(net_profit) AS profit,
        AVG(profit_margin_percent) AS margin
    FROM ozon_orders_economics_view
    WHERE order_status = 'delivered' 
      AND order_date >= CURRENT_DATE - INTERVAL '30 days'
),
previous_period AS (
    SELECT 
        COUNT(DISTINCT posting_number_short) AS orders,
        SUM(total_product_revenue) AS revenue,
        SUM(net_profit) AS profit,
        AVG(profit_margin_percent) AS margin
    FROM ozon_orders_economics_view
    WHERE order_status = 'delivered' 
      AND order_date >= CURRENT_DATE - INTERVAL '60 days'
      AND order_date < CURRENT_DATE - INTERVAL '30 days'
)
SELECT 
    '📊 КЛЮЧЕВЫЕ ПОКАЗАТЕЛИ (30 дней)' AS metric_group,
    '' AS metric_name,
    NULL AS current_value,
    NULL AS previous_value,
    NULL AS change_percent
UNION ALL
SELECT 
    '├── Заказы' AS metric_group,
    'Количество заказов' AS metric_name,
    cp.orders AS current_value,
    pp.orders AS previous_value,
    ROUND(((cp.orders - pp.orders) * 100.0 / NULLIF(pp.orders, 0))::NUMERIC, 1) AS change_percent
FROM current_period cp, previous_period pp
UNION ALL
SELECT 
    '├── Выручка' AS metric_group,
    'Выручка, руб' AS metric_name,
    ROUND(cp.revenue::NUMERIC, 0) AS current_value,
    ROUND(pp.revenue::NUMERIC, 0) AS previous_value,
    ROUND(((cp.revenue - pp.revenue) * 100.0 / NULLIF(pp.revenue, 0))::NUMERIC, 1) AS change_percent
FROM current_period cp, previous_period pp
UNION ALL
SELECT 
    '├── Прибыль' AS metric_group,
    'Чистая прибыль, руб' AS metric_name,
    ROUND(cp.profit::NUMERIC, 0) AS current_value,
    ROUND(pp.profit::NUMERIC, 0) AS previous_value,
    ROUND(((cp.profit - pp.profit) * 100.0 / NULLIF(pp.profit, 0))::NUMERIC, 1) AS change_percent
FROM current_period cp, previous_period pp
UNION ALL
SELECT 
    '└── Рентабельность' AS metric_group,
    'Средняя маржа, %' AS metric_name,
    ROUND(cp.margin::NUMERIC, 1) AS current_value,
    ROUND(pp.margin::NUMERIC, 1) AS previous_value,
    ROUND((cp.margin - pp.margin)::NUMERIC, 1) AS change_percent
FROM current_period cp, previous_period pp;

-- 💡 Применение: Еженедельные отчеты руководству, презентации инвесторам
```

---

## 🔧 ПОЛЕЗНЫЕ ФУНКЦИОНАЛЬНЫЕ ЗАПРОСЫ

### Экспорт данных для внешних систем
```sql
-- Стандартизированный экспорт для интеграции с BI-системами
SELECT 
    posting_number AS shipment_id,
    order_date AS order_timestamp,
    sku AS product_sku,
    offer_id AS vendor_code,
    product_name,
    quantity,
    product_price AS unit_price,
    total_product_revenue AS revenue,
    net_profit,
    profit_margin_percent AS margin_pct,
    shop_name AS store,
    region AS delivery_region,
    order_status AS status
FROM ozon_orders_economics_view
WHERE order_date >= '2024-01-01'  -- Параметр для выборки периода
ORDER BY order_date DESC;

-- 💡 Применение: Интеграция с внешними системами аналитики
```

## 📚 Рекомендации по использованию

### ⚡ Оптимизация производительности:
1. **Всегда используйте фильтры по датам** для ограничения объема данных
2. **Добавляйте индексы** на часто используемые поля (order_date, sku, shop_name)
3. **Используйте LIMIT** для больших выборок данных

### 🎯 Практические советы:
1. **Фильтруйте по order_status = 'delivered'** для точной аналитики
2. **Исключайте NULL значения** в критических расчетах через COALESCE
3. **Группируйте по posting_number_short** для анализа заказов целиком
4. **Используйте HAVING** для фильтрации агрегированных данных

### 📈 Регулярность анализа:
- **Ежедневно**: Мониторинг ключевых показателей
- **Еженедельно**: Анализ товаров и трендов
- **Ежемесячно**: Глубокая аналитика, стратегические решения
- **Ежеквартально**: ABC-анализ, пересмотр ассортимента