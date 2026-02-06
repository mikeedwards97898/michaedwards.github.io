# Customer Analytics SQL Project
## Business Analyst Portfolio - Michael Edwards

### Project Overview
This SQL analysis demonstrates proficiency in customer analytics, focusing on metrics critical for subscription-based businesses like Starlink. The analysis covers Customer Acquisition Cost (CAC), Lifetime Value (LTV), churn analysis, and cohort performance.

---

## Table of Contents
1. [Database Schema](#database-schema)
2. [Customer Acquisition Analysis](#customer-acquisition-analysis)
3. [Lifetime Value (LTV) Calculation](#lifetime-value-calculation)
4. [Churn Analysis](#churn-analysis)
5. [Cohort Performance](#cohort-performance)
6. [Revenue Forecasting](#revenue-forecasting)
7. [Key Insights & Recommendations](#key-insights-recommendations)

---

## Database Schema

```sql
-- Customers table
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    signup_date DATE,
    acquisition_channel VARCHAR(50),
    acquisition_cost DECIMAL(10,2),
    region VARCHAR(50),
    plan_type VARCHAR(50)
);

-- Subscriptions table
CREATE TABLE subscriptions (
    subscription_id INT PRIMARY KEY,
    customer_id INT,
    start_date DATE,
    end_date DATE,
    monthly_price DECIMAL(10,2),
    plan_type VARCHAR(50),
    status VARCHAR(20),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Transactions table
CREATE TABLE transactions (
    transaction_id INT PRIMARY KEY,
    customer_id INT,
    transaction_date DATE,
    amount DECIMAL(10,2),
    transaction_type VARCHAR(50),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

---

## 1. Customer Acquisition Analysis

### Total Customers by Acquisition Channel
```sql
SELECT 
    acquisition_channel,
    COUNT(DISTINCT customer_id) as total_customers,
    AVG(acquisition_cost) as avg_cac,
    SUM(acquisition_cost) as total_acquisition_spend
FROM customers
WHERE signup_date >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY acquisition_channel
ORDER BY total_customers DESC;
```

### Monthly Acquisition Trends
```sql
SELECT 
    DATE_FORMAT(signup_date, '%Y-%m') as signup_month,
    acquisition_channel,
    COUNT(DISTINCT customer_id) as new_customers,
    AVG(acquisition_cost) as avg_cac
FROM customers
GROUP BY signup_month, acquisition_channel
ORDER BY signup_month DESC, new_customers DESC;
```

### CAC by Region and Channel
```sql
SELECT 
    region,
    acquisition_channel,
    COUNT(DISTINCT customer_id) as customers,
    AVG(acquisition_cost) as avg_cac,
    MIN(acquisition_cost) as min_cac,
    MAX(acquisition_cost) as max_cac
FROM customers
WHERE signup_date >= '2024-01-01'
GROUP BY region, acquisition_channel
ORDER BY region, avg_cac DESC;
```

---

## 2. Lifetime Value (LTV) Calculation

### Customer LTV with Revenue Breakdown
```sql
WITH customer_revenue AS (
    SELECT 
        c.customer_id,
        c.signup_date,
        c.acquisition_cost,
        c.acquisition_channel,
        SUM(t.amount) as total_revenue,
        COUNT(DISTINCT t.transaction_id) as total_transactions,
        DATEDIFF(
            COALESCE(MAX(s.end_date), CURRENT_DATE), 
            c.signup_date
        ) / 30.0 as lifetime_months
    FROM customers c
    LEFT JOIN transactions t ON c.customer_id = t.customer_id
    LEFT JOIN subscriptions s ON c.customer_id = s.customer_id
    GROUP BY c.customer_id, c.signup_date, c.acquisition_cost, c.acquisition_channel
)
SELECT 
    acquisition_channel,
    COUNT(customer_id) as total_customers,
    AVG(total_revenue) as avg_ltv,
    AVG(acquisition_cost) as avg_cac,
    AVG(total_revenue) - AVG(acquisition_cost) as avg_profit_per_customer,
    AVG(total_revenue) / NULLIF(AVG(acquisition_cost), 0) as ltv_cac_ratio,
    AVG(lifetime_months) as avg_customer_lifetime_months
FROM customer_revenue
GROUP BY acquisition_channel
ORDER BY avg_ltv DESC;
```

### LTV by Cohort (Signup Month)
```sql
WITH monthly_cohorts AS (
    SELECT 
        c.customer_id,
        DATE_FORMAT(c.signup_date, '%Y-%m') as cohort_month,
        c.acquisition_cost,
        SUM(t.amount) as total_revenue
    FROM customers c
    LEFT JOIN transactions t ON c.customer_id = t.customer_id
    GROUP BY c.customer_id, cohort_month, c.acquisition_cost
)
SELECT 
    cohort_month,
    COUNT(customer_id) as cohort_size,
    AVG(total_revenue) as avg_ltv,
    AVG(acquisition_cost) as avg_cac,
    AVG(total_revenue) / NULLIF(AVG(acquisition_cost), 0) as ltv_cac_ratio
FROM monthly_cohorts
GROUP BY cohort_month
ORDER BY cohort_month DESC;
```

---

## 3. Churn Analysis

### Monthly Churn Rate
```sql
WITH monthly_status AS (
    SELECT 
        DATE_FORMAT(start_date, '%Y-%m') as month,
        COUNT(DISTINCT CASE WHEN status = 'active' THEN customer_id END) as active_start,
        COUNT(DISTINCT CASE WHEN status = 'cancelled' AND 
              DATE_FORMAT(end_date, '%Y-%m') = DATE_FORMAT(start_date, '%Y-%m') 
              THEN customer_id END) as churned
    FROM subscriptions
    GROUP BY month
)
SELECT 
    month,
    active_start,
    churned,
    ROUND((churned * 100.0 / NULLIF(active_start, 0)), 2) as churn_rate_pct
FROM monthly_status
ORDER BY month DESC;
```

### Churn Analysis by Plan Type
```sql
SELECT 
    s.plan_type,
    COUNT(DISTINCT s.customer_id) as total_customers,
    COUNT(DISTINCT CASE WHEN s.status = 'cancelled' THEN s.customer_id END) as churned_customers,
    ROUND(
        COUNT(DISTINCT CASE WHEN s.status = 'cancelled' THEN s.customer_id END) * 100.0 / 
        NULLIF(COUNT(DISTINCT s.customer_id), 0), 
        2
    ) as churn_rate_pct,
    AVG(DATEDIFF(s.end_date, s.start_date) / 30.0) as avg_subscription_length_months
FROM subscriptions s
GROUP BY s.plan_type
ORDER BY churn_rate_pct ASC;
```

### Churn Prediction - At-Risk Customers
```sql
WITH customer_metrics AS (
    SELECT 
        c.customer_id,
        c.signup_date,
        s.plan_type,
        COUNT(DISTINCT t.transaction_id) as transaction_count,
        MAX(t.transaction_date) as last_transaction_date,
        DATEDIFF(CURRENT_DATE, MAX(t.transaction_date)) as days_since_last_transaction,
        AVG(t.amount) as avg_transaction_value
    FROM customers c
    LEFT JOIN subscriptions s ON c.customer_id = s.customer_id
    LEFT JOIN transactions t ON c.customer_id = t.customer_id
    WHERE s.status = 'active'
    GROUP BY c.customer_id, c.signup_date, s.plan_type
)
SELECT 
    customer_id,
    plan_type,
    transaction_count,
    last_transaction_date,
    days_since_last_transaction,
    avg_transaction_value,
    CASE 
        WHEN days_since_last_transaction > 60 THEN 'High Risk'
        WHEN days_since_last_transaction > 30 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END as churn_risk_category
FROM customer_metrics
WHERE days_since_last_transaction > 30
ORDER BY days_since_last_transaction DESC;
```

---

## 4. Cohort Performance

### Cohort Retention Analysis
```sql
WITH cohorts AS (
    SELECT 
        c.customer_id,
        DATE_FORMAT(c.signup_date, '%Y-%m') as cohort_month,
        DATE_FORMAT(t.transaction_date, '%Y-%m') as transaction_month,
        PERIOD_DIFF(
            DATE_FORMAT(t.transaction_date, '%Y%m'),
            DATE_FORMAT(c.signup_date, '%Y%m')
        ) as months_since_signup
    FROM customers c
    LEFT JOIN transactions t ON c.customer_id = t.customer_id
),
cohort_sizes AS (
    SELECT 
        cohort_month,
        COUNT(DISTINCT customer_id) as cohort_size
    FROM cohorts
    GROUP BY cohort_month
)
SELECT 
    c.cohort_month,
    cs.cohort_size,
    c.months_since_signup,
    COUNT(DISTINCT c.customer_id) as active_customers,
    ROUND(
        COUNT(DISTINCT c.customer_id) * 100.0 / NULLIF(cs.cohort_size, 0), 
        2
    ) as retention_rate_pct
FROM cohorts c
JOIN cohort_sizes cs ON c.cohort_month = cs.cohort_month
WHERE c.months_since_signup IS NOT NULL
GROUP BY c.cohort_month, cs.cohort_size, c.months_since_signup
ORDER BY c.cohort_month DESC, c.months_since_signup;
```

### Cohort Revenue Performance
```sql
WITH cohort_revenue AS (
    SELECT 
        DATE_FORMAT(c.signup_date, '%Y-%m') as cohort_month,
        PERIOD_DIFF(
            DATE_FORMAT(t.transaction_date, '%Y%m'),
            DATE_FORMAT(c.signup_date, '%Y%m')
        ) as months_since_signup,
        SUM(t.amount) as total_revenue,
        COUNT(DISTINCT c.customer_id) as active_customers
    FROM customers c
    LEFT JOIN transactions t ON c.customer_id = t.customer_id
    GROUP BY cohort_month, months_since_signup
)
SELECT 
    cohort_month,
    months_since_signup,
    total_revenue,
    active_customers,
    ROUND(total_revenue / NULLIF(active_customers, 0), 2) as revenue_per_customer
FROM cohort_revenue
WHERE months_since_signup IS NOT NULL
ORDER BY cohort_month DESC, months_since_signup;
```

---

## 5. Revenue Forecasting

### Monthly Recurring Revenue (MRR) Trend
```sql
SELECT 
    DATE_FORMAT(transaction_date, '%Y-%m') as month,
    COUNT(DISTINCT customer_id) as active_subscribers,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_revenue_per_customer,
    SUM(amount) / COUNT(DISTINCT customer_id) as mrr_per_customer
FROM transactions
WHERE transaction_type = 'subscription'
GROUP BY month
ORDER BY month DESC;
```

### Revenue Forecast Based on Historical Growth
```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_FORMAT(transaction_date, '%Y-%m') as month,
        SUM(amount) as revenue
    FROM transactions
    GROUP BY month
),
revenue_growth AS (
    SELECT 
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) as previous_month_revenue,
        (revenue - LAG(revenue) OVER (ORDER BY month)) / 
        NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100 as growth_rate_pct
    FROM monthly_revenue
)
SELECT 
    month,
    revenue,
    previous_month_revenue,
    ROUND(growth_rate_pct, 2) as growth_rate_pct,
    ROUND(AVG(growth_rate_pct) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) 
        as three_month_avg_growth
FROM revenue_growth
ORDER BY month DESC;
```

---

## 6. Advanced Analytics

### Customer Segmentation by Value
```sql
WITH customer_value AS (
    SELECT 
        c.customer_id,
        c.region,
        c.acquisition_channel,
        SUM(t.amount) as total_revenue,
        COUNT(DISTINCT t.transaction_id) as transaction_count,
        DATEDIFF(CURRENT_DATE, c.signup_date) / 30.0 as customer_age_months
    FROM customers c
    LEFT JOIN transactions t ON c.customer_id = t.customer_id
    GROUP BY c.customer_id, c.region, c.acquisition_channel, c.signup_date
)
SELECT 
    CASE 
        WHEN total_revenue >= 1000 THEN 'High Value'
        WHEN total_revenue >= 500 THEN 'Medium Value'
        ELSE 'Low Value'
    END as customer_segment,
    COUNT(customer_id) as customer_count,
    AVG(total_revenue) as avg_revenue,
    AVG(transaction_count) as avg_transactions,
    AVG(customer_age_months) as avg_customer_age_months
FROM customer_value
GROUP BY customer_segment
ORDER BY avg_revenue DESC;
```

### Regional Performance Analysis
```sql
SELECT 
    c.region,
    COUNT(DISTINCT c.customer_id) as total_customers,
    SUM(t.amount) as total_revenue,
    AVG(c.acquisition_cost) as avg_cac,
    SUM(t.amount) / COUNT(DISTINCT c.customer_id) as revenue_per_customer,
    COUNT(DISTINCT CASE WHEN s.status = 'cancelled' THEN c.customer_id END) * 100.0 / 
        COUNT(DISTINCT c.customer_id) as churn_rate_pct
FROM customers c
LEFT JOIN transactions t ON c.customer_id = t.customer_id
LEFT JOIN subscriptions s ON c.customer_id = s.customer_id
GROUP BY c.region
ORDER BY total_revenue DESC;
```

---

## Key Insights & Recommendations

### SQL Skills Demonstrated:
✅ Complex JOIN operations across multiple tables
✅ Window functions (LAG, AVG OVER, ROW_NUMBER)
✅ Common Table Expressions (CTEs) for query organization
✅ Advanced aggregations and GROUP BY with HAVING
✅ Date calculations and time-based analysis
✅ CASE statements for conditional logic
✅ Cohort analysis and retention metrics
✅ Revenue forecasting techniques
✅ Customer segmentation
✅ Churn prediction modeling

### Business Metrics Covered:
- Customer Acquisition Cost (CAC)
- Lifetime Value (LTV)
- LTV:CAC Ratio
- Monthly Recurring Revenue (MRR)
- Churn Rate
- Cohort Retention
- Revenue Forecasting
- Customer Segmentation

### Sample Business Recommendations:
1. **Optimize Acquisition Channels**: Focus investment on channels with LTV:CAC ratio > 3.0
2. **Reduce Churn**: Target at-risk customers (no transaction in 30+ days) with retention campaigns
3. **Improve LTV**: Upsell high-value customers to premium plans based on usage patterns
4. **Regional Expansion**: Prioritize regions with low CAC and high retention rates

---

## Technologies Used
- **SQL** (MySQL/PostgreSQL)
- **Database Design**
- **Statistical Analysis**
- **Business Intelligence**
- **Data Visualization** (queries optimized for Tableau/Power BI integration)

---

*This project demonstrates proficiency in SQL analytics for subscription-based business models, with direct applications to roles requiring customer analytics, growth analysis, and data-driven decision making.*
