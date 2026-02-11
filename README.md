# SALES ANALYSIS OF A COFFEE SHOP - SQL PROJECT

<img width="1024" height="1536" alt="fd2bee46-dd98-435f-a1f0-aaad23b620b7" src="https://github.com/user-attachments/assets/14bd1d8c-1618-4ead-b56a-48b992f0ba68" />

### Project Overview
The project aimed to summarise the sales data of a coffee shop: Monday Coffee. They have been selling their products online since January 2023, they now want to open new stores in three cities of india, to determine which cities, an EDA was done on their data based on consumer demand and sales performance.

## Key Questions Answered
1. ### Coffee Consumer Count
How many people in each city are estimated to consume, given that about 25% of the population does like and consume coffee?

```sql
SELECT
	city_name,
	population,
	ROUND((population * 0.25)/1000000, 2) as coffee_consumers_in_millions,
	city_rank
	FROM city
ORDER BY 2 DESC
```

2. ### Sales Count for Each Product
How many units of each coffee amount per customer in each city?
```sql
SELECT
	ci.city_name,
	SUM(total) as total_revenue
FROM sales s
	JOIN customers c ON
	c.customer_id = s.customer_id
	JOIN city ci ON 
	c.city_id = ci.city_id
WHERE 
	EXTRACT(YEAR FROM s.sale_date) = 2023
	AND
	EXTRACT(quarter FROM s.sale_date) = 4
GROUP BY 1
ORDER BY 2 DESC
```

3. ### Sales count for each product
How may units for each coffee product have been sold
```sql
SELECT 
	p.product_name,
	COUNT(s.sale_id) as total_orders
FROM products as p
LEFT JOIN sales s ON
p.product_id = s.product_id
GROUP BY 1
ORDER BY 2 DESC
```

4. ### Average sales amount per city
What is the average sales amount per customer in each city
```sql
SELECT
	ci.city_name,
	COUNT(DISTINCT c.customer_id) as customers_per_city,
	SUM(s.total) as total_sales,
	ROUND(SUM(s.total)::numeric/
			COUNT(DISTINCT c.customer_id)::numeric, 2) as avg_sales
FROM sales s
	JOIN customers c ON
	c.customer_id = s.customer_id
	JOIN city ci ON 
	c.city_id = ci.city_id
GROUP BY 1
ORDER BY 3 DESC
```

