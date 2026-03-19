# Лекция 2: Математика запросов. SELECT и операции реляционной алгебры
*Теория: 30% / Практика: 70%*

2.1 Реляционная алгебра (по Кодду)
Реляционная алгебра — это набор операций над отношениями, результатом которых является новое отношение.

Базовые операции
Операция	Обозначение	Смысл	SQL-аналог
Выборка (Selection)	σ<sub>условие</sub>(R)	Выбрать строки по условию	WHERE
Проекция (Projection)	π<sub>атрибуты</sub>(R)	Выбрать столбцы	SELECT
Объединение (Union)	R ∪ S	Все кортежи из R и S	UNION
Разность (Difference)	R - S	Кортежи из R, которых нет в S	EXCEPT
Декартово произведение	R × S	Комбинация всех строк	CROSS JOIN
Пересечение (Intersection)	R ∩ S	Кортежи, присутствующие в обоих	INTERSECT
Визуализация операций
ОПЕРАЦИЯ ВЫБОРКИ (σ):

    Исходная таблица R:                      σ salary > 60000 (R):
    ┌──────┬─────────┬────────┐              ┌──────┬─────────┬────────┐
    │ id   │ name    │ salary │              │ id   │ name    │ salary │
    ├──────┼─────────┼────────┤       →      ├──────┼─────────┼────────┤
    │ 1    │ Иван    │ 50000  │              │ 2    │ Мария   │ 70000  │
    │ 2    │ Мария   │ 70000  │              │ 4    │ Петр    │ 90000  │
    │ 3    │ Ольга   │ 55000  │              └──────┴─────────┴────────┘
    │ 4    │ Петр    │ 90000  │              
    └──────┴─────────┴────────┘              

ОПЕРАЦИЯ ПРОЕКЦИИ (π):

    Исходная таблица R:                      π name, salary (R):
    ┌──────┬─────────┬────────┐              ┌─────────┬────────┐
    │ id   │ name    │ salary │              │ name    │ salary │
    ├──────┼─────────┼────────┤       →      ├─────────┼────────┤
    │ 1    │ Иван    │ 50000  │              │ Иван    │ 50000  │
    │ 2    │ Мария   │ 70000  │              │ Мария   │ 70000  │
    │ 3    │ Ольга   │ 55000  │              │ Ольга   │ 55000  │
    │ 4    │ Петр    │ 90000  │              │ Петр    │ 90000  │
    └──────┴─────────┴────────┘              └─────────┴────────┘
2.2 Глубокое погружение в SELECT
Полный синтаксис SELECT
SELECT [DISTINCT] список_столбцов_или_выражений
FROM источник_данных
[WHERE условия_фильтрации]
[GROUP BY столбцы_для_группировки]
[HAVING условия_для_групп]
[ORDER BY столбцы_для_сортировки [ASC|DESC]]
[LIMIT количество_строк]
[OFFSET смещение];
Фильтрация WHERE
Операторы сравнения:

-- Равенство и неравенство
SELECT * FROM products WHERE price = 1000;
SELECT * FROM products WHERE price != 1000;
SELECT * FROM products WHERE price <> 1000;  -- альтернатива !=

-- Диапазоны
SELECT * FROM products WHERE price BETWEEN 1000 AND 5000;
SELECT * FROM products WHERE price >= 1000 AND price <= 5000;  -- то же самое

-- Принадлежность множеству
SELECT * FROM products WHERE category IN ('Электроника', 'Софт');

-- Работа с текстом
SELECT * FROM products WHERE name LIKE 'Ноут%';     -- начинается с "Ноут"
SELECT * FROM products WHERE name LIKE '%book%';    -- содержит "book"
SELECT * FROM products WHERE name LIKE '_апка';     -- ровно один символ перед "апка"

-- Проверка на NULL
SELECT * FROM products WHERE description IS NULL;
SELECT * FROM products WHERE description IS NOT NULL;
Сортировка ORDER BY
sql
-- Простая сортировка
SELECT name, price FROM products ORDER BY price;                    -- по возрастанию (ASC)
SELECT name, price FROM products ORDER BY price DESC;               -- по убыванию

-- Сортировка по нескольким полям
SELECT name, category, price FROM products 
ORDER BY category ASC, price DESC;  -- сначала по категории, внутри категории по убыванию цены
Уникальные значения DISTINCT
sql
-- Только уникальные категории
SELECT DISTINCT category FROM products;

-- Уникальные комбинации
SELECT DISTINCT category, supplier FROM products;
2.3 Агрегация данных
Агрегатные функции
-- COUNT: количество строк
SELECT COUNT(*) FROM products;                    -- всего товаров
SELECT COUNT(DISTINCT category) FROM products;     -- количество уникальных категорий

-- SUM: сумма
SELECT SUM(price) FROM products;                   -- общая стоимость всех товаров
SELECT SUM(quantity * price) FROM order_items;     -- сумма заказа

-- AVG: среднее
SELECT AVG(price) FROM products;                    -- средняя цена

-- MIN/MAX: минимум и максимум
SELECT MIN(price), MAX(price) FROM products;        -- мин и макс цена
Группировка GROUP BY

-- Средняя цена по каждой категории
SELECT 
    category,
    COUNT(*) as product_count,
    AVG(price) as avg_price,
    MIN(price) as min_price,
    MAX(price) as max_price
FROM products
GROUP BY category;
Визуализация работы GROUP BY:


Исходная таблица:
┌────────────┬───────────┐
│ category   │ price     │
├────────────┼───────────┤
│ Электроника│ 1000      │
│ Электроника│ 2000      │
│ Книги      │ 500       │
│ Книги      │ 700       │
│ Книги      │ 800       │
│ Одежда     │ 1500      │
└────────────┴───────────┘

После GROUP BY category:
┌────────────┬───────────┬────────────┐
│ category   │ COUNT(*)  │ AVG(price) │
├────────────┼───────────┼────────────┤
│ Электроника│ 2         │ 1500       │
│ Книги      │ 3         │ 666.67     │
│ Одежда     │ 1         │ 1500       │
└────────────┴───────────┴────────────┘
Фильтрация групп HAVING

-- Только категории с количеством товаров > 1
SELECT 
    category,
    COUNT(*) as product_count,
    AVG(price) as avg_price
FROM products
GROUP BY category
HAVING COUNT(*) > 1;
Важное различие:

WHERE — фильтрует строки ДО группировки

HAVING — фильтрует группы ПОСЛЕ группировки

2.4 Практикум: аналитические запросы
Учебная база данных "Интернет-магазин"

-- Создаем таблицы
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    city VARCHAR(50),
    joined_date DATE
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    order_date DATE,
    total_amount DECIMAL(10,2)
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    product_name VARCHAR(100),
    quantity INT,
    price DECIMAL(10,2)
);

-- Заполняем данными (упрощенно)
INSERT INTO customers VALUES 
(1, 'Иванов Иван', 'Москва', '2023-01-15'),
(2, 'Петрова Мария', 'СПб', '2023-02-20'),
(3, 'Сидоров Алексей', 'Москва', '2023-03-10');

INSERT INTO orders VALUES
(101, 1, '2024-01-10', 15000),
(102, 1, '2024-02-15', 23000),
(103, 2, '2024-01-20', 8500),
(104, 3, '2024-03-05', 12700);
