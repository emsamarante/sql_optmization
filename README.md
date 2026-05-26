# SQL Query Optimization for Product Analytics

## Introduction

In data-driven companies, product and business teams depend on fast, reliable, and scalable analytical queries to monitor performance, identify opportunities, and support decision-making. However, as datasets grow and business questions become more complex, poorly optimized SQL queries can become slow, expensive, and difficult to maintain. This project focuses on improving the performance of analytical SQL queries while preserving the accuracy and business meaning of the results.

The main objective of this project is to answer basic and intermediate business questions using SQL, analyze the performance of the initial queries, and then improve them through query optimization techniques. For each business question, a baseline query is created first. After that, the query is evaluated using PostgreSQL execution plans and optimized through better filtering, joins, indexing strategies, aggregation logic, and analytical modeling when appropriate. The final goal is to compare the performance before and after optimization and document the impact of each improvement.

This project uses the **Brazilian E-Commerce Public Dataset by Olist**, a public dataset that contains real e-commerce marketplace data from Brazil. The dataset includes information about orders, customers, products, sellers, payments, reviews, and delivery details. Because the data is distributed across multiple relational tables, it provides a realistic environment for practicing SQL joins, aggregations, date analysis, revenue calculations, customer behavior analysis, and marketplace performance metrics.

Optimized queries are especially important in Product Analytics because many product decisions depend on recurring analyses, dashboards, and ad-hoc investigations. A query that works well on a small dataset may become inefficient when used on larger tables, repeated daily, or connected to a dashboard used by business stakeholders. Slow queries can delay decision-making, increase computational costs, and reduce trust in analytical workflows. By improving query performance, data professionals can make insights more accessible, dashboards more responsive, and analytical processes more reliable.

This project demonstrates not only the ability to write correct SQL queries, but also the ability to think about performance, scalability, and analytical usability. The focus is not on database administration, but on applying practical SQL optimization techniques to real business questions in a product analytics context.

## Solving Business Questions

**_Question 1: Which product categories generated the highest total revenue in 2018, considering only delivered orders?_**

> Part 1: Solution of business question

```sql
SELECT
    p.product_category_name,
    SUM(oi.price) AS total_revenue,
    COUNT(DISTINCT o.order_id) AS number_delivered_orders,
    COUNT(oi.order_item_id) AS number_sold_items,
    ROUND(SUM(oi.price) / COUNT(DISTINCT o.order_id), 2) AS avg_revenue_per_order
FROM olist.orders o
JOIN olist.order_items oi
    ON o.order_id = oi.order_id
JOIN olist.products p
    ON oi.product_id = p.product_id
WHERE
    o.order_status = 'delivered'
    AND o.order_delivered_customer_date >= DATE '2018-01-01'
    AND o.order_delivered_customer_date < DATE '2019-01-01'
GROUP BY
    p.product_category_name
ORDER BY
    total_revenue DESC;
```

> Parte 2: Optimizing the query

Looking at the plan of execution using the following command:

```sql
EXPLAIN (ANALYZE, BUFFERS ) 'my query'
```

I got `Planning Time: 0.817 ms` and `Execution Time: 366.344 ms`.

Now, I am going to starting its optimization:

> First, let`s start with the indexes creation.
>
> - Table olist.order: Creating a compose index to filtering columns

```sql
CREATE INDEX idx_orders_status_date ON olist.orders(order_status, order_delivered_customer_date);
```

I got `Planning Time: 0.892 ms` and `Execution Time: 361.326 ms`.

> - Creating index to foreign keys used on joins:

```sql
CREATE INDEX idx_oi_order_product ON olist.order_items(order_id, product_id);
```

I got `Planning Time: 1.102 ms` and `Execution Time: 367.542 ms`. So, as we can see, the result is worse than before, that's why we need to `DROP`this index.

> - Creating index to grouping column:

```sql
CREATE INDEX idx_products_category ON olist.products(product_id, product_category_name);
```

The creation of this index became the performance worse than with only one index.

**_Question 2: What is the monthly trend of delivered orders and product revenue in 2018?_**

```sql
SELECT
	COUNT(o.order_id) AS number_delivered_orders
FROM olist.orders o
WHERE order_status = 'delivered';


SELECT
	COUNT(o.order_id) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
WHERE order_status = 'delivered';

SELECT
	COUNT(o.order_id) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
WHERE o.order_status = 'delivered'
	AND DATE_PART('year', o.order_purchase_timestamp) = 2018;


