# ANALYZING NORTHWIND TRADERS: A SQL-BASED APPROACH TO BUSINESS INTELLIGENCE

## Table of contents

- [Project Overview](project_overview)
- [Objectives](objectives)
- [Tools](tools)
- [Data Dictionary](data_dictionary)
- [Data Cleaning/Preparations](data_cleaning/preparations)
- [Exporatory Data Analysis](exporatory_data_analysis)
- [Data Analysis](data_analysis)
- [Results/Findings](results/findings)
- [Recommendations](recommendations)
- [Limitations](limitations)
- [References](references)
### Project Overview

This project analyzes the Northwind Traders database, a widely used sample dataset that models a global food and beverage trading company. Through the application of 12 structured SQL queries, I investigated sales performance, customer behavior, shipping operations, and regional trends to identify actionable business insights. The database consists of seven interconnected tables, such as customers, orders, order details, and products, and records a total revenue of $1.26 million from 51,300 orders placed by 830 customers. Each query was formulated to address a specific business question, thereby transforming raw data into insights that support strategic decision-making.

### Objectives

- Sales trends – Monthly and quarterly sales analysis
- Customer analysis – Top customers, dormant accounts, delay impacts
- Shipping performance – Costs, delays by product, customer, region
- Product performance – Top products, regional preferences
- Regional insights – Revenue leaders, underperformers, highest revenue country

### Tools

- SQL Server – Database engine for query execution
- T-SQL – CTEs, window functions, date functions, subqueries, complex joins
- Excel – Results organization, validation and visualization

### Data Dictionary

- Category: This table contains information about categories, including category ID, category name and description.
- Customers: This table contains information about customers, including customer ID, company name, contact name, contact title, city and country.
- Employees: This table contains information about employees, including employee ID, employee name, title, city, country and reports to.
- Order.Details: This table contains information about orders, including order ID, product ID, unit price, quantity and discount.
- Order: This table contains information about the order record, including order ID, customer ID, employee ID, order date, required date, shipped date, shipper ID and freight.
- Products: This table contains information about the products, including product ID, product name, quantity per unit, unit price, discontinued and category ID.
- Shippers: This table contains information about the shipper, including shipper ID and company name.

### Data Cleaning/Preparations

- Handled NULL values in shippedDate (excluded from delay calculations)
- Treated NULL discounts as zero using IS NULL
- Extracted year, month, and quarter using DATEPART
- Calculated actual sales: (unitPrice × quantity) – (unitPrice × quantity × discount)
- Identified shipping delays: DATEDIFF(DAY, shippedDate, requiredDate) < 0
- Created dynamic 6-month cutoff: DATEADD(MONTH, -6, MAX(orderDate))
- Rounded currency values to 2 decimals for readability
- Used DENSE_RANK for regional product rankings
- Verified no negative values or duplicates existed

### Exploratory Data Analysis

EDA involves exploring the data to answer key questions, such as:

1. What is the monthly sales trend over the years
2. What are the top 10 customers by total revenue
3. What is the quarter-over-quarter growth rate
4. Who are the customers that hasn’t ordered in the last 6 months
5. What are the average shipping costs by destination country/region
6. Which products are most frequently causing shipping delays
7. Which customer has the most shipping delays
8. Which regions have the most shipping delays
9. Identify underperforming or overperforming regions by total sales
10. What are the top 5 products by quantity sold
11. Do the top products by quantity sold perform consistently across all region
12. Which country generated the highest revenue

### Data Analysis

1.  What is the monthly sales trend over the years
```sql
WITH sales AS (
	SELECT orderID, 
	(unitPrice * quantity) AS total_sales,
	(unitPrice * quantity * discount) AS total_discount
	FROM order_details
), actual_sales AS (
	SELECT orderID, (total_sales - total_discount) AS actual_sales
	FROM sales
)
SELECT DATEPART(year, O.orderDate) AS sales_year,
	DATEPART(month, O.orderDate) AS sales_month,
	SUM(A.actual_sales) AS totalsales
FROM actual_sales AS A
INNER JOIN orders AS O 
ON A.orderID = O.orderID
GROUP BY DATEPART(year, O.orderDate), DATEPART(month, O.orderDate)
ORDER BY sales_year, sales_month;
```

2. What are the top 10 customers by total revenue
``` sql
WITH sales AS (
	SELECT orderID, 
	(unitPrice * quantity) AS total_sales,
	(unitPrice * quantity * discount) AS total_discount
	FROM order_details
), actual_sales AS (
	SELECT orderID, (total_sales - total_discount) AS actual_sales
	FROM sales
) 
SELECT TOP 10 C.contactName, SUM(A.actual_sales) AS total_sales
FROM actual_sales AS A
INNER JOIN orders AS O
ON A.orderID = O.orderID
INNER JOIN customers AS C
ON O.customerID = c.customerID
GROUP BY C.contactName 
ORDER BY total_sales DESC;
```

