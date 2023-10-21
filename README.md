### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

```
SELECT CONCAT(ROUND(SUM(INDEX_LENGTH) / (SUM(INDEX_LENGTH)+SUM(DATA_LENGTH)) * 100), ' %') AS percent
FROM INFORMATION_SCHEMA.TABLES;
```

<img width="326" alt="Снимок экрана 2023-10-21 в 20 01 33" src="https://github.com/otuzi/index/assets/61628386/bf0b0772-caed-4278-ba93-9af791a870bd">


### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;

В выводе видим что мы сортируем по 2-м большим таблицам и проходим по большому количеству строк. Так как дополнительная сортировка по названию фильмов нигде больше не используется, то уберем ее и таблицу film. Еще отсутствует необходимость в таблице inventory и проверке условия inventory_id

```
EXPLAIN ANALYZE select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id
```
```
-> Limit: 400 row(s)  (cost=0..0 rows=0) (actual time=14.9..15.1 rows=391 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=14.9..15 rows=391 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=14.9..14.9 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id )   (actual time=13.2..14.6 rows=642 loops=1)
                -> Sort: c.customer_id  (actual time=13.1..13.2 rows=642 loops=1)
                    -> Stream results  (cost=23632 rows=16003) (actual time=0.127..12.9 rows=642 loops=1)
                        -> Nested loop inner join  (cost=23632 rows=16003) (actual time=0.12..12.4 rows=642 loops=1)
                            -> Nested loop inner join  (cost=18031 rows=16003) (actual time=0.11..11.4 rows=642 loops=1)
                                -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1606 rows=15813) (actual time=0.0887..9.33 rows=634 loops=1)
                                    -> Table scan on p  (cost=1606 rows=15813) (actual time=0.0688..6.96 rows=16044 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.938 rows=1.01) (actual time=0.0021..0.00286 rows=1.01 loops=634)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.00126..0.0013 rows=1 loops=642)
```

Основное назначение OVER PARTITION BY - это группировка по значению для sum, такая группировка оставляет все строки, GROUP BY сокращает количество строк в запросе с помощью их группировки, попробуем использовать GROUP BY

```
EXPLAIN ANALYZE select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id
GROUP BY c.customer_id;
```

---

- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

```
CREATE INDEX inx_pay_date ON payment(payment_date);

EXPLAIN ANALYZE select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, customer c
where payment_date >= '2005-07-30' and payment_date < '2005-07-31' and p.customer_id = c.customer_id 
GROUP BY p.customer_id;
```
```
-> Limit: 400 row(s)  (actual time=4.11..4.2 rows=391 loops=1)
    -> Sort with duplicate removal: `concat(c.last_name, ' ', c.first_name)`, `sum(p.amount)`  (actual time=4.11..4.16 rows=391 loops=1)
        -> Table scan on <temporary>  (actual time=3.72..3.8 rows=391 loops=1)
            -> Aggregate using temporary table  (actual time=3.72..3.72 rows=391 loops=1)
                -> Nested loop inner join  (cost=507 rows=634) (actual time=0.05..2.91 rows=634 loops=1)
                    -> Index range scan on p using inx_pay_date over ('2005-07-30 00:00:00' <= payment_date < '2005-07-31 00:00:00'), with index condition: ((p.payment_date >= TIMESTAMP'2005-07-30 00:00:00') and (p.payment_date < TIMESTAMP'2005-07-31 00:00:00'))  (cost=286 rows=634) (actual time=0.0385..1.7 rows=634 loops=1)
                    -> Single-row index lookup on c using PRIMARY (customer_id=p.customer_id)  (cost=0.25 rows=1) (actual time=0.00163..0.00167 rows=1 loops=634)
```

<img width="814" alt="Снимок экрана 2023-10-21 в 20 02 52" src="https://github.com/otuzi/index/assets/61628386/45fab448-d152-4db3-8a22-4cf4a3bc47c7">
