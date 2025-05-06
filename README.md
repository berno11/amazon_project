# amazon_project
---

# **Amazon USA Sales Analysis Project**

### **Difficulty Level: Advanced**

---

## **Project Overview**

I have worked on analyzing a dataset of over 20,000 sales records from an Amazon-like e-commerce platform. This project involves extensive querying of customer behavior, product performance, and sales trends using PostgreSQL. Through this project, I have tackled various SQL problems, including revenue analysis, customer segmentation, and inventory management.

The project also focuses on data cleaning, handling null values, and solving real-world business problems using structured queries.

An ERD diagram is included to visually represent the database schema and relationships between tables.

---

![ERD Scratch](https://github.com/najirh/amazon_usa_project5/blob/main/erd2.png)

## **Database Setup & Design**

### **Schema Structure**

```sql
CREATE TABLE category
(
  category_id	INT PRIMARY KEY,
  category_name VARCHAR(20)
);

-- customers TABLE
CREATE TABLE customers
(
  customer_id INT PRIMARY KEY,	
  first_name	VARCHAR(20),
  last_name	VARCHAR(20),
  state VARCHAR(20),
  address VARCHAR(5) DEFAULT ('xxxx')
);

-- sellers TABLE
CREATE TABLE sellers
(
  seller_id INT PRIMARY KEY,
  seller_name	VARCHAR(25),
  origin VARCHAR(15)
);

-- products table
  CREATE TABLE products
  (
  product_id INT PRIMARY KEY,	
  product_name VARCHAR(50),	
  price	FLOAT,
  cogs	FLOAT,
  category_id INT, -- FK 
  CONSTRAINT product_fk_category FOREIGN KEY(category_id) REFERENCES category(category_id)
);

-- orders
CREATE TABLE orders
(
  order_id INT PRIMARY KEY, 	
  order_date	DATE,
  customer_id	INT, -- FK
  seller_id INT, -- FK 
  order_status VARCHAR(15),
  CONSTRAINT orders_fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
  CONSTRAINT orders_fk_sellers FOREIGN KEY (seller_id) REFERENCES sellers(seller_id)
);

CREATE TABLE order_items
(
  order_item_id INT PRIMARY KEY,
  order_id INT,	-- FK 
  product_id INT, -- FK
  quantity INT,	
  price_per_unit FLOAT,
  CONSTRAINT order_items_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id),
  CONSTRAINT order_items_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- payment TABLE
CREATE TABLE payments
(
  payment_id	
  INT PRIMARY KEY,
  order_id INT, -- FK 	
  payment_date DATE,
  payment_status VARCHAR(20),
  CONSTRAINT payments_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE shippings
(
  shipping_id	INT PRIMARY KEY,
  order_id	INT, -- FK
  shipping_date DATE,	
  return_date	 DATE,
  shipping_providers	VARCHAR(15),
  delivery_status VARCHAR(15),
  CONSTRAINT shippings_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE inventory
(
  inventory_id INT PRIMARY KEY,
  product_id INT, -- FK
  stock INT,
  warehouse_id INT,
  last_stock_date DATE,
  CONSTRAINT inventory_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
  );
```

---

## **Task: Data Cleaning**

I cleaned the dataset by:
- **Removing duplicates**: Duplicates in the customer and order tables were identified and removed.
- **Handling missing values**: Null values in critical fields (e.g., customer address, payment status) were either filled with default values or handled using appropriate methods.

---

## **Handling Null Values**

Null values were handled based on their context:
- **Customer addresses**: Missing addresses were assigned default placeholder values.
- **Payment statuses**: Orders with null payment statuses were categorized as “Pending.”
- **Shipping information**: Null return dates were left as is, as not all shipments are returned.

---

## **Objective**

The primary objective of this project is to showcase SQL proficiency through complex queries that address real-world e-commerce business challenges. The analysis covers various aspects of e-commerce operations, including:
- Customer behavior
- Sales trends
- Inventory management
- Payment and shipping analysis
- Forecasting and product performance
  

## **Identifying Business Problems**

Key business problems identified:
1. Low product availability due to inconsistent restocking.
2. High return rates for specific product categories.
3. Significant delays in shipments and inconsistencies in delivery times.
4. High customer acquisition costs with a low customer retention rate.

---

## **Solving Business Problems**

### Solutions Implemented:
1. Top Selling Products
Query the top 10 products by total sales value.
Challenge: Include product name, total quantity sold, and total sales value.

```sql
ALTER TABLE order_items
DROP COLUMN total_sales_amount;

---add total_sales_amount column to the order_items table

ALTER TABLE order_items
ADD COLUMN total_sales FLOAT;

--updating price qty * price_per_unit
UPDATE order_items
SET total_sales = quantity * price_per_unit;

---JOIN tables orders,order_items, product table


SELECT
        oi.product_id,
        p.product_name,
        ROUND(SUM(oi.total_sales::NUMERIC),2) as total_sale,
		COUNT(oi.quantity) as total_quantity
FROM orders o
JOIN order_items oi ON oi.order_id =o.order_id
JOIN products p ON p.product_id = oi.product_id
GROUP BY 1,2
ORDER BY 3 DESC

---QUERY with currency formatting

SELECT
    oi.product_id,
    p.product_name,
	COUNT(oi.quantity) as total_quantity,
    TO_CHAR(ROUND(SUM(oi.total_sales::NUMERIC), 2), 'FM$999,999,999.00') AS total_sale   
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON p.product_id = oi.product_id
GROUP BY 1, 2
ORDER BY ROUND(SUM(oi.total_sales::NUMERIC), 2) DESC
LIMIT 10;


--::NUMERIC is the correct way to cast when you want to round to decimal places in PostgreSQL.

---FLOAT / DOUBLE PRECISION will work with ROUND() only without the second argument â€” i.e., rounding to nearest integer.


```

2. Revenue by Category
Calculate total revenue generated by each product category.
Challenge: Include the percentage contribution of each category to total revenue.

```sql
---option 1 for calculating the percentage of total
SELECT 
    c.category_name,
    ROUND(SUM(oi.total_sales::NUMERIC),2) AS total_revenue,
    ROUND(SUM(oi.total_sales::NUMERIC) * 100.0 /SUM(SUM(oi.total_sales::NUMERIC)) OVER (),2) AS percentage_of_total
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
JOIN category c ON p.category_id = c.category_id
GROUP BY c.category_name
ORDER BY total_revenue DESC;

---option 2 for calculating percentage of total

SELECT
	p.category_id,
	c.category_name,
	ROUND(SUM(oi.total_saleS::NUMERIC),2) as total_revenue,
	ROUND(SUM(oi.total_saleS::NUMERIC)/(SELECT SUM(total_sales::NUMERIC) FROM order_items),3)*100 as contribution
FROM order_items as oi
JOIN
products as p
ON p.product_id = oi.product_id
LEFT JOIN category as c
ON c.category_id = p.category_id
GROUP BY 1,2
ORDER BY 3 DESC
```

3. Average Order Value (AOV)
Compute the average order value for each customer.
Challenge: Include only customers with more than 5 orders.

```sql
SELECT
c.customer_id,
CONCAT(c.first_name,' ',c.last_name)	AS full_name,	   
COUNT( o.order_id) as num_of_orders,	   
ROUND(SUM(oi.total_sales::NUMERIC)/count(o.order_id::NUMERIC),2) as AOV
	FROM orders o
	JOIN customers c
	ON c.customer_id = o.customer_id
	JOIN order_items oi
	ON oi.order_id = o.order_id
	GROUP BY 1,2  
HAVING COUNT( o.order_id)>5
ORDER BY 3 DESC
```

4. Monthly Sales Trend
Query monthly total sales over the past year.
Challenge: Display the sales trend, grouping by month, return current_month sale, last month sale!

```sql
SELECT 
	year,
	month,
	total_sale as current_month_sale,
	LAG(total_sale, 1) OVER(ORDER BY year, month) as previous_month_sale
FROM
(SELECT
 EXTRACT(MONTH FROM o.order_date) as month,
 EXTRACT(YEAR FROM o.order_date) as year,
 ROUND(SUM(oi.total_sales::NUMERIC),2) as total_sale
FROM orders o
JOIN order_items oi
ON oi.order_id = o.order_id
WHERE order_date >= (CURRENT_DATE - INTERVAL '24 months')
GROUP BY 1,2
ORDER BY 2,1 ) as T1
```


5. Customers with No Purchases
Find customers who have registered but never placed an order.
Challenge: List customer details and the time since their registration.

```sql
Approach 1
SELECT 
  *
  FROM customers
  WHERE customer_id NOT IN
(SELECT DISTINCT customer_id
   FROM orders)

```
```sql
-- Approach 2
SELECT
*
FROM customers c
LEFT JOIN orders o
ON o.customer_id = c.customer_id
WHERE o.customer_id is null
```

6. Least-Selling Categories by State
Identify the least-selling product category for each state.
Challenge: Include the total sales for that category within each state.

```sql
SELECT
	state,
    category_name,
	total_sale
FROM
(SELECT 
 	c.state,
   	cc.category_name,
	ROUND(SUM(oi.total_sales::NUMERIC),2) as total_sale,
    RANK() OVER(PARTITION BY state ORDER BY ROUND(SUM(oi.total_sales::NUMERIC),2))rnk
FROM customers c
JOIN orders o
ON c.customer_id = o.customer_id
JOIN order_items oi
ON oi.order_id = o.order_id
JOIN products p
ON p.product_id = oi.product_id
JOIN category cc
ON cc.category_id = p.category_id
GROUP BY 1,2)tt
WHERE rnk = 1
```


7. Customer Lifetime Value (CLTV)
Calculate the total value of orders placed by each customer over their lifetime.
Challenge: Rank customers based on their CLTV.

```sql
SELECT
	c.customer_id,
	CONCAT(c.first_name,' ',c.last_name) as full_name,
	ROUND(SUM(oi.total_sales::NUMERIC),2) CLTV,
	DENSE_RANK() OVER(ORDER BY ROUND(SUM(oi.total_sales::NUMERIC),2) DESC )
FROM customers c
	JOIN orders o
	ON c.customer_id = o.customer_id
	JOIN order_items oi
	ON oi.order_id = o.order_id
	GROUP BY 1,2
```


8. Inventory Stock Alerts
Query products with stock levels below a certain threshold (e.g., less than 10 units).
Challenge: Include last restock date and warehouse information.

```sql
SELECT 
	i.inventory_id,
	p.product_name,
	i.stock as current_stock_left,
	i.last_stock_date,
	i.warehouse_id
FROM inventory as i
join 
products as p
ON p.product_id = i.product_id
WHERE stock < 10
```

9. Shipping Delays
Identify orders where the shipping date is later than 3 days after the order date.
Challenge: Include customer, order details, and delivery provider.

```sql
SELECT 
 CONCAT(c.first_name,' ',last_name) as full_name,
 o.*,
 shipping_date,
 s.shipping_providers as delivery_provider,
 (s.shipping_date::DATE - o.order_date::DATE) AS date_diff 
FROM shipping s
JOIN orders o
ON s.order_id = o.order_id
JOIN customers c
ON c.customer_id = o.customer_id
WHERE (s.shipping_date::DATE - o.order_date::DATE)>3
```

10. Payment Success Rate 
Calculate the percentage of successful payments across all orders.
Challenge: Include breakdowns by payment status (e.g., failed, pending).

```sql
SELECT 
	p.payment_status,
	COUNT(*) AS total_cnt,
ROUND(COUNT(*)::NUMERIC/(SELECT COUNT(*)::NUMERIC FROM payments)*100,2) percentage_of_successesfullpayments
FROM orders o
JOIN 
payments as p
ON o.order_id = p.order_id
GROUP BY 1
```

11. Top Performing Sellers
Find the top 5 sellers based on total sales value.
Challenge: Include both successful and failed orders, and display their percentage of successful orders.

```sql
WITH top_5_sellers as
(SELECT
	 s.seller_id,
	 s.seller_name,
	 ROUND(SUM(oi.total_sales::NUMERIC),2) as total_value
FROM sellers s
JOIN orders o
ON s.seller_id = o.seller_id
JOIN order_items oi
ON oi.order_id = o.order_id
GROUP BY 1,2
ORDER BY 3 DESC
LIMIT 5),
order_status as
(SELECT
   o.seller_id,
   ts.seller_name,
	o.order_status,
    count(*) as total_orders
 FROM sellers ts
 JOIN orders o
 ON ts.seller_id = o.seller_id
 WHERE o.order_status NOT IN('Inprogress', 'Returned')
 GROUP BY 1,2,3
 ORDER BY 1)
SELECT
   seller_id,
   seller_name,
SUM(CASE WHEN order_status = 'Completed' THEN total_orders
   ELSE 0 END) as completed_orders,
SUM(CASE WHEN order_status = 'Cancelled' THEN total_orders
   ELSE 0 END) as cancelled_orders, 
 SUM(total_orders) as total_orders,
 ROUND(SUM(CASE WHEN order_status = 'Completed' THEN total_orders
   ELSE 0 END)::NUMERIC/SUM(total_orders)::NUMERIC * 100,2)
   as successful_orders_percentage,
FROM order_status
GROUP BY 1,2
ORDER BY 5 DESC
```


12. Product Profit Margin
Calculate the profit margin for each product (difference between price and cost of goods sold).
Challenge: Rank products by their profit margin, showing highest to lowest.
*/


```sql
SELECT 
	p.product_id,
	p.product_name,
	ROUND(SUM(oi.total_sales - (p.cogs * oi.quantity))::NUMERIC,2) as profit_margin,
	DENSE_RANK() OVER(ORDER BY ROUND(SUM(oi.total_sales - (p.cogs * oi.quantity))::NUMERIC,2) DESC )product_rank
FROM products p
JOIN order_items oi
ON p.product_id = oi.product_id
GROUP BY 1,2
```

13. Most Returned Products
Query the top 10 products by the number of returns.
Challenge: Display the return rate as a percentage of total units sold for each product.

```sql
WITH returned_orders as
(SELECT 
	p.product_id,
	p.product_name,
	o.order_status,
	COUNT(*)Num_of_returns
FROM products p
JOIN order_items oi
ON p.product_id = oi.product_id
JOIN orders o
ON o.order_id = oi.order_id
WHERE order_status = 'Returned'
GROUP BY 1,2,3
ORDER BY 4 DESC
LIMIT 10),
total_orders as
(SELECT 
   ro.product_id,
   ro.product_name,
   SUM(oi.quantity) total_units_sold,
   ro.Num_of_returns
 FROM returned_orders ro
 JOIN order_items oi
 ON ro.product_id = oi.product_id
 GROUP BY 1,2,4)
 
 SELECT 
   product_id,
   product_name,
   total_units_sold,
   num_of_returns,
   round(num_of_returns::NUMERIC/total_units_sold::NUMERIC *100,2) as return_rate
  FROM total_orders
ORDER BY 5 DESC
```

14. Inactive Sellers
Identify sellers who haven’t made any sales in the last 6 months.
Challenge: Show the last sale date and total sales from those sellers.

```sql
WITH cte1 -- as these sellers has not done any sale in last 6 month
AS
(SELECT * FROM sellers
WHERE seller_id NOT IN (SELECT seller_id FROM orders WHERE order_date >= DATE '2024-07-30'- INTERVAL '6 month')
)

SELECT 
o.seller_id,
MAX(o.order_date) as last_sale_date,
MAX(oi.total_sales) as last_sale_amount
FROM orders as o
JOIN 
cte1
ON cte1.seller_id = o.seller_id
JOIN order_items as oi
ON o.order_id = oi.order_id
GROUP BY 1
```


15. IDENTITY customers into returning or new
if the customer has done more than 5 return categorize them as returning otherwise new
Challenge: List customers id, name, total orders, total returns

```sql
SELECT
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(o.order_id) AS total_orders,
    SUM(CASE WHEN o.order_status = 'Returned' THEN 1 ELSE 0 END) AS total_returns,
    CASE 
        WHEN SUM(CASE WHEN o.order_status = 'Returned' THEN 1 ELSE 0 END) > 5 THEN 'Returning'
        ELSE 'New'
    END AS customer_type
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
GROUP BY 1
ORDER BY 2 DESC;
```


16. Top 5 Customers by Orders in Each State
Identify the top 5 customers with the highest number of orders for each state.
Challenge: Include the number of orders and total sales for each customer.
```sql
WITH cte as
(SELECT
	CONCAT(c.first_name,' ',last_name) full_name,
	c.state,
    ROUND(SUM(oi.total_sales::NUMERIC),2) AS total_sales,
	count(o.order_id)total_number_of_orders,
	DENSE_RANK() OVER(PARTITION BY state ORDER BY COUNT(o.order_id)DESC )RNK
FROM customers c
JOIN orders o
ON c.customer_id = o.customer_id
JOIN order_items oi
ON oi.order_id = o.order_id
GROUP BY 1,2)

SELECT 
  full_name,
  total_sales,
  total_number_of_orders
  FROM cte
  WHERE rnk <=5
```

17. Revenue by Shipping Provider
Calculate the total revenue handled by each shipping provider.
Challenge: Include the total number of orders handled and the average delivery time for each provider.

```sql
SELECT 
	s.shipping_providers,
	COUNT(o.order_id) as order_handled,
	SUM(oi.total_sale) as total_sale,
	COALESCE(AVG(s.return_date - s.shipping_date), 0) as average_days
FROM orders as o
JOIN 
order_items as oi
ON oi.order_id = o.order_id
JOIN 
shippings as s
ON 
s.order_id = o.order_id
GROUP BY 1
```

18. Top 10 product with highest decreasing revenue ratio compare to last year(2022) and current_year(2023)
Challenge: Return product_id, product_name, category_name, 2022 revenue and 2023 revenue decrease ratio at end Round the result
Note: Decrease ratio = cr-ls/ls* 100 (cs = current_year ls=last_year)

```sql
WITH last_year_sales as
(SELECT
   p.product_id,
   p.product_name,
   ROUND(SUM(oi.total_sales::NUMERIC),2) AS last_year_revenue
FROM products p
JOIN order_items oi
ON p.product_id = oi.product_id
JOIN orders o
ON o.order_id = oi.order_id
WHERE EXTRACT(YEAR FROM o.order_date)=2022
GROUP BY 1,2),

current_year_sales as
(SELECT
   p.product_id,
   p.product_name,
  ROUND(SUM(oi.total_sales::NUMERIC),2) AS current_year_revenue
FROM products p
JOIN order_items oi
ON p.product_id = oi.product_id
JOIN orders o
ON o.order_id = oi.order_id
WHERE EXTRACT(YEAR FROM o.order_date)=2023
GROUP BY 1,2)

SELECT 
     cy.product_id,
	 cy.product_name,
	 cy.current_year_revenue,
	 ly.last_year_revenue,
	 (ly.last_year_revenue - cy.current_year_revenue) as diff,
	ROUND((cy.current_year_revenue - ly.last_year_revenue)::NUMERIC/ly.last_year_revenue::NUMERIC *100,2) AS revenue_decrease_ratio
FROM current_year_sales cy
JOIN last_year_sales ly
ON cy.product_id = ly.product_id
WHERE ly.last_year_revenue > cy.current_year_revenue
ORDER BY 6 desc
LIMIT 10
```


19. Final Task: Stored Procedure
Create a stored procedure that, when a product is sold, performs the following actions:
Inserts a new sales record into the orders and order_items tables.
Updates the inventory table to reduce the stock based on the product and quantity purchased.
The procedure should ensure that the stock is adjusted immediately after recording the sale.

```SQL
--20. Store Procedure
--create a function as soon as the product is sold the the same quantity should reduced from inventory table
--after adding any sales records it should update the stock in the inventory table based on the product and qty purchased


/* TO automate what happens when a sale is made:
1. Insert a new order and order item
2. Check stock availability
3. Update the inventory stock level
4. Send a message dependinh on whether the item was available or not*/



SELECT * FROM products
-- product_id 1 -- airpod 3rd gen -- 55stock
-- produ id 2 airpod max --39

SELECT * FROM inventory
WHERE product_id = 1;

SELECT * FROM orders;
SELECT * FROM order_items;
SELECT * FROM inventory;
SELECT * FROM products
order_id,
order_date,
customer_id,
seller_id,
order_item_id,
product_id,
quantity,

/*This procedure takes 6 inputs; orderID, customerID, sellerID, orderitemID, productID,and qty purchased.*/
CREATE OR REPLACE PROCEDURE add_sales
(
p_order_id INT,
p_customer_id INT,
p_seller_id INT,
p_order_item_id INT,
p_product_id INT,
p_quantity INT
)
LANGUAGE plpgsql
AS $$

/*Declare temporary variables;

v_price: to store the unit price of the product.

v_product: to store the name of the product.

v_count: used to check if the inventory has enough stock.*/

DECLARE 
v_count INT;
v_price FLOAT;
v_product VARCHAR(50);

BEGIN
/*Step 1: Get Product Info - Looks up the price and name for the product being odered and stores them in variables */
	SELECT 
		price, product_name
		INTO
		v_price, v_product
	FROM products
	WHERE product_id = p_product_id;
	
/* Step 2: checking stock and product availability in inventory. Counts if there's at least one inventory recored where the stock is
greater than or equal to the purchase quantity */
	SELECT 
		COUNT(*) 
		INTO v_count
	FROM inventory
		WHERE product_id = p_product_id
		AND stock >= p_quantity;
/* Step 3: If product Is in stock. If there's enough stock,proceed to */		
	IF v_count > 0 THEN
	-- add into orders and order_items table
	-- update inventory
	/*3a. Insert a new record into orders*/
	INSERT INTO orders(order_id, order_date, customer_id, seller_id)
	VALUES (p_order_id, CURRENT_DATE, p_customer_id, p_seller_id);
    /*3b. Insert a new record into order_items*/
		INSERT INTO order_items(order_item_id, order_id, product_id, quantity, price_per_unit, total_sales)
		VALUES
		(p_order_item_id, p_order_id, p_product_id, p_quantity, v_price, v_price*p_quantity);

		/*3c.updating inventory*/
		UPDATE inventory
		SET stock = stock - p_quantity
		WHERE product_id = p_product_id;
		/*3d.Show confirmation message*/
		
		RAISE NOTICE 'Thank you product: % sale has been added also inventory stock updates',v_product; 
/*Step 4: If Product Is not in stock prints a message that the product is not available in sufficicent quantity*/
	ELSE
		RAISE NOTICE 'Thank you for for your info the product: % is not available', v_product;

	END IF;


END;
$$

/*USAGE*
Means: Order ID -25005, CustomerID - 2, Seller Id - 5, Order_item id - 25004, product ID -1, Quantity -14*/

call add_sales
(
25005, 2, 5, 25004, 1, 14
);


(
p_order_id INT,
p_customer_id INT,
p_seller_id INT,
p_order_item_id INT,
p_product_id INT,
p_quantity INT

/*How to test it. run the procedure with a CALL, Re-run the inventory check to verify the stock reduced.*/
SELECT COUNT(*) 
FROM inventory
WHERE 
	product_id = 1
	AND 
	stock >= 56
```

---

## **Learning Outcomes**

This project enabled me to:
- Design and implement a normalized database schema.
- Clean and preprocess real-world datasets for analysis.
- Use advanced SQL techniques, including window functions, subqueries, and joins.
- Conduct in-depth business analysis using SQL.
- Optimize query performance and handle large datasets efficiently.

---

## **Conclusion**

This advanced SQL project successfully demonstrates my ability to solve real-world e-commerce problems using structured queries. From improving customer retention to optimizing inventory and logistics, the project provides valuable insights into operational challenges and solutions.

By completing this project, I have gained a deeper understanding of how SQL can be used to tackle complex data problems and drive business decision-making.

---

### **Entity Relationship Diagram (ERD)**
![ERD](https://github.com/najirh/amazon_usa_project5/blob/main/erd.png)

---
