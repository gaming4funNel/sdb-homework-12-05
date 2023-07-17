# Домашнее задание к занятию «Индексы» - Иванов Игорь

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

SELECT SUM(INDEX_LENGTH) AS index_size, SUM(DATA_LENGTH + INDEX_LENGTH) AS table_size, (SUM(INDEX_LENGTH) / (SUM(DATA_LENGTH + INDEX_LENGTH))) * 100 AS index_to_table_ratio FROM information_schema.TABLES WHERE TABLE_SCHEMA = 'sakila';

![sql](https://github.com/gaming4funNel/sdb-homework-12-05/blob/main/img/sql1.png)

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

До оптимизации.

![sql](https://github.com/gaming4funNel/sdb-homework-12-05/blob/main/img/before.png)

1. Добавление индексов сократило время запроса не значительно.

CREATE INDEX idx_payment_payment_date ON payment (payment_date);
CREATE INDEX idx_rental_rental_date ON rental (rental_date);
CREATE INDEX idx_customer_customer_id ON customer (customer_id);
CREATE INDEX idx_inventory_inventory_id ON inventory (inventory_id);
CREATE INDEX idx_film_film_id ON film (film_id);

![sql](https://github.com/gaming4funNel/sdb-homework-12-05/blob/main/img/index1.png)

2. Переписал запрос с использованием JOIN и ON вместо перечисления таблиц в блоке FROM с использованием запятых, что сократило время запроса значительно.

SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title)
FROM payment p
JOIN rental r ON p.payment_date = r.rental_date
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON i.inventory_id = r.inventory_id
JOIN film f ON i.film_id = f.film_id
WHERE DATE(p.payment_date) = '2005-07-30';

![sql](https://github.com/gaming4funNel/sdb-homework-12-05/blob/main/img/index2.png)

3. Использование подзапроса вместо вместо SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title), дополнительно сократило время запроса.

SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), t.total_amount
FROM (
    SELECT c.customer_id, f.title, SUM(p.amount) AS total_amount
    FROM payment p
    JOIN rental r ON p.payment_date = r.rental_date
    JOIN customer c ON r.customer_id = c.customer_id
    JOIN inventory i ON i.inventory_id = r.inventory_id
    JOIN film f ON i.film_id = f.film_id
    WHERE DATE(p.payment_date) = '2005-07-30'
    GROUP BY c.customer_id, f.title
) t
JOIN customer c ON t.customer_id = c.customer_id;

![sql](https://github.com/gaming4funNel/sdb-homework-12-05/blob/main/img/index3.png)

4. Правки к ДЗ. 

CREATE INDEX idx_paymentdate ON payment (payment_date);

EXPLAIN ANALYZE 
SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title)
FROM payment p, rental r, customer c, inventory i, film f
WHERE p.payment_date >= '2005-07-30' AND p.payment_date < DATE_ADD('2005-07-30', INTERVAL 1 DAY) 
  AND p.payment_date = r.rental_date AND r.customer_id = c.customer_id AND i.inventory_id = r.inventory_id;

![sql](https://github.com/gaming4funNel/sdb-homework-12-05/blob/main/img/index4.png)

## Дополнительные задания (со звёздочкой*)
Эти задания дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.

### Задание 3*

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

*Приведите ответ в свободной форме.*

Индекс по частичному соответствию (Partial Index): в PostgreSQL вы можете создать индекс только для строк, которые удовлетворяют определенному условию. Это позволяет сократить размер индекса и улучшить производительность запросов, которые используют этот индекс.

Индекс сортировки NULL (NULLS FIRST / NULLS LAST Index): в PostgreSQL вы можете указать, как будут сортироваться NULL значения в индексе. Это полезно, когда вам нужно отсортировать данные в определенном порядке, например, сначала NULL значения, а затем не-NULL значения.

Индекс функции (Functional Index): в PostgreSQL вы можете создать индекс на основе выражения или функции, а не только на столбце. Это позволяет вам создавать индексы для вычисляемых значений или применять функции к столбцам во время поиска.

Индекс на массив (Array Index): в PostgreSQL вы можете создать индекс на столбец с типом данных массива. Это позволяет эффективно искать значения в массиве и улучшить производительность запросов, связанных с массивами.

