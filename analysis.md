# Pens and Pencils — аналитический разбор

## Описание проекта
В рамках проекта был проведён анализ работы онлайн-компании по продаже
канцелярских товаров на рынке США.  
Цель анализа — оценить эффективность продаж, структуру клиентов и логистику,
а также определить наиболее перспективный штат для открытия офлайн-магазина.

---

## 1. Анализ продаж

### 1.1 Динамика выручки

**Задача:**  
Оценить, как меняется выручка компании во времени и есть ли сезонность.

```sql
SELECT
    date_trunc('month', order_date)::date AS month,
    round(sum(price * quantity * (1 - discount))) AS revenue
FROM sql.store_delivery sd
JOIN sql.store_carts sc ON sd.order_id = sc.order_id
JOIN sql.store_products sp ON sc.product_id = sp.product_id
GROUP BY 1
ORDER BY 1;
````

**Вывод:**  
Выручка компании демонстрирует устойчивый рост.  
При этом наблюдается сезонность:

- падение выручки летом и зимой,
    
- наибольшие значения — осенью, что может быть связано с началом учебного года  
    и активностью корпоративных клиентов.
    

---

### 1.2 Выручка по категориям и подкатегориям

**Задача:**  
Определить, какие категории товаров приносят наибольшую выручку.

```sql
SELECT
    category,
    subcategory,
    round(sum(price * quantity * (1 - discount))) AS revenue
FROM sql.store_carts sc
JOIN sql.store_products sp ON sc.product_id = sp.product_id
GROUP BY 1, 2
ORDER BY 3 DESC;
```

**Вывод:**  
Наибольшую выручку приносят офисная мебель и оборудование.  
Особенно выделяются офисные кресла и крупная офисная техника.

---

### 1.3 Топ-25 товаров по выручке

**Задача:**  
Выявить товары, формирующие наибольшую долю выручки.

```sql
WITH total_revenue AS (
    SELECT sum(price * quantity * (1 - discount)) AS total
    FROM sql.store_carts sc
    JOIN sql.store_products sp ON sc.product_id = sp.product_id
)
SELECT
    product_nm,
    round(sum(price * quantity * (1 - discount)), 2) AS revenue,
    sum(quantity) AS quantity,
    round(
        sum(price * quantity * (1 - discount)) / avg(total) * 100, 2
    ) AS percent_from_total
FROM sql.store_carts sc
JOIN sql.store_products sp ON sc.product_id = sp.product_id
JOIN total_revenue ON true
GROUP BY 1
ORDER BY 2 DESC
LIMIT 25;
```

**Вывод:**  
Лидерами по выручке являются:

- копировальная техника Canon,
    
- офисные кресла премиум-класса,
    
- оборудование для 3D-печати.
    

Высокая доля оборудования для 3D-печати указывает на потенциал развития  
ассортимента под корпоративных клиентов.

---

## 2. Анализ клиентов

### 2.1 B2B и B2C клиенты

**Задача:**  
Сравнить количество клиентов и выручку по сегментам.

```sql
SELECT
    sc2.category,
    count(DISTINCT sc2.cust_id) AS cust_cnt,
    round(sum(price * quantity * (1 - discount))) AS revenue
FROM sql.store_delivery sd
JOIN sql.store_carts sc ON sd.order_id = sc.order_id
JOIN sql.store_products sp ON sc.product_id = sp.product_id
JOIN sql.store_customers sc2 ON sc2.cust_id = sd.cust_id
GROUP BY 1
ORDER BY 3 DESC;
```

**Вывод:**  
Корпоративные клиенты (B2B):

- составляют основную часть клиентской базы,
    
- формируют значительно большую долю выручки по сравнению с B2C.
    

---

### 2.2 Привлечение новых корпоративных клиентов

**Задача:**  
Оценить динамику привлечения новых B2B-клиентов.

```sql
WITH first_order AS (
    SELECT
        sd.cust_id,
        min(date_trunc('month', order_date))::date AS first_date
    FROM sql.store_delivery sd
    JOIN sql.store_customers sc ON sd.cust_id = sc.cust_id
    WHERE sc.category = 'Corporate'
    GROUP BY 1
)
SELECT
    first_date,
    count(cust_id) AS new_customers
