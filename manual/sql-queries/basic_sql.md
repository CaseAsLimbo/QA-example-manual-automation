## Здесь представлены примеры SQL запросов(DDL, DML, DQL)
Контекст: **таблица пользователей**
<br>Используется для регистрации и входа. Требуется проверить, корректно ли записались данные через форму, или подготовить/удалить тестового пользователя. Смотри `-- пояснения к каждому запросу в комментариях. =)`

```sql
-- DDL: создание таблицы
CREATE TABLE users (
    user_id    INT PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    password   VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- DML: добавление тестового пользователя (подготовка тестовых данных)
INSERT INTO users (user_id, email, password)
VALUES (1, 'testuser@example.com', 'hashed_password_123');

-- Простой SELECT: получить все записи
SELECT * FROM users;
-- Пояснение: используется для проверки, что в таблице нет лишних записей или наоборот – что нужная запись создалась.

-- SELECT с условием WHERE: найти пользователя по email
SELECT * FROM users WHERE email = 'testuser@example.com';
-- Пояснение: проверка уникальности email и корректности регистронезависимого поиска.

-- UPDATE: изменение пароля (после теста восстановления пароля)
UPDATE users
SET password = 'new_hashed_password'
WHERE user_id = 1;
-- Пояснение: имитация смены пароля, после чего проводится проверка на вход с новым паролем.

-- DELETE: удаление тестового пользователя (очистка после теста)
DELETE FROM users WHERE user_id = 1;
-- Пояснение: удаление созданных данных для восстановления исходного состояния БД.
```
