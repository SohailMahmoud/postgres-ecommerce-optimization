# PostgreSQL E-Commerce Database Optimization Lab

## Overview
This repository contains a **PostgreSQL e-commerce database** designed to simulate a **realistic, large-scale workload** (millions of rows).  
The project focuses on **schema design, bulk data generation, query analysis, and SQL query optimization**.

The database is intentionally populated with large datasets to demonstrate:
- Query performance issues
- Indexing strategies
- Join optimization
- Aggregate optimization
- Real-world SQL tuning techniques


## Database Schema

### Entity Relationship Overview
ERD image here

- Category
- Product
- Customer
- Orders
- Order Details


## Table Schemas

### 1. Category
| Column | Type | Description |
|------|-----|-------------|
| category_id | INT (PK) | Unique category identifier |
| category_name | VARCHAR | Category name |

---

### 2. Product
| Column | Type | Description |
|------|-----|-------------|
| product_id | INT (PK) | Product identifier |
| category_id | INT (FK) | References `category(category_id)` |
| name | VARCHAR | Product name |
| description | TEXT | Product description |
| price | NUMERIC(12,2) | Product price |
| stock_quantity | INT | Available stock |

---

### 3. Customer
| Column | Type | Description |
|------|-----|-------------|
| customer_id | INT (PK) | Customer identifier |
| first_name | VARCHAR | First name |
| last_name | VARCHAR | Last name |
| email | VARCHAR | Email address |
| password | VARCHAR | Hashed password |

---

### 4. Orders
| Column | Type | Description |
|------|-----|-------------|
| order_id | INT (PK) | Order identifier |
| customer_id | INT (FK) | References `customer(customer_id)` |
| order_date | DATE | Order date |
| total_amount | NUMERIC(12,2) | Total order value |

---

### 5. Order Details
| Column | Type | Description |
|------|-----|-------------|
| order_detail_id | INT (PK) | Order detail identifier |
| order_id | INT (FK) | References `orders(order_id)` |
| product_id | INT (FK) | References `product(product_id)` |
| quantity | INT | Quantity ordered |
| unit_price | NUMERIC(12,2) | Price per unit |


## Database DDL

A SQL script to create the database and its tables:
```
CREATE DATABASE ecommerce;

CREATE TABLE category (
    category_id SERIAL PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE customer (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name  VARCHAR(100) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    password   TEXT NOT NULL
);

CREATE TABLE product (
    product_id SERIAL PRIMARY KEY,
    category_id INTEGER NOT NULL,
    name VARCHAR(150) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock_quantity INTEGER NOT NULL CHECK (stock_quantity >= 0),

    CONSTRAINT fk_product_category
        FOREIGN KEY (category_id)
        REFERENCES category (category_id)
        ON DELETE RESTRICT
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total_amount NUMERIC(12, 2) NOT NULL CHECK (total_amount >= 0),

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id)
        REFERENCES customer (customer_id)
        ON DELETE RESTRICT
);

CREATE TABLE order_details (
    order_detail_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),

    CONSTRAINT fk_order_details_order
        FOREIGN KEY (order_id)
        REFERENCES orders (order_id)
        ON DELETE CASCADE,

    CONSTRAINT fk_order_details_product
        FOREIGN KEY (product_id)
        REFERENCES product (product_id)
        ON DELETE RESTRICT
);

```

A SQL script to populate the tables with the required data:
```
CREATE OR REPLACE FUNCTION populate_categories()
RETURNS void AS $$
BEGIN
    INSERT INTO category (category_name)
    SELECT
        'Category_' || gs
    FROM generate_series(1, 100000) AS gs;
END;
$$ LANGUAGE plpgsql;

SELECT populate_categories();

CREATE OR REPLACE FUNCTION populate_products()
RETURNS void AS $$
BEGIN
    INSERT INTO product (category_id, name, description, price, stock_quantity)
    SELECT
        ((gs - 1) % 100000) + 1,
        'Product_' || gs,
        'Description ' || gs,
        (gs + 1)::numeric(12,2),
        gs + 1
    FROM generate_series(1, 5000000) AS gs;
END;
$$ LANGUAGE plpgsql;

SELECT populate_products();

CREATE OR REPLACE FUNCTION populate_customers()
RETURNS void AS $$
BEGIN
    INSERT INTO customer (first_name, last_name, email, password)
    SELECT
        'First_' || gs,
        'Last_' || gs,
        'user' || gs || '@mail.com',
        'hashed_password_' || gs
    FROM generate_series(1, 1000000) AS gs;
END;
$$ LANGUAGE plpgsql;

SELECT populate_customers();

CREATE OR REPLACE FUNCTION populate_orders()
RETURNS void AS $$
BEGIN
    INSERT INTO orders (customer_id, order_date, total_amount)
    SELECT
        ((gs - 1) % 1000000) + 1,
        DATE '2024-01-01',
        (gs + 10)::numeric(12,2)
    FROM generate_series(1, 20000000) AS gs;
END;
$$ LANGUAGE plpgsql;

SELECT populate_orders();

CREATE OR REPLACE FUNCTION populate_order_details()
RETURNS void AS $$
BEGIN
    INSERT INTO order_details (order_id, product_id, quantity, unit_price)
    SELECT
        ((gs - 1) % 20000000) + 1,
        ((gs - 1) % 5000000) + 1,
        (gs % 5) + 1,
        ((gs % 100) + 1)::numeric(12,2)
    FROM generate_series(1, 50000000) AS gs;
END;
$$ LANGUAGE plpgsql;

SELECT populate_order_details();
```

