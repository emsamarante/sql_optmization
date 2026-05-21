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

Looking at the plan of execution, I got `Planning Time: 0.817 ms` and `Execution Time: 366.344 ms`.

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
