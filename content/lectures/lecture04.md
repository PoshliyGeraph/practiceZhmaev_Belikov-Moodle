## Лекция 4: Транзакции, индексы и оптимизация
*Теория: 60% / Практика: 40%*

4.1 Транзакции и ACID
Что такое транзакция?
Транзакция — это логическая единица работы, которая должна быть выполнена полностью или не выполнена вовсе.

Классический пример (банковский перевод):

sql
-- Начало транзакции
BEGIN;

-- Списать 1000 со счета А
UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';

-- Зачислить 1000 на счет Б
UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';

-- Если всё хорошо — фиксируем
COMMIT;

-- Если ошибка — откатываем
ROLLBACK;
Свойства ACID
Свойство	Название	Описание	Пример нарушения
A	Atomicity (Атомарность)	Транзакция выполняется целиком или откатывается	Деньги списались, но не зачислились
C	Consistency (Согласованность)	Данные всегда соответствуют правилам	Баланс ушел в минус (нарушен CHECK)
I	Isolation (Изолированность)	Конкурирующие транзакции не мешают друг другу	Два клиента купили последний товар
D	Durability (Надежность)	Зафиксированные данные не теряются	Сбой питания после COMMIT
Проблемы параллельного доступа
1. Потерянное обновление (Lost Update)

ascii
Транзакция 1:                    Транзакция 2:
|                                |
| READ balance = 1000            |
|                                | READ balance = 1000
| balance = 1000 + 500 = 1500    |
|                                | balance = 1000 + 200 = 1200
| WRITE balance = 1500           |
|                                | WRITE balance = 1200
|                                |
Результат: 1200 (потеряли +500)
2. Грязное чтение (Dirty Read)

ascii
Транзакция 1:                    Транзакция 2:
|                                |
| UPDATE balance = 1500          |
| (не закоммичено)               |
|                                | READ balance = 1500
| ROLLBACK (откат до 1000)       |
|                                | (работает с несуществующими данными)
3. Неповторяющееся чтение (Non-repeatable Read)

ascii
Транзакция 1:                    Транзакция 2:
|                                |
| READ balance = 1000            |
|                                | UPDATE balance = 1500
|                                | COMMIT
| READ balance = 1500            |
| (данные изменились)            |
4. Фантомное чтение (Phantom Read)

ascii
Транзакция 1:                    Транзакция 2:
| SELECT COUNT(*) FROM accounts |
| WHERE balance > 1000 = 5       |
|                                | INSERT INTO accounts (balance=2000)
|                                | COMMIT
| SELECT COUNT(*) ... = 6        |
| (появилась новая строка)       |
Уровни изоляции (SQL Standard)
Уровень	Грязное чтение	Неповторяющееся чтение	Фантомы
READ UNCOMMITTED	Возможно	Возможно	Возможно
READ COMMITTED	Невозможно	Возможно	Возможно
REPEATABLE READ	Невозможно	Невозможно	Возможно
SERIALIZABLE	Невозможно	Невозможно	Невозможно
Установка уровня изоляции:

sql
-- Для текущей транзакции
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Для сессии
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
Блокировки
Пессимистическая блокировка:

sql
-- Явная блокировка строки
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Другие транзакции не могут изменить эту строку
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
Оптимистическая блокировка (через версионирование):

sql
-- Добавляем поле версии
ALTER TABLE accounts ADD COLUMN version INT DEFAULT 1;

-- При обновлении проверяем версию
UPDATE accounts 
SET balance = balance - 100, version = version + 1
WHERE id = 1 AND version = 5;
-- Если версия изменилась — обновление не сработает
4.2 Индексы
Что такое индекс?
Индекс — это структура данных, ускоряющая поиск строк по определенным колонкам.

Аналогия:

📚 Без индекса: ищем слово в книге, читая каждую страницу (полное сканирование)

🔍 С индексом: смотрим в алфавитный указатель и сразу знаем страницу

Типы индексов
B-Tree (сбалансированное дерево) — универсальный индекс по умолчанию

ascii
                        [50]
                      /      \
                  [20,30]    [70,80]
                 /   |   \   /   |   \
               10   25   35 60   75   90
Hash-индекс — только для точного равенства (=), очень быстрый
GiST/GIN — для полнотекстового поиска и JSON

Создание и использование индексов
sql
-- Простой индекс на одной колонке
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Уникальный индекс (автоматически для PRIMARY KEY)
CREATE UNIQUE INDEX idx_products_sku ON products(sku);

-- Составной индекс (важен порядок колонок!)
CREATE INDEX idx_orders_date_status ON orders(order_date, status);

-- Частичный индекс (только для интересующих строк)
CREATE INDEX idx_orders_pending ON orders(order_date) 
WHERE status = 'pending';

-- Индекс для полнотекстового поиска
CREATE INDEX idx_products_description ON products 
USING GIN(to_tsvector('russian', description));
Когда индекс полезен?
Ускоряет:

Поиск по условию (WHERE customer_id = 123)

Сортировка (ORDER BY order_date)

Соединение таблиц (JOIN ... ON ...)

Уникальность (UNIQUE)

Замедляет:

Вставку (INSERT) — нужно обновлять индекс

Обновление (UPDATE) индексированных колонок

Удаление (DELETE)

Анализ эффективности запросов
sql
-- PostgreSQL: план запроса
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- С детальным анализом
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 123;

-- Результат может показать:
-- Seq Scan — последовательное сканирование (плохо для больших таблиц)
-- Index Scan — сканирование индекса (хорошо)
-- Bitmap Heap Scan — комбинированный подход
Пример: сравнение с индексом и без
sql
-- Создаем большую таблицу
CREATE TABLE big_table (
    id SERIAL PRIMARY KEY,
    value INT,
    text VARCHAR(100)
);

-- Вставляем 1 миллион строк
INSERT INTO big_table (value, text)
SELECT random() * 1000000, 'text' || g
FROM generate_series(1, 1000000) g;

-- Поиск без индекса (Seq Scan)
EXPLAIN ANALYZE SELECT * FROM big_table WHERE value = 500000;
-- Время: ~150-200 ms

-- Создаем индекс
CREATE INDEX idx_big_value ON big_table(value);

-- Поиск с индексом (Index Scan)
EXPLAIN ANALYZE SELECT * FROM big_table WHERE value = 500000;
-- Время: ~0.1-0.5 ms
4.3 Оптимизация запросов
Типичные ошибки и как их исправить
1. SELECT * — выбираем всё подряд

sql
-- Плохо: тянет все колонки, даже не нужные
SELECT * FROM orders JOIN customers ON ...;

-- Хорошо: только нужные поля
SELECT orders.id, orders.total, customers.name FROM ...
2. Функции на индексированных колонках

sql
-- Плохо: индекс по date не используется
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- Хорошо: используется индекс
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' 
  AND order_date < '2025-01-01';
3. LIKE с ведущим %

sql
-- Плохо: полное сканирование
SELECT * FROM products WHERE name LIKE '%телефон%';

-- Хорошо: можно использовать полнотекстовый поиск
SELECT * FROM products 
WHERE to_tsvector('russian', name) @@ to_tsquery('телефон');
4. OR вместо UNION

sql
-- Плохо: сложно использовать индексы
SELECT * FROM orders 
WHERE customer_id = 123 OR status = 'pending';

-- Хорошо: можно оптимизировать через UNION
SELECT * FROM orders WHERE customer_id = 123
UNION
SELECT * FROM orders WHERE status = 'pending';