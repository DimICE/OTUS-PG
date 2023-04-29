# Домашнее задание
## Работа с базами данных, пользователями и правами
### Цель:
* создание новой базы данных, схемы и таблицы
* создание роли для чтения данных из созданной схемы созданной базы данных
* создание роли для чтения и записи из созданной схемы созданной базы данных
---
> 1. создайте новый кластер PostgresSQL 14
> 2. зайдите в созданный кластер под пользователем postgres
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
sudo -u postgres psql
```
> 3. создайте новую базу данных testdb
> 4. зайдите в созданную базу данных под пользователем postgres
> 5. создайте новую схему testnm
> 6. создайте новую таблицу t1 с одной колонкой c1 типа integer
> 7. вставьте строку со значением c1=1
```
CREATE DATABASE testdb;
\c testdb
CREATE SCHEMA testnm;
CREATE TABLE t1 (c1 integer);
INSERT INTO t1 (c1) VALUES (1);
```
> 8. создайте новую роль readonly
> 9. дайте новой роли право на подключение к базе данных testdb
> 10. дайте новой роли право на использование схемы testnm
> 11. дайте новой роли право на select для всех таблиц схемы testnm
```
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
```
> 12. создайте пользователя testread с паролем test123
> 13. дайте роль readonly пользователю testread
> 14. зайдите под пользователем testread в базу данных testdb
> 15. сделайте select * from t1;
> 16. получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
> 17. напишите что именно произошло в тексте домашнего задания
> 18. у вас есть идеи почему? ведь права то дали?
> 19. посмотрите на список таблиц
> 20. подсказка в шпаргалке под пунктом 20
> 21. а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
```
CREATE USER testread WITH PASSWORD 'test123';
GRANT readonly TO testread;
\q
psql -U testread -h localhost -d testdb
select * from t1;
\dt
```
Выдало ошибку ERROR:  permission denied for table t1  
Потому что таблица в схеме public.  
Таблица попала в схему public, т.к. мы не указали явно в какой схеме создавать таблицу, права на схему public для роли readonly не выдавались.
> 22. вернитесь в базу данных testdb под пользователем postgres
> 23. удалите таблицу t1
> 24. создайте ее заново но уже с явным указанием имени схемы testnm
> 25. вставьте строку со значением c1=1
> 26. зайдите под пользователем testread в базу данных testdb
> 27. сделайте select * from testnm.t1;
> 28. получилось?
> 29. есть идеи почему? если нет - смотрите шпаргалку
```
\q
sudo -u postgres psql
\c testdb
DROP TABLE t1;
CREATE TABLE testnm.t1 (c1 integer);
INSERT INTO testnm.t1 (c1) VALUES (1);
\q
psql -U testread -h localhost -d testdb
select * from testnm.t1;
```
Выдало ошибку ERROR:  permission denied for table t1
Потому что таблицу создавали после выдачи прав на всех таблицы в схеме testnm.
> 30. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
> 31. сделайте select * from testnm.t1;
> 32. получилось?
> 33. есть идеи почему? если нет - смотрите шпаргалку
> 31. сделайте select * from testnm.t1;
> 32. получилось?
> 33. ура!
```
\q
sudo -u postgres psql
\c testdb
ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly; 
\q
psql -U testread -h localhost -d testdb
select * from t1;
select * from testnm.t1;
SHOW search_path; 
```
Выдали права на чтение из всех таблиц в схеме и поменяли умолчание для того, чтобы права выдавались для всех вновь создаваемых таблиц в схем.  
select * from t1; - выдает ERROR:  relation "t1" does not exist  
select * from testnm.t1; - работает, значит проблема в search_path  
SHOW search_path; - выдает "$user", public, поэтому PG не находит таблицу в иной схеме  
> 34. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
> 35. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
> 36. есть идеи как убрать эти права? если нет - смотрите шпаргалку
> 37. если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
> 38. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
> 39. расскажите что получилось и почему
```
create table t2(c1 integer); insert into t2 values (2);
\q
sudo -u postgres psql
\c testdb
REVOKE CREATE on SCHEMA public FROM public; 
REVOKE ALL on DATABASE testdb FROM public; 
\q
psql -U testread -h localhost -d testdb
create table t3(c1 integer); insert into t3 values (2);
```
Таблица создается в схеме public, а права на нее есть у всех пользователей по умолчанию.  
Командой REVOKE CREATE мы убрали права у роли public на создание таблиц в схеме public.  
Второй командой REVOKE ALL мы убрали права у роли public на все операции в базе testdb.  
При создании таблиц теперь получаем ошибку  
ERROR:  permission denied for schema public  
т.к. права на создание таблиц отсутствуют.
    