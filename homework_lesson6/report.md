# Домашнее задание
## Установка и настройка PostgreSQL
### Цель:
* создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
* переносить содержимое базы данных PostgreSQL на дополнительный диск
* переносить содержимое БД PostgreSQL между виртуальными машинами
---
> создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере  
> поставьте на нее PostgreSQL 15 через sudo apt  
> проверьте что кластер запущен через sudo -u postgres pg_lsclusters  
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
sudo -u postgres pg_lsclusters
```
> зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
> остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
```
sudo -u postgres psql
create table test(c1 text);
insert into test values('1');
\q
sudo -u postgres pg_ctlcluster 15 main stop
```  
> создайте новый диск к ВМ размером 10GB  
> добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk  
> проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux  
> перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
```
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
sudo mkdir -p /mnt/data
sudo mount -o defaults /dev/sdb /mnt/data
sudo nano /etc/fstab
UUID=db2964cd-85df-4925-8770-7ee065c0418d /mnt/data ext4 defaults 0 2
sudo reboot
```
> сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/  
> перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data  
> попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
> напишите получилось или нет и почему  
```
sudo chown -R postgres:postgres /mnt/data/
sudo mv /var/lib/postgresql/15 /mnt/data
sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
Запустить не удалось, т.к. переместили директорию с данными в новое место.
> задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его  
> напишите что и почему поменяли  
> попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start  
> напишите получилось или нет и почему
```
cd /etc/postgresql/15/main
sudo nano postgresql.conf
data_directory = '/mnt/data/main';
sudo -u postgres pg_ctlcluster 15 main start
```
В postgresql.conf в параметре data_directory поменяли расположение директории с данными, после чего кластер успешно запустился.
> зайдите через через psql и проверьте содержимое ранее созданной таблицы
```
sudo -u postgres psql
select * from test;
```  
Данные на месте.


> задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
```  
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
sudo -u postgres pg_ctlcluster 15 main stop
sudo rm -rf /var/lib/postgresql
```  
отключаем диск от первого инстанса, подключаем ко второму и монтируем его в нужную директорию
```  
sudo mount -o defaults /dev/sdb /var/lib/postgresql
sudo -u postgres pg_ctlcluster 15 main start
sudo -u postgres psql
select * from test;
```  
Данные на месте.