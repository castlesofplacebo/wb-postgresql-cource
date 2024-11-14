# Домашнее задание по лекции №6

### Описание работы
- Развернуть асинхронную реплику (можно использовать 1 ВМ, просто рядом кластер развернуть и подключиться через localhost):

❖ тестируем производительность по сравнению с сингл инстансом

### Ход работы
- Работа проводилась на одной виртуальной машины, для тестирования на ней же разворачивался второй кластер PostgreSQL - `lab6`
```
postgres@postgres:/home/sheludchenko.a4$ psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret$123';"
CREATE ROLE

postgres@postgres:/home/sheludchenko.a4$ psql -c "SELECT pg_create_physical_replication_slot('test');"
 pg_create_physical_replication_slot
-------------------------------------
 (test,)
(1 row)

postgres@postgres:/home/sheludchenko.a4$ pg_createcluster 16 lab6

sheludchenko.a4@postgres:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  lab6    5433 online postgres /var/lib/postgresql/16/lab6 /var/log/postgresql/postgresql-16-lab6.log
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

postgres@postgres:~$ pg_ctlcluster 16 lab6 stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-lab6

postgres@postgres:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  lab6    5433 down   postgres /var/lib/postgresql/16/lab6 /var/log/postgresql/postgresql-16-lab6.log
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

postgres@postgres:~$ rm -rf /var/lib/postgresql/16/lab6

postgres@postgres:~$ pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
16  lab6    5433 down   <unknown> /var/lib/postgresql/16/lab6 /var/log/postgresql/postgresql-16-lab6.log
16  main    5432 online postgres  /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

postgres@postgres:~$ pg_basebackup -h localhost -p 5432 -U replicator -R -S test -D /var/lib/postgresql/16/lab6

postgres@postgres:~$ pg_lsclusters
Ver Cluster Port Status        Owner    Data directory              Log file
16  lab6    5433 down,recovery postgres /var/lib/postgresql/16/lab6 /var/log/postgresql/postgresql-16-lab6.log
16  main    5432 online        postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

postgres@postgres:~$
```
С помощью команд выше, на кластере `lab6` создали физическую реплику кластера `main`

2. Протестируем производительность кластера `main` до включения асинхронной реплики
тест на скорость вставки
```
postgres@postgres:~$ cat > ~/workload2.sql << EOL
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
        ceil(random()*100)
        , (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
    ,('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
    , ceil(random()*100));

EOL

postgres@postgres:~$ /usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 50261
number of failed transactions: 0 (0.000%)
latency average = 1.592 ms
initial connection time = 11.684 ms
tps = 5026.257322 (without initial connection time)
```

тест на скорость чтения
```
postgres@postgres:~$ cat > ~/workload.sql << EOL
\set r random(1, 5000000)
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
EOL

postgres@postgres:~$ /usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 266231
number of failed transactions: 0 (0.000%)
latency average = 0.300 ms
initial connection time = 14.773 ms
tps = 26661.460509 (without initial connection time)
```

Включим асинхронную реплику
```
postgres@postgres:/home/sheludchenko.a4$ pg_ctlcluster 16 lab6 start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@16-lab6

postgres@postgres:/home/sheludchenko.a4$ pg_lsclusters
Ver Cluster Port Status          Owner    Data directory              Log file
16  lab6    5433 online,recovery postgres /var/lib/postgresql/16/lab6 /var/log/postgresql/postgresql-16-lab6.log
16  main    5432 online          postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

postgres@postgres:/home/sheludchenko.a4$ psql -p 5433 -d thai -c "select pg_is_in_recovery();"
 pg_is_in_recovery
-------------------
 t
(1 row)
```

Проверим изменение производительности на кластере `main`.
тест на скорость вставки
```
postgres@postgres:/home/sheludchenko.a4$ /usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 31694
number of failed transactions: 0 (0.000%)
latency average = 2.523 ms
initial connection time = 11.740 ms
tps = 3170.418338 (without initial connection time)
```

тест на скорость чтения
```
postgres@postgres:/home/sheludchenko.a4$ /usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 253001
number of failed transactions: 0 (0.000%)
latency average = 0.316 ms
initial connection time = 10.881 ms
tps = 25304.809225 (without initial connection time)
```

Сравним результаты тестирования лидера.
```
Тест на вставку значений 5026 -> 3170
Тест на скорость чтения 26661 -> 25304
```

Также для сравнения, тест скорости на чтение с асинхронной реплики составил 25670

### Итоги
В результате добавления асинхронной реплики, видим падение в скорости на вставку.
Скорость на чтение на лидере и реплике примерно одинаковые