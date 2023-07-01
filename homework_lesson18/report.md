# Домашнее задание
## Работа с индексами, join'ами, статистикой
### Цель:
* знать и уметь применять основные виды индексов PostgreSQL
* строить и анализировать план выполнения запроса 
* уметь оптимизировать запросы для с использованием индексов 
* знать и уметь применять различные виды join'ов 
* строить и анализировать план выполенения запроса 
* оптимизировать запрос 
* уметь собирать и анализировать статистику для таблицы
---

Создаем БД для задания из 4 таблиц (каталог книг)
- книги (id книги, ид автора, наименование, год издания, признак удаления)
- авторы (id автора, фио, признак удаления)
```
CREATE TABLE book (book_id bigint, author_id bigint, book_name text, book_year int, book_deleted boolean);
CREATE TABLE author (author_id bigint, author_name text, author_deleted boolean);
INSERT INTO author VALUES (1, 'Пушкин А.С.', false);
INSERT INTO author VALUES (2, 'Лермонтов М.Ю.', false);
INSERT INTO author VALUES (3, 'Толстой Л.Н.', false);
INSERT INTO author VALUES (4, 'Иванов И.И.', false);
INSERT INTO book VALUES (1, 1, 'Сказка о царе Салтане', 2017, false);
INSERT INTO book VALUES (2, 1, 'Евгений Онегин', 2018, false);
INSERT INTO book VALUES (3, 3, 'Война и мир', 2018, false);
INSERT INTO book SELECT i, 2, 'Книга №' || i::text, 2000, false FROM generate_series(4, 100) AS t(i);
INSERT INTO book SELECT i, 4, 'Книга №' || i::text, 2000, false FROM generate_series(1, 1000) AS t(i); 
```

1 вариант:  
> 1. Создать индекс к какой-либо из таблиц вашей БД 
> 2. Прислать текстом результат команды explain, в которой используется данный индекс

Создаем индекс в таблице с книгами по идентификатору автора и делаем выборку:
```
CREATE INDEX book_author_id_index ON book (author_id);
EXPLAIN SELECT * FROM book WHERE author_id = 2;
```
Видим, что индекс используется при выборке:  
```
Index Scan using book_author_id_index on book  (cost=0.15..9.85 rows=97 width=38)  
Index Cond: (author_id = 2)  
```

> 3. Реализовать индекс для полнотекстового поиска

Создаем GIN индекс в таблице с книгами по наименованию книги и делаем выборку:
```
CREATE INDEX book_name_index ON book USING GIN (to_tsvector('russian', book_name));
EXPLAIN SELECT * FROM book WHERE to_tsvector('russian', book_name) @@ to_tsquery('russian', 'онегин');
```
Видим, что индекс используется при выборке:
```
Bitmap Heap Scan on book  (cost=8.04..19.50 rows=6 width=38)  
  Recheck Cond: (to_tsvector('russian'::regconfig, book_name) @@ '''онегин'''::tsquery)
  ->  Bitmap Index Scan on book_name_index  (cost=0.00..8.04 rows=6 width=0)
        Index Cond: (to_tsvector('russian'::regconfig, book_name) @@ '''онегин'''::tsquery)  
```

> 4. Реализовать индекс на часть таблицы или индекс на поле с функцией 

Создаем индекс по году издания книги на часть таблицы (только не удаленные записи):
```
CREATE INDEX book_year_index ON book (book_year) WHERE book_deleted = false;
EXPLAIN SELECT * FROM book WHERE book_year = 2018 and book_deleted = false;
```
Видим, что индекс используется при выборке:
```
Index Scan using book_year_index on book  (cost=0.15..8.32 rows=2 width=38)
  Index Cond: (book_year = 2018)  
```

> 5. Создать индекс на несколько полей

Создаем индекс по автору и году издания книги и делаем выборку:
```
CREATE INDEX book_author_id_book_year_index ON book (author_id, book_year);
EXPLAIN SELECT * FROM book WHERE author_id = 2 and book_year = 2018;
```
Видим, что индекс используется при выборке:
```
Index Scan using book_author_id_book_year_index on book  (cost=0.15..8.17 rows=1 width=38)
  Index Cond: ((author_id = 2) AND (book_year = 2018))  
```

2 вариант:  
>  1. Реализовать прямое соединение двух или более таблиц
>  2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц 
>  3. Реализовать кросс соединение двух или более таблиц 
>  4. Реализовать полное соединение двух или более таблиц 
>  5. Реализовать запрос, в котором будут использованы разные типы соединений 
>  6. Сделать комментарии на каждый запрос 
>  7. К работе приложить структуру таблиц, для которых выполнялись соединения
