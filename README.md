# Instacart Market Basket Analysis

## Overview
This project analyzes the Instacart Market Basket dataset using PostgreSQL, AWS S3, and Snowflake. The goal was to explore, process, and analyze customer purchase patterns.

## Dataset
The dataset is sourced from the [Instacart Market Basket Analysis competition on Kaggle](https://www.kaggle.com/competitions/instacart-market-basket-analysis). It consists of several CSV files containing details about orders, products, aisles, and departments.

### Key Tables:
- **aisles.csv**: Contains aisle IDs and names.
- **departments.csv**: Contains department IDs and names.
- **products.csv**: Contains product details with aisle and department mapping.
- **orders.csv**: Contains order details including order time and user ID.
- **order_products.csv**: Maps products to orders, including reordering information.

## Project Phases
### Part 1: Local Processing with PostgreSQL
1. Loaded CSV files into a PostgreSQL database using Python.
2. Due to the large dataset (32M orders, 3.4M order-product records), loading took approximately 1.5 hours.
3. Explored data locally with SQL queries.

### Part 2: Cloud-Based Processing with AWS S3 & Snowflake
1. Uploaded raw CSV files to an AWS S3 bucket.
2. Created a Snowflake database and loaded the data using **COPY INTO** statements.
3. Created structured tables for optimized querying.
4. Built a star schema data warehouse:
   - **Dimension Tables**: Users, Products, Aisles, Departments, Orders
   - **Fact Table**: Order-Product relationships

## Data Loading in Snowflake
```sql
CREATE STAGE my_stage
URL = "s3://instacart-data-analysis/data/"
CREDENTIALS = (AWS_KEY_ID = 'XXXX' AWS_SECRET_KEY = 'XXXX');

CREATE OR REPLACE FILE FORMAT csv_file_format
TYPE = 'CSV'
FIELD_DELIMITER = ','
SKIP_HEADER = 1
FIELD_OPTIONALLY_ENCLOSED_BY = '"';

COPY INTO aisles FROM @my_stage/aisles.csv FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');
COPY INTO departments FROM @my_stage/departments.csv FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');
COPY INTO products FROM @my_stage/products.csv FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');
COPY INTO orders FROM @my_stage/orders.csv FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');
COPY INTO order_products FROM @my_stage/order_products.csv FILE_FORMAT = (FORMAT_NAME = 'csv_file_format');
```

## Data Warehouse Schema
```sql
CREATE OR REPLACE TABLE dim_users AS (
  SELECT user_id FROM orders
);

CREATE OR REPLACE TABLE dim_products AS (
  SELECT product_id, product_name FROM products
);

CREATE OR REPLACE TABLE dim_aisles AS (
  SELECT aisle_id, aisle FROM aisles
);

CREATE OR REPLACE TABLE dim_departments AS (
  SELECT department_id, department FROM departments
);

CREATE OR REPLACE TABLE dim_orders AS (
  SELECT order_id, order_number, order_dow, order_hour_of_day, days_since_prior_order FROM orders
);

CREATE TABLE fact_order_products AS (
  SELECT op.order_id, op.product_id, o.user_id, p.department_id, p.aisle_id, op.add_to_cart_order, op.reordered
  FROM order_products op
  JOIN orders o ON op.order_id = o.order_id
  JOIN products p ON op.product_id = p.product_id
);
```

## Data Analysis Queries
### 1. Total Number of Products Ordered Per Department
```sql
SELECT d.department, COUNT(*) AS total_products_ordered
FROM fact_order_products fop
JOIN dim_departments d ON fop.department_id = d.department_id
GROUP BY d.department;
```

### 2. Top 5 Aisles with Highest Reordered Products
```sql
SELECT a.aisle, COUNT(*) AS total_reordered
FROM fact_order_products fop
JOIN dim_aisles a ON fop.aisle_id = a.aisle_id
WHERE fop.reordered = TRUE
GROUP BY a.aisle
ORDER BY total_reordered DESC
LIMIT 5;
```

### 3. Average Products Added to Cart Per Order by Day of the Week
```sql
SELECT o.order_dow, AVG(fop.add_to_cart_order) AS avg_products_per_order
FROM fact_order_products fop
JOIN dim_orders o ON fop.order_id = o.order_id
GROUP BY o.order_dow;
```

### 4. Top 10 Users with Highest Unique Products Ordered
```sql
SELECT u.user_id, COUNT(DISTINCT fop.product_id) AS unique_products_ordered
FROM fact_order_products fop
JOIN dim_users u ON fop.user_id = u.user_id
GROUP BY u.user_id
ORDER BY unique_products_ordered DESC
LIMIT 10;
```

## Technologies Used
- **Python** (for local data loading into PostgreSQL)
- **PostgreSQL** (local database analysis)
- **AWS S3** (storage for raw data files)
- **Snowflake** (cloud-based data warehouse)
- **SQL** (data analysis queries)

## Conclusion
This project demonstrates the process of migrating a large dataset from a local PostgreSQL environment to a cloud-based Snowflake warehouse, optimizing data for analytical queries. The use of AWS S3 for staging ensures efficient data management and retrieval.

## Future Enhancements
- Automate data pipeline using Airflow.
- Optimize Snowflake queries with clustering and materialized views.

## Author
Anish Babu Gogineni

## License
This project is licensed under the MIT License.