FROM first_order
GROUP BY 1
ORDER BY 1;
```

**Вывод:**  
После 2018 года количество новых корпоративных клиентов резко снизилось.  
Рост выручки обеспечивается в основном за счёт существующих клиентов,  
что может указывать на проблемы в привлечении новых B2B-партнёров.

---

### 2.3 Портрет корпоративного клиента

**Задача:**  
Сформировать усреднённый профиль корпоративного клиента.

```sql
WITH orders AS (
    SELECT
        d.cust_id,
        d.order_id,
        count(DISTINCT c.product_id) AS product_qty,
        sum(c.quantity * p.price * (1 - c.discount)) AS order_amt
    FROM sql.store_carts c
    JOIN sql.store_products p ON p.product_id = c.product_id
    JOIN sql.store_delivery d ON d.order_id = c.order_id
    JOIN sql.store_customers sc ON d.cust_id = sc.cust_id
    WHERE sc.category = 'Corporate'
    GROUP BY 1, 2
),
cust_zip AS (
    SELECT
        sc.cust_id,
        count(DISTINCT d.zip_code) AS address_num
    FROM sql.store_customers sc
    JOIN sql.store_delivery d ON d.cust_id = sc.cust_id
    WHERE sc.category = 'Corporate'
    GROUP BY 1
)
SELECT
    round(avg(o.product_qty), 1) AS avg_product_qty,
    round(avg(o.order_amt), 1) AS avg_order_amount,
    round(avg(z.address_num), 1) AS avg_address_number
FROM orders o
CROSS JOIN cust_zip z;
```

**Вывод:**  
Типичный корпоративный клиент:

- средний чек ≈ 286 $
    
- около 2 различных товаров в заказе
    
- доставка осуществляется в среднем в 6 офисов
    

---

## 3. Анализ логистики

### 3.1 Эффективность доставки

**Задача:**  
Оценить долю заказов, доставленных вовремя, по типам доставки.

```sql
WITH schedule AS (
    SELECT 'Standard Class' AS ship_mode, 6 AS days_plan
    UNION ALL SELECT 'Second Class', 4
    UNION ALL SELECT 'First Class', 3
    UNION ALL SELECT 'Same Day', 0
),
total_orders AS (
    SELECT ship_mode, count(*) AS orders_cnt
    FROM sql.store_delivery
    GROUP BY 1
),
late_orders AS (
    SELECT sd.ship_mode, count(*) AS late_orders_cnt
    FROM sql.store_delivery sd
    JOIN schedule s
        ON sd.ship_mode = s.ship_mode
       AND sd.ship_date - sd.order_date > s.days_plan
    GROUP BY 1
)
SELECT
    t.ship_mode,
    t.orders_cnt,
    l.late_orders_cnt,
    round(
        (t.orders_cnt - l.late_orders_cnt) * 100.0 / t.orders_cnt, 2
    ) AS success_rate
FROM total_orders t
LEFT JOIN late_orders l ON t.ship_mode = l.ship_mode
ORDER BY success_rate;
```

**Вывод:**  
Наибольшая доля задержек наблюдается у доставки Second Class.  
Это потенциальная точка риска для удовлетворённости клиентов.

---

### 3.2 География доставок и выручки

**Задача:**  
Определить наиболее перспективный штат для открытия офлайн-магазина.

```sql
SELECT
    state,
    count(DISTINCT dlv.order_id) AS orders_cnt,
    sum(price * quantity * (1 - discount)) AS revenue
FROM sql.store_delivery dlv
JOIN sql.store_carts crt ON dlv.order_id = crt.order_id
JOIN sql.store_products prd ON prd.product_id = crt.product_id
GROUP BY state
ORDER BY orders_cnt DESC;
```

**Вывод:**  
Лидерами по количеству заказов и объёму выручки являются:

- Калифорния
    
- Нью-Йорк
    
- Техас
    

---

## Итоговые выводы

- Компания демонстрирует рост выручки, но сильно зависит от существующих B2B-клиентов
    
- Привлечение новых корпоративных клиентов требует дополнительного анализа
    
- Логистика, особенно доставка Second Class, нуждается в оптимизации
    
- Калифорния является наиболее логичным выбором для открытия первого офлайн-магазина
    
