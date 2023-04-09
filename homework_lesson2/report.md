# Домашнее задание
## Работа с уровнями изоляции транзакции в PostgreSQL
### Цель:
* научиться работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)
* научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

> создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере   
далее создать инстанс виртуальной машины с дефолтными параметрами  
добавить свой ssh ключ в metadata ВМ  
зайти удаленным ssh (первая сессия), не забывайте про ssh-add  
поставить PostgreSQL  
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```
> зайти вторым ssh (вторая сессия)  
запустить везде psql из под пользователя postgres  
выключить auto commit
```
sudo -u postgres psql
\set AUTOCOMMIT OFF
\echo :AUTOCOMMIT
```
> сделать в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;  
посмотреть текущий уровень изоляции: show transaction isolation level  
начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции  
в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');  
сделать select * from persons во второй сессии  
видите ли вы новую запись и если да то почему?  

**ОТВЕТ:**  
Запись не видно.
По умолчанию в PostgreSQL уровень изоляции транзакций read committed, с этим уровнем внутри транзакции не видно изменения из незавершенных транзакций. 
> завершить первую транзакцию - commit;  
сделать select * from persons во второй сессии  
видите ли вы новую запись и если да то почему?  

**ОТВЕТ:**  
Запись видно.
Первая транзакция была завершена, при уровне изоляции транзакций read committed внутри транзакции видно зафиксированные изменения из других транзакций.
> завершите транзакцию во второй сессии  
начать новые но уже repeatable read транзации
``` 
set transaction isolation level repeatable read;
```  
> в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');  
сделать select * from persons во второй сессии  
видите ли вы новую запись и если да то почему?  

**ОТВЕТ:**  
Запись не видно.
При уровне изоляции транзакций repeatable read так же, как и при read committed не видно записи из незавершенных транзакций.
> завершить первую транзакцию - commit;  
сделать select * from persons во второй сессии  
видите ли вы новую запись и если да то почему?  

**ОТВЕТ:**  
Запись не видно.
При уровне изоляции транзакций repeatable read отсутствуют феномены неповторяющегося чтения и фантомного чтения, поэтому изменения из завершенных транзакций так же не видно. 
> завершить вторую транзакцию  
сделать select * from persons во второй сессии  
видите ли вы новую запись и если да то почему?  

**ОТВЕТ:** 
Запись видно.
Транзакция завершена, вне транзакций видно все изменения по всем завершенным транзакциям.
