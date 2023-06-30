# Домашнее задание
## Репликация
### Цель:
* реализовать свой миникластер на 3 ВМ.
---

Создаем 4 ВМ в Google Cloud:  
instance-1: 10.128.0.8  
instance-2: 10.128.0.9  
instance-3: 10.128.0.10  
instance-4: 10.128.0.11  

На всех 3 ВМ:
1. ставим PostgreSQL
2. создаем базу test
3. создаем таблицы test и test2
4. создаем пользователя admin
5. меняем настройку listen_addresses на *, чтобы можно было подключаться с любого адреса
6. устанавливаем wal_level в значение logical
7. настраиваем доступ с любого адреса в pg_hba.conf (host all all 0.0.0.0/0 md5)
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
sudo -u postgres psql
CREATE DATABASE test;
\c test
CREATE TABLE test (value bigint);
CREATE TABLE test2 (value bigint);
CREATE USER admin SUPERUSER encrypted PASSWORD 'admin';
ALTER SYSTEM SET listen_addresses TO '*';
ALTER SYSTEM SET wal_level TO 'logical';
\q
sudo nano /etc/postgresql/15/main/pg_hba.conf
sudo pg_ctlcluster 15 main restart
```

> 1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
> 2. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.

На первой ВМ:
```
sudo -u postgres psql
\c test
CREATE PUBLICATION test_pub FOR TABLE test;
CREATE SUBSCRIPTION test2_sub CONNECTION 'host=10.128.0.9 port=5432 user=admin password=admin dbname=test' PUBLICATION test2_pub WITH (copy_data = true);
```
> 3. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.   
> 4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.

На второй ВМ:
```
sudo -u postgres psql
\c test
CREATE PUBLICATION test2_pub FOR TABLE test2;
CREATE SUBSCRIPTION test_sub CONNECTION 'host=10.128.0.8 port=5432 user=admin password=admin dbname=test' PUBLICATION test_pub WITH (copy_data = true);
```

> 5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).

На третьей ВМ:
```
sudo -u postgres psql
\c test
CREATE SUBSCRIPTION test_sub_from3 CONNECTION 'host=10.128.0.8 port=5432 user=admin password=admin dbname=test' PUBLICATION test_pub WITH (copy_data = true);
CREATE SUBSCRIPTION test2_sub_from3 CONNECTION 'host=10.128.0.9 port=5432 user=admin password=admin dbname=test' PUBLICATION test2_pub WITH (copy_data = true);
```

Проверяем, что всё работает:
1. записываем данные на ВМ1 в таблицу test
```
INSERT INTO test VALUES (1), (2), (3);
```
2. записываем данные на ВМ2 в таблицу test2
```
INSERT INTO test2 VALUES (1), (2), (3);
```
3. проверяем наличие данных на ВМ1/ВМ2/ВМ3
```
SELECT FROM test;
SELECT FROM test2;
```
Данные появились на всех 3 БД.

> Задачка под звездочкой: реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

На 3 ВМ настраиваем доступ с любого адреса в pg_hba.conf для репликации (host replication all 0.0.0.0/0 md5)  
Затем на 4 ВМ:
```
sudo pg_ctlcluster 15 main stop
sudo su - postgres
rm -rf /var/lib/postgresql/15/main/*
pg_basebackup --host=10.128.0.10 --port=5432 --username=admin --pgdata=/var/lib/postgresql/15/main/ --progress --write-recovery-conf --create-slot --slot=replica1
pg_ctlcluster 15 main start
```
Проверяем, добавляем данные на ВМ1, делаем выборку на ВМ4, данные появились, реплика работает.
