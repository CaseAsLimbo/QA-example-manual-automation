## Проверка согласованности стоимости заказа
Контекст: После оформления заказа в UI итоговая сумма должна совпадать с суммой позиций в БД.

```sql
-- DDL и DML (кратко)
-- Таблицы orders, order_items, products созданы как выше.
-- Допустим, оформлен заказ #102 пользователем 1 на продукт 10 (2 шт).
INSERT INTO orders VALUES (102, 1, '2026-06-10');
INSERT INTO order_items VALUES (4, 102, 10, 2);

-- Запрос для проверки:
-- Шаг 1: вычисляем сумму позиций
SELECT SUM(oi.quantity * p.price) AS calculated_total
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
WHERE oi.order_id = 102;
-- Шаг 2: сравниваем с полем `total` или с тем, что вернул API.
-- Пояснение: автоматизация сверки: если calculated_total не равен
-- значению из ответа сервера или UI, значит ошибка в логике расчёта скидок.
```

---

## Проверка целостности жизненного цикла отправлений

**Контекст:** В сервисе отслеживания каждое отправление проходит по фиксированному workflow:  
`created -> picked_up  -> in_transit -> delivered`.  
Допускается отмена (`cancelled`) из любого состояния, после чего дальнейшие переходы запрещены.  
Изменения статусов логируются в отдельной таблице истории. Требуется удостовериться, что:

- Текущий статус в карточке отправления совпадает с последней записью истории.
- Нет пропущенных обязательных этапов (например, из `created` сразу в `delivered`).
- Хронология статусов не нарушена (записи идут строго по возрастанию времени и в соответствии с разрешёнными переходами).
- Нет бизнес-аномалий (мгновенная доставка).

### DDL (структура таблиц)

```sql
-- Основная таблица отправлений
CREATE TABLE shipments (
    shipment_id   INT PRIMARY KEY,
    current_status VARCHAR(20) NOT NULL,
    created_at     TIMESTAMP NOT NULL,
    updated_at     TIMESTAMP NOT NULL,
    CONSTRAINT chk_status CHECK (current_status IN (
        'created','picked_up','in_transit','delivered','cancelled'
    ))
);

-- История смены статусов (аудиторский лог)
CREATE TABLE shipment_status_history (
    history_id  SERIAL PRIMARY KEY,
    shipment_id INT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    changed_at  TIMESTAMP NOT NULL,
    FOREIGN KEY (shipment_id) REFERENCES shipments(shipment_id)
);
```

### DML (тестовые данные: нормальные и дефектные сценарии)

```sql
-- 1. Корректный полный цикл: created -> picked_up -> in_transit -> delivered
INSERT INTO shipments VALUES (1, 'delivered', '2026-07-01 10:00', '2026-07-02 16:00');
INSERT INTO shipment_status_history (shipment_id, status, changed_at) VALUES
(1, 'created',    '2026-07-01 10:00'),
(1, 'picked_up',  '2026-07-01 12:00'),
(1, 'in_transit', '2026-07-02 08:00'),
(1, 'delivered',  '2026-07-02 16:00');

-- 2. Корректный с отменой: created → cancelled
INSERT INTO shipments VALUES (2, 'cancelled', '2026-07-01 11:00', '2026-07-01 15:00');
INSERT INTO shipment_status_history (shipment_id, status, changed_at) VALUES
(2, 'created',   '2026-07-01 11:00'),
(2, 'cancelled', '2026-07-01 15:00');

-- 3. Некорректный: пропущен этап in_transit (created → picked_up → delivered)
INSERT INTO shipments VALUES (3, 'delivered', '2026-07-03 09:00', '2026-07-03 21:00');
INSERT INTO shipment_status_history (shipment_id, status, changed_at) VALUES
(3, 'created',   '2026-07-03 09:00'),
(3, 'picked_up', '2026-07-03 12:00'),
(3, 'delivered', '2026-07-03 21:00');

-- 4. Некорректный: нарушен порядок во времени (in_transit записан раньше picked_up)
INSERT INTO shipments VALUES (4, 'delivered', '2026-07-04 08:00', '2026-07-04 20:00');
INSERT INTO shipment_status_history (shipment_id, status, changed_at) VALUES
(4, 'created',    '2026-07-04 08:00'),
(4, 'in_transit', '2026-07-04 10:00'),  -- записан раньше, чем picked_up
(4, 'picked_up',  '2026-07-04 12:00'),  -- нарушение хронологии
(4, 'delivered',  '2026-07-04 20:00');

-- 5. Текущий статус не соответствует истории (в shipments указан 'in_transit', а последний в истории 'delivered')
INSERT INTO shipments VALUES (5, 'in_transit', '2026-07-05 08:00', '2026-07-05 10:00');
INSERT INTO shipment_status_history (shipment_id, status, changed_at) VALUES
(5, 'created',    '2026-07-05 08:00'),
(5, 'in_transit', '2026-07-05 10:00'),
(5, 'delivered',  '2026-07-05 18:00');  -- история уже ушла вперёд

-- 6. Мгновенная доставка (подозрительно короткий интервал)
INSERT INTO shipments VALUES (6, 'delivered', '2026-07-06 09:00', '2026-07-06 09:10');
INSERT INTO shipment_status_history (shipment_id, status, changed_at) VALUES
(6, 'created',    '2026-07-06 09:00'),
(6, 'picked_up',  '2026-07-06 09:02'),
(6, 'in_transit', '2026-07-06 09:05'),
(6, 'delivered',  '2026-07-06 09:10');
```

