# Case Study 1 - Danny's Diner
Uncover insights into customer behaviour to determine how to expand the current loyalty program.

## Entity Relationship Diagram
This case study included three tables:
- members: captures the join date for customers part of the loyalty program
- menu: captures the diner's menu options and their costs
- sales: captures orders including the customer id and what product was ordered

<img width="429" height="270" alt="image" src="https://github.com/user-attachments/assets/aba8e458-7c5d-4a1e-9c71-83fc1a7af8f0" />

\
**Assumptions**
- Purchases made on the same day were purchased in one order.
- If the customer joined the loyalty scheme on a day they made a purchase, this purchase is counted as part of the loyalty scheme.

## Case Study Questions
**Question 1**: What is the total amount each customer spent at the restaurant?


```sql
SELECT
	customer_id,
	SUM(price) AS total_spent
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
GROUP BY
	customer_id
ORDER BY 
	customer_id;
```

|customer_id|total_spent|
|---|---|
|A|76|
|B|74|
|C|36|

This required a JOIN between the sales table and the menu table to retrieve the product prices. The SUM aggregation determined the amount each customer spent. The customers who are part of the loyalty program, A and B, have been spending more money at the diner.

\
**Question 2**: How many days has each customer visited the restaurant?

```sql
SELECT 
	customer_id,
	COUNT(DISTINCT order_date) AS days_visited
FROM dannys_diner.sales
GROUP BY 
	customer_id 
ORDER BY 
	customer_id;
```

|customer_id|days_visited|
|---|---|
|A|4|
|B|6|
|C|2|

The DISTINCT keyword is used to makes sure orders on the same day are not included in the count. The customers who are part of the loyalty program have visited the restaurant the most. 

\
**Question 3**: What was the first item from the menu purchased by each customer?

```sql
WITH purchase_order AS
	(SELECT 
		customer_id,
		product_id,
		DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS transaction_id
	FROM dannys_diner.sales)
SELECT DISTINCT
	customer_id,
	product_name
FROM purchase_order AS p
JOIN dannys_diner.menu AS m
ON p.product_id = m.product_id
WHERE transaction_id = 1
ORDER BY 
	customer_id;
```

|customer_id|product_name|
|---|---|
|A|curry|
|A|sushi|
|B|curry|
|C|ramen|

The DENSE_RANK() window function gives an order to customer purchases. This is added in a CTE due to the order of execution and window functions not being allowed in the WHERE clause. The JOIN is used to get the product name. Curry was bought by two customers in their first order.

\
**Question 4**: What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
SELECT 
	product_name,
	COUNT(*) AS order_amount
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
GROUP BY 
	product_name
ORDER BY 
	order_amount DESC
LIMIT 1;
```

|product_name|order_amount|
|---|---|
|ramen|8|

The JOIN is used to retrieve the product name. The number of orders for each product is determined with COUNT. Ordering in descending order of order amount and limiting the result by 1 row gives the most purchased product. Ramen is the most purchased product by all customers.

\
**Question 5**: Which item was the most popular for each customer?

```sql
WITH order_amounts AS
	(SELECT 
		customer_id,
		product_name,
		COUNT(*) AS order_amount
	FROM dannys_diner.sales AS s
	JOIN dannys_diner.menu AS m
	ON s.product_id = m.product_id
	GROUP BY 
 		customer_id,
		product_name)
SELECT *
FROM order_amounts AS o1
WHERE order_amount = 
	(SELECT 
		MAX(order_amount)
	FROM order_amounts AS o2
	WHERE o1.customer_id = o2.customer_id)
ORDER BY 
	customer_id;
```

|customer_id|product_id|order_amount|
|---|---|---|
|A|ramen|3|
|B|sushi|2|
|B|curry|2|
|B|ramen|2|
|C|ramen|3|

A CTE was used to create a table with the order amounts of each product for each customer to be used in the correlated subquery. The correlated subquery only allows the query to return the rows with the products where the order amount is the highest for each customer. Customer B has ordered an equal amount of all products. Ramen is the most purchases item between customers, and is the top purchase for each customer.

\
**Question 6**: Which item was purchased first by the customer after they became a member?

```sql
WITH purchase_order AS
	(SELECT 
		customer_id,
		product_id,
		order_date,
		DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS transaction_id
	FROM dannys_diner.sales)
