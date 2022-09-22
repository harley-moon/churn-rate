# churn-rate
This project uses fictional data to calculate subscription churn rates for a streaming service. These insights were then used to compare two segments of users to determine which segment should be expanded. 

This analysis uses the *subscriptions* table; there are two user segments (87 and 30). 
```sql
SELECT *
FROM subscriptions
LIMIT 20;

SELECT DISTINCT segment
FROM subscriptions;
```
<img width="800" alt="churnrate_1" src="https://user-images.githubusercontent.com/96267643/191808274-7e40a483-5255-40f5-bb7e-6e051a17894b.png">

This query determines the range of months provided in the data.
* The months with available subscription start and end data are January, February and March 2017.
```sql
SELECT MIN (subscription_start),
MAX (subscription_start)
FROM subscriptions;
```
<img width="509" alt="churnrate_2" src="https://user-images.githubusercontent.com/96267643/191808609-5ddcb372-7af9-4eb8-9cc8-51e4a80d82e0.png">

To calculate the churn rates for user segments 87 and 30 in January-March, I have broken down the query as follows:

Creation of temporary table *months* with first_day and last_day columns.
```sql
WITH months AS
(SELECT
  '2017-01-01' AS 'first_day',
  '2017-01-31' AS 'last_day'
UNION
SELECT
  '2017-02-01' AS 'first_day',
  '2017-02-28' AS 'last_day'
UNION
SELECT
  '2017-03-01' AS 'first_day',
  '2017-03-31' AS 'last_day'
),
```
<img width="509" alt="churnrate_3" src="https://user-images.githubusercontent.com/96267643/191808918-6647fac3-7067-4938-b8e5-62ce4bfc1e0b.png">

Creation of temporary table *cross_join* to cross join the *subscriptions* table and the *months* table.
```sql
cross_join AS
(SELECT * FROM subscriptions
CROSS JOIN months
),
```
<img width="850" alt="churnrate_4" src="https://user-images.githubusercontent.com/96267643/191809695-fec452db-6a81-4fba-a7ee-de342adb48c8.png">

Creation of temporary table *status*, using case statements to show which segment each user is in and if they had an active or canceled subscription during each month.
```sql
status AS
(SELECT id,
  first_day AS 'month',
  CASE
    WHEN (subscription_start < first_day) AND
    (subscription_end > first_day OR subscription_end IS NULL) AND
    (segment = 87) THEN 1
      ELSE 0
    END AS 'is_active_87',
  CASE
    WHEN (subscription_start < first_day) AND
    (subscription_end > first_day OR subscription_end IS NULL) AND
    (segment = 30) THEN 1
      ELSE 0
    END AS 'is_active_30',
CASE
      WHEN (subscription_end BETWEEN first_day AND last_day) AND 
      (segment = 87) THEN 1
        ELSE 0
      END AS 'is_canceled_87',
    CASE
      WHEN (subscription_end BETWEEN first_day AND last_day) AND 
      (segment = 30) THEN 1
        ELSE 0
      END AS 'is_canceled_30'
FROM cross_join
),
```
<img width="850" alt="churnrate_5" src="https://user-images.githubusercontent.com/96267643/191809994-41e79b84-fb91-45f8-b97b-a44c7b4a1b9f.png">

Creation of temporary table *status_aggregate* to calculate the sum of active and canceled subscriptions for each user segment in each month. 
```sql
status_aggregate AS
(SELECT month,
  SUM(is_active_87) AS 'sum_active_87',
  SUM(is_active_30) AS 'sum_active_30',
  SUM(is_canceled_87) AS 'sum_canceled_87',
  SUM(is_canceled_30) AS 'sum_canceled_30'
FROM status
GROUP BY month
)
```
<img width="917" alt="churnrate_6" src="https://user-images.githubusercontent.com/96267643/191810240-95638625-046f-4ca0-95c2-48af6539f9a7.png">

This query calculates churn rates over the three month period.
```sql
SELECT month,
    1.0 * sum_canceled_87/sum_active_87 AS 'churn_rate_87',
    1.0 * sum_canceled_30/sum_active_30 AS 'churn_rate_30'
FROM status_aggregate;
```

Put together, the entire query to calculate the churn rates for each user segment, each month is:
```sql
WITH months AS
(SELECT
  '2017-01-01' AS 'first_day',
  '2017-01-31' AS 'last_day'
UNION
SELECT
  '2017-02-01' AS 'first_day',
  '2017-02-28' AS 'last_day'
UNION
SELECT
  '2017-03-01' AS 'first_day',
  '2017-03-31' AS 'last_day'
),
cross_join AS
(SELECT * FROM subscriptions
CROSS JOIN months
),
status AS
(SELECT id,
  first_day AS 'month',
  CASE
    WHEN (subscription_start < first_day) AND
    (subscription_end > first_day OR subscription_end IS NULL) AND
    (segment = 87) THEN 1
      ELSE 0
    END AS 'is_active_87',
  CASE
    WHEN (subscription_start < first_day) AND
    (subscription_end > first_day OR subscription_end IS NULL) AND
    (segment = 30) THEN 1
      ELSE 0
    END AS 'is_active_30',
    CASE
      WHEN (subscription_end BETWEEN first_day AND last_day) AND 
      (segment = 87) THEN 1
        ELSE 0
      END AS 'is_canceled_87',
    CASE
      WHEN (subscription_end BETWEEN first_day AND last_day) AND 
      (segment = 30) THEN 1
        ELSE 0
      END AS 'is_canceled_30'
FROM cross_join
), 
status_aggregate AS
(SELECT month,
  SUM(is_active_87) AS 'sum_active_87',
  SUM(is_active_30) AS 'sum_active_30',
  SUM(is_canceled_87) AS 'sum_canceled_87',
  SUM(is_canceled_30) AS 'sum_canceled_30'
FROM status
GROUP BY month
) 
SELECT month,
    1.0 * sum_canceled_87/sum_active_87 AS 'churn_rate_87',
    1.0 * sum_canceled_30/sum_active_30 AS 'churn_rate_30'
FROM status_aggregate;
```
<img width="557" alt="churnrate_7" src="https://user-images.githubusercontent.com/96267643/191811645-825e8859-e4b6-44a3-a051-a73ef6a28f99.png">

Segment 30 had the lower churn rate, losing less subscribers than segment 87. This indicates that segment 30 should be expanded.

Rounded churn rates:
* Segment 87
  * January: 25%
  * February: 32%
  * March: 49%
* Segment 30
  * January: 8%
  * February: 7%
  * March: 12%
