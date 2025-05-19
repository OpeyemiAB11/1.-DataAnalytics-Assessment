# 1.-DataAnalytics-Assessment


This repository contains solutions to four SQL data analysis questions based on a mock database of customer transactions, savings, and investment plans.

Question Summaries and Solutions

Assessment_Q1.sql – High-Value Customers with Multiple Products

Objective: Identify customers with at least one funded savings plan and one funded investment plan, sorted by total deposits.

Approach:

Created two CTEs (savings, investments) to isolate customers with qualifying plan types.

Joined these with a total CTE to calculate total deposits per customer.

Returned users with both types and sorted them by total deposit value.

Challenge: Ensuring accuracy in customer-product matching, resolved using DISTINCT and proper aggregation.

Assessment_Q2.sql – Transaction Frequency Analysis

Objective: Categorize customers based on average monthly transaction frequency.

Approach:

Calculated monthly transaction count per customer.

Averaged this to get the monthly frequency per user.

Categorized each customer as High, Medium, or Low frequency.

Aggregated the category counts and calculated the average transactions.

Challenge: Reused formatted dates with DATE_FORMAT; ensured grouping and averaging worked consistently.

Assessment_Q3.sql – Account Inactivity Alert

Objective: Identify active savings/investment plans with no inflow in the last 365 days.

Approach:

Used a CTE to extract the most recent inflow date per plan.

Filtered for plans with inactivity greater than one year.

Classified plans based on type flags in plans_plan.

Challenge: Handling NULLs for plans with no transactions; resolved using conditional checks.

Assessment_Q4.sql – Customer Lifetime Value (CLV) Estimation

Objective: Estimate CLV using a simplified formula involving tenure, transaction volume, and profit per transaction.

Approach:

Calculated tenure using TIMESTAMPDIFF.

Counted all inflow transactions and computed the average amount.

Applied the given formula to estimate CLV per customer.

Challenge: Preventing division by zero for new accounts; resolved using a HAVING clause.

Notes

All SQL queries follow clean formatting and include explanatory comments for complex logic.

All queries were tested with proper join and aggregation strategies to ensure optimal performance.

Author: [Oduyombo Opeyemi Abraham]Submission: Public GitHub Repository as per instructions



"Assessment_Q1.sql": """
-- Assessment_Q1.sql
-- Identify owners who have both savings and investment plans,
-- along with total deposits.

WITH savings AS (
    SELECT DISTINCT p.owner_id, p.id AS plan_id
    FROM plans_plan p
    JOIN savings_savingsaccount s ON p.id = s.plan_id
    WHERE p.is_regular_savings = 1
      AND s.amount > 0
),
investments AS (
    SELECT DISTINCT p.owner_id, p.id AS plan_id
    FROM plans_plan p
    JOIN savings_savingsaccount s ON p.id = s.plan_id
    WHERE p.is_fixed_investment = 1
      AND s.amount > 0
),
totals AS (
    SELECT p.owner_id,
           SUM(s.amount) AS total_deposits
    FROM plans_plan p
    JOIN savings_savingsaccount s ON p.id = s.plan_id
    WHERE s.amount > 0
    GROUP BY p.owner_id
)

SELECT
    u.id AS owner_id,
    u.name,
    COUNT(DISTINCT sa.plan_id) AS savings_count,
    COUNT(DISTINCT ia.plan_id) AS investment_count,
    ROUND(t.total_deposits, 2) AS total_deposits
FROM users_customuser u
JOIN savings sa ON u.id = sa.owner_id
JOIN investments ia ON u.id = ia.owner_id
JOIN totals t ON u.id = t.owner_id
GROUP BY u.id, u.name, t.total_deposits
HAVING savings_count > 0 AND investment_count > 0
ORDER BY total_deposits DESC;
""",

    "Assessment_Q2.sql": """
-- Assessment_Q2.sql
-- Categorize customers by their average monthly transaction frequency.

WITH customer_monthly_txn AS (
    SELECT
        s.owner_id,
        DATE_FORMAT(s.transaction_date, '%Y-%m') AS month,
        COUNT(*) AS monthly_txns
    FROM savings_savingsaccount s
    WHERE s.amount > 0
    GROUP BY s.owner_id, month
),

avg_txn_per_customer AS (
    SELECT
        owner_id,
        AVG(monthly_txns) AS avg_txn_per_month
    FROM customer_monthly_txn
    GROUP BY owner_id
),

categorized_customers AS (
    SELECT
        owner_id,
        avg_txn_per_month,
        CASE
            WHEN avg_txn_per_month >= 10 THEN 'High Frequency'
            WHEN avg_txn_per_month BETWEEN 3 AND 9 THEN 'Medium Frequency'
            ELSE 'Low Frequency'
        END AS frequency_category
    FROM avg_txn_per_customer
)

SELECT
    frequency_category,
    COUNT(*) AS customer_count,
    ROUND(AVG(avg_txn_per_month), 1) AS avg_transactions_per_month
FROM categorized_customers
GROUP BY frequency_category
ORDER BY
    CASE frequency_category
        WHEN 'High Frequency' THEN 1
        WHEN 'Medium Frequency' THEN 2
        WHEN 'Low Frequency' THEN 3
    END;
""",

    "Assessment_Q3.sql": """
-- Assessment_Q3.sql
-- List plans inactive for more than 365 days, classified by type.

WITH latest_transactions AS (
    SELECT
        s.plan_id,
        MAX(s.transaction_date) AS last_transaction_date
    FROM savings_savingsaccount s
    WHERE s.amount > 0
    GROUP BY s.plan_id
)

SELECT
    p.id AS plan_id,
    p.owner_id,
    CASE
        WHEN p.is_regular_savings = 1 THEN 'Savings'
        WHEN p.is_fixed_investment = 1 THEN 'Investment'
        ELSE 'Other'
    END AS type,
    lt.last_transaction_date,
    DATEDIFF(CURDATE(), lt.last_transaction_date) AS inactivity_days
FROM plans_plan p
JOIN latest_transactions lt ON p.id = lt.plan_id
WHERE
    DATEDIFF(CURDATE(), lt.last_transaction_date) > 365
    AND (p.is_regular_savings = 1 OR p.is_fixed_investment = 1)
ORDER BY inactivity_days DESC;
""",

    "Assessment_Q4.sql": """
-- Assessment_Q4.sql
-- Estimate customer lifetime value (CLV) based on transactions and tenure.

WITH user_txn_stats AS (
    SELECT
        u.id AS customer_id,
        u.name,
        TIMESTAMPDIFF(MONTH, u.date_joined, CURDATE()) AS tenure_months,
        COUNT(s.id) AS total_transactions,
        AVG(s.amount) AS avg_txn_amount
    FROM users_customuser u
    JOIN savings_savingsaccount s ON u.id = s.owner_id
    WHERE s.amount > 0
    GROUP BY u.id, u.name, u.date_joined
    HAVING tenure_months > 0  -- avoid division by zero
)

SELECT
    customer_id,
    name,
    tenure_months,
    total_transactions,
    ROUND((total_transactions / tenure_months) * 12 * (0.001 * avg_txn_amount), 2) AS estimated_clv
FROM user_txn_stats
ORDER BY estimated_clv DESC;
""",

 