3. What is the quarter-over-quarter growth rate
``` sql
WITH sales AS (
	SELECT orderID, 
	(unitPrice * quantity) AS total_sales,
	(unitPrice * quantity * discount) AS total_discount
	FROM order_details
), actual_sales AS (
	SELECT orderID, (total_sales - total_discount) AS actual_sales
	FROM sales
), sales_quarter AS (
SELECT DATEPART(year, O.orderDate) AS sales_year,
	DATEPART(QUARTER, O.orderDate) AS sales_quarter,
	SUM(A.actual_sales) AS totalsales
FROM actual_sales AS A
INNER JOIN orders AS O 
ON A.orderID = O.orderID
GROUP BY DATEPART(year, O.orderDate), DATEPART(QUARTER, O.orderDate)
)
SELECT sales_year, sales_quarter, totalsales,
		ROUND(LAG(totalsales, 4) OVER ( ORDER BY sales_year, sales_quarter),2) AS previous_quarter,
		ROUND((totalsales - (LAG(totalsales, 4) OVER ( ORDER BY sales_year, sales_quarter))) / (LAG(totalsales, 4) OVER ( ORDER BY sales_year, sales_quarter)) * 100,2) AS growth_rate
FROM sales_quarter;
```

4. Who are the customers that hasn’t ordered in the last 6 months
``` sql
 SELECT C.contactName,C.country, MAX(O.orderDate) AS last_orderdate, (SELECT DATEADD(MONTH,-6, MAX(orderDate)) FROM orders) AS last_six_months
 FROM orders AS O 
 INNER JOIN customers AS C
 ON C.customerID = O.customerID
 WHERE orderdate NOT IN (SELECT orderDate 
			FROM orders 
			WHERE orderDate  >= DATEADD(MONTH,-6, (SELECT MAX(orderDate) FROM orders)))
GROUP BY C.contactName, C.country;
```

5. What are the average shipping costs by destination country/region
``` sql
SELECT C.country,ROUND(AVG(O.freight), 2) AS avg_shipping_cost
FROM orders AS O
INNER JOIN customers AS C 
ON O.customerID = C.customerID
GROUP BY C.country
ORDER BY avg_shipping_cost DESC;
```

6. Which products are most frequently causing shipping delays
``` sql
SELECT productName, COUNT(delivery_day) AS frequency
FROM 
(SELECT  P.productName, O.shippedDate, O.requiredDate, DATEDIFF(DAY, shippedDate, requiredDate) AS delivery_day
FROM orders AS O
INNER JOIN order_details AS D
ON O.orderID = D.orderID
INNER JOIN products AS P
ON D.productID = P.productID
WHERE DATEDIFF(DAY, shippedDate, requiredDate) < 0 AND DATEDIFF(DAY, shippedDate, requiredDate) IS NOT NULL)AS sub_query 
GROUP BY productName
ORDER BY frequency DESC;
```

7. Which customer has the most shipping delays
``` sql
SELECT contactName, COUNT(delivery_day) AS frequency
FROM 
(SELECT  c.contactName, O.shippedDate, O.requiredDate, DATEDIFF(DAY, shippedDate, requiredDate) AS delivery_day
FROM orders AS O
INNER JOIN customers AS C
ON O.customerID = C.customerID
WHERE DATEDIFF(DAY, shippedDate, requiredDate) < 0 AND DATEDIFF(DAY, shippedDate, requiredDate) IS NOT NULL)AS sub_query 
GROUP BY contactName
ORDER BY frequency DESC;
```

8. Which regions have the most shipping delays
``` sql
SELECT country, COUNT(delivery_day) AS frequency
FROM 
(SELECT  c.country, O.shippedDate, O.requiredDate, DATEDIFF(DAY, shippedDate, requiredDate) AS delivery_day
FROM orders AS O
INNER JOIN customers AS C
ON O.customerID = C.customerID
WHERE DATEDIFF(DAY, shippedDate, requiredDate) < 0 AND DATEDIFF(DAY, shippedDate, requiredDate) IS NOT NULL)AS sub_query 
GROUP BY country
ORDER BY frequency DESC;
```

9. Identify underperforming or overperforming regions by total sales
``` sql
   WITH sales AS (
	SELECT O.orderID, O.customerID, 
	(unitPrice * quantity) AS total_sales,
	(unitPrice * quantity * discount) AS total_discount
	FROM order_details AS D
	INNER JOIN orders AS O
	ON D.orderID = O.orderID
), actual_sales AS (
	SELECT orderID ,customerID, (total_sales - total_discount) AS actual_sales
	FROM sales
)
SELECT C.country, ROUND(SUM(A.actual_sales), 2) AS total_sales
FROM actual_sales AS A
INNER JOIN customers AS C
ON A.customerID = C.customerID
GROUP BY C.country
ORDER BY total_sales DESC;
```

