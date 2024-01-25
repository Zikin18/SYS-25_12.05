# Домашнее задание к занятию «Индексы» - Юрий Белобородов


### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

Ответ:

```
SELECT 
	SUM(t.index_length) / (SUM(t.data_length) + SUM(t.index_length)) * 100 AS percentage 
FROM information_schema.TABLES t
WHERE t.table_schema = 'sakila'
```

![1-1.png](https://github.com/Zikin18/SYS-25_12.05/blob/master/1-1.png)


### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

Ответ:

Вывод explain analyze
```
-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=5480..5480 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=5480..5480 rows=391 loops=1)
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=2183..5272 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=2182..2248 rows=642000 loops=1)
                -> Stream results  (cost=22.1e+6 rows=16.3e+6) (actual time=0.341..1680 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=22.1e+6 rows=16.3e+6) (actual time=0.336..1409 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=20.5e+6 rows=16.3e+6) (actual time=0.333..1245 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=18.8e+6 rows=16.3e+6) (actual time=0.329..1062 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1.61e+6 rows=16.1e+6) (actual time=0.318..60.7 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.68 rows=16086) (actual time=0.0274..7.47 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.68 rows=16086) (actual time=0.0186..5.04 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=112 rows=1000) (actual time=0.0334..0.207 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.969 rows=1.01) (actual time=0.00101..0.00144 rows=1.01 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=250e-6 rows=1) (actual time=143e-6..162e-6 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=250e-6 rows=1) (actual time=119e-6..139e-6 rows=1 loops=642000)
```

![2_slow.png](https://github.com/Zikin18/SYS-25_12.05/blob/master/2_slow.png)

Из вывода explain analyze видно что таблицы `payment`, `rental`, `inventory`, `customer` циклически связываются между собой `Nested loop inner join` с очень большим колличеством повторений `loops=642000` при этом происходит связывание каждой строки каждой таблици со строками в остальных таблицах, что приводит к декартовому соединению таблиц. При этом в таблице `rental` уже есть ссылка на `customer`, и так как прочие таблицы не участвуют в фильтрации и в результирующей выборке значит можно от них избавиться. Теперь можно заменить `distinct` на `GROUP BY`, а таблицу `payment` присоединить по индексу `fk_payment_customer` что в итоге позволит получить сумму платежей обычной агрегатной функцией, без использования оконной функции, так как строки больше не повторяются.

Запрос после оптимизации:
```
explain analyze 
SELECT
	concat(c.last_name, ' ', c.first_name), 
	sum(p.amount)
FROM customer c
JOIN payment p ON p.customer_id = c.customer_id
WHERE date(p.payment_date) = '2005-07-30'
GROUP BY c.customer_id
```

Explain analyze после оптимизации
```
-> Group aggregate: sum(p.amount)  (cost=7300 rows=599) (actual time=0.129..15.3 rows=391 loops=1)
    -> Nested loop inner join  (cost=5691 rows=16086) (actual time=0.0996..15 rows=634 loops=1)
        -> Index scan on c using PRIMARY  (cost=61.2 rows=599) (actual time=0.0339..0.236 rows=599 loops=1)
        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=6.72 rows=26.9) (actual time=0.0223..0.0245 rows=1.06 loops=599)
            -> Index lookup on p using idx_fk_customer_id (customer_id=c.customer_id)  (cost=6.72 rows=26.9) (actual time=0.0193..0.0224 rows=26.8 loops=599)
```

Исходный запрос выполянялся порядка 4 сек, оптимизированный стал выполнятся ~0.02 сек

![2_faster.png](https://github.com/Zikin18/SYS-25_12.05/blob/master/2_faster.png)
