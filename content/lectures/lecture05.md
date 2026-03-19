## Лекция 5: Администрирование и современные тренды (NoSQL)
*Теория: 50% / Итоговая работа: 50%*

5.1 Администрирование баз данных
Пользователи и привилегии
sql
-- Создание пользователя
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Дать права на подключение
GRANT CONNECT ON DATABASE shop TO app_user;

-- Дать права на схему
GRANT USAGE ON SCHEMA public TO app_user;

-- Дать права на таблицы
GRANT SELECT, INSERT, UPDATE ON ALL TABLES 
IN SCHEMA public TO app_user;

-- Дать права на последовательности (для SERIAL)
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Для будущих таблиц
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
GRANT SELECT, INSERT, UPDATE ON TABLES TO app_user;

-- Отозвать права
REVOKE DELETE ON orders FROM app_user;

-- Удалить пользователя
DROP USER app_user;
Резервное копирование
PostgreSQL:

bash
# Логический бэкап (дамп)
pg_dump -U postgres -d mydb > backup.sql

# Сжатый бэкап
pg_dump -U postgres -d mydb | gzip > backup.sql.gz

# Бэкап только схемы (без данных)
pg_dump -U postgres -d mydb --schema-only > schema.sql

# Восстановление
psql -U postgres -d mydb < backup.sql

# Физический бэкап (на уровне файлов)
# Нужно остановить БД и скопировать директорию с данными
Репликация — Master-Slave:

ascii
┌──────────┐          ┌──────────┐
│ Master   │─────────>│ Slave 1  │
│ (запись) │    └────>│ (только  │
└──────────┘     ┌───>│  чтение) │
                 │    └──────────┘
                 │    ┌──────────┐
                 └───>│ Slave 2  │
                      │ (только  │
                      │  чтение) │
                      └──────────┘
5.2 Введение в NoSQL
Почему появился NoSQL?
Проблемы реляционных БД:

Горизонтальное масштабирование сложно

Жесткая схема (нужно заранее все продумать)

Неудобно для некоторых типов данных (графы, иерархии)

Когда нужен NoSQL:

Миллиарды записей

Высокая нагрузка на запись

Часто меняющаяся структура данных

Геораспределенные системы

Теорема CAP
ascii
                    ┌─────────┐
                    │  CA     │
                    │ (RDBMS) │
           ┌────────┴─────────┴────────┐
           │                           │
    ┌──────┴──────┐             ┌──────┴──────┐
    │   CP        │             │    AP       │
    │ (MongoDB    │             │ (Cassandra, │
    │  в нек.     │             │  DynamoDB)  │
    │  режимах)   │             │             │
    └─────────────┘             └─────────────┘
Consistency (согласованность) — все видят одни и те же данные

Availability (доступность) — система всегда отвечает

Partition tolerance (устойчивость к разрывам) — работает при потере связи между узлами

Теорема: можно выбрать только 2 из 3.

5.3 Типы NoSQL БД
1. Ключ-значение (Key-Value)
Примеры: Redis, Memcached, Riak

ascii
┌────────────┬─────────────────────────┐
│ Key        │ Value                   │
├────────────┼─────────────────────────┤
│ user:123   │ {"name": "Иван", "age":30}│
│ session:xyz│ {"user_id": 123, "expire":...}│
│ product:456│ {"title": "Ноутбук", "price":75000}│
└────────────┴─────────────────────────┘
Redis пример:

bash
# Установка значений
SET user:123:name "Иван"
SET user:123:age 30

# Получение
GET user:123:name

# Счетчики
INCR page:visits:2024-01-01

# Списки
LPUSH recent:visits user:123

# Истечение срока
EXPIRE session:xyz 3600  # через час удалится
2. Документо-ориентированные (Document)
Примеры: MongoDB, CouchDB

javascript
// MongoDB — коллекция пользователей
db.users.insertOne({
    _id: ObjectId("507f1f77bcf86cd799439011"),
    name: "Иван Петров",
    email: "ivan@email.com",
    address: {
        city: "Москва",
        street: "Ленина",
        zip: "101000"
    },
    interests: ["программирование", "музыка"],
    orders: [
        { date: new Date(), total: 75000 },
        { date: new Date(), total: 1500 }
    ]
});

// Поиск
db.users.find({ "address.city": "Москва" });
db.users.find({ interests: "программирование" });
3. Колоночные (Wide Column)
Примеры: Apache Cassandra, HBase

ascii
Row Key: user:123
┌─────────────────────────────────────┐
│ name   : Иван                       │
│ email  : ivan@email.com             │
│ address: Москва                     │
│ orders:2024-01: {...}               │
│ orders:2024-02: {...}               │
└─────────────────────────────────────┘
Cassandra пример:

sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    email TEXT
);

CREATE TABLE orders_by_user (
    user_id UUID,
    order_date DATE,
    order_id UUID,
    amount DECIMAL,
    PRIMARY KEY (user_id, order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);
4. Графовые (Graph)
Примеры: Neo4j, Amazon Neptune

ascii
        [Иван] —(друг)—> [Мария]
         |                 |
      (работает)        (работает)
         |                 |
         ▼                 ▼
      [Компания А]     [Компания Б]
Neo4j Cypher запросы:

cypher
// Создание узлов и связей
CREATE (ivan:Person {name: 'Иван', age: 30})
CREATE (maria:Person {name: 'Мария', age: 28})
CREATE (companyA:Company {name: 'ООО Рога'})
CREATE (companyB:Company {name: 'ООО Копыта'})

CREATE (ivan)-[:WORKS_AT {since: 2020}]->(companyA)
CREATE (maria)-[:WORKS_AT {since: 2021}]->(companyB)
CREATE (ivan)-[:FRIEND_WITH]->(maria)

// Поиск друзей, работающих в одной компании
MATCH (p:Person)-[:FRIEND_WITH]->(friend)-[:WORKS_AT]->(company)
WHERE p.name = 'Иван'
RETURN friend.name, company.name
5.4 SQL vs NoSQL: сравнение
Критерий	SQL (PostgreSQL)	NoSQL (MongoDB)
Схема	Фиксированная, жесткая	Гибкая, динамическая
Масштабирование	Вертикальное (мощнее сервер)	Горизонтальное (больше серверов)
Транзакции	ACID (строгие)	BASE (мягкие)
Связи	Через JOIN	Вложенные документы или ссылки
Запросы	SQL (стандарт)	Специфичные API
Когда использовать	Финансы, CRM, где важна целостность	Логи, каталоги, соцсети
Что такое BASE?
Basically Available — система всегда доступна

Soft state — состояние может меняться без внешнего воздействия

Eventual consistency — данные в конечном счете согласуются