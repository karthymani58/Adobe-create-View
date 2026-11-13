# Sales & Customer Order Line-Item Report (PostgreSQL / Magento 2)

## Overview
This SQL query extracts line-item level sales, customer acquisition (`New` vs. `Existing`), payment classification (`COD` vs. `Prepaid`), product categorization, and shipping/billing demographic data from a Magento 2 e-commerce database (PostgreSQL dialect).

---

## SQL Query

```sql
WITH fiscal_year_start AS (
    SELECT 
        MAKE_DATE(
            CASE 
                WHEN EXTRACT(MONTH FROM CURRENT_DATE) >= 4 THEN EXTRACT(YEAR FROM CURRENT_DATE)::INT
                ELSE EXTRACT(YEAR FROM CURRENT_DATE)::INT - 1
            END, 4, 1
        ) AS fy_start,
        MAKE_DATE(
            CASE 
                WHEN EXTRACT(MONTH FROM CURRENT_DATE) >= 4 THEN EXTRACT(YEAR FROM CURRENT_DATE)::INT + 1
                ELSE EXTRACT(YEAR FROM CURRENT_DATE)::INT
            END, 4, 1
        ) AS fy_end
),
valid_orders AS (
    SELECT 
        so.entity_id,
        so.increment_id,
        so.created_at,
        LOWER(TRIM(so.customer_email)) AS customer_email,
        so.customer_firstname,
        so.customer_lastname,
        so.coupon_code,
        so.grand_total AS order_grand_total
    FROM sales_order so
    WHERE so.status IN (
        'complete', 'processing', 'delivered', 'Packed', 'shipped',
        'in-transit', 'dispatching_process', 'partial_delivered', 'out-for-delivery'
    )
    AND so.store_id = 1
),
first_purchase AS (
    SELECT 
        customer_email,
        MIN(created_at) AS first_ever_order_date
    FROM valid_orders
    GROUP BY customer_email
)
SELECT
    vo.increment_id AS order_id,
    vo.created_at AS order_date,
    COALESCE(sop.method, 'Unknown') AS payment_method,
    CASE 
        WHEN sop.method IN ('cashondelivery', 'srcheckout_cod') THEN 'COD'
        ELSE 'Prepaid'
    END AS payment_type,
    vo.coupon_code,
    CASE 
        WHEN fp.first_ever_order_date >= fys.fy_start 
             AND fp.first_ever_order_date < fys.fy_end THEN 'New'
        ELSE 'Existing'
    END AS customer_type,
    fp.first_ever_order_date AS ltv_date,
    soi.sku,
    soi.name AS item_name,
    COALESCE(ctm.l1, 'Uncategorized') AS category,
    vo.customer_email AS email,
    COALESCE(ship.telephone, bill.telephone) AS mobile,
    COALESCE(ship.firstname, bill.firstname, vo.customer_firstname) AS firstname,
    COALESCE(ship.lastname, bill.lastname, vo.customer_lastname) AS lastname,
    COALESCE(ship.city, bill.city) AS city,
    COALESCE(ship.region, bill.region) AS state,
    COALESCE(ship.postcode, bill.postcode) AS pincode,
    soi.qty_ordered AS order_qty,
    soi.base_price AS base_price,
    (soi.qty_ordered * soi.base_price) AS item_revenue
FROM valid_orders vo
INNER JOIN sales_order_item soi ON vo.entity_id = soi.order_id
INNER JOIN first_purchase fp ON vo.customer_email = fp.customer_email
CROSS JOIN fiscal_year_start fys
LEFT JOIN sales_order_payment sop ON vo.entity_id = sop.parent_id
LEFT JOIN category_table_mapping_india ctm ON soi.sku = ctm.sku
LEFT JOIN sales_order_address ship ON vo.entity_id = ship.parent_id AND ship.address_type = 'shipping'
LEFT JOIN sales_order_address bill ON vo.entity_id = bill.parent_id AND bill.address_type = 'billing'
WHERE soi.parent_item_id IS NULL;
```

---

## Business Logic & Rules

### 1. Fiscal Year Calculation (`fiscal_year_start`)
* Follows the **Indian Financial Year cycle (April 1 – March 31)**.
* Dynamically determines `fy_start` and `fy_end` based on `CURRENT_DATE`:
  * If current month is **April (4) or later**, the fiscal year runs from April 1 of the current year to April 1 of the following year.
  * If current month is **January–March (1–3)**, the fiscal year runs from April 1 of the previous year to April 1 of the current year.

