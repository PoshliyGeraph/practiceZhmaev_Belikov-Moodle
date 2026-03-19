## Лекция 2: Агрегация и группировка. Функции работы с данными
*Теория: 30% / Практика: 70%*

2.1 Агрегатные функции
Базовые агрегатные функции
sql
-- COUNT: количество строк
SELECT COUNT(*) FROM employees;                    -- всего сотрудников
SELECT COUNT(email) FROM employees;                 -- не считает NULL
SELECT COUNT(DISTINCT department) FROM employees;   -- уникальных отделов

-- SUM: сумма
SELECT SUM(salary) FROM employees;                   -- общий фонд оплаты труда
SELECT SUM(salary) FROM employees WHERE department = 'IT';

-- AVG: среднее
SELECT AVG(salary) FROM employees;                   -- средняя зарплата
SELECT AVG(salary) FROM employees WHERE department = 'IT';

-- MIN/MAX: минимум и максимум
SELECT 
    MIN(salary) as min_salary,
    MAX(salary) as max_salary,
    MAX(salary) - MIN(salary) as salary_range
FROM employees;
Примеры с агрегацией
sql
-- Статистика по зарплатам
SELECT 
    COUNT(*) as total_employees,
    AVG(salary) as avg_salary,
    SUM(salary) as total_salary,
    MIN(salary) as min_salary,
    MAX(salary) as max_salary
FROM employees;

-- Количество сотрудников по отделам
SELECT 
    department,
    COUNT(*) as employee_count
FROM employees
GROUP BY department;
2.2 Группировка GROUP BY
Принцип работы GROUP BY
ascii
Исходная таблица employees:
┌────────────┬────────────┬────────┐
│ name       │ department │ salary │
├────────────┼────────────┼────────┤
│ Иван       │ IT         │ 120000 │
│ Мария      │ IT         │ 90000  │
│ Петр       │ HR         │ 80000  │
│ Ольга      │ Sales      │ 75000  │
│ Анна       │ IT         │ 130000 │
│ Дмитрий    │ Sales      │ 70000  │
│ Елена      │ HR         │ 150000 │
└────────────┴────────────┴────────┘

После GROUP BY department:
┌────────────┬──────────────┬──────────────┬──────────────┐
│ department │ COUNT(*)     │ AVG(salary)  │ SUM(salary)  │
├────────────┼──────────────┼──────────────┼──────────────┤
│ IT         │ 3            │ 113333.33    │ 340000       │
│ HR         │ 2            │ 115000       │ 230000       │
│ Sales      │ 2            │ 72500        │ 145000       │
└────────────┴──────────────┴──────────────┴──────────────┘
Базовый GROUP BY
sql
-- Количество сотрудников по отделам
SELECT 
    department,
    COUNT(*) as employee_count
FROM employees
GROUP BY department;

-- Средняя зарплата по должностям
SELECT 
    position,
    AVG(salary) as avg_salary,
    COUNT(*) as employee_count
FROM employees
GROUP BY position
ORDER BY avg_salary DESC;

-- Группировка по нескольким полям
SELECT 
    department,
    position,
    COUNT(*) as count,
    AVG(salary) as avg_salary
FROM employees
GROUP BY department, position
ORDER BY department, position;
GROUP BY с выражениями
sql
-- Группировка по году найма
SELECT 
    EXTRACT(YEAR FROM hire_date) as hire_year,
    COUNT(*) as hired_count
FROM employees
GROUP BY EXTRACT(YEAR FROM hire_date)
ORDER BY hire_year;

-- Группировка по диапазону зарплат
SELECT 
    CASE 
        WHEN salary < 80000 THEN 'Низкая'
        WHEN salary BETWEEN 80000 AND 120000 THEN 'Средняя'
        ELSE 'Высокая'
    END as salary_grade,
    COUNT(*) as employee_count
FROM employees
GROUP BY salary_grade;
2.3 Фильтрация групп HAVING
Отличие WHERE от HAVING
sql
-- WHERE — фильтрует строки ДО группировки
-- HAVING — фильтрует группы ПОСЛЕ группировки

SELECT 
    department,
    AVG(salary) as avg_salary,
    COUNT(*) as emp_count
FROM employees
WHERE hire_date >= '2023-01-01'  -- сначала отфильтровали новых сотрудников
GROUP BY department
HAVING COUNT(*) >= 2              -- потом оставили отделы с >= 2 сотрудниками
ORDER BY avg_salary DESC;
Примеры HAVING
sql
-- Отделы с минимальной зарплатой выше 70000
SELECT 
    department,
    MIN(salary) as min_salary,
    MAX(salary) as max_salary
FROM employees
GROUP BY department
HAVING MIN(salary) > 70000;

-- Должности, где средняя зарплата > 100000 и сотрудников >= 2
SELECT 
    position,
    AVG(salary) as avg_salary,
    COUNT(*) as emp_count
FROM employees
GROUP BY position
HAVING AVG(salary) > 100000 AND COUNT(*) >= 2;

-- Отделы с суммарной зарплатой > 200000
SELECT 
    department,
    SUM(salary) as total_salary
FROM employees
GROUP BY department
HAVING SUM(salary) > 200000;
2.4 Строковые функции
Основные строковые функции
sql
-- Конкатенация (объединение строк)
SELECT 
    first_name || ' ' || last_name as full_name,
    CONCAT(first_name, ' ', last_name) as full_name_v2  -- альтернатива
FROM employees;

-- Длина строки
SELECT 
    name,
    LENGTH(name) as name_length
FROM employees;

