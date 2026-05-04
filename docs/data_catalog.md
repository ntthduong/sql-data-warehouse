# Data Catalog for Gold Layer

## Overview

The Gold layer is the business-ready layer of the data warehouse.  
It exposes cleaned and integrated data from the Silver layer as analytical views for reporting and exploration.

The Gold layer contains:

- `gold.dim_customers`
- `gold.dim_products`
- `gold.fact_sales`

These views follow a star schema design, where the sales fact view connects to customer and product dimension views through surrogate keys.

---

## 1. `gold.dim_customers`

### Purpose

Stores customer profile information enriched with demographic and geographic attributes.

This dimension is used to analyze sales by customer, country, marital status, gender, and customer creation date.

### Columns

| Column Name | Data Type | Description |
|---|---:|---|
| `customer_key` | `INT` | Surrogate key generated for each customer record in the Gold layer. |
| `customer_id` | `INT` | Original customer ID from the CRM source system. |
| `customer_number` | `NVARCHAR(50)` | Customer business identifier used for joining CRM and ERP customer data. |
| `first_name` | `NVARCHAR(50)` | Customer first name from the CRM source. |
| `last_name` | `NVARCHAR(50)` | Customer last name from the CRM source. |
| `country` | `NVARCHAR(50)` | Customer country information from the ERP location source. |
| `marital_status` | `NVARCHAR(50)` | Standardized marital status, such as `Single`, `Married`, or `n/a`. |
| `gender` | `NVARCHAR(50)` | Standardized customer gender. CRM gender is prioritized, with ERP gender used as a fallback. |
| `birthdate` | `DATE` | Customer birth date from the ERP customer source. |
| `create_date` | `DATE` | Date when the customer record was created in the CRM source system. |

### Source Tables

| Source Table | Usage |
|---|---|
| `silver.crm_cust_info` | Main customer profile source |
| `silver.erp_cust_az12` | Customer birthdate and fallback gender |
| `silver.erp_loc_a101` | Customer country information |

### Notes

- `customer_key` is generated using `ROW_NUMBER()` in the Gold view.
- CRM is treated as the primary source for gender.
- ERP gender is only used when CRM gender is unavailable or marked as `n/a`.

---

## 2. `gold.dim_products`

### Purpose

Stores product information enriched with category and subcategory details.

This dimension is used to analyze sales by product, category, subcategory, product line, and maintenance status.

### Columns

| Column Name | Data Type | Description |
|---|---:|---|
| `product_key` | `INT` | Surrogate key generated for each product record in the Gold layer. |
| `product_id` | `INT` | Original product ID from the CRM product source. |
| `product_number` | `NVARCHAR(50)` | Product business identifier used to link sales transactions to products. |
| `product_name` | `NVARCHAR(50)` | Product name from the CRM product source. |
| `category_id` | `NVARCHAR(50)` | Product category identifier extracted from the CRM product key. |
| `category` | `NVARCHAR(50)` | High-level product category from the ERP product category source. |
| `subcategory` | `NVARCHAR(50)` | More detailed product classification from the ERP product category source. |
| `maintenance` | `NVARCHAR(50)` | Indicates whether the product category requires maintenance. |
| `cost` | `INT` | Product cost from the CRM product source. |
| `product_line` | `NVARCHAR(50)` | Standardized product line name. |
| `start_date` | `DATE` | Date when the product record became active. |

### Source Tables

| Source Table | Usage |
|---|---|
| `silver.crm_prd_info` | Main product information source |
| `silver.erp_px_cat_g1v2` | Product category, subcategory, and maintenance details |

### Notes

- `product_key` is generated using `ROW_NUMBER()` in the Gold view.
- Only current product records are included.
- Historical product records are excluded by filtering rows where `prd_end_dt IS NULL`.
- The column is named `maintenance` to match the Gold view definition.

---

## 3. `gold.fact_sales`

### Purpose

Stores sales transaction records linked to the customer and product dimensions.

This fact view is used to analyze sales amount, quantity, unit price, order dates, shipping dates, and due dates.

### Grain

Each row represents one sales order line item.

### Columns

| Column Name | Data Type | Description |
|---|---:|---|
| `order_number` | `NVARCHAR(50)` | Sales order number from the CRM sales source. |
| `product_key` | `INT` | Surrogate key linking the sales record to `gold.dim_products`. |
| `customer_key` | `INT` | Surrogate key linking the sales record to `gold.dim_customers`. |
| `order_date` | `DATE` | Date when the order was placed. |
| `shipping_date` | `DATE` | Date when the order was shipped. |
| `due_date` | `DATE` | Date when the order was due. |
| `sales_amount` | `INT` | Total sales amount for the order line. |
| `quantity` | `INT` | Number of units sold in the order line. |
| `price` | `INT` | Unit price of the product in the order line. |

### Source Tables and Views

| Source Object | Usage |
|---|---|
| `silver.crm_sales_details` | Main sales transaction source |
| `gold.dim_products` | Provides `product_key` for product relationship |
| `gold.dim_customers` | Provides `customer_key` for customer relationship |

### Notes

- `fact_sales` joins to `dim_products` using product number.
- `fact_sales` joins to `dim_customers` using customer ID.
- The fact view stores cleaned sales values from the Silver layer.

---

## Relationship Summary

| Fact View | Foreign Key | Dimension View | Dimension Key |
|---|---|---|---|
| `gold.fact_sales` | `customer_key` | `gold.dim_customers` | `customer_key` |
| `gold.fact_sales` | `product_key` | `gold.dim_products` | `product_key` |

---

## Recommended Quality Checks

The Gold layer should be validated with the following checks:

| Check | Expected Result |
|---|---|
| Duplicate `customer_key` in `gold.dim_customers` | No rows returned |
| Duplicate `product_key` in `gold.dim_products` | No rows returned |
| Missing customer relationship in `gold.fact_sales` | No rows returned |
| Missing product relationship in `gold.fact_sales` | No rows returned |

The related SQL checks are stored in:

```text
tests/quality_checks_gold.sql
