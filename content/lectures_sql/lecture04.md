## Лекция 4: Манипуляция данными (DML) и транзакции
*Теория: 40% / Практика: 60%*

4.1 Вставка данных INSERT
Базовая вставка
sql
-- Вставка одной строки (с указанием колонок)
INSERT INTO customers (id, name, email, city, joined_date)
VALUES (5, 'Анна Новикова', 'anna@email.com', 'Екатеринбург', '2024-03-15');

-- Вставка без указания колонок (все поля в порядке определения)
INSERT INTO customers
VALUES (6, 'Дмитрий Волков', 'dmitry@email.com', 'Новосибирск', '2024-03-20');

-- Вставка нескольких строк
INSERT INTO products (id, name, category, price) VALUES
    (7, 'Планшет', 'Электроника', 45000),
    (8, 'Чехол для планшета', 'Аксессуары', 1500),
    (9, 'Стилус', 'Аксессуары', 3000);
Вставка с SELECT
sql
-- Вставка результатов запроса
INSERT INTO products_archive (product_id, name, price, archived_date)
SELECT id, name, price, CURRENT_DATE
FROM products
WHERE price < 2000;

-- Копирование данных между таблицами
INSERT INTO premium_customers (customer_id, name, email, total_spent)
SELECT 
    c.id,
    c.name,
    c.email,
    SUM(oi.quantity * oi.price) as spent
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
GROUP BY c.id, c.name, c.email
HAVING SUM(oi.quantity * oi.price) > 100000;
INSERT ... ON CONFLICT (UPSERT)
sql
-- PostgreSQL: вставка или обновление при конфликте
INSERT INTO products (id, name, category, price)
VALUES (1, 'Ноутбук Pro', 'Электроника', 85000)
ON CONFLICT (id) DO UPDATE
SET 
    name = EXCLUDED.name,
    price = EXCLUDED.price,
    updated_at = CURRENT_TIMESTAMP;

-- MySQL: REPLACE
REPLACE INTO products (id, name, category, price)
VALUES (1, 'Ноутбук Pro', 'Электроника', 85000);
4.2 Обновление данных UPDATE
Базовое обновление
sql
-- Обновление одного поля
UPDATE products
SET price = 1600
WHERE id = 2;

-- Обновление нескольких полей
UPDATE customers
SET 
    email = 'ivan.new@email.com',
    city = 'СПб'
WHERE id = 1;

-- Обновление с вычислениями
UPDATE products
SET price = price * 1.1  -- повышение на 10%
WHERE category = 'Электроника';
UPDATE с JOIN
sql
-- Обновление на основе данных из другой таблицы
UPDATE products p
SET total_sold = (
    SELECT SUM(quantity)
    FROM order_items oi
    WHERE oi.product_id = p.id
)
WHERE EXISTS (
    SELECT 1
    FROM order_items oi
    WHERE oi.product_id = p.id
);

-- PostgreSQL: UPDATE с FROM
UPDATE products p
SET total_sold = sub.total
FROM (
    SELECT product_id, SUM(quantity) as total
    FROM order_items
    GROUP BY product_id
) sub
WHERE p.id = sub.product_id;

-- MySQL: UPDATE с JOIN
UPDATE products p
JOIN (
    SELECT product_id, SUM(quantity) as total
    FROM order_items
    GROUP BY product_id
) sub ON p.id = sub.product_id
SET p.total_sold = sub.total;
UPDATE с CASE
sql
-- Массовое обновление с условиями
UPDATE products
SET price = CASE
    WHEN category = 'Электроника' THEN price * 1.1
    WHEN category = 'Книги' THEN price * 1.05
    WHEN price < 1000 THEN price * 1.2
    ELSE price
END;
4.3 Удаление данных DELETE
Базовое удаление
sql
-- Удаление по условию
DELETE FROM products
WHERE id = 9;

-- Удаление нескольких записей
DELETE FROM order_items
WHERE order_id = 105;

-- Удаление всех записей (но таблица остается)
DELETE FROM products_archive;
DELETE с подзапросом
sql
-- Удаление товаров, которые никогда не продавались
DELETE FROM products
WHERE id NOT IN (
    SELECT DISTINCT product_id
    FROM order_items
);

-- Удаление старых заказов
DELETE FROM orders
WHERE order_date < '2023-01-01'
  AND status = 'delivered';
TRUNCATE vs DELETE
sql
-- DELETE: построчное удаление (медленно, но срабатывают триггеры)
DELETE FROM logs;

-- TRUNCATE: быстрое удаление всех строк (нельзя откатить в некоторых СУБД)
TRUNCATE TABLE temp_data;

-- TRUNCATE с RESTART IDENTITY (сброс счетчика)
TRUNCATE TABLE products RESTART IDENTITY CASCADE;
4.4 Транзакции
Основы транзакций
sql
-- BEGIN (или START TRANSACTION)
BEGIN;

-- Несколько операций
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

-- Если всё хорошо
COMMIT;

-- Если ошибка
ROLLBACK;
Точки сохранения (SAVEPOINT)
sql
BEGIN;

INSERT INTO orders (id, customer_id, order_date) VALUES (106, 1, CURRENT_DATE);

SAVEPOINT order_created;

INSERT INTO order_items (order_id, product_id, quantity, price) 
VALUES (106, 1, 1, 75000);

