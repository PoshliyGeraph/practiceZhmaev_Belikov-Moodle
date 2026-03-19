## Лекция 5: Продвинутый SQL: оконные функции, CTE и оптимизация
*Теория: 40% / Практика: 60%*

5.1 Оконные функции (Window Functions)
Что такое оконные функции?
Оконные функции выполняют вычисления над набором строк, связанных с текущей строкой, не сворачивая их в одну строку (в отличие от GROUP BY).

ascii
GROUP BY:
┌─────────┬─────────┐    ┌─────────┬─────────┐
│ отдел   │ зарплата│    │ отдел   │ AVG     │
├─────────┼─────────┤ →  ├─────────┼─────────┤
│ IT      │ 120000  │    │ IT      │ 113333  │
│ IT      │ 90000   │    │ HR      │ 115000  │
│ HR      │ 150000  │    │ Sales   │ 72500   │
│ Sales   │ 75000   │    └─────────┴─────────┘
│ Sales   │ 70000   │    
└─────────┴─────────┘    

Оконные функции:
┌─────────┬─────────┬─────────┐
│ отдел   │ зарплата│ dept_avg│
├─────────┼─────────┼─────────┤
│ IT      │ 120000  │ 113333  │ ← сохраняем детали
│ IT      │ 90000   │ 113333  │   и добавляем
│ HR      │ 150000  │ 115000  │   агрегаты
│ Sales   │ 75000   │ 72500   │
│ Sales   │ 70000   │ 72500   │
└─────────┴─────────┴─────────┘
Синтаксис оконных функций
sql
<функция> OVER (
    [PARTITION BY выражение]  -- группировка (опционально)
    [ORDER BY выражение]      -- сортировка внутри окна
    [ROWS/RANGE BETWEEN ...]  -- рамки окна (опционально)
)
Агрегатные оконные функции
sql
SELECT 
    name,
    department,
    salary,
    -- Средняя по отделу (сохраняя все строки)
    AVG(salary) OVER (PARTITION BY department) as dept_avg,
    -- Средняя по компании
    AVG(salary) OVER () as company_avg,
    -- Максимальная в отделе
    MAX(salary) OVER (PARTITION BY department) as dept_max,
    -- Минимальная в отделе
    MIN(salary) OVER (PARTITION BY department) as dept_min,
    -- Сумма по отделу
    SUM(salary) OVER (PARTITION BY department) as dept_total
FROM employees;
Ранжирующие функции
sql
SELECT 
    name,
    department,
    salary,
    -- Ранжирование с пропусками (1,1,3,4...)
    RANK() OVER (ORDER BY salary DESC) as rank,
    -- Ранжирование без пропусков (1,1,2,3...)
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank,
    -- Номер строки (уникальный)
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num,
    -- Процентиль (0-1)
    PERCENT_RANK() OVER (ORDER BY salary DESC) as percent_rank
FROM employees;

-- Топ-3 по зарплате в каждом отделе
SELECT * FROM (
    SELECT 
        name,
        department,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
    FROM employees
) ranked
WHERE rn <= 3;
Функции смещения
sql
SELECT 
    name,
    department,
    salary,
    hire_date,
    -- Предыдущая запись
    LAG(salary, 1) OVER (PARTITION BY department ORDER BY hire_date) as prev_salary,
    -- Следующая запись
    LEAD(salary, 1) OVER (PARTITION BY department ORDER BY hire_date) as next_salary,
    -- Первая запись в окне
    FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY hire_date) as first_salary,
    -- Последняя запись в окне
    LAST_VALUE(salary) OVER (PARTITION BY department ORDER BY hire_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as last_salary
FROM employees
ORDER BY department, hire_date;
Пример: анализ продаж с накоплением
sql
SELECT 
    sale_date,
    product_name,
    amount,
    -- Накопительная сумма (по дате)
    SUM(amount) OVER (ORDER BY sale_date) as running_total,
    -- Скользящее среднее за 3 дня
    AVG(amount) OVER (ORDER BY sale_date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg_3d,
    -- Сумма за предыдущие 7 дней
    SUM(amount) OVER (ORDER BY sale_date 
        ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING) as prev_week_total
FROM sales
ORDER BY sale_date;
5.2 Общие табличные выражения (CTE)
Базовые CTE
sql
-- Простое CTE
WITH high_earners AS (
    SELECT name, salary, department
    FROM employees
    WHERE salary > 100000
)
SELECT department, COUNT(*) as high_earners_count
FROM high_earners
GROUP BY department;

-- Множественные CTE
WITH 
dept_stats AS (
    SELECT 
        department,
        AVG(salary) as avg_salary,
        COUNT(*) as emp_count
    FROM employees
    GROUP BY