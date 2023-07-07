# Домашнее задание
## Триггеры, поддержка заполнения витрин
### Цель:
* Создать триггер для поддержки витрины в актуальном состоянии.
---

> Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ  
> В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).  
> Есть запрос для генерации отчета – сумма продаж по каждому товару.  
> БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.  
> Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)  
> Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE  

Создаем таблицы из файла hw_triggers.sql (goods, sales, good_sum_mart).
Проверим работу отчета и заполним витрину первичными данными:
```
INSERT INTO sum_sale
SELECT G.good_name, sum(G.good_price * S.sales_qty) as sum_sale
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

SELECT * FROM sum_sale;
```

| good_name                | sum_sale     |
|--------------------------|--------------|
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозяйственные     | 65.5         |

Напишем функцию для наполнения витрины (good_sum_mart) и сделаем ее вызов в триггерах:
```
CREATE OR REPLACE FUNCTION tf_edit_sales()
RETURNS trigger
AS
$$
DECLARE
    row_dbg record;
BEGIN
    CASE TG_OP
        WHEN 'INSERT' THEN
            UPDATE
                sum_sale S
            SET
                sum_sale = sum_sale + (G.good_price * NEW.sales_qty)
            FROM
                goods G
            WHERE
                S.good_name = G.good_name
                AND G.goods_id = NEW.good_id;

            RETURN NEW;

        WHEN 'DELETE' THEN
            UPDATE
                sum_sale S
            SET
                sum_sale = sum_sale - (G.good_price * OLD.sales_qty)
            FROM
                goods G
            WHERE
                S.good_name = G.good_name
                AND G.goods_id = OLD.good_id;

            RETURN OLD;

        WHEN 'UPDATE' THEN
            UPDATE
                sum_sale S
            SET
                sum_sale = sum_sale - (G.good_price * OLD.sales_qty)
            FROM
                goods G
            WHERE
                S.good_name = G.good_name
                AND G.goods_id = OLD.good_id;

            UPDATE
                sum_sale S
            SET
                sum_sale = sum_sale + (G.good_price * NEW.sales_qty)
            FROM
                goods G
            WHERE
                S.good_name = G.good_name
                AND G.goods_id = NEW.good_id;

            RETURN NEW;
    END CASE;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_edit_sales
AFTER INSERT OR UPDATE OR DELETE
ON sales
FOR EACH ROW
EXECUTE FUNCTION tf_edit_sales();
```

Произведем несколько покупок (insert), изменений покупок (update) и отмен (delete) в таблице sales и проверим, каков результат получен в витрине:
```
INSERT INTO sales (good_id, sales_qty) VALUES (1, 12300), (1, 32100), (2, 1);
UPDATE sales SET sales_qty = 10 WHERE sales_qty = 12300;
DELETE FROM sales WHERE sales_qty = 32100;

SELECT * FROM sum_sale;
```

| good_name                | sum_sale     |
|--------------------------|--------------|
| Автомобиль Ferrari FXX K | 370000000.02 |
| Спички хозяйственные     | 70.50        |

 Видим, что в витрину, как и ожидалось, добавилась покупка одного автомобиля и покупка десяти спичек, т.е. необходимый результат достигнут.

> Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?  
> Подсказка: В реальной жизни возможны изменения цен.  

При использовании такой витрины нам не нужно хранить отдельно историю цен и учитывать её при формировании отчета, в витрину будет записываться актуальная цена товара на момент срабатывания триггера.