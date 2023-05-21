# Домашнее задание
## Механизм блокировок
### Цель:
* понимать как работает механизм блокировок объектов и строк
---
> Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

Включаем логирование ожиданий блокировок (log_lock_waits), устанавливаем таймаут после которого пишется сообщение в лог (deadlock_timeout) и перечитываем конфиг (pg_reload_conf).  
Стартуем транзакцию и накладываем блокировку на строку с помощью update:
```
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = 200;
SELECT pg_reload_conf();
BEGIN;
UPDATE accounts SET amount = 500 WHERE acc_no = 1;
```
В новой сессии делаем update той же строки:
```
UPDATE accounts SET amount = 500 WHERE acc_no = 1;
```
Смотрим журнал в /var/log/postgresql/postgresql-15-main.log и видим сообщение об ожидании блокировки:
```
2023-05-21 11:18:23.047 UTC [5242] postgres@locks СООБЩЕНИЕ:  процесс 5242 продолжает ожидать в режиме ShareLock блокировку "транзакция 739" в течение 203.576 мс
2023-05-21 11:18:23.047 UTC [5242] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 4820. Wait queue: 5242.
2023-05-21 11:18:23.047 UTC [5242] postgres@locks КОНТЕКСТ:  при изменении кортежа (0,4) в отношении "accounts"
2023-05-21 11:18:23.047 UTC [5242] postgres@locks ОПЕРАТОР:  UPDATE accounts SET amount = 500 WHERE acc_no = 1;
```
> Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

В сессиях стартуем транзакцию и выполняем update одной и той же строки:
```
BEGIN;
UPDATE accounts SET amount = 500 WHERE acc_no = 1;
```
В представлении pg_locks получаем:
```
 pid  |   locktype    |   lockid   |       mode       | granted 
------+---------------+------------+------------------+---------
 4820 | relation      | accounts   | RowExclusiveLock | t
 4820 | transactionid | 741        | ExclusiveLock    | t
 5310 | tuple         | accounts:6 | ExclusiveLock    | t
 5310 | transactionid | 741        | ShareLock        | f
 5310 | transactionid | 742        | ExclusiveLock    | t
 5310 | relation      | accounts   | RowExclusiveLock | t
 5381 | tuple         | accounts:6 | ExclusiveLock    | f
 5381 | transactionid | 743        | ExclusiveLock    | t
 5381 | relation      | accounts   | RowExclusiveLock | t
```
Все три сессии захватили эксклюзивную блокировку (ExclusiveLock) на номер собственной транзакции (transactionid) и эксклюзивную блокировку RowExclusiveLock на строку таблицы accounts.  
Помимо этого:  
Вторая сессия (5310) не смогла получить ShareLock номера первой транзакции и создала эксклюзивную блокировку кортежа (tuple) и находится в режиме ожидания.  
Третья сессия (5381) не смогла получить эксклюзивную блокировку на кортеж (tuple) и находится в режиме ожидания.  
Сообщения об этом видно и в логе:  
```
2023-05-21 11:24:26.558 UTC [5310] postgres@locks СООБЩЕНИЕ:  процесс 5310 продолжает ожидать в режиме ShareLock блокировку "транзакция 741" в течение 200.704 мс
2023-05-21 11:24:28.608 UTC [5381] postgres@locks СООБЩЕНИЕ:  процесс 5381 продолжает ожидать в режиме ExclusiveLock блокировку "кортеж (0,6) отношения 16389 базы данных 16388" в течение 201.807 мс
```
> Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

В первой сессии стартуем транзакцию и выполняем update первой строки:
```
BEGIN;
UPDATE accounts SET amount = 500 WHERE acc_no = 1;
```
Во второй сессии стартуем транзакцию и выполняем update второй строки и затем первой строки:
```
BEGIN;
UPDATE accounts SET amount = 500 WHERE acc_no = 2;
UPDATE accounts SET amount = 500 WHERE acc_no = 1;
```
В третьей сессии стартуем транзакцию и выполняем update третьей строки и затем второй строки:
```
BEGIN;
UPDATE accounts SET amount = 500 WHERE acc_no = 3;
UPDATE accounts SET amount = 500 WHERE acc_no = 2;
```
Возвращаемся в первую сессию и выполняем update третьей строки:
```
UPDATE accounts SET amount = 500 WHERE acc_no = 3;
```
И получаем ошибку о взаимоблокировке.  
Изучая журнал сообщений мы можем увидеть из-за чего получилась взаимоблокировка:
```
2023-05-21 12:19:25.498 UTC [4820] postgres@locks СООБЩЕНИЕ:  процесс 4820 обнаружил взаимоблокировку, ожидая в режиме ShareLock блокировку "транзакция 747" в течение 205.901 мс
2023-05-21 12:19:25.498 UTC [4820] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 5381. Wait queue: .
2023-05-21 12:19:25.498 UTC [4820] postgres@locks КОНТЕКСТ:  при изменении кортежа (0,3) в отношении "accounts"
2023-05-21 12:19:25.498 UTC [4820] postgres@locks ОПЕРАТОР:  UPDATE accounts SET amount = 500 WHERE acc_no = 3;
2023-05-21 12:19:25.498 UTC [4820] postgres@locks ОШИБКА:  обнаружена взаимоблокировка
2023-05-21 12:19:25.498 UTC [4820] postgres@locks ПОДРОБНОСТИ:  Процесс 4820 ожидает в режиме ShareLock блокировку "транзакция 747"; заблокирован процессом 5381.
	Процесс 5381 ожидает в режиме ShareLock блокировку "транзакция 746"; заблокирован процессом 5310.
	Процесс 5310 ожидает в режиме ShareLock блокировку "транзакция 745"; заблокирован процессом 4820.
	Процесс 4820: UPDATE accounts SET amount = 500 WHERE acc_no = 3;
	Процесс 5381: UPDATE accounts SET amount = 500 WHERE acc_no = 2;
	Процесс 5310: UPDATE accounts SET amount = 500 WHERE acc_no = 1;
```
Т.е. второй процесс ждал первого, третий ждал второго, первый начал ждать третьего.

> Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Могут, если транзакции будут выполнять update строк в разном порядке.

> Задание со звездочкой* Попробуйте воспроизвести такую ситуацию.

В 1 сессии:
```
truncate accounts;
insert into accounts (acc_no) (SELECT generate_series(1, 9999999));
BEGIN;
UPDATE accounts SET amount = (select acc_no from accounts order by acc_no asc limit 1 for update);
```

Во 2 сессии:
```
BEGIN;
UPDATE accounts SET amount = (select acc_no from accounts order by acc_no desc limit 1 for update);
```
Получаем ошибку
```
ПОДРОБНОСТИ:  Процесс 4820 ожидает в режиме ShareLock блокировку "транзакция 772"; заблокирован процессом 5310.
Процесс 5310 ожидает в режиме ShareLock блокировку "транзакция 771"; заблокирован процессом 4820.
ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
КОНТЕКСТ:  при изменении кортежа (44247,177) в отношении "accounts"
```