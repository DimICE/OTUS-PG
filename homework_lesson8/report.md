# Домашнее задание
## Настройка autovacuum с учетом особенностей производительности
### Цель:
* запустить нагрузочный тест pgbench
* настроить параметры autovacuum
* проверить работу autovacuum
---
> Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB  
> Установить на него PostgreSQL 15 с дефолтными настройками  
> Создать БД для тестов: выполнить pgbench -i postgres  
> Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
sudo su postgres
pgbench -i postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
exit
```
duration: 60 s  
number of transactions actually processed: 40718  
number of failed transactions: 0 (0.000%)  
latency average = 11.782 ms  
latency stddev = 8.430 ms  
initial connection time = 32.781 ms  
tps = 678.775412 (without initial connection time)  
> Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла  
> Протестировать заново  
```
sudo nano /etc/postgresql/15/main/postgresql.conf
sudo service postgresql restart
sudo su postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
duration: 60 s  
number of transactions actually processed: 45685  
number of failed transactions: 0 (0.000%)  
latency average = 10.501 ms  
latency stddev = 4.556 ms  
initial connection time = 34.576 ms  
tps = 761.590258 (without initial connection time)  
> Что изменилось и почему?  

Производительность повысилась за счет более подходящей настройки параметров под ресурсы ВМ.

> Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
> Посмотреть размер файла с таблицей
```
psql
CREATE TABLE test (field text);
INSERT INTO test(field) SELECT 'noname' FROM generate_series(1,1000000);
SELECT pg_size_pretty(pg_relation_size('test'));
```
35 MB
> 5 раз обновить все строчки и добавить к каждой строчке любой символ  
```
UPDATE test SET field = field || '1';
UPDATE test SET field = field || '1';
UPDATE test SET field = field || '1';
UPDATE test SET field = field || '1';
UPDATE test SET field = field || '1';
SELECT pg_size_pretty(pg_relation_size('test'));
```
238 MB
> Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум  
> Подождать некоторое время, проверяя, пришел ли автовакуум
> 5 раз обновить все строчки и добавить к каждой строчке любой символ  
> Посмотреть размер файла с таблицей
```
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test';
UPDATE test SET field = field || '2';
UPDATE test SET field = field || '2';
UPDATE test SET field = field || '2';
UPDATE test SET field = field || '2';
UPDATE test SET field = field || '2';
SELECT pg_size_pretty(pg_relation_size('test'));
```
261 MB

> Отключить Автовакуум на конкретной таблице  
> 10 раз обновить все строчки и добавить к каждой строчке любой символ  
> Посмотреть размер файла с таблицей  
> Объясните полученный результат  
> Не забудьте включить автовакуум)
```
ALTER TABLE test SET (autovacuum_enabled = off);
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
UPDATE test SET field = field || '3';
SELECT pg_size_pretty(pg_relation_size('test'));
```
570 MB  
При отключенном автовакууме за 10 апдейтов получили 10 миллионов мертвых строк, поэтому размер таблицы вырос в 2 раза  
Под 5 миллионов мертвых строк место уже было занято, хоть строки и были ранее очищены автовакуумом + под еще 5 млн мертвых строк выделилось дополнительное место.

> Задание со *:  
> Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.  
> Не забыть вывести номер шага цикла.
```
DO $$ BEGIN
    FOR i IN 1..10 LOOP
        RAISE NOTICE 'step = %', i;
        UPDATE test SET field = field || i;
    END LOOP;
END$$;
```