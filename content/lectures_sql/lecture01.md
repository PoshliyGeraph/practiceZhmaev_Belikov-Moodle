## Лекция 1: Основы SQL. SELECT и фильтрация данных
*Теория: 30% / Практика: 70%*

1.1 Введение в SQL
Что такое SQL?
SQL (Structured Query Language) — это специализированный язык программирования, предназначенный для управления данными в реляционных базах данных.

Возможности SQL:

Создание и изменение структуры БД (DDL)

Чтение и изменение данных (DML)

Управление доступом (DCL)

Управление транзакциями (TCL)

Классификация команд SQL
ascii
┌─────────────────────────────────────────────────────────────┐
│                           SQL                               │
├──────────────┬──────────────┬──────────────┬───────────────┤
│     DDL      │     DML      │     DCL      │     TCL       │
│  Data        │  Data        │  Data        │  Transaction  │
│  Definition  │  Manipulation│  Control     │  Control      │
├──────────────┼──────────────┼──────────────┼───────────────┤
│ CREATE       │ SELECT       │ GRANT        │ BEGIN         │
│ ALTER        │ INSERT       │ REVOKE       │ COMMIT        │
│ DROP         │ UPDATE       │              │ ROLLBACK      │
│ TRUNCATE     │ DELETE       │              │ SAVEPOINT     │
│ RENAME       │ MERGE        │              │               │
└──────────────┴──────────────┴──────────────┴───────────────┘
1.2 Типы данных в SQL
Стандартные типы SQL
sql
-- Числовые типы
INT, INTEGER              -- целое число (обычно 4 байта)
SMALLINT                  -- малое целое (2 байта)
BIGINT                    -- большое целое (8 байт)
DECIMAL(p,s), NUMERIC     -- дробное с фиксированной точностью
FLOAT(p)                  -- число с плавающей точкой
REAL, DOUBLE PRECISION    -- числа с плавающей точкой

-- Строковые типы
CHAR(n)                   -- строка фиксированной длины (дополняется пробелами)
VARCHAR(n)                -- строка переменной длины (до n символов)
TEXT                      -- длинный текст (ограничение зависит от СУБД)

-- Дата и время
DATE                      -- дата (год-месяц-день)
TIME                      -- время (часы:минуты:секунды)
TIMESTAMP                 -- дата и время
TIMESTAMP WITH TIME ZONE  -- дата и время с часовым поясом
INTERVAL                  -- интервал времени

-- Логический тип
BOOLEAN                   -- TRUE, FALSE, NULL

-- Бинарные данные
BYTEA, BLOB               -- бинарные данные (изображения, файлы)

-- JSON (в современных СУБД)
JSON, JSONB               -- данные в формате JSON
Пример создания таблицы с разными типами
sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) CHECK (price >= 0),
    in_stock BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    attributes JSONB,
    CONSTRAINT positive_price CHECK (price >= 0)
);
1.3 Базовая структура SELECT
Простейший SELECT
sql
-- Выбрать все колонки из таблицы
SELECT * FROM employees;

-- Выбрать конкретные колонки
SELECT first_name, last_name, salary FROM employees;

-- Выбрать с вычисляемым полем
SELECT 
    first_name,
    last_name,
    salary * 12 AS annual_salary
FROM employees;

-- Выбрать с константой
SELECT 
    'Сотрудник' as type,
    first_name || ' ' || last_name AS full_name
FROM employees;
Псевдонимы (ALIAS)
sql
-- Псевдоним для колонки (через AS или без)
SELECT 
    first_name AS name,
    last_name AS surname,
    salary AS sal
FROM employees;

-- Можно без AS (но с AS читаемее)
SELECT first_name name FROM employees;

-- Псевдоним для таблицы
SELECT e.first_name, e.last_name
FROM employees e;
1.4 Фильтрация с WHERE
Операторы сравнения
sql
-- Равенство
SELECT * FROM products WHERE category = 'Electronics';

