# Bicycle-Store-SQL-Data-Cleaning-and-EDA

This readme file provides a step-by-step description of a data cleaning and analysis project for a bicycle shop dataset.

## 1. Downloading [the dataset from Kaggle](https://www.kaggle.com/datasets/rohitsahoo/bicycle-store-dataset)

I chose this particular dataset because it contains a large amount of data, which would make its analysis meaningful.

## 2. Preparation for data import into PostgreSQL

While viewing the data on Kaggle, I noticed that some files contain a lot of empty columns that are marked as containing data. Also, the "Customer Demographic.csv" file has an extra 'Default' column with some unclear data of various formats, so it should not be imported into PostgreSQL either. Using Microsoft Excel, I opened these files and removed the unnecessary columns.

## 3. Creating a database and tables in PostgreSQL
### Creating a database
```
CREATE DATABASE Bicycle_Store;
```
### Creating tables

I reviewed the files, and noticed that the "Customer List.csv" file contains only data that is already present in the "Customer Demographic.csv" and "Customer Address.csv" tables, so importing it makes no sense. Therefore, I will only import 3 files and create a table for each of them accordingly.

### Customer Demographic.csv
```
CREATE TABLE customer_demographic (
    customer_id INTEGER PRIMARY KEY,
    first_name TEXT,
    last_name TEXT,
    gender TEXT,
    past_3_years_bike_related_purchases INTEGER,
    DOB DATE,
    job_title TEXT,
    job_industry_category TEXT,
    wealth_segment TEXT,
    deceased_indicator TEXT,
    owns_car TEXT,
    tenure INTEGER
);
```
### Customer Address.csv

In this table, I created an additional *address_id* column with auto-increment, which will serve as the Primary Key for this table. This is because using *customer_id* as a Primary Key in two tables would violate normalization rules and complicate working with them.
```
CREATE TABLE customer_address (
	address_id SERIAL PRIMARY KEY,
    	customer_id INTEGER,
    	FOREIGN KEY (customer_id) REFERENCES customer_demographic(customer_id),
    	address TEXT,
    	postcode INTEGER,
    	state TEXT,
    	country TEXT,
    	property_valuation INTEGER
);
```
### Transactions.csv
```
CREATE TABLE Transactions (
    	transaction_id INTEGER PRIMARY KEY,
    	product_id INTEGER,
    	customer_id INTEGER,
	FOREIGN KEY (customer_id) REFERENCES customer_demographic(customer_id),
	transaction_date DATE,
	online_order BOOL,
	order_status TEXT,
	brand TEXT,
	product_line TEXT,
	product_class TEXT,
	product_size TEXT,
	standart_cost TEXT,
	product_first_sold_date DATE
);
```
## 4. Filling the tables

I use the built-in Postgres "Import/Export data" tool for importing data. I specify the path to the file from which I am importing the data for each table. I indicate that they contain a header row, and select the columns to be imported. After that, I select the first 10 rows of each table to check if the data is present.

