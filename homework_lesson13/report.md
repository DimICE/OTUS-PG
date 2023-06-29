# Домашнее задание
## Бэкапы
### Цель:
* Применить логический бэкап. Восстановиться из бэкапа
---

> 1. Создаем ВМ/докер c ПГ.  
> 2. Создаем БД, схему и в ней таблицу.  
> 3. Заполним таблицы автосгенерированными 100 записями.  

Ставим pgsql и создаем таблицу с данными:
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
sudo -u postgres psql
CREATE DATABASE test;
\c test
CREATE SCHEMA test;
CREATE TABLE test.test (c1 integer);
INSERT INTO test.test(c1) SELECT generate_series FROM generate_series(1,100);
```

> 4. Под линукс пользователем Postgres создадим каталог для бэкапов   
> 5. Сделаем логический бэкап используя утилиту COPY   
> 6. Восстановим во вторую таблицу данные из бэкапа.  

Создаем каталог для бэкапов, делаем бэкап и восстановаливаем его в новую таблицу, проверяем, что данные успешно восстановлены:
```
sudo su postgres
mkdir ~/backup
psql
\c test
\copy test.test to '/var/lib/postgresql/backup/test.sql';
CREATE TABLE test.test2 (c1 integer);
\copy test.test2 from '/var/lib/postgresql/backup/test.sql';
select * from test.test2;
```
Видим, что данные были успешно восстановлены.

> 7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц   
> 8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!  

С помощью pg_dump делаем дамп, с помощью pg_restore восстаналиваем из него одну из таблиц, указывая ее в параметре -t:
```
pg_dump -d test -U postgres -Fc > /var/lib/postgresql/backup/test.gz
psql
create database test2;
\c test2
create schema test;
\q
pg_restore -d test2 -U postgres -t test2 /var/lib/postgresql/backup/test.gz
psql
\c test2
select * from test.test;
select * from test.test2;
```
Видим, что восстановилась только таблица test.test2.  
Предварительно пришлось создать базу и схему, т.к. при указании конкретной таблицы в pg_restore команда ругалась на отсутствующую схему.