5. ### City population and coffee consumers
Privide a list of cities along with their populations and estimated coffee consumers.
```sql
WITH city_table 
AS
(
	SELECT 
		city_name,
		ROUND((population * 0.25)/1000000, 2) as coffee_consumers_in_millions
	FROM city
),
customers_table 
AS
(
	SELECT
		ci.city_name,
		COUNT(DISTINCT c.customer_id) as unique_cx
	FROM sales s
		JOIN customers as c
		ON c.customer_id = s.customer_id
		JOIN city as ci ON 
		ci.city_id = c.cIty_id
	GROUP BY 1
)
SELECT 
	customers_table.city_name,
	city_table.coffee_consumers_in_millions,
	customers_table.unique_cx
FROM city_table
JOIN customers_table ON 
city_table.city_name = customers_table.city_name
```
6. ### Top selling products by city
What are the top 3 selling products in each city based on sales volume?
```sql
WITH cte 
AS
(
SELECT 
	ci.city_name,
	p.product_name,
	COUNT(s.sale_id) as total_orders,
	DENSE_RANK() OVER(PARTITION BY ci.city_name ORDER BY COUNT(s.sale_id) DESC) as rank
FROM  sales as s
	JOIN products as p ON
	s.product_id = p.product_id
	JOIN customers as c ON
	c.customer_id = s.customer_id
	JOIN city as ci ON
	ci.city_id = c.city_id
	GROUP BY 1, 2
)
SELECT 
	*
FROM cte
WHERE rank <= 3
```
7. ### Customer segmentation by city
How many unique customers are there in each city who have purchased coffee products?
```sql
SELECT 
	ci.city_name,
	COUNT(DISTINCT c.customer_id) as unique_cx
FROM city as ci
	LEFT JOIN customers as c ON
	c.city_id = ci.city_id
	JOIN sales as s ON
	s.customer_id = c.customer_id
WHERE 
	s.product_id IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14)
GROUP BY 1
```
8. ### Average sales vs rent
Find each city and their avarage sale per customer and avg per customer
```sql
SELECT 
	ci.city_name,
	COUNT(DISTINCT c.customer_id) as total_customers,
	SUM(s.total) as total_sales,
	ROUND(SUM(s.total)::numeric/
			COUNT(DISTINCT c.customer_id)::numeric, 2) as avg_sales,
	ci.estimated_rent,
	ROUND((ci.estimated_rent)::numeric/
			COUNT(DISTINCT c.customer_id)::numeric, 2) as avg_rent
FROM sales as s
LEFT JOIN customers as c ON 
c.customer_id = s.customer_id
JOIN city as ci ON 
c.city_id = ci.city_id
GROUP BY 1, 5
ORDER BY 4 DESC
```
9. ### Monthly sales Growth
Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly)
```sql
WITH monthly_sales 
AS
(
	SELECT
		ci.city_name,
		EXTRACT( MONTH FROM sale_date) as month,
		EXTRACT(YEAR FROM sale_date) as year,
		SUM(s.total) as total_sale
	FROM  sales as s
		JOIN products as p ON
		s.product_id = p.product_id
		JOIN customers as c ON
		c.customer_id = s.customer_id
		JOIN city as ci ON
		ci.city_id = c.city_id
	GROUP BY 1, 2, 3
	ORDER BY 1, 3, 2
),
growth_ratio
AS
(
	SELECT
		city_name,
		month,
		year,
		total_sale as cr_month_sale,
		LAG(total_sale, 1) OVER(PARTITION BY city_name ORDER BY year, month) as last_month_sale,
		total_sale - (LAG(total_sale, 1) OVER(PARTITION BY city_name ORDER BY year, month)) as sales_change
FROM monthly_sales
)
SELECT 
	*,
	ROUND(
			(cr_month_sale - last_month_sale)::numeric/last_month_sale::numeric * 100, 
			2) as growth_ratio
FROM growth_ratio
WHERE last_month_sale IS NOT NULL
```
10. ### Market Potential Analysis
Identify top 3 city based on highest sales, return city name, total sales, total rent, total customers and estimated coffeeconsumers
```sql
SELECT 
	ci.city_name,
	COUNT(DISTINCT c.customer_id) as total_customers,
	SUM(s.total) as total_sales,
	ROUND(SUM(s.total)::numeric/
			COUNT(DISTINCT c.customer_id)::numeric, 2) as avg_sales,
	ci.estimated_rent as total_rent,
	ROUND((ci.estimated_rent)::numeric/
			COUNT(DISTINCT c.customer_id)::numeric, 2) as avg_rent,
	ROUND((ci.population * 0.25)/1000000, 3) as estimated_coffee_consumers_in_millions
FROM sales as s
LEFT JOIN customers as c ON 
c.customer_id = s.customer_id
JOIN city as ci ON 
c.city_id = ci.city_id
GROUP BY 1, 5, 7
ORDER BY 3 DESC
```

# RECOMMENDATIONS 

After analyzing the data, the recommendations top three cities for new store openings are:

### City: Pune
1. Average rent per customer is very low
2. Highest total revenue.
3. Average sales per customer is also high

### City 2: Jaipur
1. Highest number of customers (69)
2. Average rent per customer is very low at 156
3. Average sales per customer is better at 11.6K

### City 3: Delhi
1. Highest estimated coffee consumer at 7.7M
2. Average rent per customer is 300: which is still under 300
3. Highest total number of customers which is 68
