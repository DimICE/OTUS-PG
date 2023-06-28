# Домашнее задание
## Нагрузочное тестирование и тюнинг PostgreSQL
### Цель:
* сделать нагрузочное тестирование PostgreSQL
* настроить параметры PostgreSQL для достижения максимальной производительности
---

> • развернуть виртуальную машину любым удобным способом  
> • поставить на неё PostgreSQL 15 любым способом  
> • настроить кластер PostgreSQL 15 на максимальную производительность не
обращая внимание на возможные проблемы с надежностью в случае
аварийной перезагрузки виртуальной машины
> • нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)  
> • написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему

Для тестирования используем виртуальную машину с ОС Ubuntu в Parallels Desktop: 2 ядра, 2 гб памяти, ssd диск  
Устанавливаем PostgreSQL 15 и проводим первичное тестирование с помощью pgbench:
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
sudo su postgres
pgbench -i postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
Производительность до настойки:  
tps = 3735.676896

С помощью утилиты https://pgtune.leopard.in.ua определяем оптимальные настройки и применяем их, после чего проводим повторное тестирование:
```
sudo -u postgres psql
ALTER SYSTEM SET
 max_connections = '200';
ALTER SYSTEM SET
 shared_buffers = '512MB';
ALTER SYSTEM SET
 effective_cache_size = '1536MB';
ALTER SYSTEM SET
 maintenance_work_mem = '128MB';
ALTER SYSTEM SET
 checkpoint_completion_target = '0.9';
ALTER SYSTEM SET
 wal_buffers = '16MB';
ALTER SYSTEM SET
 default_statistics_target = '100';
ALTER SYSTEM SET
 random_page_cost = '1.1';
ALTER SYSTEM SET
 effective_io_concurrency = '200';
ALTER SYSTEM SET
 work_mem = '1310kB';
ALTER SYSTEM SET
 min_wal_size = '1GB';
ALTER SYSTEM SET
 max_wal_size = '4GB';
select pg_reload_conf();
\q
sudo pg_ctlcluster 15 main restart
sudo su postgres
pgbench -i postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
Производительность после настойки:  
tps = 3792.549559  
Видим, что производительность практически не изменилась.  
Настроим еще 4 параметра, которые могут повлиять на надежность системы:    
synchronous_commit = off  - pg не дожидается записи WAL на диск  
fsync = off  - pg не выполняет системные вызовы fsync  
full_page_writes = off  - pg не производит запись страниц в WAL при первом их изменении     
checkpoint_timeout = 24h  - таймаут чекпоинтов сильно увеличен  
```
sudo -u postgres psql
ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM SET fsync = off;
ALTER SYSTEM SET full_page_writes = off;
ALTER SYSTEM SET checkpoint_timeout = 24h;
\q
sudo pg_ctlcluster 15 main restart
sudo su postgres
pgbench -i postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
После применения данных настроек производительность существенно выросла:  
tps = 6590.017616  
Это объясняется тем, что PG при данных настройках выполняет меньше операций с диском и не дожидается ответа при записи на диск.