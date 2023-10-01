#### Data Cleaning
Every table is checked for duplicates, missing values, and inconsistent data types.
Order_status was inconsistent:
```sql
UPDATE orders
SET order_status = 3
WHERE order_date = required_date
AND shipped_date IS NULL;
```
Checking for duplicates in products:
```sql
SELECT product_name, COUNT(*)
FROM products
GROUP BY product_name
HAVING COUNT(*) > 1;
```
The same products with different categories assigned to them:
```sql
SELECT product_name, COUNT(*), group_concat(CAST(category_name AS CHAR) SEPARATOR ', ') AS category_ids
FROM products as p
JOIN categories as c
ON p.category_id = c.category_id
GROUP BY product_name
HAVING COUNT(*) > 1;
```
This could be a duplicate or a completely different product; for the purpose of this project, this will be treated as duplicate entries. I will remove the duplicates and only keep the first entry.
Product_id is also used in the order_items table so the values need to be updated there as well. First, I check if there are any products with more than one duplicate entry:
```sql
SELECT product_name, COUNT(*)
FROM products
GROUP BY product_name
HAVING COUNT(*) > 2
```
There is only one item that has more than one duplicate so I will change it manually:
```sql
SELECT *
FROM products
WHERE product_name = 'Electra Townie Go! 8i - 2017/2018';
```
The first entry has product_id of 192 so in the order_items table I will change every product with id 250 or 303 to display 192:
```sql
UPDATE order_items
SET product_id = 192
WHERE product_id IN(250, 303);
```
In the products table, I will just delete those rows:
```sql
DELETE FROM products
WHERE product_id IN(250, 303);
```
Now, I need to create a column for the products with one duplicate that will return the first product_id entry. First, I use row_number() to identify duplicates:
```sql
SELECT 
  	*, 
	ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
FROM products
```
Then I create the column with first product_id for each product using CASE function:
```sql
SELECT	
  	*,
	CASE
		WHEN rnk = 1 THEN product_id
		ELSE MIN(product_id) OVER(partition by product_name)
	END AS first_id
FROM (
	SELECT 
    		*, 
		ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
	FROM products
        ) as sub
```
Now I can use this query in a common table expression (CTE) to identify products with different product_id than first_id in the order_items table:
```sql
WITH rnk_cte AS (
	SELECT	
    		*,
		CASE
			WHEN rnk = 1 THEN product_id
			ELSE MIN(product_id) OVER(partition by product_name)
		END AS first_id
	FROM (
		SELECT 
      			*, 
			ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
		FROM products
        ) as sub
)

SELECT a.product_id, product_name, first_id
FROM order_items as a
JOIN rnk_cte as b
ON a.product_id = b.product_id
WHERE a.product_id <> first_id;
```
Now I can replace the product_id with the first_id
```sql
WITH rnk_cte AS (
	SELECT	
    		*,
		CASE
			WHEN rnk = 1 THEN product_id
			ELSE MIN(product_id) OVER(partition by product_name)
		END AS first_id
	FROM (
		SELECT 
     			*, 
			ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
		FROM products
        ) as sub
)

UPDATE order_items as a
JOIN rnk_cte as b
	ON a.product_id = b.product_id
SET a.product_id = first_id
WHERE a.product_id <> first_id;
```
Now we can check if this worked by using the previous statement, as you can see the duplicate product entries have been replaced:
 llllllllllllllllllllllllllllllllllllllllllllllllllllllllllllllllll
