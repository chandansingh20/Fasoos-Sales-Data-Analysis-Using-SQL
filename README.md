# Fasoos-Sales-Data-Analysis-Using-SQL

drop table if exists driver;
CREATE TABLE driver(driver_id integer,reg_date date); 

INSERT INTO driver(driver_id,reg_date) 
 VALUES (1,'01-01-2021'),
(2,'01-03-2021'),
(3,'01-08-2021'),
(4,'01-15-2021');


drop table if exists ingredients;
CREATE TABLE ingredients(ingredients_id integer,ingredients_name varchar(60)); 

INSERT INTO ingredients(ingredients_id ,ingredients_name) 
 VALUES (1,'BBQ Chicken'),
(2,'Chilli Sauce'),
(3,'Chicken'),
(4,'Cheese'),
(5,'Kebab'),
(6,'Mushrooms'),
(7,'Onions'),
(8,'Egg'),
(9,'Peppers'),
(10,'schezwan sauce'),
(11,'Tomatoes'),
(12,'Tomato Sauce');

drop table if exists rolls;
CREATE TABLE rolls(roll_id integer,roll_name varchar(30)); 

INSERT INTO rolls(roll_id ,roll_name) 
 VALUES (1	,'Non Veg Roll'),
(2	,'Veg Roll');

drop table if exists rolls_recipes;
CREATE TABLE rolls_recipes(roll_id integer,ingredients varchar(24)); 

INSERT INTO rolls_recipes(roll_id ,ingredients) 
 VALUES (1,'1,2,3,4,5,6,8,10'),
(2,'4,6,7,9,11,12');

drop table if exists driver_order;
CREATE TABLE driver_order(order_id integer,driver_id integer,pickup_time timestamp,distance VARCHAR(7),duration VARCHAR(10),cancellation VARCHAR(23));
INSERT INTO driver_order(order_id,driver_id,pickup_time,distance,duration,cancellation) 
 VALUES(1,1,'01-01-2021 18:15:34','20km','32 minutes',''),
(2,1,'01-01-2021 19:10:54','20km','27 minutes',''),
(3,1,'01-03-2021 00:12:37','13.4km','20 mins','NaN'),
(4,2,'01-04-2021 13:53:03','23.4','40','NaN'),
(5,3,'01-08-2021 21:10:57','10','15','NaN'),
(6,3,null,null,null,'Cancellation'),
(7,2,'01-08-2021 21:30:45','25km','25mins',null),
(8,2,'01-10-2021 00:15:02','23.4 km','15 minute',null),
(9,2,null,null,null,'Customer Cancellation'),
(10,1,'01-11-2021 18:50:20','10km','10minutes',null);


drop table if exists customer_orders;
CREATE TABLE customer_orders(order_id integer,customer_id integer,roll_id integer,not_include_items VARCHAR(4),extra_items_included VARCHAR(4),order_date timestamp);
INSERT INTO customer_orders(order_id,customer_id,roll_id,not_include_items,extra_items_included,order_date)
values (1,101,1,'','','01-01-2021  18:05:02'),
(2,101,1,'','','01-01-2021 19:00:52'),
(3,102,1,'','','01-02-2021 23:51:23'),
(3,102,2,'','NaN','01-02-2021 23:51:23'),
(4,103,1,'4','','01-04-2021 13:23:46'),
(4,103,1,'4','','01-04-2021 13:23:46'),
(4,103,2,'4','','01-04-2021 13:23:46'),
(5,104,1,null,'1','01-08-2021 21:00:29'),
(6,101,2,null,null,'01-08-2021 21:03:13'),
(7,105,2,null,'1','01-08-2021 21:20:29'),
(8,102,1,null,null,'01-09-2021 23:54:33'),
(9,103,1,'4','1,5','01-10-2021 11:22:59'),
(10,104,1,null,null,'01-11-2021 18:34:49'),
(10,104,1,'2,6','1,4','01-11-2021 18:34:49');

select * from customer_orders;
select * from driver_order;
select * from ingredients;
select * from driver;
select * from rolls;
select * from rolls_recipes;

--A. roll metrics