-- Неравенство
SELECT * FROM products WHERE category != 'Electronics';
SELECT * FROM products WHERE category <> 'Electronics'; -- альтернатива

-- Больше/меньше
SELECT * FROM products WHERE price > 1000;
SELECT * FROM products WHERE price >= 1000;
SELECT * FROM products WHERE price < 1000;
SELECT * FROM products WHERE price <= 1000;
Логические операторы
sql
-- AND (все условия должны быть истинны)
SELECT * FROM products 
WHERE category = 'Electronics' 
  AND price < 50000 
  AND in_stock = TRUE;

-- OR (хотя бы одно условие истинно)
SELECT * FROM products 
WHERE category = 'Electronics' 
   OR category = 'Computers';

-- NOT (отрицание)
SELECT * FROM products 
WHERE NOT category = 'Electronics';

-- Комбинация с приоритетом (скобки важны!)
SELECT * FROM products 
WHERE (category = 'Electronics' OR category = 'Computers')
  AND price < 100000;
Специальные операторы WHERE
BETWEEN — диапазон значений:

sql
SELECT * FROM products 
WHERE price BETWEEN 1000 AND 5000;
-- эквивалентно: price >= 1000 AND price <= 5000

SELECT * FROM orders 
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';
IN — принадлежность множеству:

sql
SELECT * FROM products 
WHERE category IN ('Electronics', 'Computers', 'Phones');
-- эквивалентно длинной цепочке OR

SELECT * FROM employees 
WHERE department_id IN (10, 20, 30);
LIKE — шаблон строки:

sql
-- % — любая последовательность символов (включая пустую)
SELECT * FROM products 
WHERE name LIKE 'Ноут%';        -- начинается с "Ноут"
WHERE name LIKE '%бук';         -- заканчивается на "бук"
WHERE name LIKE '%телефон%';     -- содержит "телефон"

-- _ — ровно один символ
SELECT * FROM employees 
WHERE phone LIKE '+7 ___ ___ __ __';  -- шаблон российского номера

-- Экранирование спецсимволов
SELECT * FROM files 
WHERE name LIKE '%\_secret%' ESCAPE '\';  -- поиск "_secret"
IS NULL / IS NOT NULL — проверка на NULL:

sql
-- Найти записи с пропущенными значениями
SELECT * FROM employees 
WHERE phone IS NULL;

-- Найти заполненные записи
SELECT * FROM employees 
WHERE email IS NOT NULL;
1.5 Сортировка ORDER BY
Простая сортировка
sql
-- По возрастанию (по умолчанию)
SELECT name, price FROM products 
ORDER BY price;

-- По убыванию
SELECT name, price FROM products 
ORDER BY price DESC;

-- По нескольким полям
SELECT name, category, price FROM products 
ORDER BY category ASC, price DESC;
Сортировка с выражениями
sql
-- Сортировка по вычисляемому полю
SELECT 
    first_name,
    last_name,
    salary * 12 as annual
FROM employees
ORDER BY annual DESC;

-- Сортировка с CASE (особый порядок)
SELECT name, status FROM orders
ORDER BY 
    CASE status
        WHEN 'pending' THEN 1
        WHEN 'processing' THEN 2
        WHEN 'shipped' THEN 3
        WHEN 'delivered' THEN 4
        ELSE 5
    END;
1.6 Ограничение выборки LIMIT и OFFSET
sql
-- Первые 10 записей
SELECT * FROM products 
ORDER BY price DESC 
LIMIT 10;

-- Следующие 10 записей (пагинация)
SELECT * FROM products 
ORDER BY price DESC 
LIMIT 10 OFFSET 10;

-- Топ-3 самых дорогих товара
SELECT name, price FROM products 
ORDER BY price DESC 
LIMIT 3;
1.7 Удаление дубликатов DISTINCT
sql
-- Уникальные категории
SELECT DISTINCT category FROM products;

-- Уникальные комбинации
SELECT DISTINCT category, supplier_id FROM products;

-- Количество уникальных категорий
SELECT COUNT(DISTINCT category) FROM products;