### 2. Valid Orders Filter (`valid_orders`)
* Filters `sales_order` to **Store ID `1`** (`so.store_id = 1`).
* Includes only fulfilled, active, or in-progress order statuses:
  * `complete`, `processing`, `delivered`, `Packed`, `shipped`, `in-transit`, `dispatching_process`, `partial_delivered`, `out-for-delivery`
* Excludes canceled, closed, refunded, or pending payment orders.
* Normalizes `customer_email` using `LOWER(TRIM(...))` for accurate customer grouping.

### 3. Customer Acquisition & LTV (`first_purchase`)
* **`ltv_date` (`first_ever_order_date`)**: The timestamp of the customer's earliest valid order across all historical data.
* **`customer_type`**:
  * **`New`**: Customer's first-ever valid order occurred within the current fiscal year (`fy_start <= first_ever_order_date < fy_end`).
  * **`Existing`**: Customer's first-ever valid order occurred prior to the current fiscal year.

### 4. Payment Classification
* **`payment_type`**:
  * **`COD`**: Payment method is `cashondelivery` or `srcheckout_cod`.
  * **`Prepaid`**: All other payment methods (including `Unknown` if payment record is missing).

### 5. Line-Item Deduplication
* Filters `WHERE soi.parent_item_id IS NULL` on `sales_order_item` to prevent double-counting configurable or bundled products (retaining only the top-level parent line item).

### 6. Address Fallback Logic
* Customer contact and location fields (`mobile`, `firstname`, `lastname`, `city`, `state`, `pincode`) prioritize the **Shipping Address** (`address_type = 'shipping'`), falling back to the **Billing Address** (`address_type = 'billing'`), and finally to the order-level customer name fields.

---

## Source Tables Used

| Table | Alias | Join Type | Purpose |
| :--- | :--- | :--- | :--- |
| `sales_order` | `so` | Base | Core order headers, timestamps, status, and customer email |
| `sales_order_item` | `soi` | `INNER JOIN` | Line-item SKUs, product names, quantities, and unit prices |
| `sales_order_payment` | `sop` | `LEFT JOIN` | Payment gateway method codes |
| `category_table_mapping_india` | `ctm` | `LEFT JOIN` | Maps product SKUs to L1 product categories |
| `sales_order_address` | `ship` | `LEFT JOIN` | Shipping address contact and location details |
| `sales_order_address` | `bill` | `LEFT JOIN` | Fallback billing address contact and location details |

---

## Output Schema

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `order_id` | `VARCHAR` | Human-readable Magento order increment ID (`increment_id`) |
| `order_date` | `TIMESTAMP` | Order creation timestamp (`created_at`) |
| `payment_method` | `VARCHAR` | Raw payment method code (defaults to `'Unknown'`) |
| `payment_type` | `VARCHAR` | Classified as `'COD'` or `'Prepaid'` |
| `coupon_code` | `VARCHAR` | Applied promo/discount code (if any) |
| `customer_type` | `VARCHAR` | `'New'` (acquired in current FY) or `'Existing'` |
| `ltv_date` | `TIMESTAMP` | Timestamp of the customer's first valid order |
| `sku` | `VARCHAR` | Product SKU |
| `item_name` | `VARCHAR` | Product display name |
| `category` | `VARCHAR` | L1 category from mapping table (defaults to `'Uncategorized'`) |
| `email` | `VARCHAR` | Normalized customer email address |
| `mobile` | `VARCHAR` | Customer phone number (Shipping -> Billing) |
| `firstname` | `VARCHAR` | Customer first name (Shipping -> Billing -> Order) |
| `lastname` | `VARCHAR` | Customer last name (Shipping -> Billing -> Order) |
| `city` | `VARCHAR` | Delivery/billing city |
| `state` | `VARCHAR` | Delivery/billing state/region |
| `pincode` | `VARCHAR` | Delivery/billing postal code |
| `order_qty` | `NUMERIC` | Quantity ordered for the line item |
| `base_price` | `NUMERIC` | Base unit price of the item |
| `item_revenue` | `NUMERIC` | Gross line-item revenue (`qty_ordered * base_price`) |

---

## Notes & Caveats
* **Revenue Calculation:** `item_revenue` represents gross item revenue (`qty_ordered * base_price`) before item-level discounts, tax, or shipping charges.
* **Bundle/Configurable Items:** Because `soi.parent_item_id IS NULL` is applied, ensure `base_price` on parent rows reflects the intended revenue (in Magento 2, configurable products store pricing on the parent row, whereas certain bundle configurations store pricing on child rows).
