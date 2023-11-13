# SQL_Ecommerce_Exploring
Utilized SQL in Google BigQuery to write and execute queries to find the desired data.

**1. INTRODUCTION**
   
The sample dataset contains obfuscated Google Analytics 360 data from the Google Merchandise Store, a real ecommerce store. The Google Merchandise Store sells Google branded merchandise. The data is typical of what you would see for an ecommerce website. It includes the following kinds of information:

- Traffic source data: information about where website visitors originate. This includes data about organic traffic, paid search traffic, display traffic, etc.
- Content data: information about the behavior of users on the site. This includes the URLs of pages that visitors look at, how they interact with content, etc.
- Transactional data: information about the transactions that occur on the Google Merchandise Store website.

=> Based on this dataset I will explore it by using Google Bigquery  to have an outlook on the business situation, marketing activity efficiency analyzing the products.


**2. THE GOAL OF PROJECT**


**3. DATASET ACCESS**

To access the dataset:

1. Go to https://console.cloud.google.com/bigquery.
2. If you're new to BigQuery (or you don't have a project set up yet) [visit BigQuery sandbox](https://cloud.google.com/bigquery/docs/sandbox).
3. Once you have a project, use [this link](https://console.cloud.google.com/marketplace/product/obfuscated-ga360-data/obfuscated-ga360-data?project=lexical-script-761&folder=&organizationId=) to access the dataset. Click **View Dataset** to open the dataset in your project.
4. Click **Query Table** to run a query.

=> In the future you can access the dataset within BigQuery by selecting the **bigquery-public-data** project from the left-hand navigation panel, then select the **ga_sessions** table under the **google_analytics_sample** dataset.

**4. READ AND EXPLAIN DATASET**

| Field Name|Data Type|Description|
| :------------:|:-------------:|:-----:|
|fullVisitorId|STRING     | The unique visitor ID.    |
|date|RECORD     |  This section contains aggregate values across the session.   |
|totals|RECORD         |   This section contains aggregate values across the session.  |
|totals.bounces   |INTEGER   |  Total bounces (for convenience). For a bounced session, the value is 1, otherwise it is null.  |
|totals.hits  |INTEGER     |   Total number of hits within the session.   |
|totals.pageviews |INTEGER         |    Total number of pageviews within the session. |
|totals.visits |INTEGER     | The number of sessions (for convenience). This value is 1 for sessions with interaction events. The value is null if there are no interaction events in the session. |
|trafficSource.source      |STRING    |  The source of the traffic source. Could be the name of the search engine, the referring hostname, or a value of the utm_source URL parameter.   |
|hits    |RECORD         |   This row and nested fields are populated for any and all types of hits.  |
|hits.eCommerceAction       |RECORD  |  This section contains all of the ecommerce hits that occurred during the session. This is a repeated field and has an entry for each hit that was collected.  |
|hits.eCommerceAction.action_type        |STRING |  The action type. Click through of product lists = 1, Product detail views = 2, Add product(s) to cart = 3, Remove product(s) from cart = 4, Check out = 5, Completed purchase = 6, Refund of purchase = 7, Checkout options = 8, Unknown = 0.Usually this action type applies to all the products in a hit, with the following exception: when hits.product.isImpression = TRUE, the corresponding product is a product impression that is seen while the product action is taking place (i.e., a product in list view).Example query to calculate number of products in list views:SELECTCOUNT(hits.product.v2ProductName)FROM [foo-160803:123456789.ga_sessions_20170101]WHERE hits.product.isImpression == TRUEExample query to calculate number of products in detailed view:SELECTCOUNT(hits.product.v2ProductName),FROM[foo-160803:123456789.ga_sessions_20170101]WHEREhits.ecommerceaction.action_type = 2AND ( BOOLEAN(hits.product.isImpression) IS NULL OR BOOLEAN(hits.product.isImpression) == FALSE ) |
|hits.product   |RECORD        |   This row and nested fields will be populated for each hit that contains Enhanced Ecommerce PRODUCT data. |
|hits.product.productQuantity     |INTEGER         |    The quantity of the product purchased. |
|hits.product.productRevenue    |INTEGER         |    The revenue of the product, expressed as the value passed to Analytics multiplied by 10^6 (e.g., 2.40 would be given as 2400000). |
|hits.product.productSKU      |STRING        |    Product SKU. |
|hits.product.v2ProductName     |STRING            |   Product Name. |

More detail: https://support.google.com/analytics/answer/3437719?hl=en

**5. EXPLORING DATASET**

***Query 01:*** Calculate total visit, pageview, transaction for Jan, Feb and March 2017 (order by month)

**CODE:**
```sql
SELECT
  LEFT(date,6) AS month,
  SUM(totals.visits) AS visits,
  SUM(totals.pageviews) AS pageviews,
  SUM(totals.transactions) AS transactions
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE
  _table_suffix BETWEEN '0101'AND '0331'
GROUP BY 1
ORDER BY 1;
```
**RESULT**

|      month       |      visits       | pageviews     |transactions|
| :------------:|:-------------:|:-----:|:-----:|
|    201701          |        64694      |  257708    |  713    |
|     201702         |        62192      |   233373   |  733    |
|     201703         | 69931             |    259522  |  993    |

***Query 02:*** Calculate total visit, pageview, transaction for Jan, Feb and March 2017 (order by month)

**CODE:**
```sql
SELECT
  trafficSource.source AS SOURCE,
  SUM(totals.visits) AS total_visits,
  SUM(totals.bounces) AS total_no_of_bounces,
  ROUND(100.0 * (SUM(totals.bounces)/SUM(totals.visits)),2) AS bounce_rate
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
WHERE
  _table_suffix BETWEEN '01'AND '31'
GROUP BY 1
ORDER BY 3 DESC;
```
**RESULT**

|      SOURCE      |      total_visits       | total_no_of_bounces     |bounce_rate|
| :------------:|:-------------:|:-----:|:-----:|
|    google          |        38400      |  19798    |  51.56    |
|     (direct)         |        19891      |   8606   |  43.27    |
|     youtube.com      | 6351             |    4238  |  66.73    |
|    analytics.google.com      |        1972      |   1064   |  53.96   |
|     Partners   | 1788             |    936  |  52.35   |
|     ...   | ...             |    ...  |  ...   |

***Query 03:*** Revenue by traffic source by week, by month in June 2017

**CODE:**
```sql
WITH raw_month AS(
  SELECT
    'Month' AS time_type,
    LEFT(date,6) AS month,
    trafficSource.source AS SOURCE,
    ROUND(SUM(product.productRevenue)/1000000,4) AS revenue
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST (hits) hits,
    UNNEST (hits.product) product
  WHERE
    _table_suffix BETWEEN '01'AND '30'
  GROUP BY 1,2,3
  ORDER BY 3,4 
  ),

  raw_week AS(
  SELECT
    'Week' AS time_type,
    LEFT(date,6) AS month,
    trafficSource.source AS SOURCE,
    ROUND(SUM(product.productRevenue)/1000000,4) AS revenue
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST (hits) hits,
    UNNEST (hits.product) product
  WHERE
    _table_suffix BETWEEN '01'AND '30'
  GROUP BY 1,2,3
  ORDER BY 2,3 
  )
  
SELECT *
FROM raw_month
UNION ALL
SELECT *
FROM raw_week
ORDER BY 4 DESC
```

**RESULT**

|      time_type      |      month       | SOURCE     |revenue|
| :------------:|:-------------:|:-----:|:-----:|
|    Week          |        201706      |  (direct)    |  97333.6197   |
|     Month        |        201706      |   (direct)   |  97333.6197   |
|     Week     | 201706             |    google  |  18757.1799  |
|    Month      |        201706      |   google   |  18757.1799   |
|     Month   | 201706             |    dfa  |  8862.23 |
|     ...   | ...             |    ...  |  ...   |

***Query 04:*** Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.

**CODE:**
```sql
WITH purchase AS (
  SELECT
    LEFT(date,6) AS month,
    ROUND(SUM(totals.pageviews)/COUNT(DISTINCT(fullVisitorId)),2) AS avg_pageviews_purchase
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST (hits) hits,
    UNNEST (hits.product) product
  WHERE
    _table_suffix BETWEEN '0601'AND '0731'
    AND totals.transactions >=1
    AND product.productRevenue IS NOT NULL
  GROUP BY 1
  ORDER BY 1
  ),

  non_purchase AS(
  SELECT
    LEFT(date,6) AS month,
    ROUND(SUM(totals.pageviews)/COUNT(DISTINCT(fullVisitorId)),2) AS avg_pageviews_non_purchase
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST (hits) hits,
    UNNEST (hits.product) product
  WHERE
    _table_suffix BETWEEN '0601'AND '0731'
    AND totals.transactions IS NULL
    AND product.productRevenue IS NULL
  GROUP BY 1
  ORDER BY 1
  )
  
SELECT
  p.month,
  p.avg_pageviews_purchase,
  n.avg_pageviews_non_purchase
FROM
  purchase AS p
LEFT JOIN
  non_purchase AS n
ON
  p.month = n.month
```

**RESULT**

|      month      |      avg_pageviews_purchase       | avg_pageviews_non_purchase     |
| :------------:|:-------------:|:-----:|
|    201706          |        94.02      |  316.87   |
|     201707        |        124.24     |   334.06   |

***Query 05:*** Average number of transactions per user that made a purchase in July 2017

**CODE:**
```sql
SELECT
  LEFT(date,6) AS month,
  ROUND(SUM(totals.transactions)/COUNT(DISTINCT(fullVisitorId)),5) AS Avg_total_transactions_per_user
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
WHERE
  _table_suffix BETWEEN '0701'AND '0731'
  AND totals.transactions >=1
  AND product.productRevenue IS NOT NULL
GROUP BY 1
ORDER BY 1
```

**RESULT**

|      month      |      Avg_total_transactions_per_user       |
| :------------:|:-------------:|
|     201707        |        4.1639   |  

***Query 06:*** Average amount of money spent per session. Only include purchaser data in July 2017

**CODE:**
```sql
SELECT
  LEFT(date,6) AS month,
  TRUNC((SUM(product.productRevenue)/SUM(totals.visits)/1000000),2) AS avg_revenue_by_user_per_visit
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
WHERE
  _table_suffix BETWEEN '0701'AND '0731'
  AND totals.transactions IS NOT NULL
  AND product.productRevenue IS NOT NULL
GROUP BY 1
ORDER BY 1
```

**RESULT**

|      month      |      avg_revenue_by_user_per_visit       |
| :------------:|:-------------:|
|     201707        |        43.85   |  

***Query 07:*** Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered.

**CODE:**
```sql
WITH buyer AS(
  SELECT
    DISTINCT(fullVisitorId),
    product.v2ProductName
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST (hits) hits,
    UNNEST (hits.product) product
  WHERE
    _table_suffix BETWEEN '0701'AND '0731'
    AND product.productRevenue IS NOT NULL
    AND product.v2ProductName = "YouTube Men's Vintage Henley")

SELECT
  product.v2ProductName AS other_purchased_products,
  SUM(productQuantity) AS quantity
FROM
  `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
WHERE
  _table_suffix BETWEEN '0701'AND '0731'
  AND fullVisitorId IN (SELECT fullVisitorId FROM buyer)
  AND product.productRevenue IS NOT NULL
GROUP BY 1
ORDER BY 2 DESC
```

**RESULT**

|month|avg_revenue_by_user_per_visit|
| :------------:|:-------------:|
|Google Sunglasses|20|  
|YouTube Men's Vintage Henley|13|  
|Google Women's Vintage Hero Tee Black|7|  
|SPF-15 Slim & Slender Lip Balm|6|  
|Google Women's Short Sleeve Hero Tee Red Heather|4|  

***Query 08:*** Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017. For example, 100% product view then 40% add_to_cart and 10% purchase.
Add_to_cart_rate = number product  add to cart/number product view. Purchase_rate = number product purchase/number product view. The output should be calculated in product level.

**CODE:**
```sql
WITH product_cart AS(
  SELECT
    LEFT(date,6) AS month,
    SUM(
      CASE
        WHEN eCommerceAction.action_type = '2' THEN 1
      ELSE 0
    END) AS num_product_view,
    SUM(
      CASE
        WHEN eCommerceAction.action_type = '3' THEN 1
      ELSE 0
    END) AS num_addtocart
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST (hits) hits,
    UNNEST (hits.product) product
  WHERE
    _table_suffix BETWEEN '0101'AND '0331'
  GROUP BY 1
  ORDER BY 1
  ),

  purchase AS(
  SELECT
    LEFT(date,6) AS month,
    SUM(
      CASE
        WHEN eCommerceAction.action_type = '6' THEN 1
      ELSE 0
    END) AS num_purchase
  FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST (hits) hits,
    UNNEST (hits.product) product
  WHERE
    _table_suffix BETWEEN '0101'AND '0331'
    AND product.productRevenue IS NOT NULL
  GROUP BY 1
  ORDER BY 1
  )

SELECT
  pc.month,
  pc.num_product_view,
  pc.num_addtocart,
  p.num_purchase,
  ROUND(100.0 * (pc.num_addtocart/pc.num_product_view),2) AS add_to_cart_rate,
  ROUND(100.0 * (p.num_purchase/pc.num_product_view),2) AS purchase_rate
FROM
  product_cart AS pc
LEFT JOIN
  purchase AS p
ON
  pc.month = p.month
ORDER BY 1
```

**RESULT**

|      month       |      num_product_view       | num_addtocart     |num_purchase|  add_to_cart_rate  |  purchase_rate  |   
| :------------:|:-------------:|:-----:|:-----:|:-----:|:-----:|
|    201701          |        25787      |  7342    |  2143    |  28.47    |  8.31    |
|     201702         |        21489      |   7360   |  2060    |  34.25   |  9.59    |
|     201703         | 23549             |    8782  |  2977    |  37.29   |  12.64    |