10. What are the top 5 products by quantity sold
``` sql
SELECT TOP 5 P.productName, SUM(D.quantity) AS product_quantity
FROM products AS P 
INNER JOIN order_details AS D
ON p.productID = D.productID
GROUP BY P.productName
ORDER BY product_quantity DESC;
```

11. Do the top products by quantity sold perform consistently across all region
``` sql
WITH top_ranking AS (
SELECT C.country, P.productName, SUM(D.quantity) AS product_quantity,
		DENSE_RANK() OVER(PARTITION BY C.country ORDER BY SUM(D.quantity) DESC) AS ranking
FROM products AS P 
INNER JOIN order_details AS D ON p.productID = D.productID
INNER JOIN orders AS O ON D.orderID = O.orderID
INNER JOIN customers AS C ON O.customerID = C.customerID
GROUP BY C.country, P.productName
)
SELECT country, productName, product_quantity, ranking
FROM top_ranking
WHERE ranking <= 3
ORDER BY country, productName, ranking;
```

12. Which country generated the highest revenue
``` sql
SELECT  TOP 3 C.country, ROUND(SUM(unitPrice * quantity), 2) AS total_sales
FROM customers AS C
INNER JOIN orders AS O 
ON c.customerID = O.customerID
INNER JOIN order_details AS D
ON O.orderID = D.orderID
GROUP BY C.country
ORDER BY total_sales DESC;
```

### Results/Findings

The analysis results are summarized as follows

#### Sales Performance

- Total revenue: $1.26M from 51,300 orders
- Monthly sales grew 4.4x ($27.8K → $123.8K)
- Q1 2015: +116% YoY growth
- Q2 2015: -0.73% (first plateau)

#### Customers

- Top 3: Horst Kloss ($110K), Roland Mendel ($105K), José Pavarotti ($104K)
- 81 dormant customers – including top revenue generators
- Most delayed: Patricia McKenna (3 delays)

#### Shipping

- Cost range: Austria ($184) to Poland ($25) – 7x difference
- Delay hotspots: USA (7), UK (4), Germany (4)
- Problem products: Sir Rodney's Marmalade (5), Camembert (4)

#### Regional

- Top markets: USA ($245K), Germany ($230K), Austria ($128K)
- UK underperforms: $59K (1/4 of Germany's revenue)
- Austria overperforms: #3 market despite small population

#### Products

- Top sellers: Camembert (1,577), Raclette (1,496), Gorgonzola (1,397)
- Global: Cheese dominates everywhere
- Local favorites: Gnocchi (USA), Guarana (Austria), Outback Lager (Venezuela)
- Camembert: Top seller + top delay offender

### Recommendations

Based on the analysis we recommend the following actions:

Customer Retention
- 81 dormant customers identified, including top earners Jose Pavarotti and Roland Mendel
- Launch an immediate win-back campaign with personal outreach for high-value accounts
- Retaining customers costs 5x less than acquiring new ones

Shipping Improvements
- USA leads with 7 shipping delays, affecting top customers like Horst Kloss
- Problem products: Sir Rodney's Marmalade (5 delays), Camembert Pierrot (4 delays)
- Audit US operations and create VIP shipping protocol for top clients

Regional Strategy
- The USA and Germany generate nearly half of the total revenue
- Austria overperforms despite a small population
- UK underperforms dramatically compared to Germany – investigate why
- Poland and Norway have low revenue, but shipping cost advantages worth exploring

Product Strategy
- Cheese sells everywhere, but local preferences matter
- Gnocchi dominates in the USA, Guarana in Austria, Outback Lager in Venezuela
- Stock global favorites but celebrate regional heroes

Growth Planning
- Explosive Q1 growth (+116%) followed by Q2 plateau (-0.73%)
- Study what worked in Q1 and prepare for seasonal softening

### Limitations

The data only extends through 2015, so recent trends and changes are not captured. External factors such as competitor activity, economic conditions, or market shifts were not considered in this analysis. Customer information is limited to basic contact details, with no demographic data like age, income, or preferences that might explain purchasing behavior. The analysis focuses on revenue rather than profitability, meaning products with high sales but low margins were not identified. Additionally, no data exists on returns, refunds, or customer satisfaction beyond shipping delays. Marketing campaign data was not available, making it impossible to link sales patterns to specific promotions. Some countries, like Poland, had very few orders, reducing the reliability of their performance metrics. Finally, this analysis is descriptive rather than predictive; it shows what happened but does not forecast future trends. These factors should be considered when interpreting the findings. 

### References

- Microsoft T-SQL Documentation – Reference for SQL functions and syntax
- Northwind Dataset – Original source of all data analyzed in this project