--1.how many rolls were ordered?

SELECT
	count(roll_id) as total_orders
from customer_orders

--2.how many unique customers orders were made?

SELECT
		count(DISTINCT customer_id)
FROM customer_orders

--3.how many successful orders were delievered by each driver?

SELECT
	dr.driver_id,
	count(d.order_id) as succesful_deliveries
FROM driver_order as d
join driver as dr
on d.driver_id = dr.driver_id
WHERE
	(d.cancellation is null or d.cancellation not in ('Cancellation','Customer Cancellation'))
group by 1

--4.how many of each type of roll were delivered?


SELECT
	roll_name,
	count(co.order_id) as no_of_orders
FROM customer_orders as co
JOIN rolls as r
on co.roll_id = r.roll_id
JOIN driver_order as d
on co.order_id = d.order_id
where
	(d.cancellation is null or d.cancellation not in ('Cancellation','Customer Cancellation'))
group by 1

--5.how many veg and non-veg rolls were ordered by each customer

SELECT
	co.customer_id,
	sum(case when r.roll_name = 'Veg Roll' then 1 else 0 end) as veg_rolls_ordered,
	sum(case when r.roll_name = 'Non Veg Roll' then 1 else 0 end) as non_veg_rolls_ordered
FROM customer_orders as co
join rolls as r
on co.roll_id = r.roll_id
GROUP by 1

--6.what was maximum number of rolls delievered in single order?

with orders_roll_count as
(
SELECT
	co.order_id,
	count(co.roll_id) as order_count
FROM customer_orders as co
join driver_order as d
on co.order_id = d.order_id
WHERE
	(d.cancellation is null or d.cancellation not in ('Cancellation','Customer Cancellation'))
group by 1
)
SELECT
	order_id,
	order_count as max_rolls_in_single_order
FROM orders_roll_count
WHERE
	order_count = (select max(order_count) from orders_roll_count)

--7.for each customer, how many delivered rolls had at least 1 change and how many had no change?

with succesful_orders as
(
select
	co.order_id,
	co.customer_id,
	co.not_include_items,
	co.extra_items_included
from customer_orders as co
join driver_order as d
on co.order_id = d.order_id
where
	(d.cancellation is null or d.cancellation not in ('Cancellation','Customer Cancellation'))
)
select
	customer_id,
	count(case when (not_include_items IS NOT NULL AND not_include_items != '') OR 
                        (extra_items_included IS NOT NULL AND extra_items_included != '') 
                THEN 1 END) AS rolls_with_changes,
	COUNT(CASE WHEN (not_include_items IS NULL OR not_include_items = '') AND 
                        (extra_items_included IS NULL OR extra_items_included = '') 
                THEN 1 END) AS rolls_with_no_changes
from succesful_orders
group by 1

--8.how many rolls were delivered that had both exclusions and extras

with succesful_orders as
(
SELECT
	co.order_id,
	co.roll_id,
	co.not_include_items,
	co.extra_items_included
from customer_orders as co
join driver_order as d
on d.order_id = co.order_id
where
	(d.cancellation is null or d.cancellation not in ('Cancellation','Customer Cancellation'))
)
select
	order_id,
	count(*) as rolls_with_both_exclusions_and_extras
FROM 
    succesful_orders
where
	(not_include_items is not null and not_include_items != '')
	and
	(extra_items_included is not null and extra_items_included != '')
group by 1

--9.what was total order ordered for each hour of the DAY?

SELECT
	extract(hour from order_date) as hours,
	count(roll_id) as total_orders
from customer_orders
group by 1
order by 1

SELECT
	concat(extract(hour from order_date),'-',(extract(hour from order_date)+1)) as hours_group,
	count(roll_id) as total_orders
from customer_orders
group by 1
order by 1

--10.what was the number of orders for each day of the week

SELECT
	extract(dow from order_date) as dow,
	count(distinct order_id) as total_orders
from customer_orders
group by 1

SELECT
	to_char(order_date,'day') as dow,
	count(distinct order_id) as total_orders
from customer_orders
group by 1

--B. Drivers and Customer experience