## Query Optimization Case Studies:

### 1. Total Number of Products per Category:
| Simple Query | Execution time before optimization | Optimization Technique | Rewrite Query | Execution time after optimization |
| ---- | ---- | ----- | ----- | ----- |
| ```select category_id, count(*) as total_products from product group by category_id;``` | 832.168 ms | Adding non-clustered index on the category_id column | ```CREATE INDEX idx_product_category_id ON product(category_id); select category_id, count(*) as total_products from product group by category_id;``` | 474.947 ms |

By adding the index on the category_id column here, we can avoid:
- Full table scan
- Hash aggregation
- Sorting

As a result, we improved performance by approximately 43%.

### 2. Top Customers by Total Spending:
| Simple Query | Execution time before optimization | Optimization Technique | Rewrite Query | Execution time after optimization |
| ---- | ---- | ----- | ----- | ----- |
| ```select customer_id, sum(total_amount) from orders group by customer_id order by sum(total_amount)desc limit 10;``` | 28389.099 ms | Creating a materilzed view using the same query | ```create materialized view mv_cust as select customer_id, sum(total_amount) from orders group by customer_id order by sum(total_amount)desc limit 10;``` | 0.024 ms |

The query is slow because PostgreSQL must aggregate 20 million rows into 5 million customer groups, causing hash aggregation to spill over 700 MB to disk, which dominates execution time. 

Possible solutions:
- If we attempt to add an index on the customer_id column, the execution time increases to 41192.940 ms. Although an index enabled streaming aggregation via GroupAggregate, the overall execution time increased because index scans are significantly slower than sequential scans when reading the entire table.
- We can cluster the table based on the same index created above in point number 1, and the execution time decreased slightly to 26969.724 ms because we improved the table read instead of random I/O to seq. I/O.
- We may create a materialized view so that we can reduce the execution time significantly by 99%, but it comes with its cons:
    - Consumes more storage
    - Must be refreshed periodically to keep the stored data up-to-date and in sync

### 3. Most Recent Orders with Customer Information (Top 1000):
| Simple Query | Execution time before optimization | Optimization Technique | Rewrite Query | Execution time after optimization |
| ---- | ---- | ----- | ----- | ----- |
| ```select category_id, count(*) as total_products from product group by category_id;``` | 832.168 ms | Adding non-clustered index on the category_id column | ```CREATE INDEX idx_product_category_id ON product(category_id); select category_id, count(*) as total_products from product group by category_id;``` | 474.947 ms |

### 4. Products with Low Stock Quantity (< 10):
| Simple Query | Execution time before optimization | Optimization Technique | Rewrite Query | Execution time after optimization |
| ---- | ---- | ----- | ----- | ----- |
| ```select * from product where stock_quantity < 10;``` | 281.313 ms | Adding non-clustered index on the stock_quantity column | ```CREATE INDEX idx_product_stock_quantity ON product(stock_quantity); select * from product where stock_quantity < 10;``` | 0.067 ms |

### 5. Revenue Generated per Product Category:
| Simple Query | Execution time before optimization | Optimization Technique | Rewrite Query | Execution time after optimization |
| ---- | ---- | ----- | ----- | ----- |
| ```SELECT product.category_id, SUM(quantity * unit_price) AS revenue FROM order_details JOIN product ON product.product_id = order_details.product_id GROUP BY product.category_id;``` | 14476.996 ms | Creating a pre-aggregated table product_revenue | ```SELECT p.category_id, SUM(pr.total_revenue) FROM product_revenue pr JOIN product p ON p.product_id = pr.product_id GROUP BY p.category_id;``` | 4.831 ms |

PostgreSQL scanned all order_details in parallel, cached product lookups using Memoize, aggregated revenue per category efficiently in memory, and merged results with minimal overhead. 

Possible optimization approaches: 
- We can create a materialized view as we did on query number 2.
- We can use a pre-aggregated table to reduce the number of rows scanned. We call it product_revenue:
```
CREATE TABLE product_revenue (
    product_id     BIGINT PRIMARY KEY,
    total_revenue  NUMERIC(18,2) NOT NULL
);
```

After that, we can populate it with the data using the following query:
```
INSERT INTO product_revenue (product_id, total_revenue)
SELECT
    product_id,
    SUM(quantity * unit_price)
FROM order_details
GROUP BY product_id;
```

Then the final query will be:
```
SELECT
    p.category_id,
    SUM(pr.total_revenue)
FROM product_revenue pr
JOIN product p ON p.product_id = pr.product_id
GROUP BY p.category_id;
```

The execution time is down to only 4.831 ms. As always, there is a trade-off here, which is that we must keep the product_revenue up-to-date by periodically - for example - running a batch refresh.
