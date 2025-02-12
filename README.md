# Bicycle-Store-SQL-Data-Cleaning-and-EDA
У цьому файлі представлено покроковий опис виконання проєкту з дата клінінгу та аналізу датасету магазина велосипедів
## 1. Завантаження [датасету із Kaggle](https://www.kaggle.com/datasets/rohitsahoo/bicycle-store-dataset)
Я обрав саме цей датасет через те, що він містить велику кількість даних, завдяки чому його аналіз буде мати сенс.
## 2. Підготовка до імпорту у PostgreSQL
При перегляді даних на Kaggle я помітив, що у деяких файлах є дуже багато порожніх стовпців, які позначені як ті, що містять дані. Також у файлі "Customer Demographic.csv" є зайвий стовпець Default з якимись незрозумілими даними різного вигляду, тому його теж не варто імпортувати в PostgreSQL. Використовуючи Microsoft Excel я відкрив ці файли та прибрав зайві стовпці.
## 3. Створення бази даних та таблиць у PostgreSQL
### Створення бази даних
```
CREATE DATABASE Bicycle_Store;
```
### Створення таблиць
Переглянувши файли я звернув увагу, що файл "Customer List.csv" містить лише дані, що вже є в таблицях "Customer Demographic.csv" та "Customer Address.csv", тому імпортувати його немає сенсу. Тому імпортуватиму лише 3 файли і відповідно створю таблицю для кожного з них.
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
В цій таблиці я створив додатковий стовпець address_id з автоінкрементом, який буде виступати Primary Key для цієї таблиці, оскільки використовування customer_id як Primary Key у двох таблицях було б порушенням нормалізації та заплутувало б при роботі з ними.
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
## 4. Заповнення таблиць
Щоб імпортувати дані я використовую вбудований у Postgres інструмент "Import/Export data", для кожної таблиці вказую шлях файлу з якого імпортую дані, вказую що вони містять заголовковий рядок та обираю стовпці які потрібно імпортувати. Після цього виводжу перші 10 рядків кожної таблиці щоб перевірити чи дані присутні.
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
## 5. Очищення таблиць
### Підготовка до очищення таблиць
Щоб очистити таблиці та привести їх до гарного вигляду потрібно спершу створити ідентичні запасні таблиці. Це потрібно для того щоб випадково не втратити якісь важливі дані в процесі видалення непотрібних. Продемонструю це на прикладі таблиці customer_demographic
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
Таблиці ідентичні. Далі роблю те саме для таблиць customer_address та transactions.
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
### Видалення дублікатів
Для того щоб виявити дублікати у таблицях я використовую функцію ROW_NUMBER(), яка генерує унікальний номер для рядка в межах певної групи. Ці групи я визначаю через PARTITION BY.

Також щоб виводились лише ті рядки, де номер рядка більше 1 (тобто якраз ті, що є дублікатами) використовую CTE (Common Table Expression), який дозволить використовувати WHERE для тимчасово створеного стовпця що показує номер рядка.

Таблиця demographic_staging
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

Рядків немає, отже дублікатів у цій таблиці немає. 
У таблиці address_staging ситуація аналогічна, а от у таблиці transactions_staging ситуація цікавіша
Таблиця transactions_staging
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