Now we can also remove the duplicates from the products table:
```sql
WITH rnk_cte AS (
	SELECT	
    		*,
		CASE
			WHEN rnk = 1 THEN product_id
			ELSE MIN(product_id) OVER(partition by product_name)
		END AS first_id
	FROM (
		SELECT 
     			*, 
			ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
		FROM products
        ) as sub
)

DELETE a
FROM products as a
JOIN rnk_cte as b
	ON a.product_id = b.product_id
WHERE a.product_id <> first_id;
```
#### Data Analysis
For analysis and later visualisation in Power BI, I will create various, different views.
##### VIEW nr 1
```sql
CREATE VIEW orders_view AS (
SELECT
	a.order_id,
	customer_id, 
    	order_status, 
    	order_date,
    	SUM(ROUND(list_price * quantity * (1 - discount),2)) as final_price,
    	SUM(quantity) as no_of_products,
    	CASE
		WHEN shipped_date <= required_date THEN 'On time'
        	WHEN shipped_date > required_date THEN 'Late'
        	ELSE 'N/A'
	END as order_performance,
    	store_id
FROM orders as a
JOIN order_items as b
	ON a.order_id = b.order_id
GROUP BY
	a.order_id, 
    	customer_id, 
    	order_status, 
    	order_date,
	order_performance,
    	store_id
);
```
##### VIEW nr 2
```sql
CREATE VIEW store_view AS (
WITH cte_1 AS (
SELECT 
	order_id,
	CASE
		WHEN order_performance = 'Late' THEN 1
        ELSE 0
	END as Late
FROM orders_view 
)

SELECT 
	a.store_id, 
    	city,
    	DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS start_of_month,
    	YEAR(order_date) as year,
    	COUNT(a.order_id) as orders,
    	SUM(final_price) as revenue,
    	COUNT(DISTINCT customer_id) as uq_customers,
    	ROUND(SUM(Late)/COUNT(*)*100,2) as late_rate
FROM orders_view a
JOIN stores b
	ON a.store_id = b.store_id
JOIN cte_1 c
	ON a.order_id = c.order_id
WHERE order_status <> 3
GROUP BY
	year,
	a.store_id,
	city,
	start_of_month
ORDER BY year, a.store_id
);
```
##### VIEW nr 3
```sql
CREATE VIEW top_cust AS (
WITH cte_2 AS (
SELECT
	*,
	RANK() OVER(PARTITION BY month(order_date),year(order_date), store_id ORDER BY total_revenue DESC) as rnk
FROM (
SELECT
	order_date,
	customer_id,
    	SUM(final_price) as total_revenue,
    	store_id
FROM orders_view
WHERE order_status <> 3
GROUP BY
	order_date,
	customer_id,
    	store_id
) as subb
ORDER BY year(order_date), month(order_date), total_revenue DESC
)

SELECT
	store_id,
	DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS start_of_month,
    	CONCAT(first_name, ' ', last_name) AS full_name,
    	total_revenue
FROM cte_2 a
JOIN customers b
	ON a.customer_id = b.customer_id
WHERE rnk = 1
ORDER BY year(order_date), month(order_date), store_id
);
```
##### VIEW nr 4
```sql
CREATE VIEW prod_view AS (
WITH cte_3 as (
SELECT 
	product_id,
    	SUM(CASE WHEN store_id = 1 THEN quantity ELSE 0 END) AS store_id_1,
    	SUM(CASE WHEN store_id = 2 THEN quantity ELSE 0 END) AS store_id_2,
    	SUM(CASE WHEN store_id = 3 THEN quantity ELSE 0 END) AS store_id_3
FROM stocks
GROUP BY product_id
)

SELECT 
	a.product_id,
    	product_name,
    	COUNT(DISTINCT order_id) as total_orders,
	SUM(ROUND(b.list_price * quantity * (1 - discount),2)) as total_revenue,
    	SUM(quantity) as units_sold,
    	store_id_1,
    	store_id_2,
    	store_id_3
FROM products a
JOIN order_items b
	ON a.product_id = b.product_id
JOIN cte_3 c
	ON a.product_id = c.product_id
GROUP BY
	a.product_id,
    	product_name,
    	store_id_1,
    	store_id_2,
    	store_id_3
);
```
##### VIEW nr 5
```sql
CREATE VIEW cat_view AS (
WITH category_cte AS (
SELECT
	product_id,
    	category_name
FROM products a
JOIN categories b
	ON a.category_id = b.category_id
)

SELECT
	category_name,
    	SUM(total_orders) as total_orders,
    	SUM(total_revenue) as total_revenue,
    	SUM(units_sold) as units_sold
FROM prod_view a
JOIN category_cte b
	ON a.product_id = b.product_id
GROUP BY category_name
);
```
##### VIEW nr 6
```sql
CREATE VIEW brand_view AS (
WITH brand_cte AS (
SELECT
	product_id,
    	brand_name
FROM products a
JOIN brands b
	ON a.brand_id = b.brand_id
)

SELECT
	brand_name,
    	SUM(total_orders) as total_orders,
    	SUM(total_revenue) as total_revenue,
    	SUM(units_sold) as units_sold
FROM prod_view a
JOIN brand_cte b
	ON a.product_id = b.product_id
GROUP BY brand_name
);
```
