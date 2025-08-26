# 🚀 RFM Customer Segmentation Project with SQL & Power BI  

## 📌 Project Overview  
This project demonstrates **end-to-end customer segmentation** using the **RFM (Recency, Frequency, Monetary) model**.  
The goal is to analyze customer purchase behavior, identify key customer groups (like **Champions, Loyal Customers, At Risk, etc.**) and visualize insights in **Power BI** for better business decisions.  

---

## 🛠 Tools & Technologies  
- **Python** – Bulk insert sales data into SQL database  
- **MySQL (SQL)** – Data preprocessing & RFM calculation  
- **Power BI** – Interactive dashboards & visualizations  

---

## 📂 Workflow  

1️⃣ **Data Insertion with Python**  
- Bulk inserted sample sales dataset into MySQL database.  

2️⃣ **SQL RFM Segmentation**  
- Converted `ORDERDATE` into proper date format.  
- Calculated **Recency, Frequency, and Monetary values** for each customer.  
- Applied `NTILE(5)` window functions to generate **R, F, M scores (1–5 scale)**.  
- Combined into **RFM Combination Codes** (e.g., 555, 454, 332).  
- Segmented customers into groups:  
  - Champions  
  - Loyal Customers  
  - Promising Customers  
  - About to Sleep  
  - At Risk  
  - Other  

🔑 **Sample SQL Code Snippet**  

```sql
USE RFM_SALES;

SELECT * FROM SAMPLE_SALES_DATA;

-- DROP TABLE SAMPLE_SALES_DATA;
SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SAMPLE_SALES_DATA; -- 2005-05-31 [LAST BUSINESS DAY] (Convereverrted and max date)
SELECT MIN(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SAMPLE_SALES_DATA; -- 2003-01-06 (Convereverrted and min date)

SELECT
	CUSTOMERNAME,
    ROUND(SUM(SALES),0) AS CLV, -- Customer Live Value(CLV)
    COUNT(DISTINCT ORDERNUMBER) AS FREQUENCY, -- How many time customer buy product/Total Ordered
    SUM(QUANTITYORDERED) AS TOTAL_QTY_ORDERED, 
    MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) AS CUSTOMER_LAST_TRANSACTION_DATE, -- [LAST BUSINESS DAY] (Convereverrted and max date)
    DATEDIFF((SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SAMPLE_SALES_DATA), MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y'))) AS CUSTOMER_RECENCY 
    --  Datediff(Whole dateset max-percustomer date)
FROM SAMPLE_SALES_DATA
GROUP BY CUSTOMERNAME;


-- RFM SEGMENTATION
--  CREATE VIEW RFM_SEGMENTATION_DATA AS
WITH CLV AS 
(SELECT
	CUSTOMERNAME,
	MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) AS CUSTOMER_LAST_TRANSACTION_DATE, 
    DATEDIFF((SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SAMPLE_SALES_DATA), MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y'))) AS RECENCY_VALUE, -- when is the latest purchase date.
    COUNT(DISTINCT ORDERNUMBER) AS FREQUENCY_VALUE, -- how frequently do they make purchases.
    SUM(QUANTITYORDERED) AS TOTAL_QTY_ORDERED,
    ROUND(SUM(SALES),0) AS MONETARY_VALUE --  Total spending /Customer Live Value(CLV)
FROM SAMPLE_SALES_DATA
GROUP BY CUSTOMERNAME),

RFM_SCORE AS
(SELECT 
	C.*, -- CLV table used above CTE 
    NTILE(5) OVER(ORDER BY RECENCY_VALUE DESC) AS R_SCORE,
    NTILE(5) OVER(ORDER BY FREQUENCY_VALUE ASC) AS F_SCORE,
    NTILE(5) OVER(ORDER BY MONETARY_VALUE ASC) AS M_SCORE
FROM CLV AS C),

RFM_COMBINATION AS
(SELECT
	R.*,
    R_SCORE + F_SCORE + M_SCORE AS TOTAL_RFM_SCORE,
    CONCAT_WS('', R_SCORE, F_SCORE, M_SCORE) AS RFM_COMBINATION
FROM RFM_SCORE AS R)

SELECT
	RC.*,
    CASE
		WHEN RFM_COMBINATION IN (555, 554, 545) THEN "Champions"
        WHEN RFM_COMBINATION IN (355, 454, 455, 344, 445) THEN 'Loyal Customers'
        WHEN RFM_COMBINATION IN (433, 434, 332, 343) THEN 'Potential Loyalists'
        WHEN RFM_COMBINATION IN (511, 411, 311) THEN 'New Customers'
        WHEN RFM_COMBINATION IN (332, 333, 323) THEN 'Promising Customers'
        WHEN RFM_COMBINATION IN (212, 213, 221, 222) THEN 'Needs Attention'
        WHEN RFM_COMBINATION IN (112, 113, 114) THEN 'About to Sleep'
        WHEN RFM_COMBINATION IN (511, 512, 513) THEN 'Cannot lose them'
        WHEN RFM_COMBINATION IN (212, 213, 221, 222) THEN 'At risk'
        WHEN RFM_COMBINATION IN (111, 112) THEN 'Hibernating customers'
        ELSE "Other"
        END AS CUSTOMER_SEGMENT
FROM RFM_COMBINATION RC;




SELECT
	CUSTOMER_SEGMENT,
    SUM(MONETARY_VALUE) AS TOTAL_SPENDING,
    ROUND(AVG(MONETARY_VALUE),0) AS AVERAGE_SPENDING,
    SUM(FREQUENCY_VALUE) AS TOTAL_ORDER,
    SUM(TOTAL_QTY_ORDERED) AS TOTAL_QTY_ORDERED
FROM RFM_SEGMENTATION_DATA
GROUP BY CUSTOMER_SEGMENT;

SELECT * FROM RFM_SEGMENTATION_DATA;  -- used view (vartual table
SELECT count(distinct(customername))  FROM RFM_SEGMENTATION_DATA
Where Customer_Segment='Other'
;



