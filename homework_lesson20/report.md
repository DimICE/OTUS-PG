# Домашнее задание
## Секционирование таблицы
### Цель:
* научиться секционировать таблицы.
---

> Секционировать большую таблицу из демо базы flights

Скачиваем демо-базу flights: https://edu.postgrespro.com/demo-big-en.zip  
В скрипт создания таблицы flights добавим секционирование по диапазону по полю scheduled_departure:
```
CREATE TABLE flights (
    flight_id integer NOT NULL,
    flight_no character(6) NOT NULL,
    scheduled_departure timestamp with time zone NOT NULL,
    scheduled_arrival timestamp with time zone NOT NULL,
    departure_airport character(3) NOT NULL,
    arrival_airport character(3) NOT NULL,
    status character varying(20) NOT NULL,
    aircraft_code character(3) NOT NULL,
    actual_departure timestamp with time zone,
    actual_arrival timestamp with time zone,
    CONSTRAINT flights_check CHECK ((scheduled_arrival > scheduled_departure)),
    CONSTRAINT flights_check1 CHECK (((actual_arrival IS NULL) OR ((actual_departure IS NOT NULL) AND (actual_arrival IS NOT NULL) AND (actual_arrival > actual_departure)))),
    CONSTRAINT flights_status_check CHECK (((status)::text = ANY (ARRAY[('On Time'::character varying)::text, ('Delayed'::character varying)::text, ('Departed'::character varying)::text, ('Arrived'::character varying)::text, ('Scheduled'::character varying)::text, ('Cancelled'::character varying)::text])))
) partition by range (scheduled_departure);
```

После создания таблицы создадим партиции (одну по умолчанию и по одной на каждый год):
```
CREATE TABLE flights_default partition of flights default;
CREATE TABLE flights_2016 partition of flights for values from ('2016-01-01') to ('2017-01-01');
CREATE TABLE flights_2017 partition of flights for values from ('2017-01-01') to ('2018-01-01');
CREATE TABLE flights_2018 partition of flights for values from ('2018-01-01') to ('2019-01-01');
```

Импортируем файл в БД:
```
psql -h localhost -U test
\i /Users/dimice/Downloads/demo-big-en-20170815.sql
```

Проверим, что партиции создались и используются при запросах:
```
select * from flights_2016 limit 100;
select * from flights_2017 limit 100;
explain analyze select * from flights where scheduled_departure between '2017-02-02' and '2017-02-05';
```
Видим, что данные в партициях есть, а при обращении к основной таблице с фильтром по дате отправления видим, что обращение идет к нужной партиции:
```
Seq Scan on flights_2017 flights  (cost=0.00..3792.94 rows=1693 width=63) (actual time=0.141..21.785 rows=1612 loops=1)
Filter: ((scheduled_departure >= '2017-02-02 00:00:00+00'::timestamp with time zone) AND (scheduled_departure <= '2017-02-05 00:00:00+00'::timestamp with time zone))
Rows Removed by Filter: 137851
Planning Time: 0.212 ms
Execution Time: 21.881 ms
(5 rows)
```