# Домашнее задание
## Установка и настройка PostgteSQL в контейнере Docker
### Цель:
* установить PostgreSQL в Docker контейнере
* настроить контейнер для внешнего подключения
---
Создаем ВМ с Ubuntu 22.04 в Parallels Desktop.
Устанавливаем Docker согласно документации https://docs.docker.com/engine/install/ubuntu/
```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Создаем каталог /var/lib/postgres:
```
sudo mkdir /var/lib/postgres
```
Создаем сеть pg-net и разворачиваем контейнер с образом PostgreSQL 15 (postgres:15), пробрасываем из контейнера порт 5432 и монтируем папку /var/lib/postgres внутрь контейнера:
```
sudo docker network create pg-net
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Разворачиваем контейнер с образом postgres:15 и сразу запускаем в нем утилиту psql с подключением к контейнеру с сервером:
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```
Создаем базу и таблицу с парой строк:
```
CREATE DATABASE test;
\c test
CREATE TABLE people (name varchar, surname varchar);
INSERT INTO people (name, surname) VALUES ('Ivan', 'Petrov');
INSERT INTO people (name, surname) VALUES ('Petr', 'Sidorov');
\q
```
Проверяем подключение к серверу извне виртуальной машины, используя ее ip-адрес:
```
psql -h 10.211.55.11 -U postgres
```
Останавливаем и удаляем контейнер:
```
sudo docker stop pg-server
sudo docker rm pg-server
```
Создаем снова контейнер с сервером:
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Подключаемся снова из контейнера с клиентом к контейнеру с сервером:
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```
Подключаемся к базе и проверяем, что данные на месте:
```
\c test
SELECT * FROM people;
```
