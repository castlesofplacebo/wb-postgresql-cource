# Домашнее задание по лекции №8

### Описание работы
1. Создать таблицу с продажами
2. Реализовать функцию выбор трети года (1-4 месяц - первая треть, 5-8 - вторая и т.д.)
    a. через case
    b. * (бонуса в виде зачета дз не будет) используя математическую операцию (лучше 2+ варианта)
    c. предусмотреть NULL на входе
3. Вызвать эту функцию в SELECT из таблицы с продажами, уведиться, что всё отработало

### Ход работы
1. Создадим таблицу с продажами, и наполним данными для тестов
```
postgres=# CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    amount NUMERIC NOT NULL
);
CREATE TABLE
postgres=# INSERT INTO sales (sale_date, amount)
VALUES
    ('2024-01-15', 500),
    ('2024-01-16', 1000),
    ('2024-05-17', 750),
    ('2024-07-18', 1200),
    ('2024-10-19', 800),
    ('2024-12-20', 800);
INSERT 0 6
```

2. Создадим функцию, которая будет принимать на вход номер трети года (1, 2 или 3), и которая будет возвращать соответсвтующие номеру трети записи о продажах
```
CREATE OR REPLACE FUNCTION get_sales_from_third_part_of_year(num INT)
RETURNS TABLE(id int, sale_date DATE, amount NUMERIC) AS $$
BEGIN
    IF num IS NULL THEN
        RAISE EXCEPTION 'Input can not be null';
    END IF;

    IF num < 1 OR num > 3 THEN
        RAISE EXCEPTION 'Invalid input number: %, must be 1, 2 or 3', num;
    END IF;

    RETURN QUERY
    SELECT *
    FROM sales as base_sales
    WHERE CASE
            WHEN num = 1 THEN EXTRACT(MONTH FROM base_sales.sale_date) BETWEEN 1 AND 4
            WHEN num = 2 THEN EXTRACT(MONTH FROM base_sales.sale_date) BETWEEN 5 AND 8
            WHEN num = 3 THEN EXTRACT(MONTH FROM base_sales.sale_date) BETWEEN 9 AND 12
          END;
END;
$$ LANGUAGE plpgsql;
```

3. Тестируем работоспособность функции
```
postgres=# SELECT * FROM get_sales_from_third_part_of_year(2);
 id | sale_date  | amount
----+------------+--------
  3 | 2024-05-17 |    750
  4 | 2024-07-18 |   1200
(2 rows)

postgres=# SELECT * FROM get_sales_from_third_part_of_year(1);
 id | sale_date  | amount
----+------------+--------
  1 | 2024-01-15 |    500
  2 | 2024-01-16 |   1000
(2 rows)

postgres=# SELECT * FROM get_sales_from_third_part_of_year(3);
 id | sale_date  | amount
----+------------+--------
  5 | 2024-10-19 |    800
  6 | 2024-12-20 |    800
(2 rows)

postgres=# SELECT * FROM get_sales_from_third_part_of_year(4);
ERROR:  Invalid input number: 4, must be 1, 2 or 3
CONTEXT:  PL/pgSQL function get_sales_from_third_part_of_year(integer) line 4 at RAISE

postgres=# SELECT * FROM get_sales_from_third_part_of_year(-1);
ERROR:  Invalid input number: -1, must be 1, 2 or 3
CONTEXT:  PL/pgSQL function get_sales_from_third_part_of_year(integer) line 8 at RAISE

postgres=# SELECT * FROM get_sales_from_third_part_of_year(NULL);
ERROR:  Input can not be null
CONTEXT:  PL/pgSQL function get_sales_from_third_part_of_year(integer) line 4 at RAISE
postgres=#
```