### Customer Demographic.csv
```
SELECT * FROM customer_demographic
LIMIT 10
```
![image](https://github.com/user-attachments/assets/0e85adde-34a1-4bf7-b58b-46f6e3d3c680)

### Customer Demographic.csv
```
SELECT * FROM customer_address
LIMIT 10
```
![image](https://github.com/user-attachments/assets/162f2255-c47c-40b8-ab9b-a0c85de94515)

### Transactions.csv
```
SELECT * FROM transactions
LIMIT 10
```
![image](https://github.com/user-attachments/assets/8c1dbd19-9cad-4764-a8aa-f71606bcd436)

## 5. Cleaning the tables
### Preparation for cleaning

To clean the tables and make them looking good, at first it's necessary to create identical backup tables. This is needed to prevent any accidental loss of important data during the process of deleting unnecessary information. I will demonstrate this using the *customer_demographic* table as an example.

```
CREATE TABLE demographic_staging
(LIKE customer_demographic);

INSERT INTO demographic_staging
SELECT *
FROM customer_demographic;

SELECT *
FROM demographic_staging
```
![image](https://github.com/user-attachments/assets/d3db37f2-78e7-4c27-83aa-04573bc53504)

The tables are identical. Next, I do the same for the *customer_address* and *transactions* tables.

```
CREATE TABLE address_staging
(LIKE customer_address);

INSERT INTO address_staging
SELECT *
FROM customer_address;

CREATE TABLE transactions_staging
(LIKE transactions);

INSERT INTO transactions_staging
SELECT *
FROM transactions;
```
### Deleting duplicates

To detect duplicates in tables, I use the ROW_NUMBER() function, which generates a unique number for a row within a specific group. I define these groups by using PARTITION BY.

Also, to display only those rows where the row number is greater than 1 (precisely those that are duplicates), I use a CTE (Common Table Expression), which allows using WHERE for a temporary column that shows the row number.

demographic_staging table
```
with duplicate_cte as
(
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY first_name, last_name, dob) as row_num
FROM demographic_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;
```
![image](https://github.com/user-attachments/assets/f2aad0b6-a7f1-455b-bc8e-33c338f5d14f)

There are no rows, so there are no duplicates in this table.
The situation is similar in the *address_staging* table, but in the *transactions_staging* table the situation is more interesting.

transactions_staging table
```
with duplicate_cte as
(
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY product_id, customer_id, transaction_date, online_order, order_status) as row_num
FROM transactions_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;
```
![image](https://github.com/user-attachments/assets/7b0da640-4e1d-4b93-a5a2-afdc253f74fa)

There is one row, meaning there is 1 duplicate in the table. To confirm this I decided to see these two records.

```
SELECT *
FROM transactions_staging
WHERE product_id = 0 and customer_id = 1840 and transaction_date='2017-07-11'
```
![image](https://github.com/user-attachments/assets/bf3f4e42-0761-4d16-b916-995fea174001)

Here I noticed that although the *product_id* is identical, the *brand*, *product_size*, and *standard_cost* columns differ. Therefore, I decided to look at more records with this *product_id*.

```
SELECT product_id, brand, product_line, product_class, product_size, standart_cost
FROM transactions_staging
WHERE product_id = 0;
```
![image](https://github.com/user-attachments/assets/fb070cc4-5570-4cac-aec6-3d94e73c8445)

It is noticeable that completely different values are encountered in all columns describing the product. From this, we can understand that the product_id column is not informative and needs to be replaced, and this will be done in the following steps.

### Data Standardization
#### 1) demographic_staging table
##### a) gender column
```
SELECT DISTINCT gender
FROM demographic_staging
```
![image](https://github.com/user-attachments/assets/51aaa9f4-00f9-41dc-8a5e-f09a48942111)

It needs to be changed so that there are only Female, Male and Unknown.

```
UPDATE demographic_staging
SET gender = 
    CASE
        WHEN gender IN ('Femal', 'F') THEN 'Female'
        WHEN gender = 'M' THEN 'Male'
        WHEN gender = 'U' THEN 'Unknown'
	ELSE gender
    END;
```
![image](https://github.com/user-attachments/assets/b93d8e7e-86cd-465d-990c-2e86e3d87988)

I decided to see how many customers have their gender listed as 'Unknown'.

```
SELECT * FROM demographic_staging
WHERE gender = 'Unknown';
```
![image](https://github.com/user-attachments/assets/38b5fcf5-b5b6-4da3-a203-6e11110657a7)

We see that the date of birth is not specified for everyone, except for one customer whose date of birth is not realistic, so it should be cleaned.

```
UPDATE demographic_staging
SET dob = NULL
WHERE dob = '1843-12-21';
```
![image](https://github.com/user-attachments/assets/15beda61-cb86-4874-9513-1ce23802b70a)

##### b) job_title column
```
SELECT DISTINCT job_title
FROM demographic_staging
ORDER BY job_title;
```
![image](https://github.com/user-attachments/assets/80e4b047-a3fc-455e-b2ca-97b7b5e1e962)

Some professions end with the Roman numerals. Let's remove them to merge identical professions.

```
UPDATE demographic_staging
SET job_title = regexp_replace(job_title, ' (I|II|III|IV)$', '')
WHERE job_title ~ ' (I|II|III|IV)$';
```
![image](https://github.com/user-attachments/assets/e3a75678-dc56-4a58-a2af-63b39b54c220)

##### c) deceased_indicator and owns_car columns
```
SELECT DISTINCT deceased_indicator, owns_car
FROM demographic_staging;
```
![image](https://github.com/user-attachments/assets/8d9b5f27-415d-42d9-9c48-cc1ef778138f)

Let's change the data type of these columns to boolean, after standardizing the values.

```
UPDATE demographic_staging
SET deceased_indicator = CASE
    WHEN deceased_indicator = 'Y' THEN 'TRUE'
    WHEN deceased_indicator = 'N' THEN 'FALSE'
END,
	owns_car = CASE
    WHEN owns_car = 'Yes' THEN 'TRUE'
    WHEN owns_car = 'No' THEN 'FALSE'
END;

ALTER TABLE demographic_staging
ALTER COLUMN deceased_indicator TYPE BOOLEAN USING deceased_indicator::BOOLEAN,
ALTER COLUMN owns_car TYPE BOOLEAN USING owns_car::BOOLEAN;
```
![image](https://github.com/user-attachments/assets/5e300fc0-aa1e-4ead-b356-09a91baf6db0)

#### 2) address_staging table
##### a) state column
```
SELECT DISTINCT state
FROM address_staging;
```
![image](https://github.com/user-attachments/assets/1315d579-c2a6-4bc8-bc01-fba23117380b)

Not all data is in the same format.
```
UPDATE address_staging
SET state = CASE
	WHEN state = 'Victoria' THEN 'VIC'
	WHEN state = 'New South Wales' THEN 'NSW'
	ELSE state
END;
```
![image](https://github.com/user-attachments/assets/5ec466bc-a1c1-4ba1-9014-03e08f49d28d)

#### 3) transactions_staging table

Before standardizing the table data, it is necessary to resolve the normalization issue. There are *transaction_id* and *product_id* columns in the table. Columns describing transaction characteristics depend on *transaction_id*, while those describing product characteristics depend on *product_id*. This violates the conditions of the 2nd normal form, so another table needs to be created where only products will be stored.

##### a) table normalisation

Also, the problem of the product_id column being uninformative was mentioned earlier, so it needs to be resolved as well. Let's add another column that will be populated with a hash using the built-in **MD5 hashing algorithm**. It will be unique for each unique combination of the *brand*, *product_line*, *product_class*, *product_size*, and *standard_cost* columns.

```
ALTER TABLE transactions_staging ADD COLUMN product_hash_id TEXT;

UPDATE transactions_staging
SET product_hash_id = md5(brand || product_line || product_class || product_size || standart_cost);

SELECT *
FROM transactions_staging
order by product_hash_id;
```
![image](https://github.com/user-attachments/assets/b957b71d-9abe-4438-b496-76976b980684)

I am not considering the *product_first_sold_date* column because it can differ even when all other characteristics are identical. Therefore, this column will be deleted later.

Next, let's create a *products* table and populate it with product values using the unique hash from the previous table.

```
CREATE TABLE products (
	product_id SERIAL PRIMARY KEY,
	product_hash_id TEXT,
	brand TEXT,
	product_line TEXT,
	product_class TEXT,
	product_size TEXT,
	standart_cost TEXT
);

INSERT INTO products (product_hash_id, brand, product_line, product_class, product_size, standart_cost)
SELECT DISTINCT product_hash_id, brand, product_line, product_class, product_size, standart_cost
FROM transactions_staging
ORDER BY product_hash_id;

SELECT * FROM products
ORDER BY product_id;
```
![image](https://github.com/user-attachments/assets/a3bbfeea-3811-49c7-abe0-94765de5bb56)

After this, the uninformative *product_id* in the transactions_staging table has to be replaced with the newly created *product_id* from *products*.

```
UPDATE transactions_staging
SET product_id = products.product_id
FROM products
WHERE transactions_staging.product_hash_id = products.product_hash_id;

SELECT *
FROM transactions_staging
WHERE product_id > 0
ORDER BY product_id;
```
![image](https://github.com/user-attachments/assets/ac1c4654-9a63-4f68-a20b-ac9ac679b6a7)

There are many transactions with no available information about the product. Such data is not informative, so it has to be deleted.

```
DELETE FROM transactions_staging
WHERE product_id = 0;
```

Now, let's delete the unnecessary columns.

```
ALTER TABLE transactions_staging
DROP COLUMN product_hash_id,
DROP COLUMN brand,
DROP COLUMN product_line,
DROP COLUMN product_class,
DROP COLUMN product_size,
DROP COLUMN standart_cost,
DROP COLUMN product_first_sold_date;

SELECT *
FROM transactions_staging
order by product_id;
```
![image](https://github.com/user-attachments/assets/8eb07ab0-25e2-4ef5-b323-55148d8fc783)

##### b) order_status column

There are only two unique values, so let's change the type to boolean.

```
UPDATE transactions_staging
SET order_status = CASE
    WHEN order_status = 'Approved' THEN 'TRUE'
    WHEN order_status = 'Cancelled' THEN 'FALSE'
END;

ALTER TABLE transactions_staging
RENAME COLUMN order_status TO approved_order_status;

ALTER TABLE transactions_staging
ALTER COLUMN approved_order_status TYPE BOOLEAN USING approved_order_status::BOOLEAN;

SELECT DISTINCT approved_order_status
FROM transactions_staging;
```
![image](https://github.com/user-attachments/assets/188af8f2-df9f-4db4-a542-74160b430976)

#### 4) products table
##### a) product_hash_id column and empty row
![image](https://github.com/user-attachments/assets/fd7c0ee5-ae0b-4bd8-b513-b6247141eff0)

Let's remove empty row and unnecessary column.

```
DELETE FROM products
WHERE product_hash_id is null;

ALTER TABLE products
DROP COLUMN product_hash_id;
```
![image](https://github.com/user-attachments/assets/7856a051-0807-4810-9a62-6ce5dd3e7bcf)

##### b) standart_cost column

Let's change the data type to numeric, removing the '$' and ',' symbols from the entries.

```
ALTER TABLE products
ALTER COLUMN standart_cost TYPE NUMERIC USING REPLACE(REPLACE(standart_cost, '$', ''), ',', '')::NUMERIC;
```
![image](https://github.com/user-attachments/assets/b4485bba-0315-4d3c-8e26-085e180fdd1e)

## 6. Transferring data from backup tables

Let's transfer all data to the original tables.

### 1) customer_demographic table
```
TRUNCATE TABLE customer_demographic CASCADE;

INSERT INTO customer_demographic
SELECT * FROM demographic_staging;

SELECT * FROM customer_demographic
ORDER BY customer_id;
```
![image](https://github.com/user-attachments/assets/91d631dd-054e-4a3e-91e3-8c58f1069ba8)
### 2) customer_address table
```
TRUNCATE TABLE customer_address;

INSERT INTO customer_address
SELECT * FROM address_staging;

SELECT * FROM customer_address
ORDER BY address_id;
```
![image](https://github.com/user-attachments/assets/d7b3b314-48c2-48d5-93e4-8a95a9c831e2)
### 3) transactions table
```
TRUNCATE TABLE transactions;

ALTER TABLE transactions
DROP COLUMN brand,
DROP COLUMN product_line,
DROP COLUMN product_class,
DROP COLUMN product_size,
DROP COLUMN standart_cost,
DROP COLUMN product_first_sold_date;

ALTER TABLE transactions
RENAME COLUMN order_status TO approved_order_status;

ALTER TABLE transactions
ALTER COLUMN approved_order_status TYPE BOOLEAN USING approved_order_status::BOOLEAN;

INSERT INTO transactions (transaction_id, product_id, customer_id, transaction_date, online_order, approved_order_status)
SELECT transaction_id, product_id, customer_id, transaction_date, online_order, approved_order_status FROM transactions_staging;

SELECT * FROM transactions
ORDER BY transaction_id;
```
![image](https://github.com/user-attachments/assets/1df1be14-74b5-452c-9457-c7e11251dcbb)

The only thing left is to make *product_id* a foreign key that references the products table.

```
ALTER TABLE transactions
ADD CONSTRAINT transactions_product_id_fkey
FOREIGN KEY (product_id)
REFERENCES products(product_id);
```
![image](https://github.com/user-attachments/assets/34e4de19-e077-475c-b3ee-74d208493d78)

## 7. Visualization using Power BI
The data from PostgreSQL was imported into Power BI and the [visualization](https://github.com/maxymfarenyk/Bicycle-Store-SQL-Data-Cleaning-and-EDA/blob/main/Bicycle_Store.pbix) was created.

![image](https://github.com/user-attachments/assets/97e23aae-de61-403d-8e86-0324a5d95309)

## 8. Data Analysis

The data analysis revealed specific patterns that, if considered, could lead to increased sales.

### 1) Main Customers
#### a) Age
![image](https://github.com/user-attachments/assets/0f90500a-c4d1-4e9a-8bb9-b5e0be82a0f1)
![image](https://github.com/user-attachments/assets/93ee468e-3452-4b65-aa44-f441dfdf2162) 

Most purchases were made by customers born between 1974 and 1980, which, considering the dataset is from 2017, means these are people aged 37 to 43.

#### b) Job Industry
![image](https://github.com/user-attachments/assets/9aa85975-378f-4420-91a3-90d293c46276)

The most frequent customers are people working in the manufacturing, finance, and healthcare sectors.

#### c) Wealth Segment

![image](https://github.com/user-attachments/assets/7ea3dfd9-1dfc-451b-9ef7-d87c30dc9b83)

Half of all purchases were made by mass-market customers. However, each of the other segments accounted for approximately a quarter of the total purchases.

#### d) Location 
![image](https://github.com/user-attachments/assets/6ecb54c9-045e-41de-8592-f2ab2ea81d55)

Out of 19.8 thousand purchases, more than 10 thousand were made in the state of New South Wales.

#### Conclusion

Therefore, the business should focus its marketing efforts on these customer groups, as they make the majority of purchases.

### 2) Seasonality and trends
![image](https://github.com/user-attachments/assets/5cacf807-c413-40be-83a7-458ad3064700)

At the beginning of the year (from January to February), there is a slight decline in sales. This is followed by a gradual increase throughout the year until October, except for sharp drops in June and September. After that, sales decrease again in November and December. In June, sales fell by 6.1% compared to May, and in September, they dropped by 10.1% compared to August.

#### a) Sales decline in June

In this case, the sales decline could be related to the end of the financial year (which ends on June 30 in Australia), as manufacturing companies may want to use their budgets before the year ends. When filtering this chart by purchases from people working in the manufacturing sector, we observe a sharp decline specifically in this month.

![image](https://github.com/user-attachments/assets/bd3edf79-b6c0-4511-b96b-285535499302)

A decrease in purchases in June is also noticeable among people working in the healthcare sector, which could be related to certain epidemics, such as the flu, that tend to peak during the winter months (June to August in Australia).

![image](https://github.com/user-attachments/assets/be7f0396-eaef-4163-a057-1c536c9cbe33)

#### b) Sales decline in September

Among people working in the healthcare sector, the decrease in purchases may be due to the arrival of spring and the corresponding increase in the number of people suffering from seasonal allergies. Additionally, during this period, there is a noticeable sharp decline in purchases among people working in the financial sector with no clear reasons for the decrease in sales within this group.

![image](https://github.com/user-attachments/assets/a99974c4-6525-4a82-93f6-f43b2a0cb9d3)

#### Conclusion

To boost purchases during low-revenue months a store could offer some exclusive discounts and/or promotions available only during these specific periods.

### 3) Online purchases

![image](https://github.com/user-attachments/assets/7a0efca6-ea85-4269-855b-966019d1c34e)

It is clear that a very small amount of purchases are made offline. This could indicate that reducing the number of physical stores would be a strategic move, allowing the business to cut costs on rent and employee salaries. Also, this could mean that there are already very few physical stores, and that customers find it inconvenient to access them, which is the reason for the low number of offline purchases.




