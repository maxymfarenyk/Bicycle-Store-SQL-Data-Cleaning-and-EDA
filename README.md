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
Таблиця demographic_staging
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

