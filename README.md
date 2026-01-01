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


## 🧱 Database Schema

### Entity Relationship Overview
ERD image here

- Category
- Product
- Customer
- Orders
- Order Details


## 📋 Table Schemas

### 1️⃣ Category
| Column | Type | Description |
|------|-----|-------------|
| category_id | INT (PK) | Unique category identifier |
| category_name | VARCHAR | Category name |

---

### 2️⃣ Product
| Column | Type | Description |
|------|-----|-------------|
| product_id | BIGINT (PK) | Product identifier |
| category_id | INT (FK) | References `category(category_id)` |
| name | VARCHAR | Product name |
| description | TEXT | Product description |
| price | NUMERIC(12,2) | Product price |
| stock_quantity | INT | Available stock |

---

### 3️⃣ Customer
| Column | Type | Description |
|------|-----|-------------|
| customer_id | BIGINT (PK) | Customer identifier |
| first_name | VARCHAR | First name |
| last_name | VARCHAR | Last name |
| email | VARCHAR | Email address |
| password | VARCHAR | Hashed password |

---

### 4️⃣ Orders
| Column | Type | Description |
|------|-----|-------------|
| order_id | BIGINT (PK) | Order identifier |
| customer_id | BIGINT (FK) | References `customer(customer_id)` |
| order_date | DATE | Order date |
| total_amount | NUMERIC(12,2) | Total order value |

---

### 5️⃣ Order Details
| Column | Type | Description |
|------|-----|-------------|
| order_detail_id | BIGINT (PK) | Order detail identifier |
| order_id | BIGINT (FK) | References `orders(order_id)` |
| product_id | BIGINT (FK) | References `product(product_id)` |
| quantity | INT | Quantity ordered |
| unit_price | NUMERIC(12,2) | Price per unit |


## 🛠️ Database DDL

A SQL script to create the database and its tables:
```
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