-- Ой, что-то пошло не так с этим товаром
ROLLBACK TO SAVEPOINT order_created;

-- Пробуем другой товар
INSERT INTO order_items (order_id, product_id, quantity, price) 
VALUES (106, 2, 3, 1500);

COMMIT;  -- заказ создан, но без первого товара
Пример банковского перевода с проверками
sql
BEGIN;

-- Проверяем достаточно ли средств
SELECT balance INTO @balance FROM accounts WHERE id = 1 FOR UPDATE;

IF @balance >= 1000 THEN
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
    INSERT INTO transactions (from_id, to_id, amount, status)
    VALUES (1, 2, 1000, 'completed');
    COMMIT;
ELSE
    ROLLBACK;
    RAISE EXCEPTION 'Недостаточно средств';
END IF;
4.5 Управление структурой (DDL)
CREATE TABLE
sql
-- Создание таблицы с ограничениями
CREATE TABLE employees (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    hire_date DATE NOT NULL DEFAULT CURRENT_DATE,
    salary DECIMAL(10,2) CHECK (salary > 0),
    department_id INT,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_email CHECK (email LIKE '%@%.%')
);

-- Создание таблицы с внешним ключом
CREATE TABLE employee_departments (
    employee_id INT REFERENCES employees(id) ON DELETE CASCADE,
    department_id INT REFERENCES departments(id) ON DELETE CASCADE,
    start_date DATE NOT NULL,
    end_date DATE,
    PRIMARY KEY (employee_id, department_id, start_date)
);
ALTER TABLE
sql
-- Добавление колонки
ALTER TABLE employees 
ADD COLUMN middle_name VARCHAR(50);

-- Добавление колонки с DEFAULT (без блокировки в некоторых СУБД)
ALTER TABLE employees 
ADD COLUMN bonus DECIMAL(10,2) DEFAULT 0;

-- Изменение типа колонки
ALTER TABLE employees 
ALTER COLUMN phone TYPE VARCHAR(30);

-- Установка NOT NULL
ALTER TABLE employees 
ALTER COLUMN phone SET NOT NULL;

-- Добавление ограничения
ALTER TABLE employees 
ADD CONSTRAINT salary_min CHECK (salary >= 16242);

-- Удаление ограничения
ALTER TABLE employees 
DROP CONSTRAINT salary_min;

-- Переименование колонки
ALTER TABLE employees 
RENAME COLUMN middle_name TO patronymic;

-- Удаление колонки
ALTER TABLE employees 
DROP COLUMN patronymic;
Индексы
sql
-- Создание индекса
CREATE INDEX idx_employees_email ON employees(email);

-- Уникальный индекс
CREATE UNIQUE INDEX idx_employees_phone ON employees(phone);

-- Составной индекс
CREATE INDEX idx_employees_dept_salary ON employees(department_id, salary DESC);

-- Частичный индекс
CREATE INDEX idx_employees_active ON employees(hire_date) 
WHERE active = true;

-- Удаление индекса
DROP INDEX idx_employees_email;
4.6 Представления (VIEW)
Создание представлений
sql
-- Простое представление
CREATE VIEW active_customers AS
SELECT id, name, email, city
FROM customers
WHERE active = true
WITH LOCAL CHECK OPTION;

-- Представление с агрегацией
CREATE VIEW customer_summary AS
SELECT 
    c.id,
    c.name,
    c.city,
    COUNT(o.id) as order_count,
    COALESCE(SUM(oi.quantity * oi.price), 0) as total_spent,
    MAX(o.order_date) as last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
LEFT JOIN order_items oi ON o.id = oi.order_id
GROUP BY c.id, c.name, c.city;

-- Обновляемое представление (если простое)
CREATE VIEW expensive_products AS
SELECT id, name, price
FROM products
WHERE price > 10000
WITH CHECK OPTION;  -- запрещает вставку дешевых товаров через представление
Использование представлений
sql
-- Запросы к представлению как к обычной таблице
SELECT * FROM customer_summary
WHERE total_spent > 100000
ORDER BY total_spent DESC;

-- Обновление через представление
UPDATE expensive_products
SET price = 12000
WHERE id = 1;  -- работает, если представление простое

-- Вставка через представление
INSERT INTO expensive_products (id, name, price)
VALUES (10, 'Дорогой товар', 50000);  -- работает, цена > 10000

-- Попытка вставить дешевый товар (ошибка из-за WITH CHECK OPTION)
INSERT INTO expensive_products (id, name, price)
VALUES (11, 'Дешевый товар', 500);  -- ERROR
Материализованные представления
sql
-- PostgreSQL: материализованное представление (хранит данные физически)
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
    sale_date,
    COUNT(*) as transactions,
    SUM(amount) as total_sales,
    AVG(amount) as avg_sale
FROM sales
GROUP BY sale_date
WITH DATA;

-- Обновление материализованного представления
REFRESH MATERIALIZED VIEW daily_sales_summary;
4.7 Временные таблицы
sql
-- Временная таблица (существует только в рамках сессии)
CREATE TEMP TABLE temp_order_totals AS
SELECT 
    order_id,
    SUM(quantity * price) as total
FROM order_items
GROUP BY order_id;

-- Использование временной таблицы
SELECT 
    o.id,
    o.customer_id,
    t.total
FROM orders o
JOIN temp_order_totals t ON o.id = t.order_id;