-- Изменение регистра
SELECT 
    UPPER(name) as upper_name,
    LOWER(department) as lower_dept,
    INITCAP(email) as proper_email  -- первая буква заглавная
FROM employees;

-- Обрезка пробелов
SELECT 
    TRIM('  Hello  ') as trimmed,        -- 'Hello'
    LTRIM('  Hello') as left_trimmed,     -- 'Hello'
    RTRIM('Hello  ') as right_trimmed;    -- 'Hello'

-- Поиск подстроки
SELECT 
    name,
    POSITION('ов' IN name) as position_ov,  -- позиция подстроки
    STRPOS(name, 'ов') as position_ov_v2
FROM employees
WHERE name LIKE '%ов%';

-- Извлечение подстроки
SELECT 
    email,
    SUBSTRING(email FROM 1 FOR POSITION('@' IN email) - 1) as username,
    SUBSTRING(email FROM POSITION('@' IN email) + 1) as domain
FROM employees
WHERE email IS NOT NULL;

-- Замена подстроки
SELECT 
    name,
    REPLACE(department, 'IT', 'Information Technology') as dept_full
FROM employees;
2.5 Функции даты и времени
Работа с датами
sql
-- Текущая дата и время
SELECT 
    CURRENT_DATE as today,
    CURRENT_TIME as now_time,
    CURRENT_TIMESTAMP as now,
    NOW() as now_v2;

-- Извлечение частей даты
SELECT 
    hire_date,
    EXTRACT(YEAR FROM hire_date) as year,
    EXTRACT(MONTH FROM hire_date) as month,
    EXTRACT(DAY FROM hire_date) as day,
    EXTRACT(DOW FROM hire_date) as day_of_week,  -- 0=воскресенье
    EXTRACT(QUARTER FROM hire_date) as quarter
FROM employees;

-- Форматирование дат
SELECT 
    hire_date,
    TO_CHAR(hire_date, 'DD.MM.YYYY') as ru_format,
    TO_CHAR(hire_date, 'Month DD, YYYY') as us_format,
    TO_CHAR(hire_date, 'Day') as weekday
FROM employees;

-- Арифметика с датами
SELECT 
    name,
    hire_date,
    CURRENT_DATE - hire_date as days_worked,
    AGE(CURRENT_DATE, hire_date) as interval_worked,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) as years_worked
FROM employees;

-- Добавление/вычитание интервалов
SELECT 
    hire_date,
    hire_date + INTERVAL '1 year' as anniversary,
    hire_date + INTERVAL '6 months' as review_date,
    hire_date - INTERVAL '1 day' as day_before_hire
FROM employees;

-- Фильтрация по датам
SELECT * FROM employees
WHERE hire_date BETWEEN 
    (CURRENT_DATE - INTERVAL '1 year') AND CURRENT_DATE;  -- последний год

SELECT * FROM employees
WHERE EXTRACT(MONTH FROM hire_date) = 1;  -- нанятые в январе
2.6 Числовые функции
Математические функции
sql
-- Округление
SELECT 
    salary,
    ROUND(salary, -3) as rounded_thousands,  -- округление до тысяч
    ROUND(salary / 1000) * 1000 as alt_round,
    CEIL(salary / 1000) * 1000 as round_up,   -- всегда вверх
    FLOOR(salary / 1000) * 1000 as round_down  -- всегда вниз
FROM employees;

-- Абсолютное значение, знак
SELECT 
    salary - 100000 as diff,
    ABS(salary - 100000) as abs_diff,
    SIGN(salary - 100000) as sign_diff  -- -1, 0, 1
FROM employees;

-- Степень, корень
SELECT 
    POWER(10, 3) as ten_cubed,     -- 1000
    SQRT(144) as sqrt_144,          -- 12
    CBRT(27) as cube_root_27;       -- 3

-- Тригонометрия (редко в бизнес-задачах)
SELECT 
    SIN(radians(30)) as sin_30,
    COS(radians(60)) as cos_60;
2.7 Условные выражения
CASE
sql
-- Простой CASE (сравнение с константами)
SELECT 
    name,
    salary,
    CASE department
        WHEN 'IT' THEN salary * 1.1
        WHEN 'HR' THEN salary * 1.05
        ELSE salary
    END as new_salary
FROM employees;

-- Поисковый CASE (с условиями)
SELECT 
    name,
    salary,
    CASE 
        WHEN salary < 80000 THEN 'Низкая'
        WHEN salary BETWEEN 80000 AND 120000 THEN 'Средняя'
        WHEN salary > 120000 THEN 'Высокая'
        ELSE 'Не определена'
    END as salary_grade
FROM employees;

-- CASE в GROUP BY
SELECT 
    CASE 
        WHEN salary < 100000 THEN 'До 100k'
        ELSE 'От 100k'
    END as salary_range,
    COUNT(*) as employee_count,
    AVG(salary) as avg_salary
FROM employees
GROUP BY salary_range;
COALESCE и NULLIF
sql
-- COALESCE: первое не-NULL значение
SELECT 
    name,
    COALESCE(email, 'нет email') as contact,
    COALESCE(phone, email, 'нет контактов') as any_contact
FROM employees;

-- NULLIF: возвращает NULL если значения равны
SELECT 
    name,
    NULLIF(department, 'IT') as non_it_dept  -- IT станет NULL
FROM employees;

-- Пример: избегаем деления на ноль
SELECT 
    product_name,
    sales,
    COALESCE(sales / NULLIF(target, 0), 0) as percent_of_target
FROM sales_data;