## Лекция 3: Соединения таблиц (JOIN) и подзапросы
*Теория: 40% / Практика: 60%*

3.1 Нормализация и связи между таблицами
Почему данные нужно хранить в разных таблицах?
Проблема одной таблицы:

sql
-- Плохая структура: всё в одной таблице
CREATE TABLE bad_orders (
    order_id INT,
    customer_name VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    product_name VARCHAR(100),
    product_price DECIMAL,
    quantity INT,
    order_date DATE
);
-- Избыточность: данные клиента повторяются в каждом заказе
-- Аномалии обновления: нужно менять телефон во многих местах
Хорошая структура (нормализованная):

sql
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    phone VARCHAR(20),
    address TEXT
);

CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    order_date DATE,
    status VARCHAR(20)
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    product_id INT REFERENCES products(id),
    quantity INT,
    price DECIMAL(10,2)  -- цена на момент заказа
);
3.2 Внутреннее соединение INNER JOIN
Синтаксис INNER JOIN
sql
-- INNER JOIN возвращает только строки, где есть совпадения в обеих таблицах
SELECT 
    orders.id as order_id,
    customers.name as customer_name,
    orders.order_date,
    orders.status
FROM orders
INNER JOIN customers ON orders.customer_id = customers.id;
Визуализация INNER JOIN:

ascii
Таблица A (orders):          Таблица B (customers):
┌────┬────────────┐         ┌────┬──────────────┐
│ id │ customer_id│         │ id │ name         │
├────┼────────────┤         ├────┼──────────────┤
│ 101│ 1          │         │ 1  │ Иван Петров  │
│ 102│ 2          │         │ 2  │ Мария Иванова│
│ 103│ 1          │         │ 3  │ Петр Сидоров │
│ 104│ 3          │         └────┴──────────────┘
└────┴────────────┘

Результат INNER JOIN:
┌─────────┬──────────────┬────────────┐
│ order_id│ customer_name │ status     │
├─────────┼──────────────┼────────────┤
│ 101     │ Иван Петров   │ delivered  │
│ 102     │ Мария Иванова │ processing │
│ 103     │ Иван Петров   │ shipped    │
│ 104     │ Петр Сидоров  │ pending    │
└─────────┴──────────────┴────────────┘
Примеры INNER JOIN
sql
-- Соединение двух таблиц
SELECT 
    o.id as order_id,
    c.name as customer,
    o.order_date,
    o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.total_amount > 10000
ORDER BY o.total_amount DESC;

-- Соединение трех таблиц
SELECT 
    o.id as order_id,
    c.name as customer,
    p.name as product,
    oi.quantity,
    oi.price
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.id = 101;

-- JOIN с агрегацией
SELECT 
    c.name,
    COUNT(DISTINCT o.id) as order_count,
    SUM(oi.quantity * oi.price) as total_spent
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
3.3 Внешние соединения LEFT/RIGHT/FULL JOIN
LEFT JOIN (LEFT OUTER JOIN)
sql
-- LEFT JOIN возвращает ВСЕ строки из левой таблицы + совпадения из правой
SELECT 
    c.name as customer,
    o.id as order_id,
    o.total_amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
ORDER BY c.name;
Визуализация LEFT JOIN:

ascii
Таблица customers (левая):    Таблица orders (правая):
┌────┬──────────────┐         ┌────┬────────────┬────────────┐
│ id │ name         │         │ id │ customer_id│ total      │
├────┼──────────────┤         ├────┼────────────┼────────────┤
│ 1  │ Иван Петров  │         │ 101│ 1          │ 15000      │
│ 2  │ Мария Иванова│         │ 102│ 1          │ 23000      │
│ 3  │ Петр Сидоров │         │ 103│ 2          │ 8500       │
│ 4  │ Ольга Смирнова│         └────┴────────────┴────────────┘
└────┴──────────────┘

Результат LEFT JOIN:
┌───────────────┬──────────┬────────────┐
│ customer      │ order_id │ total      │
├───────────────┼──────────┼────────────┤
│ Иван Петров   │ 101      │ 15000      │
│ Иван Петров   │ 102      │ 23000      │
│ Мария Иванова │ 103      │ 8500       │
│ Петр Сидоров  │ NULL     │ NULL       │  -- нет заказов
│ Ольга Смирнова│ NULL     │ NULL       │  -- нет заказов
└───────────────┴──────────┴────────────┘
RIGHT JOIN (RIGHT OUTER JOIN)
sql
-- RIGHT JOIN возвращает ВСЕ строки из правой таблицы + совпадения из левой
SELECT 
    c.name as customer,
    o.id as order_id,
    o.total_amount
FROM orders o
RIGHT JOIN customers c ON c.id = o.customer_id;  -- то же, что LEFT JOIN выше
FULL OUTER JOIN
sql
-- FULL JOIN возвращает все строки из обеих таблиц
SELECT 
    c.name as customer,
    o.id as order_id,
    o.total_amount
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id;