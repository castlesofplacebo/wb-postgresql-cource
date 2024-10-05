# Домашнее задание по лекции №1

### Описание работы
1. Развернуть ВМ (Linux) с PostgreSQL
2. Залить Тайские перевозки
https://github.com/aeuge/postgres16book/tree/main/database
3. Посчитать количество поездок - `select count(*) from book.tickets`;

### Ход работы
1. Создана машина в wb cloud - Debian GNU/Linux 12
2. 
    - установлен PostgreSQL 16
    ```
    postgres=# select version();
                                                        version
    ---------------------------------------------------------------------------------------------------------------------
    PostgreSQL 16.4 (Debian 16.4-1.pgdg120+2) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
    (1 row)
    ```
    - выполнена заливка данных с помощью команды `wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql`
    ```
    postgres=# \l
                                                    List of databases
    Name    |  Owner   | Encoding | Locale Provider | Collate |  Ctype  | ICU Locale | ICU Rules |   Access privileges
    -----------+----------+----------+-----------------+---------+---------+------------+-----------+-----------------------
    postgres  | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           |
    template0 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           | =c/postgres          +
            |          |          |                 |         |         |            |           | postgres=CTc/postgres
    template1 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           | =c/postgres          +
            |          |          |                 |         |         |            |           | postgres=CTc/postgres
    thai      | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |            |           |
    (4 rows)
    ```
3. Выполнен подсчет количества поездок
    ```
    thai=# select count(*) from book.tickets;
    count
    ---------
    5185505
    (1 row)
    ```