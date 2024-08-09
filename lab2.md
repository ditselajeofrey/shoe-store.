![image](terraform/img/confluent-logo-300-2.png)
# Lab 2
Finishing Lab 1 is required for Lab 2. If you have not completed it, go back to [Lab 1](lab1.md).


[1. Flink Joins](lab2.md#1-flink-joins)

[2. Understand Timestamps](lab2.md#2-understand-timestamps)

[3. Understand Joins](lab2.md#3-understand-joins)

[4. Data Enrichment](lab2.md#4-data-enrichment)

[5. Loyalty Levels Calculation](lab2.md#5-loyalty-levels-calculation)

[6. Promotions Calculation](lab2.md#6-promotions-calculation)



## 1. Flink Joins

Flink SQL supports complex and flexible join operations over dynamic tables. There are a number of different types of joins to account for the wide variety of semantics that queries may require.
By default, the order of joins is not optimized. Tables are joined in the order in which they are specified in the FROM clause.

You can find more information about Flink SQL Joins [here.](https://docs.confluent.io/cloud/current/flink/reference/queries/joins.html)

### 2. Understand Timestamps
Let's first look at our data records and their timestamps. Open the Flink SQL workspace.

If you left the Flink SQL Workspace or refreshed the page, `catalog` and `database` dropdowns are reset. Make sure they are selected again. 

![image](terraform/img/catalog-and-database-dropdown.png)

Find all customer records for one customer and display the timestamps from when the events were ingested in the `shoe_customers` Kafka topic.
```
SELECT id,$rowtime 
FROM shoe_customers  
WHERE id = 'b523f7f3-0338-4f1f-a951-a387beeb8b6a';
```
NOTE: Check the timestamps from when the customer records were generated.

Find all orders for one customer and display the timestamps from when the events were ingested in the `shoe_orders` Kafka topic.
```
SELECT order_id, customer_id, $rowtime
FROM shoe_orders
WHERE customer_id = 'b523f7f3-0338-4f1f-a951-a387beeb8b6a';
```
NOTE: Check the timestamps when the orders were generated. This is important for the join operations we will do next.

### 3. Understand Joins
Now, we can look at the different types of joins available. 
We will join `order` records and `customer` records.

Join orders with non-keyed customer records (Regular Join):
```
SELECT order_id, shoe_orders.`$rowtime`, first_name, last_name
FROM shoe_orders
INNER JOIN shoe_customers 
ON shoe_orders.customer_id = shoe_customers.id
WHERE customer_id = 'b523f7f3-0338-4f1f-a951-a387beeb8b6a';
```
NOTE: Look at the number of rows returned. There are many duplicates!

Join orders with non-keyed customer records in some time windows (Interval Join):
```
SELECT order_id, shoe_orders.`$rowtime`, first_name, last_name
FROM shoe_orders
INNER JOIN shoe_customers
ON shoe_orders.customer_id = shoe_customers.id
WHERE customer_id = 'b523f7f3-0338-4f1f-a951-a387beeb8b6a' AND
  shoe_orders.`$rowtime` BETWEEN shoe_customers.`$rowtime` - INTERVAL '1' HOUR AND shoe_customers.`$rowtime`;
```

Join orders with keyed customer records (Regular Join with Keyed Table):
```
SELECT order_id, o.`$rowtime`, first_name, last_name
FROM shoe_orders o
INNER JOIN shoe_customers_keyed_<yourname> c
ON o.customer_id = c.customer_id
WHERE c.customer_id = 'b523f7f3-0338-4f1f-a951-a387beeb8b6a';
```
NOTE: Look at the number of rows returned. There are no duplicates! This is because we have only one customer record for each customer id.

Join orders with keyed customer records at the time when order was created (Temporal Join with Keyed Table):
```
SELECT order_id, o.`$rowtime`, first_name, last_name
FROM shoe_orders o
INNER JOIN shoe_customers_keyed_<yourname> FOR SYSTEM_TIME AS OF o.`$rowtime` c
ON o.customer_id = c.customer_id
WHERE c.customer_id = 'b523f7f3-0338-4f1f-a951-a387beeb8b6a';
```
NOTE 1: There might be empty result set if keyed customers tables was created after the order records were ingested in the shoe_orders topic. 

NOTE 2: You can find more information about Temporal Joins with Flink SQL [here.](https://docs.confluent.io/cloud/current/flink/reference/queries/joins.html#temporal-joins)

### 4. Data Enrichment
We can store the result of a join in a new table. 
We will join data from: Order, Customer, Product tables together in a single SQL statement.

Create a new table for `Order <-> Customer <-> Product` join result:
```
CREATE TABLE shoe_order_customer_product_<yourname>(
  order_id INT,
  first_name STRING,
  last_name STRING,
  email STRING,
  brand STRING,
  `model` STRING,
  sale_price INT,
  rating DOUBLE
)WITH (
    'changelog.mode' = 'retract'
);
```

Insert joined data from 3 tables into the new table:
```
INSERT INTO shoe_order_customer_product_<yourname>(
  order_id,
  first_name,
  last_name,
  email,
  brand,
  `model`,
  sale_price,
  rating)
SELECT
  o.order_id,
  c.first_name,
  c.last_name,
  c.email,
  p.brand,
  p.`model`,
  p.sale_price,
  p.rating
FROM 
  shoe_orders o
  INNER JOIN shoe_customers_keyed_<yourname> c 
    ON o.customer_id = c.customer_id
  INNER JOIN shoe_products_keyed_<yourname> p
    ON o.product_id = p.product_id;
```

Verify that the data was joined successfully. 
```
SELECT * FROM shoe_order_customer_product_<yourname>;
```

### 5. Loyalty Levels Calculation

Now we are ready to calculate loyalty levels for our customers.

First let's see which loyalty levels are being calculated:
```
SELECT
  email,
  SUM(sale_price) AS total,
  CASE
    WHEN SUM(sale_price) > 800000 THEN 'GOLD'
    WHEN SUM(sale_price) > 70000 THEN 'SILVER'
    WHEN SUM(sale_price) > 6000 THEN 'BRONZE'
    ELSE 'CLIMBING'
  END AS rewards_level
FROM shoe_order_customer_product_<yourname>
GROUP BY email;
```
NOTE: You might need to change the loyalty level numbers according to the amount of the data you have already ingested.


Prepare the table for loyalty levels:
```
CREATE TABLE shoe_loyalty_levels_<yourname>(
  email STRING,
  total BIGINT,
  rewards_level STRING,
  PRIMARY KEY (email) NOT ENFORCED
);
```

Now you can calculate loyalty levels and store the results in the new table.
```
INSERT INTO shoe_loyalty_levels_<yourname>(
 email,
 total,
 rewards_level)
SELECT
  email,
  SUM(sale_price) AS total,
  CASE
    WHEN SUM(sale_price) > 80000000 THEN 'GOLD'
    WHEN SUM(sale_price) > 7000000 THEN 'SILVER'
    WHEN SUM(sale_price) > 600000 THEN 'BRONZE'
    ELSE 'CLIMBING'
  END AS rewards_level
FROM shoe_order_customer_product_<yourname>
GROUP BY email;
```

Verify your results:
```
SELECT * FROM shoe_loyalty_levels_<yourname>;
```

### 6. Promotions Calculation

Let's find out if some customers are eligible for special promotions.

Find which customer should receive a special promotion for their 10th order of the same shoe brand.
```
SELECT
   email,
   COUNT(*) AS total,
   (COUNT(*) % 10) AS sequence,
   (COUNT(*) % 10) = 0 AS next_one_free
 FROM shoe_order_customer_product_<yourname>
 WHERE brand = 'Jones-Stokes'
 GROUP BY email;
 ```
NOTE: We calculate the number of orders of the brand 'Jones-Stokes' for each customer and offer a free product if it's their 10th order.

Find which customers have ordered related brands in large volumes.
```
SELECT
     email,
     COLLECT(brand) AS products,
     'bundle_offer' AS promotion_name
  FROM shoe_order_customer_product_<yourname>
  WHERE brand IN ('Braun-Bruen', 'Will Inc')
  GROUP BY email
  HAVING COUNT(DISTINCT brand) = 2 AND COUNT(brand) > 10;
```
NOTE: We sum all orders of brands 'Braun-Bruen' and 'Will Inc' for each customer and offer a special promotion if the sum is larger than ten.  

Now we are ready to store the results for all calculated promotions. 

Create a table for promotion notifications:
```
CREATE TABLE shoe_promotions_<yourname>(
  email STRING,
  promotion_name STRING,
  PRIMARY KEY (email) NOT ENFORCED
);
```

Write both calculated promotions in a single statement set to the `shoe_promotions` table.
NOTE: There is a bug in the Web UI. Remove the first two lines and the last line to be able to run the statement.

```
EXECUTE STATEMENT SET 
BEGIN

INSERT INTO shoe_promotions_<yourname>
SELECT
   email,
   'next_free' AS promotion_name
FROM shoe_order_customer_product_<yourname>
WHERE brand = 'Jones-Stokes'
GROUP BY email
HAVING COUNT(*) % 10 = 0;

INSERT INTO shoe_promotions_<yourname>
SELECT
     email,
     'bundle_offer' AS promotion_name
  FROM shoe_order_customer_product_<yourname>
  WHERE brand IN ('Braun-Bruen', 'Will Inc')
  GROUP BY email
  HAVING COUNT(DISTINCT brand) = 2 AND COUNT(brand) > 10;

END;
```


Check if all promotion notifications are stored correctly.
```
SELECT * from shoe_promotions_<yourname>;
```

All data products are created now and events are in motion. Visit the brand new data portal to get all information you need and query the data. Give it a try!

![image](terraform/img/dataportal.png)

## End of Lab2.

# If you don't need your infrastructure anymore, do not forget to delete the resources!
