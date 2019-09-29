---
layout: post
title: SQL JOIN description
category: database
tags: [education, development, copypaste]
---

Источник: [sql-join.com](http://www.sql-join.com/sql-join-types)

---

# INNER JOIN

![](https://images.squarespace-cdn.com/content/v1/5732253c8a65e244fd589e4c/1464122775537-YVL7LO1L7DU54X1MC2CI/ke17ZwdGBToddI8pDm48kMjn7pTzw5xRQ4HUMBCurC5Zw-zPPgdn4jUwVcJE1ZvWMv8jMPmozsPbkt2JQVr8L3VwxMIOEK7mu3DMnwqv-Nsp2ryTI0HqTOaaUohrI8PIvqemgO4J3VrkuBnQHKRCXIkZ0MkTG3f7luW22zTUABU/image-asset.png?format=300w)

Select all records from Table A and Table B, where the join condition is met.

# LEFT (Outer) JOIN

![](https://images.squarespace-cdn.com/content/v1/5732253c8a65e244fd589e4c/1464122797709-C2CDMVSK7P4V0FNNX60B/ke17ZwdGBToddI8pDm48kMjn7pTzw5xRQ4HUMBCurC5Zw-zPPgdn4jUwVcJE1ZvWEV3Z0iVQKU6nVSfbxuXl2c1HrCktJw7NiLqI-m1RSK4p2ryTI0HqTOaaUohrI8PIO5TUUNB3eG_Kh3ocGD53-KZS67ndDu8zKC7HnauYqqk/image-asset.png?format=300w)

Select all records from Table A, along with records from Table B for which the join condition is met (if at all).

# RIGHT JOIN

![](https://images.squarespace-cdn.com/content/v1/5732253c8a65e244fd589e4c/1464122744888-MVIUN2P80PG0YE6H12WY/ke17ZwdGBToddI8pDm48kMjn7pTzw5xRQ4HUMBCurC5Zw-zPPgdn4jUwVcJE1ZvWlExFaJyQKE1IyFzXDMUmzc1HrCktJw7NiLqI-m1RSK4p2ryTI0HqTOaaUohrI8PI-FpwTc-ucFcXUDX7aq6Z4KQhQTkyXNMGg1Q_B1dqyTU/image-asset.png?format=300w)

Select all records from Table B, along with records from Table A for which the join condition is met (if at all).

# FULL JOIN

![](https://images.squarespace-cdn.com/content/v1/5732253c8a65e244fd589e4c/1464122981217-RIYH5VL2MF1XWTU2DKVQ/ke17ZwdGBToddI8pDm48kMjn7pTzw5xRQ4HUMBCurC5Zw-zPPgdn4jUwVcJE1ZvWEV3Z0iVQKU6nVSfbxuXl2c1HrCktJw7NiLqI-m1RSK4p2ryTI0HqTOaaUohrI8PIO5TUUNB3eG_Kh3ocGD53-KZS67ndDu8zKC7HnauYqqk/image-asset.png?format=300w)

Select all records from Table A and Table B, regardless of whether the join condition is met or not.

# Example

Let's use the tables we introduced in the “What is a SQL join?” section to show examples of these joins in action. The relationship between the two tables is specified by the customer_id key, which is the "primary key" in customers table and a "foreign key" in the orders table:

**Table 1**

| customer_id | first_name | last_name | email | address | city | state | zipcode |
|-------------|------------|------------|---------------------|---------------------------|-----------------|-------|---------|
| 1 | George | Washington | gwashington@usa.gov | 3200 Mt Vernon Hwy | Mount Vernon | VA | 22121 |
| 2 | John | Adams | jadams@usa.gov | 1250 Hancock St | Quincy | MA | 02169 |
| 3 | Thomas | Jefferson | tjefferson@usa.gov | 931 Thomas Jefferson Pkwy | Charlottesville | VA | 22902 |
| 4 | James | Madison | jmadison@usa.gov | 11350 Constitution Hwy | Orange | VA | 22960 |
| 5 | James | Monroe | jmonroe@usa.gov | 2050 James Monroe Parkway | Charlottesville | VA | 22902 |

**Table 2**

| order_id | order_date | amount | customer_id |
|----------|------------|---------|-------------|
| 1 | 07/04/1776 | $234.56 | 1 |
| 2 | 03/14/1760 | $78.50 | 3 |
| 3 | 05/23/1784 | $124.00 | 2 |
| 4 | 09/03/1790 | $65.50 | 3 |
| 5 | 07/21/1795 | $25.50 | 10 |
| 6 | 11/27/1787 | $14.40 | 9 |

Note that (1) not every customer in our customers table has placed an order and (2) there are a few orders for which no customer record exists in our customers table.

# Inner Join

```sql

select first_name, last_name, order_date, order_amount
from customers c
inner join orders o
on c.customer_id = o.customer_id

```

| first_name | last_name | order_date | order_amount |
|------------|------------|------------|--------------|
| George | Washington | 07/4/1776 | $234.56 |
| John | Adams | 05/23/1784 | $124.00 |
| Thomas | Jefferson | 03/14/1760 | $78.50 |
| Thomas | Jefferson | 09/03/1790 | $65.50 |

# Left join

```sql

select first_name, last_name, order_date, order_amount
from customers c
left join orders o
on c.customer_id = o.customer_id

```

| first_name | last_name | order_date | order_amount |
|------------|------------|------------|--------------|
| George | Washington | 07/04/1776 | $234.56 |
| John | Adams | 05/23/1784 | $124.00 |
| Thomas | Jefferson | 03/14/1760 | $78.50 |
| Thomas | Jefferson | 09/03/1790 | $65.50 |
| James | Madison | NULL | NULL |
| James | Monroe | NULL | NULL |


# Right join

```sql

select first_name, last_name, order_date, order_amount
from customers c
right join orders o
on c.customer_id = o.customer_id

```

| first_name | last_name | order_date | order_amount |
|------------|------------|------------|--------------|
| George | Washington | 07/04/1776 | $234.56 |
| Thomas | Jefferson | 03/14/1760 | $78.50 |
| John | Adams | 05/23/1784 | $124.00 |
| Thomas | Jefferson | 09/03/1790 | $65.50 |
| NULL | NULL | 07/21/1795 | $25.50 |
| NULL | NULL | 11/27/1787 | $14.40 |

# Full join

```sql

select first_name, last_name, order_date, order_amount
from customers c
full join orders o
on c.customer_id = o.customer_id

```

| first_name | last_name | order_date | order_amount |
|------------|------------|------------|--------------|
| George | Washington | 07/04/1776 | $234.56 |
| Thomas | Jefferson | 03/14/1760 | $78.50 |
| John | Adams | 05/23/1784 | $124.00 |
| Thomas | Jefferson | 09/03/1790 | $65.50 |
| NULL | NULL | 07/21/1795 | $25.50 |
| NULL | NULL | 11/27/1787 | $14.40 |
| James | Madison | NULL | NULL |
| James | Monroe | NULL | NULL |