Є один рядок, тобто в таблиці є 1 дублікат. Щоб пересвідчитись я вирішив побачити ці два записи
```
SELECT *
FROM transactions_staging
WHERE product_id = 0 and customer_id = 1840 and transaction_date='2017-07-11'
```
![image](https://github.com/user-attachments/assets/bf3f4e42-0761-4d16-b916-995fea174001)

Тут я помітив, що хоч product_id ідентичний, але ствопці brand, product_size та standart_cost відрізняються.
Тому я вирішив подивитись більше записів із цим product_id
```
SELECT product_id, brand, product_line, product_class, product_size, standart_cost
FROM transactions_staging
WHERE product_id = 0;
```
![image](https://github.com/user-attachments/assets/fb070cc4-5570-4cac-aec6-3d94e73c8445)

Помітно, що у всіх стовпцях, які описують товар зустрічають зовсім різні значення. З цього ми можемо зрозуміти, що стовпець product_id не є інформативним і є необхідність його замінити, але це буде зроблено у наступних кроках

### Стандартизація даних
#### 1) Таблиця demographic_staging
##### а) стовпець gender
```
SELECT DISTINCT gender
FROM demographic_staging
```
![image](https://github.com/user-attachments/assets/51aaa9f4-00f9-41dc-8a5e-f09a48942111)

Потрібно зробити щоб було лише Female, Male, Unknown

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

Я вирішив подивитись у скількох покупців стать вказана як 'Unknown'
```
SELECT * FROM demographic_staging
WHERE gender = 'Unknown';
```
![image](https://github.com/user-attachments/assets/38b5fcf5-b5b6-4da3-a203-6e11110657a7)

Бачимо, що у всіх дата народження не вказана, окрім одного покупця, у котрого дата народження не є реалістичною, тому її варто очистити.
```
UPDATE demographic_staging
SET dob = NULL
WHERE dob = '1843-12-21';
```
![image](https://github.com/user-attachments/assets/15beda61-cb86-4874-9513-1ce23802b70a)

##### б) стовпець job_title
```
SELECT DISTINCT job_title
FROM demographic_staging
ORDER BY job_title;
```
![image](https://github.com/user-attachments/assets/80e4b047-a3fc-455e-b2ca-97b7b5e1e962)

Помітно що деякі професії закінчуються римськими цифрами. Приберемо їх щоб об'єднати однакові професії

```
UPDATE demographic_staging
SET job_title = regexp_replace(job_title, ' (I|II|III|IV)$', '')
WHERE job_title ~ ' (I|II|III|IV)$';
```
![image](https://github.com/user-attachments/assets/e3a75678-dc56-4a58-a2af-63b39b54c220)

##### в) стовпці deceased_indicator, owns_car
```
SELECT DISTINCT deceased_indicator, owns_car
FROM demographic_staging;
```
![image](https://github.com/user-attachments/assets/8d9b5f27-415d-42d9-9c48-cc1ef778138f)

Змінимо тип даних цих стовпців на boolean, попередньо стандартизувавши написання 
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

#### 2) Таблиця address_staging
##### а) стовпець state
```
SELECT DISTINCT state
FROM address_staging;
```
![image](https://github.com/user-attachments/assets/1315d579-c2a6-4bc8-bc01-fba23117380b)

Не всі дані в однаковому форматі
```
UPDATE address_staging
SET state = CASE
	WHEN state = 'Victoria' THEN 'VIC'
	WHEN state = 'New South Wales' THEN 'NSW'
	ELSE state
END;
```
![image](https://github.com/user-attachments/assets/5ec466bc-a1c1-4ba1-9014-03e08f49d28d)

#### 3) Таблиця transactions_staging
Перш ніж стандартизувати дані таблиці необхідно вирішити проблему із нормалізацією. У цій таблиці є стовпці transaction_id та product_id. Від transaction_id залежать стовпці, що описують характеристики транзакції, а від product_id - ті, що описують характеристики товару. Це порушує умови 2 нормальної форми, тому треба створити ще одну таблицю, де будуть зберігатись лише товари. 

##### а) нормалізація таблиці

Також раніше згадувалась проблема неінформативності стовпця product_id, тому треба вирішити і її. Додамо ще один стовпець, який буде заповнюватись хешем за допомогою вбудованого алгоритму хешування md5. Він буде унікальний для кожної унікальної комбінації стовпців brand, product_line, product_class, product_size, standart_cost. 
```
ALTER TABLE transactions_staging ADD COLUMN product_hash_id TEXT;

UPDATE transactions_staging
SET product_hash_id = md5(brand || product_line || product_class || product_size || standart_cost);

SELECT *
FROM transactions_staging
order by product_hash_id;
```
![image](https://github.com/user-attachments/assets/b957b71d-9abe-4438-b496-76976b980684)

Стовпець product_first_sold_date я до уваги не беру, оскільки він буває різний навіть тоді, коли всі інші характеристики ідентичні. Тому цей стовпець в подальшому буде видалений.

Далі створимо таблицю products та заповнимо її значеннями товарів з унікальним хешем із попередньої таблиці
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

Після цього замінимо неінформативний product_id у таблиці transactions_staging на новостворений product_id із products.
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
Можна помітити, що є дуже багато транзакцій, де про товар немає жодної інформації. Такі дані не є інформативними тому їх можна видалити
```
DELETE FROM transactions_staging
WHERE product_id = 0;
```
Після цього видалимо непотрібні стовпці
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

##### б) стовпець order_status

В цьому стовпці лише два унікальні значення, тому змінимо тип на bool
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

#### 4) Таблиця products
##### а) стовпець product_hash_id та рядок null
![image](https://github.com/user-attachments/assets/fd7c0ee5-ae0b-4bd8-b513-b6247141eff0)

Видалимо порожній рядок та непотрібний стовпець
```
DELETE FROM products
WHERE product_hash_id is null;

ALTER TABLE products
DROP COLUMN product_hash_id;
```
![image](https://github.com/user-attachments/assets/7856a051-0807-4810-9a62-6ce5dd3e7bcf)

##### б) стовпець standart_cost
Змінимо тип даних на numeric, прибравши із запису значки '$' та ','
```
ALTER TABLE products
ALTER COLUMN standart_cost TYPE NUMERIC USING REPLACE(REPLACE(standart_cost, '$', ''), ',', '')::NUMERIC;
```
![image](https://github.com/user-attachments/assets/b4485bba-0315-4d3c-8e26-085e180fdd1e)

## 6. Перенесення даних із запасних таблиць у фінальні
Перенесемо всі дані у початкові таблиці
### 1) Таблиця customer_demographic
```
TRUNCATE TABLE customer_demographic CASCADE;

INSERT INTO customer_demographic
SELECT * FROM demographic_staging;

SELECT * FROM customer_demographic
ORDER BY customer_id;
```
![image](https://github.com/user-attachments/assets/91d631dd-054e-4a3e-91e3-8c58f1069ba8)
### 2) Таблиця customer_address
```
TRUNCATE TABLE customer_address;

INSERT INTO customer_address
SELECT * FROM address_staging;

SELECT * FROM customer_address
ORDER BY address_id;
```
![image](https://github.com/user-attachments/assets/d7b3b314-48c2-48d5-93e4-8a95a9c831e2)
### 3) Таблиця transactions
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

Залишається лише зробити product_id foreign_key, що посилатиметься на таблицю products
```
ALTER TABLE transactions
ADD CONSTRAINT transactions_product_id_fkey
FOREIGN KEY (product_id)
REFERENCES products(product_id);
```
![image](https://github.com/user-attachments/assets/34e4de19-e077-475c-b3ee-74d208493d78)

## 7. Візуалізація у Power BI
Дані з SQL було імпортовано у Power BI та зроблено візуалізацію
![image](https://github.com/user-attachments/assets/49aa60ef-2157-4236-b5db-bf8f3538c35f)


