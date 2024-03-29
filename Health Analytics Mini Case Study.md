# Health Analytics Mini Case Study
### by Thai Tran

We’ve just received an urgent request from the General Manager of Analytics at Health Co requesting our assistance with their analysis of the health.user_logs dataset.
The Health Co analytics team have shared with us their SQL script - they unfortunately ran into a few bugs that they couldn’t fix!
We’ve been asked to quickly debug their SQL script and use the resulting query outputs to quickly answer a few questions that the GM has requested for a board meeting about their active users.

## 1. How many unique users exist in the logs dataset?
```sql
SELECT
  COUNT (DISTINCT id)
FROM health.user_logs;
```

## 2. How many total measurements do we have per user on average?
```sql
DROP TABLE IF EXISTS user_measure_count;

CREATE TEMP TABLE user_measure_count AS

SELECT

    id,

    COUNT(*) AS measure_count,

    COUNT(DISTINCT measure) as unique_measures

  FROM health.user_logs

  GROUP BY 1; 
```


## 3. What about the median number of measurements per user?
```sql
SELECT

  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY measure_count) AS median_value

FROM user_measure_count;

SELECT * FROM user_measure_count;
```
## 4. How many users have 3 or more measurements?
```sql
SELECT
  COUNT(*) 
FROM user_measure_count
WHERE measure_count >= 3;
```
## 5. How many users have 1,000 or more measurements?
```sql
SELECT
  COUNT(id)
FROM user_measure_count
WHERE measure_count >= 1000;
```
## 6. Have logged blood glucose measurements?
```sql
SELECT
  *
FROM health.user_logs

SELECT
  COUNT (DISTINCT id)
FROM health.user_logs
WHERE measure = 'blood_glucose';
```
## 7. Have at least 2 types of measurements?
```sql
SELECT

  COUNT(*)

FROM user_measure_count

WHERE unique_measures >= 2;
```

## 8. Have all 3 measures - blood glucose, weight and blood pressure?
```sql
SELECT
  COUNT(*)
FROM user_measure_count
WHERE unique_measures = 3;
```
 ## 9.  What is the median systolic/diastolic blood pressure values?
```sql
SELECT
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY systolic) AS median_systolic,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY diastolic) AS median_diastolic
FROM health.user_logs
WHERE measure = 'blood_pressure';
```