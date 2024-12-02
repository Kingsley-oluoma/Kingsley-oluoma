# Project Description: SQL Data Analysis for Business Insights

/*This project focuses on analyzing and deriving business 
insights from a dataset that includes various aspects of a music store’s sales, 
customer behavior, and playlist interactions. 
The goal was to use SQL to explore and extract meaningful patterns from transactional and metadata information,
helping inform business decisions.*/

## Chinook is an online digital media store.

/*
## Question #1:
Write a solution to find the employee_id of managers with at least 2 direct reports.


Expected column names: employee_id

*/

-- q1 solution:
WITH num_reports as 
(select
  reports_to as reports,
  COUNT (reports_to)
FROM employee
GROUP BY 
  reports_to
HAVING COUNT (reports_to)>=2)
SELECT
  employee_id
FROM employee
WHERE 
  employee_id in (select reports from num_reports);




# Question #2: 
/* Calculate total revenue for MPEG-4 video files purchased in 2024.

Expected column names: total_revenue

*/
-- q2 solution:
SELECT 
    SUM(ii.unit_price * ii.quantity) AS total_revenue
FROM 
    invoice_line ii
LEFT JOIN 
    invoice i ON i.invoice_id = ii.invoice_id
LEFT JOIN 
    track t ON t.track_id = ii.track_id
LEFT JOIN 
    media_type m ON m.media_type_id = t.media_type_id
WHERE 
    EXTRACT(YEAR FROM i.invoice_date) = 2024
    AND t.media_type_id = 3;  -- Query to calculate total revenue for media type 3 in 2024



## Question #3: 
/* For composers appearing in classical playlists, count the number of distinct playlists they appear on and 
create a comma separated list of the corresponding (distinct) playlist names.

Expected column names: composer, distinct_playlists, list_of_playlists

*/   

-- Q3 Solution:
WITH classic AS (
    SELECT 
        DISTINCT t.composer,
        COUNT(p.name) OVER (PARTITION BY t.composer) AS distinct_playlist,
        p.name,
        p.playlist_id,
        ROW_NUMBER() OVER (PARTITION BY t.composer) AS rn
    FROM 
        track t
    JOIN 
        playlist_track pt ON pt.track_id = t.track_id
    JOIN 
        playlist p ON p.playlist_id = pt.playlist_id
    WHERE 
        p.name LIKE '%Classical%' 
        AND t.composer IS NOT NULL 
    ORDER BY 
        t.composer
),

first AS (
    SELECT
        *
    FROM 
        classic
    WHERE 
        rn = 1
),

second AS (
    SELECT
        *
    FROM 
        classic
    WHERE 
        rn <> 1
),

final AS (
    SELECT
        DISTINCT f.composer,
        f.distinct_playlist,
        CONCAT(f.name, ',', s.name) AS list_of_playlist,
        DENSE_RANK() OVER (PARTITION BY f.composer ORDER BY CONCAT(f.name, ',', s.name) DESC) AS rn
    FROM 
        first f
    JOIN 
        second s ON s.composer = f.composer
)

SELECT
    composer,
    distinct_playlist,
    list_of_playlist
FROM 
    final
WHERE 
    rn = 1;  -- Query to get the first combination of playlists per composer


## Question #4: 
/*Find customers whose yearly total spending is strictly increasing*.

Expected column names: customer_id
*/
-- Q4 Solution:
WITH initial AS (
    SELECT
        i.customer_id,
        EXTRACT(YEAR FROM i.invoice_date::date) AS invoice_year,
        i.total
    FROM 
        invoice i
    JOIN 
        invoice_line ii ON ii.invoice_id = i.invoice_id
    WHERE 
        EXTRACT(YEAR FROM i.invoice_date::date) <> '2025'
    GROUP BY 
        i.customer_id, i.invoice_date, i.total
    ORDER BY 
        i.customer_id, invoice_year
),

merged AS (
    SELECT
        customer_id,
        invoice_year AS invoice_date,
        SUM(total) AS amount_spent,
        COALESCE(LEAD(SUM(total), 1) OVER (PARTITION BY customer_id ORDER BY invoice_year), 0) AS forward
    FROM 
        initial
    GROUP BY 
        customer_id, invoice_year
),

final AS (
    SELECT 
        customer_id,
        invoice_date,
        amount_spent,
        CASE 
            WHEN forward > amount_spent THEN 0 
            ELSE 1 
        END AS status,
        forward
    FROM 
        merged
    WHERE 
        forward <> 0
)

SELECT 
    DISTINCT customer_id
FROM 
    final
GROUP BY 
    customer_id
HAVING 
    SUM(status) = 0;  -- Query to find customers with no increase in spending in subsequent years    
