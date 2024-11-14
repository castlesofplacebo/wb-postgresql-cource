# Домашнее задание по лекции №4

### Описание работы
1. Создать таблицу accounts(id integer, amount numeric);
2. Добавить несколько записей и, подключившись через 2 терминала, добиться ситуации взаимоблокировки (deadlock)
3. Посмотреть логи и убедиться, что информация о дедлоке туда попала

### Ход работы
1. Создадим таблицу `accounts(id integer, amount numeric);`
```
thai=# CREATE TABLE accounts(id integer, amount numeric);
CREATE TABLE
```

2. Наполним ее несколькими значениями
```
thai=# INSERT INTO accounts VALUES (1,200.00), (2,300.00), (3,400.00);
```

Добьемся ситуации возникновения deadlock - ниже приведены команды с указанием терминала, в порядке их выполнения:
**Терминал №1** - в транзакции берем в блокировку запись с id = 1
```
thai=# BEGIN;
BEGIN
thai=*# SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
 id | amount
----+--------
  1 | 200.00
(1 row)
```

**Терминал №2** - в транзакции берем в блокировку запись с id = 2
```
thai=# BEGIN;
BEGIN
thai=*# SELECT * FROM accounts WHERE id = 2 FOR UPDATE;
 id | amount
----+--------
  2 | 300.00
(1 row)
```

**Терминал №1** - пытаемся обновить запись с id = 2 и зависаем
```
thai=*# update accounts set amount=400 where id = 2;

```

**Терминал №2** - пытаемся обновить запись с id = 1 и получаем deadlock
```
thai=*# update accounts set amount=500 where id = 1;
ERROR:  deadlock detected
DETAIL:  Process 465817 waits for ShareLock on transaction 865; blocked by process 464474.
Process 464474 waits for ShareLock on transaction 866; blocked by process 465817.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
thai=!#
```
При этом в **Терминал №1** запрос проходит
```
thai=*# update accounts set amount=400 where id = 2;
UPDATE 1
```

3. Смотрим информацию о deadlock в логах
```
thai=# \! tail -f /var/log/postgresql/postgresql-16-main.log
2024-11-14 02:01:27.143 MSK [55843] LOG:  checkpoint starting: time
2024-11-14 02:01:27.248 MSK [55843] LOG:  checkpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.101 s, sync=0.001 s, total=0.105 s; sync files=1, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=391295 kB; lsn=1/CBCCB790, redo lsn=1/CBCCB750
2024-11-14 02:05:55.849 MSK [465817] postgres@thai ERROR:  deadlock detected
2024-11-14 02:05:55.849 MSK [465817] postgres@thai DETAIL:  Process 465817 waits for ShareLock on transaction 865; blocked by process 464474.
	Process 464474 waits for ShareLock on transaction 866; blocked by process 465817.
	Process 465817: update accounts set amount=500 where id = 1;
	Process 464474: update accounts set amount=400 where id = 2;
2024-11-14 02:05:55.849 MSK [465817] postgres@thai HINT:  See server log for query details.
2024-11-14 02:05:55.849 MSK [465817] postgres@thai CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-11-14 02:05:55.849 MSK [465817] postgres@thai STATEMENT:  update accounts set amount=500 where id = 1;
2024-11-14 02:06:27.323 MSK [55843] LOG:  checkpoint starting: time
2024-11-14 02:06:27.427 MSK [55843] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.101 s, sync=0.001 s, total=0.105 s; sync files=2, longest=0.001 s, average=0.001 s; distance=1 kB, estimate=352165 kB; lsn=1/CBCCBB98, redo lsn=1/CBCCBB60
```

### Итоги
В PostgreSQL есть механизм автоматического разрешения deadlock, поэтому один из процессов был убит для разрешения блокировки