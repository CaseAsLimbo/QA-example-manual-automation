## Здесь представлены примеры продвинутого использования SQL(агрегатные функции, join-ы, подзапросы)
Контекст: интернет-магазин – заказы, товары, позиции заказа
```sql
-- DDL: три связанные таблицы
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name        VARCHAR(100)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name       VARCHAR(200),
    price      DECIMAL(10,2)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    item_id    INT PRIMARY KEY,
    order_id   INT,
    product_id INT,
    quantity   INT,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- DML: заполнение тестовыми данными
INSERT INTO customers VALUES (1, 'Иван Петров'), (2, 'Анна Сидорова');
INSERT INTO products VALUES (10, 'Ноутбук', 50000.00), (20, 'Мышка', 2000.00);
INSERT INTO orders VALUES (100, 1, '2026-06-01'), (101, 2, '2026-06-02');
INSERT INTO order_items VALUES (1, 100, 10, 1), (2, 100, 20, 2), (3, 101, 10, 1);
```
---
### Запросы с JOIN и агрегацией
```sql
-- INNER JOIN: список всех заказов с именами покупателей
SELECT o.order_id, c.name, o.order_date
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
-- Пояснение: проверка связи «заказ-клиент», полезно при тестировании корректности вывода истории заказов.

-- LEFT JOIN: покупатели и их заказы (включая тех, у кого нет заказов)
SELECT c.name, COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.name;
-- Пояснение: тестирование отчёта «Клиенты без заказов», обнаружение потерянных данных.

-- Агрегатные функции: общая стоимость каждого заказа
SELECT o.order_id,
       SUM(oi.quantity * p.price) AS total_amount
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.order_id;
-- Пояснение: проверка корректности расчёта итоговой суммы заказа (сравнивается с UI/API).

-- HAVING: заказы на сумму больше 50 000
SELECT o.order_id, SUM(oi.quantity * p.price) AS total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.order_id
HAVING SUM(oi.quantity * p.price) > 50000;
-- Пояснение: используется для тестирования фильтрации в админке магазина.

-- Подзапрос: товары, которые ещё ни разу не покупали
SELECT name FROM products
WHERE product_id NOT IN (
    SELECT DISTINCT product_id FROM order_items
);
-- Пояснение: валидация отчёта «Неактивные товары» в панели мерчанта.
```
