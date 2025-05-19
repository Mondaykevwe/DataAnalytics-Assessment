Question 1
identifying high-value customers who have both a funded savings plan and a funded investment plan, and sorting them by total deposits. I need to join the tables based on user/customer relationships, filter by funded status, and group/sort by user and total deposits. Here’s a SQL query to achieve that
SELECT * FROM adashi_staging.users_customuser;
SELECT 
    u.id AS user_id,
    u.full_name,
    SUM(CASE WHEN s.status = 'funded' THEN s.total_deposit ELSE 0 END +
        CASE WHEN p.status = 'funded' THEN p.total_deposit ELSE 0 END) AS total_deposit
FROM 
    users_customuser u
JOIN 
    savings_savingsaccount s ON s.user_id = u.id AND s.status = 'funded'
JOIN 
    plans_plan p ON p.user_id = u.id AND p.status = 'funded'
GROUP BY 
    u.id, u.full_name
ORDER BY 
    total_deposit DESC;

    

Question 2
calculates the average number of transactions per customer per month and segments them as: High Frequency: ≥10 transactions/month, Medium Frequency: 3–9 transactions/month and Low Frequency: ≤2 transactions/month. I need a table that records individual transactions.

SELECT * FROM adashi_staging.users_customuser;
WITH MonthlyTransactions AS (
    SELECT
        customer_id,
        YEAR(created_at) AS txn_year,
        MONTH(created_at) AS txn_month,
        COUNT(*) AS monthly_txn_count
    FROM transactions_transaction
    GROUP BY customer_id, YEAR(created_at), MONTH(created_at)
),
AverageTransactions AS (
    SELECT
        customer_id,
        AVG(monthly_txn_count) AS avg_txn_per_month
    FROM MonthlyTransactions
    GROUP BY customer_id
)
SELECT
    customer_id,
    ROUND(avg_txn_per_month, 2) AS avg_txn_per_month,
    CASE
        WHEN avg_txn_per_month >= 10 THEN 'High Frequency'
        WHEN avg_txn_per_month BETWEEN 3 AND 9 THEN 'Medium Frequency'
        ELSE 'Low Frequency'
    END AS frequency_category
FROM AverageTransactions
ORDER BY avg_txn_per_month DESC;



Question 3
To flag active accounts (from either plans_plan or savings_savingsaccount) that have not had any inflow transactions in the last 1 year (365 days), you'll need; A transactions table that logs all inflows (e.g., transactions_transaction), with:account_id or plan_id,account_type (if applicable to distinguish savings/investments), created_at (timestamp) and amount (should be > 0 for inflows)

SELECT * FROM adashi_staging.users_customuser;
-- Savings accounts with no inflow in the last 365 days
SELECT s.id AS account_id, 'savings' AS account_type
FROM savings_savingsaccount s
WHERE s.is_active = 1
  AND NOT EXISTS (
    SELECT 1
    FROM transactions_transaction t
    WHERE t.related_account_id = s.id
      AND t.related_account_type = 'savings'
      AND t.amount > 0
      AND t.created_at >= DATEADD(DAY, -365, GETDATE())
  )

UNION

-- Investment plans with no inflow in the last 365 days
SELECT p.id AS account_id, 'plan' AS account_type
FROM plans_plan p
WHERE p.is_active = 1
  AND NOT EXISTS (
    SELECT 1
    FROM transactions_transaction t
    WHERE t.related_account_id = p.id
      AND t.related_account_type = 'plan'
      AND t.amount > 0
      AND t.created_at >= DATEADD(DAY, -365, GETDATE())
  );



Question 4
To calculate the Customer Lifetime Value (CLV) based on tenure and transaction volume, here's what I needed
I have a transactions table with user_id, amount, and created_at, The user's signup date is in users_customuser.date_joined, Profit per transaction is 0.1% of the transaction value. Since we're calculating tenure as months between date_joined and now;

SELECT * FROM adashi_staging.users_customuser;
WITH transaction_summary AS (
    SELECT 
        t.user_id,
        COUNT(*) AS total_transactions,
        SUM(t.amount) AS total_transaction_value,
        AVG(t.amount) AS avg_transaction_value
    FROM transactions_transaction t
    GROUP BY t.user_id
),
tenure_data AS (
    SELECT 
        u.id AS user_id,
        DATEDIFF(MONTH, u.date_joined, GETDATE()) AS tenure_months
    FROM users_customuser u
),
clv_estimation AS (
    SELECT 
        u.user_id,
        t.total_transactions,
        t.total_transaction_value,
        td.tenure_months,
        -- avg_profit_per_transaction = 0.1% of avg amount
        (t.avg_transaction_value * 0.001) AS avg_profit_per_transaction,
        -- CLV formula
        ((CAST(t.total_transactions AS FLOAT) / NULLIF(td.tenure_months, 0)) * 12 * (t.avg_transaction_value * 0.001)) AS estimated_clv
    FROM transaction_summary t
    JOIN tenure_data td ON t.user_id = td.user_id
    JOIN users_customuser u ON u.id = t.user_id
)
SELECT *
FROM clv_estimation
ORDER BY estimated_clv DESC;