---

### SQL‑запросы для выявления нарушений

#### 1. Рассогласование текущего статуса и последней записи истории

```sql
SELECT s.shipment_id,
       s.current_status AS status_in_shipments,
       h.last_status AS status_in_history
FROM shipments s
JOIN (
  SELECT shipment_id,
         status AS last_status
  FROM shipment_status_history h1
  WHERE changed_at = (
    SELECT MAX(changed_at) FROM shipment_status_history h2
    WHERE h2.shipment_id = h1.shipment_id
  )
) h ON s.shipment_id = h.shipment_id
WHERE s.current_status <> h.last_status;
-- Ожидается пустой результат. Если есть записи – расхождение, возможен баг сервиса статусов.
```
---
#### 2. Пропущенные обязательные статусы

```sql
-- Разрешённый порядок (индексы для удобства):
-- 1:created, 2:picked_up, 3:in_transit, 4:delivered (cancelled вне порядка)
WITH status_order AS (
  SELECT status, seq FROM (VALUES
    ('created', 1), ('picked_up', 2), ('in_transit', 3), ('delivered', 4)
  ) AS t(status, seq)
),
shipment_path AS (
  SELECT h.shipment_id, h.status, so.seq,
         ROW_NUMBER() OVER (PARTITION BY h.shipment_id ORDER BY h.changed_at) AS step
  FROM shipment_status_history h
  JOIN status_order so ON h.status = so.status
  WHERE h.status != 'cancelled'  -- cancelled может появиться в любом месте, отдельная логика
)
SELECT shipment_id,
       STRING_AGG(status, ' -> ' ORDER BY step) AS actual_path
FROM shipment_path
GROUP BY shipment_id
HAVING STRING_AGG(status, ' -> ' ORDER BY step) NOT LIKE '%created%picked_up%in_transit%delivered%'
   AND STRING_AGG(status, ' -> ' ORDER BY step) NOT LIKE '%created%cancelled%';
-- Этот простой вариант проверяет, что строка, построенная из статусов, содержит нужную последовательность
-- для завершённых доставок. Более строгая проверка – сравнение с эталонным путём.
```
---
Более точный способ – проверить, что после сортировки по времени последовательность не нарушает порядок индексов (seq должен строго возрастать, кроме переходов с участием cancelled). Например, для некэнселенных отправлений:

```sql
WITH ranked AS (
  SELECT h.shipment_id, h.status, so.seq, h.changed_at,
         LAG(so.seq) OVER (PARTITION BY h.shipment_id ORDER BY h.changed_at) AS prev_seq,
         LAG(h.status) OVER (PARTITION BY h.shipment_id ORDER BY h.changed_at) AS prev_status
  FROM shipment_status_history h
  LEFT JOIN status_order so ON h.status = so.status
  WHERE h.status != 'cancelled'
)
SELECT DISTINCT shipment_id
FROM ranked
WHERE prev_seq IS NOT NULL
  AND seq <= prev_seq;  -- следующий статус должен иметь больший индекс
-- Если запрос возвращает строки, значит, есть перескоки или обратный ход.
```
---
#### 3. Аномально быстрая доставка (дополнительная бизнес‑проверка)

```sql
SELECT s.shipment_id,
       s.created_at,
       MIN(h.changed_at) FILTER (WHERE h.status = 'delivered') AS delivered_at,
       EXTRACT(EPOCH FROM (MIN(h.changed_at) FILTER (WHERE h.status = 'delivered') - s.created_at)) / 3600 AS hours
FROM shipments s
JOIN shipment_status_history h ON s.shipment_id = h.shipment_id
WHERE s.current_status = 'delivered'
GROUP BY s.shipment_id
HAVING MIN(h.changed_at) FILTER (WHERE h.status = 'delivered') - s.created_at < INTERVAL '1 hour';
-- Показывает отправления, доставленные менее чем за час с момента создания.
```
---
#### 4. Проверка, что отменённое отправление не имеет последующих статусов

```sql
SELECT h.shipment_id, h.changed_at AS cancelled_at, later.status AS later_status
FROM shipment_status_history h
JOIN shipment_status_history later
  ON h.shipment_id = later.shipment_id AND later.changed_at > h.changed_at
WHERE h.status = 'cancelled';
-- Если есть результаты, значит после отмены были проставлены другие статусы – нарушение.
```