--1.what was the average time in minutes it took for each driver to arrive at the fasoos hq to pickup the order?

with cte as
(
SELECT
	d.driver_id,
	co.order_id,
	abs(extract(minute from(co.order_date-d.pickup_time))) as diff,
	row_number() over(partition by co.order_id order by abs(extract(minute from(co.order_date-d.pickup_time)))) as rank
from customer_orders as co
join driver_order as d
on co.order_id = d.order_id
WHERE
	d.pickup_time is not null
)
select
	driver_id,
	round(sum(diff)/count(order_id)) as avg_time
from cte
WHERE rank = 1
group by 1

--2.is there any realationship between no of rolls and how long the orders take to prepare

WITH cte as
(
SELECT
	co.order_id,
	co.roll_id,
	abs(extract(minute from(co.order_date-d.pickup_time))) as diff
FROM customer_orders as co
JOIN driver_order as d
on co.order_id = d.order_id
WHERE
	d.pickup_time is not null
)
SELECT
	order_id,
	count(roll_id) as no_of_rolls,
	round(sum(diff)/count(roll_id)) as time_taken
FROM cte
group by 1
order by 1

--3.what was the avg distance travelled for each customer?

WITH distance_travelled as
(
SELECT
	co.customer_id,
	d.order_id,
	extract(minute from(co.order_date-d.pickup_time)) as diff,
	cast(regexp_replace(d.distance,'[^0-9.]', '','g') as numeric) as distance,
	row_number() over(partition by d.order_id order by extract(minute from(co.order_date-d.pickup_time))) as ranks
FROM customer_orders as co
join driver_order as d
on co.order_id = d.order_id
WHERE
	d.distance is not null
)
SELECT
	customer_id,
	round(avg(distance)) as avg_distance
FROM distance_travelled
where ranks = 1
group by 1
order by 1

--4.what was the difference between the longest and shortest delivery time for all orders?

SELECT 
    MAX(CAST(REGEXP_REPLACE(d.duration, '[^0-9]', '', 'g') AS INT)) - 
    MIN(CAST(REGEXP_REPLACE(d.duration, '[^0-9]', '', 'g') AS INT)) AS duration_difference_minutes
FROM 
    driver_order d
WHERE 
    d.duration IS NOT NULL
    AND REGEXP_REPLACE(d.duration, '[^0-9]', '', 'g') != '';

--5.what was the avg speed for each driver for each delivery and do you notice any trend for these values? 

SELECT 
    order_id,
    -- Extract numeric part of the distance and convert to numeric (e.g., 20km -> 20)
    CAST(REGEXP_REPLACE(distance, '[^0-9.]', '', 'g') AS numeric) AS distance_numeric,
    -- Extract numeric part of the duration and convert to numeric (e.g., 32 minutes -> 32)
    CAST(REGEXP_REPLACE(duration, '[^0-9.]', '', 'g') AS numeric) AS duration_numeric,
    -- Calculate the average speed as distance divided by duration (in km per minute)
    CAST(REGEXP_REPLACE(distance, '[^0-9.]', '', 'g') AS numeric) / 
    CAST(REGEXP_REPLACE(duration, '[^0-9.]', '', 'g') AS numeric) AS avg_speed_per_order
FROM 
    driver_order
WHERE 
    distance IS NOT NULL
    AND duration IS NOT NULL
ORDER BY 
    order_id;

--6.what is the succesful delivery percentage for each driver

SELECT 
    driver_id,
    -- Calculate the total number of orders for each driver
    COUNT(order_id) AS total_deliveries,
    -- Count the number of successful deliveries (no cancellations)
    COUNT(CASE WHEN cancellation IS NULL OR cancellation not in('Cancellation','Customer Cancellation') THEN 1 END) AS successful_deliveries,
    -- Calculate the success percentage
    (COUNT(CASE WHEN cancellation IS NULL OR cancellation not in('Cancellation','Customer Cancellation') THEN 1 END) * 100.0) / 
    COUNT(order_id) AS successful_delivery_percentage
FROM driver_order
GROUP BY driver_id
ORDER BY driver_id;