SELECT 
	po1.customer_id,
	product_name
FROM purchase_order AS po1
JOIN dannys_diner.members AS mem
ON po1.customer_id = mem.customer_id
JOIN dannys_diner.menu AS m
ON po1.product_id = m.product_id
WHERE transaction_id = 
	(SELECT
		MIN(transaction_id)
	FROM purchase_order AS po2
	WHERE order_date >= join_date
	AND po1.customer_id = po2.customer_id)
ORDER BY 
	customer_id;
```

|customer_id|product_name|
|---|---|
|A|curry|
|B|sushi|

The CTE is the same as Question 3 to give an order to the customers's purchases. The JOINS retrieve the date the customer joined the loyalty scheme and the product name. The correlated subquery chooses the transaction for each customer that was made first after joining the loyalty scheme. Customers bought different products after joining the loyalty scheme.

\
**Question 7**: Which item was purchased just before the customer became a member?

```sql
WITH purchase_order AS
	(SELECT 
		customer_id,
		product_id,
		order_date,
		DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS transaction_id
	FROM dannys_diner.sales)
SELECT 
	po1.customer_id,
	product_name
FROM purchase_order AS po1
JOIN dannys_diner.members AS mem
ON po1.customer_id = mem.customer_id
JOIN dannys_diner.menu AS m
ON po1.product_id = m.product_id
WHERE transaction_id = 
	(SELECT
		MAX(transaction_id)
	FROM purchase_order AS po2
	WHERE order_date < join_date
	AND po1.customer_id = po2.customer_id)
ORDER BY 
	customer_id;
```

|customer_id|product_name|
|---|---|
|A|sushi|
|A|curry|
|B|sushi|

This query is very similar to Question 6. The only different is in the correlated subquery so it returns the transaction made just before joining the loyalty scheme. The customers bought similar products as they did after they did just after joining the loyalty scheme. Both customers bough sushi before they joined the loyalty scheme.

\
**Question 8**: What is the total items and amount spent for each member before they became a member?

```sql
SELECT 
	s.customer_id,
	COUNT(*) AS total_items,
	SUM(price) AS amount_spent
FROM dannys_diner.sales AS s
JOIN dannys_diner.members AS mem
ON s.customer_id = mem.customer_id
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
WHERE order_date < join_date
GROUP BY 
	s.customer_id
ORDER BY
	s.customer_id;
```

|customer_id|total_items|amount_spent|
|---|---|---|
|A|2|25|
|B|3|40|

The JOINS retrieve the price of the products and the join date of the loyalty scheme members. INNER JOINS are used so customers who are not part of the loyalty scheme are not returned. Customers buy over 1 product before joining the loyalty scheme. 

\
**Question 9**: If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

```sql
SELECT 
	s.customer_id,
	SUM(CASE WHEN m.product_id = 1 THEN price*20
		ELSE price*10
		END) AS total_points
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
JOIN dannys_diner.members AS mem
ON s.customer_id = mem.customer_id
WHERE order_date >= join_date
GROUP BY 
	s.customer_id
ORDER BY 
	customer_id;
```

|customer_id|total_points|
|---|---|
|A|510|
|B|440|

The CASE expression allocates a point value to each product and the SUM aggregation adds all the points up for each customer. THE JOINS retrieve the point amounts and the join date for each customer who is part of the loyalty scheme so only purchases made on or after that date are counted. Sushi is the least popular menu item, the 2x point multiplier may convince users to increase the amount of sushi they purchase. It may be worth using on curry as it is the most expensive item on the menu, to try and increase purchases. 

\
**Question 10**: In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

```sql
SELECT 
	s.customer_id,
	SUM(CASE WHEN s.product_id <> 1 AND order_date BETWEEN join_date AND join_date+6 THEN price*20
	WHEN s.product_id = 1 THEN price*20
	ELSE price*10
	END) AS january_points
FROM dannys_diner.sales AS s
JOIN dannys_diner.members AS mem
ON s.customer_id = mem.customer_id
JOIN  dannys_diner.menu AS m
ON s.product_id = m.product_id
WHERE order_date < '2021-02-01'
AND order_date >= join_date
GROUP BY
	s.customer_id
ORDER BY 
	customer_id;
