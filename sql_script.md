### Data Cleaning
Every table is checked for duplicates, missing values, and inconsistent data types.
#### Modifying data types
As an example, **products** table will be used. It currently has 6 columns:

![products_old_data_types](https://github.com/emTgi/Project_Bike_Store/assets/114177110/2f228396-c89f-4cb1-a24d-95e63d8bfb1f)

![products_table](https://github.com/emTgi/Project_Bike_Store/assets/114177110/d52adba9-1522-445f-9a58-296ab3284788)

By exploring the table further, I noticed that _list_price_ needs to be as precise as possible as it is financial data so I will be changing it to DECIMAL data type, and _product_name_ can be changed to VARCHAR to decrease the file size and improve performance:
```sql
ALTER TABLE products
MODIFY product_name VARCHAR(255),
MODIFY list_price DECIMAL(10,2);
```
This has successfully changed the data types of both columns:

![products_new_data_types](https://github.com/emTgi/Project_Bike_Store/assets/114177110/a6e8782e-ca66-4176-a39d-531be932b0fa)

Another example is the **orders** table, where dates are stored as text:

![orders_old_data_types](https://github.com/emTgi/Project_Bike_Store/assets/114177110/9fe592e4-45b2-4efc-ae99-cd903c73ba3e)

I want to change these 3 columns to be stored as DATE data type:
```sql
ALTER TABLE orders
MODIFY order_date DATE,
MODIFY required_date DATE,
MODIFY shipped_date DATE;
```
Again this has successfully changed the data types:

![orders_new_data_types](https://github.com/emTgi/Project_Bike_Store/assets/114177110/36bd32ae-4df9-49f8-8563-e5b4b45480c5)

#### Handling missing values
Only the _shipped_date_ column in the **orders** table has missing values:
```sql
SELECT *
FROM orders
WHERE shipped_date IS NULL;
```
![NULL_values_orders](https://github.com/emTgi/Project_Bike_Store/assets/114177110/26f307ee-75c9-4cb3-9bce-d926982f6fad)

However, these are cancelled or unprocessed orders so it is expected that this column might have NULL values. For this project, I will treat all rows with NULL values as cancelled:
```sql
UPDATE orders
SET order_status = 3
WHERE order_date = required_date
AND shipped_date IS NULL;
```

#### Duplicates
Only the **products** table has duplicate rows:
```sql
SELECT
	product_name,
	COUNT(*)
FROM products
GROUP BY product_name
HAVING COUNT(*) > 1;
```
![Duplicates](https://github.com/emTgi/Project_Bike_Store/assets/114177110/663fff64-e5b4-4a69-9f1e-6dee9a2b1a57)

The only difference between duplicate rows apart from _product_id_ column is the category assigned to them:
```sql
SELECT
	product_name,
	COUNT(*),
	GROUP_CONCAT(CAST(category_name AS CHAR) SEPARATOR ', ') AS category_ids
FROM products as p
JOIN categories as c
	ON p.category_id = c.category_id
GROUP BY product_name
HAVING COUNT(*) > 1;
```
![duplicates_different_categories](https://github.com/emTgi/Project_Bike_Store/assets/114177110/3eb64d09-0141-4df8-8f20-e4a65a512649)

This could be a duplicate or a completely different product; for the purpose of this project, these will be treated as duplicate entries. I will remove the duplicates and only keep the first entry.
_Product_id_ is also used in the **order_items** table so the values need to be updated there as well. First, I check if there are any products with more than one duplicate entry:
```sql
SELECT
	product_name,
	COUNT(*)
FROM products
GROUP BY product_name
HAVING COUNT(*) > 2
```
![triple_duplicate](https://github.com/emTgi/Project_Bike_Store/assets/114177110/b88454c2-1de5-4ced-8001-a1647215a0f3)

There is only one item that has more than one duplicate so I will change it manually:
```sql
SELECT *
FROM products
WHERE product_name = 'Electra Townie Go! 8i - 2017/2018';
```
![triple_duplicate_ids](https://github.com/emTgi/Project_Bike_Store/assets/114177110/09d64249-6db4-425a-9eb1-81128ab76149)

The first entry has _product_id_ of 192 so in the **order_items** table I will change every product with id 250 or 303 to display 192:
```sql
UPDATE order_items
SET product_id = 192
WHERE product_id IN(250, 303);
```
In the **products** table, I will just delete those rows:
```sql
DELETE FROM products
WHERE product_id IN(250, 303);
```
Now, I need to create a column for the products with one duplicate that will return the first _product_id_ entry. First, I use ROW_NUMBER() to identify duplicates:
```sql
SELECT 	*, 
	ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
FROM products
```
![duplicates_rank](https://github.com/emTgi/Project_Bike_Store/assets/114177110/37b0c8db-91d0-404c-9cc4-4bd86ceb0320)

Then I create the column with first _product_id_ for each product using CASE function:
```sql
SELECT	*,
	CASE
		WHEN rnk = 1 THEN product_id
		ELSE MIN(product_id) OVER(partition by product_name)
	END AS first_id
FROM (
	SELECT 	*, 
		ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
	FROM products
        ) as sub
```
![first_id_duplicates](https://github.com/emTgi/Project_Bike_Store/assets/114177110/5ad3ea56-36cc-43b8-9e06-7cb1fc0c606a)

Now I can use this query in a common table expression (CTE) to identify products with different product_id than first_id in the order_items table:
```sql
WITH rnk_cte AS (
	SELECT	*,
		CASE
			WHEN rnk = 1 THEN product_id
			ELSE MIN(product_id) OVER(partition by product_name)
		END AS first_id
	FROM (
		SELECT 	*, 
			ROW_NUMBER() OVER(PARTITION BY product_name ORDER BY product_id) as rnk
		FROM products
        ) as sub
)

SELECT
	a.product_id,
	product_name,
	first_id
FROM order_items as a
JOIN rnk_cte as b
	ON a.product_id = b.product_id
WHERE a.product_id <> first_id;
```
![different_ids](https://github.com/emTgi/Project_Bike_Store/assets/114177110/dd901869-ea0e-4939-bef8-628e9dd1559a)

Now I can replace the product_id with the first_id
```sql
WITH rnk_cte AS (
	SELECT	*,
		CASE
			WHEN rnk = 1 THEN product_id
			ELSE MIN(product_id) OVER(partition by product_name)
		END AS first_id
	FROM (
		SELECT 	*, 
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
Now we can also remove the duplicates from the products table:
```sql
WITH rnk_cte AS (
	SELECT	*,
		CASE
			WHEN rnk = 1 THEN product_id
			ELSE MIN(product_id) OVER(partition by product_name)
		END AS first_id
	FROM (
		SELECT 	*, 
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
### Data Analysis
For analysis and visualisation in Power BI, I will create various, different views.
##### VIEW nr 1
![orders_view](https://github.com/emTgi/Project_Bike_Store/assets/114177110/08de9a84-bce2-4231-85df-6714ff04a89d)
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
SELECT	*,
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