SELECT
	COUNT(o.order_id) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND DATE_PART('year', o.order_purchase_timestamp) = 2018;

SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(o.order_id) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND DATE_PART('year', o.order_purchase_timestamp) = 2018
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;



SELECT
    DATE_PART('month', o.order_purchase_timestamp) AS order_month,
    COUNT(DISTINCT o.order_id) AS number_delivered_orders,
    COUNT(oi.order_item_id) AS number_sold_items,
    SUM(oi.price) AS revenue,
    ROUND(SUM(oi.price) / COUNT(DISTINCT o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
    ON o.order_id = oi.order_id
WHERE
    o.order_status = 'delivered'
    AND DATE_PART('year', o.order_purchase_timestamp) >= 2018
GROUP BY
    DATE_PART('month', o.order_purchase_timestamp)
ORDER BY
    order_month ASC;

-- Otimização

EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND DATE_PART('year', o.order_purchase_timestamp) = 2018
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;

/*
GroupAggregate  (cost=9089.92..9174.06 rows=482 width=88) (actual time=179.736..203.155 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=5405
  ->  Gather Merge  (cost=9089.92..9154.88 rows=570 width=51) (actual time=176.285..195.576 rows=62556 loops=1)
        Workers Planned: 1
        Workers Launched: 1
        Buffers: shared hit=5405
        ->  Sort  (cost=8089.91..8090.75 rows=335 width=51) (actual time=167.325..168.034 rows=31278 loops=2)
              Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
              Sort Method: quicksort  Memory: 3011kB
              Buffers: shared hit=5405
              Worker 0:  Sort Method: quicksort  Memory: 2925kB
              ->  Parallel Hash Join  (cost=4862.91..8075.86 rows=335 width=51) (actual time=52.013..71.794 rows=31278 loops=2)
                    Hash Cond: (oi.order_id = o.order_id)
                    Buffers: shared hit=5340
                    ->  Parallel Seq Scan on order_items oi  (cost=0.00..2961.65 rows=66265 width=37) (actual time=0.004..3.276 rows=56325 loops=2)
                          Buffers: shared hit=2299
                    ->  Parallel Hash  (cost=4859.21..4859.21 rows=296 width=80) (actual time=51.920..51.922 rows=27374 loops=2)
                          Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 7448kB
                          Buffers: shared hit=2990
                          ->  Parallel Hash Join  (cost=2836.21..4859.21 rows=296 width=80) (actual time=24.231..44.721 rows=27374 loops=2)
                                Hash Cond: (op.order_id = o.order_id)
                                Buffers: shared hit=2990
                                ->  Parallel Seq Scan on order_payments op  (cost=0.00..1792.09 rows=61109 width=39) (actual time=0.004..3.692 rows=51943 loops=2)
                                      Buffers: shared hit=1181
                                ->  Parallel Hash  (cost=2832.66..2832.66 rows=284 width=41) (actual time=24.209..24.209 rows=26392 loops=2)
                                      Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 5176kB
                                      Buffers: shared hit=1809
                                      ->  Parallel Seq Scan on orders o  (cost=0.00..2832.66 rows=284 width=41) (actual time=0.020..16.610 rows=26392 loops=2)
                                            Filter: ((order_status = 'delivered'::text) AND (date_part('year'::text, order_purchase_timestamp) = '2018'::double precision))
                                            Rows Removed by Filter: 23329
                                            Buffers: shared hit=1809
Planning:
  Buffers: shared hit=5 dirtied=1
Planning Time: 1.323 ms
Execution Time: 203.403 ms
*/

-- Primeiro passo: remover a função do WHERE
EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;

/*
GroupAggregate  (cost=19711.47..21944.40 rows=52159 width=88) (actual time=299.943..316.703 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=5289, temp read=497 written=498
  ->  Sort  (cost=19711.47..19866.30 rows=61930 width=51) (actual time=297.354..308.739 rows=62556 loops=1)
        Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
        Sort Method: external merge  Disk: 3976kB
        Buffers: shared hit=5289, temp read=497 written=498
        ->  Hash Join  (cost=8042.79..12664.86 rows=61930 width=51) (actual time=71.164..102.753 rows=62556 loops=1)
              Hash Cond: (oi.order_id = o.order_id)
              Buffers: shared hit=5289
              ->  Seq Scan on order_items oi  (cost=0.00..3425.50 rows=112650 width=37) (actual time=0.017..4.196 rows=112650 loops=1)
                    Buffers: shared hit=2299
              ->  Hash  (cost=7359.44..7359.44 rows=54668 width=80) (actual time=71.066..71.066 rows=54748 loops=1)
                    Buckets: 65536  Batches: 1  Memory Usage: 6551kB
                    Buffers: shared hit=2990
                    ->  Hash Join  (cost=4203.33..7359.44 rows=54668 width=80) (actual time=35.340..63.261 rows=54748 loops=1)
                          Hash Cond: (op.order_id = o.order_id)
                          Buffers: shared hit=2990
                          ->  Seq Scan on order_payments op  (cost=0.00..2219.86 rows=103886 width=39) (actual time=0.006..4.244 rows=103886 loops=1)
                                Buffers: shared hit=1181
                          ->  Hash  (cost=3549.22..3549.22 rows=52329 width=41) (actual time=35.310..35.310 rows=52783 loops=1)
                                Buckets: 65536  Batches: 1  Memory Usage: 4275kB
                                Buffers: shared hit=1809
                                ->  Seq Scan on orders o  (cost=0.00..3549.22 rows=52329 width=41) (actual time=0.014..21.335 rows=52783 loops=1)
                                      Filter: ((order_purchase_timestamp >= '2018-01-01'::date) AND (order_purchase_timestamp >= '2018-01-01'::date) AND (order_status = 'delivered'::text))
                                      Rows Removed by Filter: 46658
                                      Buffers: shared hit=1809
Planning Time: 0.963 ms
Execution Time: 319.988 ms
 */

-- CRIAR INDEX nas colunas usadas para filtrar
CREATE INDEX idx_order_status_date ON olist.orders(order_status, order_purchase_timestamp);
EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND DATE_PART('year', o.order_purchase_timestamp) = 2018
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;
/*
 * GroupAggregate  (cost=9089.92..9174.06 rows=482 width=88) (actual time=181.301..204.539 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=5405
  ->  Gather Merge  (cost=9089.92..9154.88 rows=570 width=51) (actual time=177.783..196.944 rows=62556 loops=1)
        Workers Planned: 1
        Workers Launched: 1
        Buffers: shared hit=5405
        ->  Sort  (cost=8089.91..8090.75 rows=335 width=51) (actual time=168.744..169.498 rows=31278 loops=2)
              Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
              Sort Method: quicksort  Memory: 2977kB
              Buffers: shared hit=5405
              Worker 0:  Sort Method: quicksort  Memory: 2959kB
              ->  Parallel Hash Join  (cost=4862.91..8075.86 rows=335 width=51) (actual time=55.155..74.845 rows=31278 loops=2)
                    Hash Cond: (oi.order_id = o.order_id)
                    Buffers: shared hit=5340
                    ->  Parallel Seq Scan on order_items oi  (cost=0.00..2961.65 rows=66265 width=37) (actual time=0.006..3.656 rows=56325 loops=2)
                          Buffers: shared hit=2299
                    ->  Parallel Hash  (cost=4859.21..4859.21 rows=296 width=80) (actual time=55.063..55.082 rows=27374 loops=2)
                          Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 7480kB
                          Buffers: shared hit=2990
                          ->  Parallel Hash Join  (cost=2836.21..4859.21 rows=296 width=80) (actual time=24.228..47.351 rows=27374 loops=2)
                                Hash Cond: (op.order_id = o.order_id)
                                Buffers: shared hit=2990
                                ->  Parallel Seq Scan on order_payments op  (cost=0.00..1792.09 rows=61109 width=39) (actual time=0.006..4.413 rows=51943 loops=2)
                                      Buffers: shared hit=1181
                                ->  Parallel Hash  (cost=2832.66..2832.66 rows=284 width=41) (actual time=24.202..24.203 rows=26392 loops=2)
                                      Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 5176kB
                                      Buffers: shared hit=1809
                                      ->  Parallel Seq Scan on orders o  (cost=0.00..2832.66 rows=284 width=41) (actual time=0.016..16.460 rows=26392 loops=2)
                                            Filter: ((order_status = 'delivered'::text) AND (date_part('year'::text, order_purchase_timestamp) = '2018'::double precision))
                                            Rows Removed by Filter: 23329
                                            Buffers: shared hit=1809
Planning:
  Buffers: shared hit=18 read=1
Planning Time: 2.098 ms
Execution Time: 204.811 ms
 */


EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;

/*
GroupAggregate  (cost=19711.47..21944.40 rows=52159 width=88) (actual time=309.173..326.295 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=5289, temp read=497 written=498
  ->  Sort  (cost=19711.47..19866.30 rows=61930 width=51) (actual time=306.535..318.220 rows=62556 loops=1)
        Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
        Sort Method: external merge  Disk: 3976kB
        Buffers: shared hit=5289, temp read=497 written=498
        ->  Hash Join  (cost=8042.79..12664.86 rows=61930 width=51) (actual time=83.518..112.726 rows=62556 loops=1)
              Hash Cond: (oi.order_id = o.order_id)
              Buffers: shared hit=5289
              ->  Seq Scan on order_items oi  (cost=0.00..3425.50 rows=112650 width=37) (actual time=0.019..3.852 rows=112650 loops=1)
                    Buffers: shared hit=2299
              ->  Hash  (cost=7359.44..7359.44 rows=54668 width=80) (actual time=83.449..83.450 rows=54748 loops=1)
                    Buckets: 65536  Batches: 1  Memory Usage: 6551kB
                    Buffers: shared hit=2990
                    ->  Hash Join  (cost=4203.33..7359.44 rows=54668 width=80) (actual time=36.198..73.147 rows=54748 loops=1)
                          Hash Cond: (op.order_id = o.order_id)
                          Buffers: shared hit=2990
                          ->  Seq Scan on order_payments op  (cost=0.00..2219.86 rows=103886 width=39) (actual time=0.006..5.654 rows=103886 loops=1)
                                Buffers: shared hit=1181
                          ->  Hash  (cost=3549.22..3549.22 rows=52329 width=41) (actual time=36.006..36.007 rows=52783 loops=1)
                                Buckets: 65536  Batches: 1  Memory Usage: 4275kB
                                Buffers: shared hit=1809
                                ->  Seq Scan on orders o  (cost=0.00..3549.22 rows=52329 width=41) (actual time=0.016..22.010 rows=52783 loops=1)
                                      Filter: ((order_purchase_timestamp >= '2018-01-01'::date) AND (order_purchase_timestamp >= '2018-01-01'::date) AND (order_status = 'delivered'::text))
                                      Rows Removed by Filter: 46658
                                      Buffers: shared hit=1809
Planning Time: 0.967 ms
Execution Time: 329.409 ms
*/


-- CRIAR INDEX nas colunas do join
CREATE INDEX idx_order_id ON olist.orders(order_id);
EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND DATE_PART('year', o.order_purchase_timestamp) = 2018
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;

/*
GroupAggregate  (cost=9089.92..9174.06 rows=482 width=88) (actual time=180.039..203.514 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=5405
  ->  Gather Merge  (cost=9089.92..9154.88 rows=570 width=51) (actual time=176.475..195.642 rows=62556 loops=1)
        Workers Planned: 1
        Workers Launched: 1
        Buffers: shared hit=5405
        ->  Sort  (cost=8089.91..8090.75 rows=335 width=51) (actual time=168.912..169.670 rows=31278 loops=2)
              Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
              Sort Method: quicksort  Memory: 3015kB
              Buffers: shared hit=5405
              Worker 0:  Sort Method: quicksort  Memory: 2921kB
              ->  Parallel Hash Join  (cost=4862.91..8075.86 rows=335 width=51) (actual time=52.081..72.416 rows=31278 loops=2)
                    Hash Cond: (oi.order_id = o.order_id)
                    Buffers: shared hit=5340
                    ->  Parallel Seq Scan on order_items oi  (cost=0.00..2961.65 rows=66265 width=37) (actual time=0.008..3.387 rows=56325 loops=2)
                          Buffers: shared hit=2299
                    ->  Parallel Hash  (cost=4859.21..4859.21 rows=296 width=80) (actual time=52.002..52.028 rows=27374 loops=2)
                          Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 7448kB
                          Buffers: shared hit=2990
                          ->  Parallel Hash Join  (cost=2836.21..4859.21 rows=296 width=80) (actual time=25.237..44.997 rows=27374 loops=2)
                                Hash Cond: (op.order_id = o.order_id)
                                Buffers: shared hit=2990
                                ->  Parallel Seq Scan on order_payments op  (cost=0.00..1792.09 rows=61109 width=39) (actual time=0.010..3.502 rows=51943 loops=2)
                                      Buffers: shared hit=1181
                                ->  Parallel Hash  (cost=2832.66..2832.66 rows=284 width=41) (actual time=25.197..25.197 rows=26392 loops=2)
                                      Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 5176kB
                                      Buffers: shared hit=1809
                                      ->  Parallel Seq Scan on orders o  (cost=0.00..2832.66 rows=284 width=41) (actual time=0.014..16.582 rows=26392 loops=2)
                                            Filter: ((order_status = 'delivered'::text) AND (date_part('year'::text, order_purchase_timestamp) = '2018'::double precision))
                                            Rows Removed by Filter: 23329
                                            Buffers: shared hit=1809
Planning:
  Buffers: shared hit=26 read=6
Planning Time: 0.966 ms
Execution Time: 203.637 ms
 */


EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;

/*
GroupAggregate  (cost=19711.47..21944.40 rows=52159 width=88) (actual time=303.219..320.147 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=5289, temp read=497 written=498
  ->  Sort  (cost=19711.47..19866.30 rows=61930 width=51) (actual time=300.623..312.139 rows=62556 loops=1)
        Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
        Sort Method: external merge  Disk: 3976kB
        Buffers: shared hit=5289, temp read=497 written=498
        ->  Hash Join  (cost=8042.79..12664.86 rows=61930 width=51) (actual time=77.647..107.619 rows=62556 loops=1)
              Hash Cond: (oi.order_id = o.order_id)
              Buffers: shared hit=5289
              ->  Seq Scan on order_items oi  (cost=0.00..3425.50 rows=112650 width=37) (actual time=0.009..3.928 rows=112650 loops=1)
                    Buffers: shared hit=2299
              ->  Hash  (cost=7359.44..7359.44 rows=54668 width=80) (actual time=77.619..77.620 rows=54748 loops=1)
                    Buckets: 65536  Batches: 1  Memory Usage: 6551kB
                    Buffers: shared hit=2990
                    ->  Hash Join  (cost=4203.33..7359.44 rows=54668 width=80) (actual time=35.049..68.328 rows=54748 loops=1)
                          Hash Cond: (op.order_id = o.order_id)
                          Buffers: shared hit=2990
                          ->  Seq Scan on order_payments op  (cost=0.00..2219.86 rows=103886 width=39) (actual time=0.003..5.055 rows=103886 loops=1)
                                Buffers: shared hit=1181
                          ->  Hash  (cost=3549.22..3549.22 rows=52329 width=41) (actual time=35.033..35.034 rows=52783 loops=1)
                                Buckets: 65536  Batches: 1  Memory Usage: 4275kB
                                Buffers: shared hit=1809
                                ->  Seq Scan on orders o  (cost=0.00..3549.22 rows=52329 width=41) (actual time=0.008..21.078 rows=52783 loops=1)
                                      Filter: ((order_purchase_timestamp >= '2018-01-01'::date) AND (order_purchase_timestamp >= '2018-01-01'::date) AND (order_status = 'delivered'::text))
                                      Rows Removed by Filter: 46658
                                      Buffers: shared hit=1809
Planning:
  Buffers: shared hit=16
Planning Time: 0.569 ms
Execution Time: 322.634 ms
 */


-- Criando o index na chave estrangeira na tabela order_items
CREATE INDEX idx_oi_order_id ON olist.order_items(order_id);

EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND DATE_PART('year', o.order_purchase_timestamp) = 2018
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;

/*
 * GroupAggregate  (cost=6041.17..6125.31 rows=482 width=88) (actual time=352.433..376.693 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=221449 read=732
  ->  Gather Merge  (cost=6041.17..6106.13 rows=570 width=51) (actual time=348.773..368.801 rows=62556 loops=1)
        Workers Planned: 1
        Workers Launched: 1
        Buffers: shared hit=221449 read=732
        ->  Sort  (cost=5041.16..5042.00 rows=335 width=51) (actual time=343.721..344.455 rows=31278 loops=2)
              Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
              Sort Method: quicksort  Memory: 2944kB
              Buffers: shared hit=221449 read=732
              Worker 0:  Sort Method: quicksort  Memory: 2992kB
              ->  Nested Loop  (cost=2836.62..5027.11 rows=335 width=51) (actual time=32.183..249.031 rows=31278 loops=2)
                    Join Filter: (o.order_id = oi.order_id)
                    Buffers: shared hit=221433 read=732
                    ->  Parallel Hash Join  (cost=2836.21..4859.21 rows=296 width=80) (actual time=31.432..48.901 rows=27374 loops=2)
                          Hash Cond: (op.order_id = o.order_id)
                          Buffers: shared hit=3041
                          ->  Parallel Seq Scan on order_payments op  (cost=0.00..1792.09 rows=61109 width=39) (actual time=0.015..3.663 rows=51943 loops=2)
                                Buffers: shared hit=1181
                          ->  Parallel Hash  (cost=2832.66..2832.66 rows=284 width=41) (actual time=31.304..31.304 rows=26392 loops=2)
                                Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 5176kB
                                Buffers: shared hit=1809
                                ->  Parallel Seq Scan on orders o  (cost=0.00..2832.66 rows=284 width=41) (actual time=0.024..23.048 rows=26392 loops=2)
                                      Filter: ((order_status = 'delivered'::text) AND (date_part('year'::text, order_purchase_timestamp) = '2018'::double precision))
                                      Rows Removed by Filter: 23329
                                      Buffers: shared hit=1809
                    ->  Index Scan using idx_oi_order_id on order_items oi  (cost=0.42..0.55 rows=1 width=37) (actual time=0.007..0.007 rows=1 loops=54748)
                          Index Cond: (order_id = op.order_id)
                          Buffers: shared hit=218392 read=732
Planning:
  Buffers: shared hit=42 read=6 dirtied=2
Planning Time: 3.482 ms
Execution Time: 376.933 ms
*/

EXPLAIN (ANALYZE, BUFFERS )
SELECT
	DATE_PART('month', o.order_purchase_timestamp) AS month,
	COUNT(DISTINCT(o.order_id)) AS number_delivered_orders,
	COUNT(oi.order_item_id) AS number_sold_items,
	SUM(op.payment_value) AS revenue,
	ROUND(1.0 * SUM(op.payment_value) / COUNT(o.order_id), 2) AS avg_revenue_per_delivered_order
FROM olist.orders o
JOIN olist.order_items oi
	ON o.order_id = oi.order_id
JOIN olist.order_payments op
	ON o.order_id = op.order_id
WHERE o.order_status = 'delivered'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
	AND o.order_purchase_timestamp >= DATE '2018-01-01'
GROUP BY DATE_PART('month', o.order_purchase_timestamp)
ORDER BY month ASC;

/*
 * GroupAggregate  (cost=19711.47..21944.40 rows=52159 width=88) (actual time=312.198..328.959 rows=8 loops=1)
  Group Key: (date_part('month'::text, o.order_purchase_timestamp))
  Buffers: shared hit=5289, temp read=497 written=498
  ->  Sort  (cost=19711.47..19866.30 rows=61930 width=51) (actual time=309.614..321.017 rows=62556 loops=1)
        Sort Key: (date_part('month'::text, o.order_purchase_timestamp)), o.order_id
        Sort Method: external merge  Disk: 3976kB
        Buffers: shared hit=5289, temp read=497 written=498
        ->  Hash Join  (cost=8042.79..12664.86 rows=61930 width=51) (actual time=80.737..116.924 rows=62556 loops=1)
              Hash Cond: (oi.order_id = o.order_id)
              Buffers: shared hit=5289
              ->  Seq Scan on order_items oi  (cost=0.00..3425.50 rows=112650 width=37) (actual time=0.029..10.359 rows=112650 loops=1)
                    Buffers: shared hit=2299
              ->  Hash  (cost=7359.44..7359.44 rows=54668 width=80) (actual time=80.363..80.364 rows=54748 loops=1)
                    Buckets: 65536  Batches: 1  Memory Usage: 6551kB
                    Buffers: shared hit=2990
                    ->  Hash Join  (cost=4203.33..7359.44 rows=54668 width=80) (actual time=34.544..70.265 rows=54748 loops=1)
                          Hash Cond: (op.order_id = o.order_id)
                          Buffers: shared hit=2990
                          ->  Seq Scan on order_payments op  (cost=0.00..2219.86 rows=103886 width=39) (actual time=0.012..5.580 rows=103886 loops=1)
                                Buffers: shared hit=1181
                          ->  Hash  (cost=3549.22..3549.22 rows=52329 width=41) (actual time=34.395..34.395 rows=52783 loops=1)
                                Buckets: 65536  Batches: 1  Memory Usage: 4275kB
                                Buffers: shared hit=1809
                                ->  Seq Scan on orders o  (cost=0.00..3549.22 rows=52329 width=41) (actual time=0.015..20.692 rows=52783 loops=1)
                                      Filter: ((order_purchase_timestamp >= '2018-01-01'::date) AND (order_purchase_timestamp >= '2018-01-01'::date) AND (order_status = 'delivered'::text))
                                      Rows Removed by Filter: 46658
                                      Buffers: shared hit=1809
Planning:
  Buffers: shared hit=32
Planning Time: 1.492 ms
Execution Time: 332.236 ms
*/
```
