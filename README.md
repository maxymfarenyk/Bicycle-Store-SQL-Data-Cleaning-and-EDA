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
    standard_cost TEXT,
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



