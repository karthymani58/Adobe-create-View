# Customer LTV & Order Summary Query

## Overview
This PostgreSQL query aggregates customer profile details, contact information, order history, Lifetime Value (LTV), payment preferences, and purchased product metadata for **Store ID 1**. It also dynamically classifies customers as **New** or **Existing** based on the Indian financial year (April 1 – March 31).

---

## Business Logic & Rules

- **Fiscal Year Classification (`fiscal_year_bounds`)**:
  - Dynamically calculates the current Indian fiscal year (`fy_start` = April 1, `fy_end` = April 1 of the next year).
  - **New Customer**: First valid order falls within the current fiscal year (`first_ever_order_date >= fy_start` and `< fy_end`).
  - **Existing Customer**: First valid order occurred prior to the current fiscal year.
- **Valid Orders (`valid_orders`)**:
  - Filters strictly for `store_id = 1` and active/fulfilled order statuses (`complete`, `processing`, `delivered`, `Packed`, `shipped`, `in-transit`, `dispatching_process`, `partial_delivered`, `out-for-delivery`).
  - Normalizes customer emails using `LOWER(TRIM(customer_email))`.
- **Latest Contact Details (`ranked_shipping` & `latest_contacts`)**:
  - Uses `ROW_NUMBER()` ordered by `created_at DESC, entity_id DESC` to pull the most recent address and phone details per customer, falling back from shipping to billing address fields.
- **LTV & Fan-Out Prevention (`order_payments` & `order_items_agg`)**:
  - Filters out child/configurable product duplicate rows using `soi.parent_item_id IS NULL`.
  - Pre-aggregates `sales_order_payment` at the order level before joining to `sales_order_item` so multiple payment records per order do not inflate `ltv` (`SUM(qty_ordered * base_price)`).

---

## Database Dependencies

| Table | Purpose |
| :--- | :--- |
| `sales_order` | Core order headers, statuses, store filter, and customer emails |
| `sales_order_address` | Shipping and billing contact details (`telephone`, `city`, `region`, `postcode`) |
| `sales_order_item` | Line-item quantities, base prices, SKUs, and product names |
| `sales_order_payment` | Payment method codes mapped to `COD` or `Prepaid` |
| `category_table_mapping_india` | Maps product `sku` to Level-1 (`l1`) product categories |

---

## Output Schema

| Column | Description |
| :--- | :--- |
| `email` | Normalized lowercase customer email address |
| `mobile` | Most recent shipping (or billing) phone number |
| `firstname` | Customer's first name from latest order/address |
| `lastname` | Customer's last name from latest order/address |
| `city` | City from the most recent order address |
| `state` | State/region from the most recent order address |
| `pincode` | Postal code from the most recent order address |
| `customer_type` | `New` (acquired in current FY) or `Existing` |
| `total_orders` | Count of distinct valid `increment_id` orders |
| `ltv` | Total Lifetime Value (`SUM(qty_ordered * base_price)`) |
| `first_order_date` | Timestamp of the customer's earliest valid order |
| `last_order_date` | Timestamp of the customer's most recent valid order |
| `payment_types_used` | Comma-separated distinct payment modes (`COD`, `Prepaid`) |
| `all_bought_categories` | Comma-separated distinct L1 product categories purchased |
| `all_bought_items` | Comma-separated distinct product names purchased |

---

## SQL Query

```sql
WITH fiscal_year_bounds AS (
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
        so.customer_lastname
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
),
ranked_shipping AS (
    SELECT 
        vo.customer_email,
        COALESCE(ship.telephone, bill.telephone) AS telephone,
        COALESCE(ship.firstname, bill.firstname, vo.customer_firstname) AS firstname,
        COALESCE(ship.lastname, bill.lastname, vo.customer_lastname) AS lastname,
        COALESCE(ship.city, bill.city) AS city,
        COALESCE(ship.region, bill.region) AS state,
        COALESCE(ship.postcode, bill.postcode) AS pincode,
        vo.created_at,
        ROW_NUMBER() OVER (
            PARTITION BY vo.customer_email 
            ORDER BY vo.created_at DESC, vo.entity_id DESC
        ) AS order_rank
    FROM valid_orders vo
    LEFT JOIN sales_order_address ship 
        ON vo.entity_id = ship.parent_id AND ship.address_type = 'shipping'
    LEFT JOIN sales_order_address bill 
        ON vo.entity_id = bill.parent_id AND bill.address_type = 'billing'
),
latest_contacts AS (
    SELECT
        customer_email,
        telephone,
        firstname,
        lastname,
        city,
        state,
        pincode,
        created_at AS last_order_date
    FROM ranked_shipping
    WHERE order_rank = 1
),
order_payments AS (
    SELECT 
        parent_id AS order_id,
        STRING_AGG(
            DISTINCT CASE 
                WHEN method IN ('cashondelivery', 'srcheckout_cod') THEN 'COD' 
                ELSE 'Prepaid' 
            END, 
            ', '
        ) AS payment_methods
    FROM sales_order_payment
    GROUP BY parent_id
),
order_items_agg AS (
    SELECT 
        vo.customer_email AS email,
        MAX(CASE 
            WHEN fp.first_ever_order_date >= fyb.fy_start 
             AND fp.first_ever_order_date < fyb.fy_end THEN 'New'
            ELSE 'Existing'
        END) AS customer_type,
        COUNT(DISTINCT vo.increment_id) AS total_orders,
        SUM(soi.qty_ordered * soi.base_price) AS ltv,
        MIN(vo.created_at) AS first_order_date,
        STRING_AGG(DISTINCT op.payment_methods, ', ') AS payment_types_used,
        STRING_AGG(DISTINCT COALESCE(ctm.l1, 'Uncategorized'), ', ') AS all_bought_categories,
        STRING_AGG(DISTINCT soi.name, ', ') AS all_bought_items
    FROM valid_orders vo
    INNER JOIN sales_order_item soi ON vo.entity_id = soi.order_id
    INNER JOIN first_purchase fp ON vo.customer_email = fp.customer_email
    CROSS JOIN fiscal_year_bounds fyb
    LEFT JOIN order_payments op ON vo.entity_id = op.order_id
    LEFT JOIN category_table_mapping_india ctm ON soi.sku = ctm.sku
    WHERE soi.parent_item_id IS NULL
    GROUP BY vo.customer_email
)
SELECT 
    lc.customer_email AS email,
    lc.telephone AS mobile,
    lc.firstname,
    lc.lastname,
    lc.city,
    lc.state,
    lc.pincode,
    oia.customer_type,
    oia.total_orders,
    oia.ltv,
    oia.first_order_date,
    lc.last_order_date,
    oia.payment_types_used,
    oia.all_bought_categories,
    oia.all_bought_items
FROM latest_contacts lc
INNER JOIN order_items_agg oia ON lc.customer_email = oia.email;