```

|customer_id|january_points|
|---|---|
|A|1020|
|B|320|

This query is similar to Question 9 but the CASE expression includes an extra WHEN clause for the extra points for the first week of joining. The WHERE clause also ensures to only include purchases in January. Customer A's point balance increased a lot as they bought several products in the first week of membership, unlike customer B.

## Bonus Questions

**Question 1**: Join All Things

```sql
SELECT
	s.customer_id,
	order_date,
	product_name,
	price,
	CASE WHEN join_date IS NULL OR order_date < join_date THEN 'N'
	ELSE 'Y'
	END AS member
FROM dannys_diner.sales AS s
LEFT JOIN dannys_diner.members AS mem
ON s.customer_id = mem.customer_id
JOIN dannys_diner.menu AS m
ON s.product_id = m.product_id
ORDER BY 
	customer_id,
	order_date;
```

|customer_id|order_date|product_name|price|member|
|---|---|---|---|---|
|A|2021-01-01|sushi|10|N|
|A|2021-01-01|curry|15|N|
|A|2021-01-07|curry|15|Y|
|A|2021-01-10|ramen|12|Y|
|A|2021-01-11|ramen|12|Y|
|A|2021-01-11|ramen|12|Y|
|B|2021-01-01|curry|15|N|
|B|2021-01-02|curry|15|N|
|B|2021-01-04|sushi|10|N|
|B|2021-01-11|sushi|10|Y|
|B|2021-01-16|ramen|12|Y|
|B|2021-02-01|ramen|12|Y|
|C|2021-01-01|ramen|12|N|
|C|2021-01-01|ramen|12|N|
|C|2021-01-07|ramen|12|N|

The CASE expression is used to create the members column, showing if the customer was a loyalty scheme member when they made the purchase. A LEFT JOIN is used when joining the sales and members tables to ensure the non-member customers are still included in the result. Customers who are part of the loyalty scheme make repeat purchases after becoming a member.

\
**Question 2**: Rank All Things

```sql
WITH joins AS
	(SELECT
		s.customer_id,
		order_date,
		product_name,
		price,
		CASE WHEN join_date IS NULL OR order_date < join_date THEN 'N'
		ELSE 'Y'
		END AS member
	FROM dannys_diner.sales AS s
	LEFT JOIN dannys_diner.members AS mem
	ON s.customer_id = mem.customer_id
	JOIN dannys_diner.menu AS m
	ON s.product_id = m.product_id)
SELECT 
	customer_id,
	order_date,
	product_name,
	price,
	member,
	CASE WHEN member ='Y' THEN DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date)
	ELSE NULL 
	END AS ranking
FROM joins;
```

|customer_id|order_date|product_name|price|member|ranking|
|---|---|---|---|---|---|
|A|2021-01-01|sushi|10|N|NULL|
|A|2021-01-01|curry|15|N|NULL|
|A|2021-01-07|curry|15|Y|1|
|A|2021-01-10|ramen|12|Y|2|
|A|2021-01-11|ramen|12|Y|3|
|A|2021-01-11|ramen|12|Y|3|
|B|2021-01-01|curry|15|N|NULL|
|B|2021-01-02|curry|15|N|NULL|
|B|2021-01-04|sushi|10|N|NULL|
|B|2021-01-11|sushi|10|Y|1|
|B|2021-01-16|ramen|12|Y|2|
|B|2021-02-01|ramen|12|Y|3|
|C|2021-01-01|ramen|12|N|NULL|
|C|2021-01-01|ramen|12|N|NULL|
|C|2021-01-07|ramen|12|N|NULL|

This solution initially had a couple more CTEs until I learnt you can add a window function in a CASE expression. The ranking column gives a cumulative total for the number of orders made by loyalty scheme members. Both customers have made 3 purchases since being loyalty scheme members.

## Conclusion - Insights and Recommendations

Customers who joined the loyalty scheme have purchased more than the customer who did not join the loyalty scheme. Ramen is the most popular product. More ramen options could be popular with customers. The loyalty scheme could be used to boost purchases of less popular products. Two of the customers bought curry in their first order; maybe the curry product entices people to try the diner. However, this case study has only used a very small amount of data. Analysis of a larger customer group and time frame would be needed to give further backing to these insights.

