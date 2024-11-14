# Домашнее задание по лекции №3

### Описание работы
1. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк
2. Посмотреть размер файла с таблицей
3. 5 раз обновить все строчки и добавить к каждой строчке любой символ
4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
5. Подождать некоторое время, проверяя, пришел ли автовакуум
6. 5 раз обновить все строчки и добавить к каждой строчке любой символ
7. Посмотреть размер файла с таблицей
8. Отключить Автовакуум на конкретной таблице
9. 10 раз обновить все строчки и добавить к каждой строчке любой символ
10. Посмотреть размер файла с таблицей
11. Объясните полученный результат
12. Не забудьте включить автовакуум)

Задание со *:
Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в
искомой таблице.
Не забыть вывести номер шага цикла.

### Ход работы
1. Создадим таблицу с текстовым полем и заполним сгенерированными данным в размере 1 млн строк
```
thai=# CREATE TABLE random_data (
    id SERIAL PRIMARY KEY,
    text_field TEXT
);

CREATE TABLE
thai=# INSERT INTO random_data (text_field)
SELECT md5(random()::text)
FROM generate_series(1, 1000000);
INSERT 0 1000000
```

2. Найдем базовый размер файла с таблицей
```
thai=# SELECT pg_relation_filepath('random_data');
 pg_relation_filepath
----------------------
 base/16388/16519
(1 row)

thai=# \! ls -lh /var/lib/postgresql/16/main/base/16388/16519
-rw------- 1 postgres postgres 66M Nov 13 23:37 /var/lib/postgresql/16/main/base/16388/16519
```

3. Обновим таблицу 5 раз, добавляя различные символы в конец каждой строки
```
thai=# DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..5 LOOP
        UPDATE random_data
        SET text_field = text_field || chr(64 + i);
    END LOOP;
END $$;
```

4. Изучим количество мертвых строчек в таблице, и когда последний раз приходил автовакуум
```
thai=# select relname AS table_name,
    n_dead_tup AS dead_tuples,last_autovacuum FROM
    pg_stat_user_tables WHERE
    schemaname = 'public';
 table_name  | dead_tuples |        last_autovacuum
-------------+-------------+-------------------------------
 random_data |           0 | 2024-11-13 23:54:16.434464+03
(1 row)
```

7. Обновим таблицу 5 раз по функции из пункта 3, после выполнения, изучим размер файла с таблицей
```
thai=# \! ls -lh /var/lib/postgresql/16/main/base/16388/16519
-rw------- 1 postgres postgres 439M Nov 14 00:08 /var/lib/postgresql/16/main/base/16388/16519
```

8. Отключаем автовакуум на таблице
```
ALTER TABLE random_data SET (autovacuum_enabled = false);
```

9. Обновляем таблицу 10 раз, используя функцию
```
thai=# DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..10 LOOP
        UPDATE random_data
        SET text_field = text_field || chr(64 + i);
    END LOOP;
END $$;
DO
```

10. Изучаем размер файла с таблицей, смотрим вывод по количеству мертвых стров и по времени последнего автовакуума
```
thai=# \! ls -lh /var/lib/postgresql/16/main/base/16388/16519
-rw------- 1 postgres postgres 880M Nov 14 00:27 /var/lib/postgresql/16/main/base/16388/16519

thai=# select relname AS table_name,
    n_dead_tup AS dead_tuples,last_autovacuum FROM
    pg_stat_user_tables WHERE
    schemaname = 'public';
 table_name  | dead_tuples |        last_autovacuum
-------------+-------------+-------------------------------
 random_data |    10000000 | 2024-11-14 00:09:48.286001+03
(1 row)
```

12. Возвращаем автовакуум =)
```
thai=# ALTER TABLE random_data SET (autovacuum_enabled = true);
ALTER TABLE
```

### Выводы по ходу выполнения работы
- после первой модификации данных (пункт 3), автовакуум пришел довольно быстро (после обновления строк таблицы), поэтому количество мертвых строк в статистике не увидели;
- после второй модификации (пунт 5), размер таблицы сильно увеличился, приход автовакуума размер файла с таблицей не изменил;
- после отключения автовакуума и модификации (пункт 9), в статистике отобразились мертвые строки, а также размер  файла с таблицей кратно увеличился;
- после возвращения автовакуума (пункт 12), спустя небольшое кол-во времени, автовакуум дочистил мертвые строки в таблице (при этом размер файла с таблицей также не изменился);

в ходе последующего эксперимента, и применения команды `VACUUM FULL`, размер файла с таблицей снова вернулся к изначальному
```
thai=# \! ls -lh /var/lib/postgresql/16/main/base/16388/16519
-rw------- 1 postgres postgres 880M Nov 14 00:48 /var/lib/postgresql/16/main/base/16388/16519

thai=# SELECT pg_size_pretty(pg_table_size('random_data'));
 pg_size_pretty
----------------
 879 MB
(1 row)

thai=# VACUUM FULL random_data;
VACUUM
thai=# SELECT pg_size_pretty(pg_table_size('random_data'));
 pg_size_pretty
----------------
 89 MB
(1 row)
```

таким образом, автовакуум размер файла с таблицей не изменяет, с помощью автовакуума происходит удаление мертвых строк