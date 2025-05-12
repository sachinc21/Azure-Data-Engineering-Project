# Azure Data Engineering Project - End-to-End Data Pipeline

## 🚀 Project Overview
This project is an end-to-end data engineering pipeline built using Microsoft Azure. The objective was to design and implement a cloud-native data pipeline that ingests data from a public API, processes it using Databricks (PySpark), stores it in Azure Data Lake Gen2, and enables data querying using Azure Synapse Analytics. The project follows the Medallion Architecture (Bronze, Silver, Gold) to structure the data in layers for data quality and analytics readiness.

## 🛠️ Tech Stack
- **Azure Data Lake Storage Gen2:** Scalable storage for raw, cleaned, and aggregated data
- **Azure Databricks:** Data processing and transformations using PySpark
- **Azure Synapse Analytics:** Data querying and analytics using SQL
- **Python & PySpark:** Data extraction, processing, and transformations
- **GitHub API:** Source data for pipeline (commits data)
- **Azure Storage Explorer:** Data monitoring and management
- **Azure Portal:** Resource management and monitoring

## ✅ Architecture
The project is structured using the Medallion Architecture to separate data processing stages:

- **Bronze Layer:** Raw data ingestion
- **Silver Layer:** Cleaned and structured data
- **Gold Layer:** Aggregated, analytics-ready data

### 📌 Data Flow Overview:
1. **Data Ingestion:** Data is extracted from the GitHub API and stored in the Bronze layer in Azure Data Lake Gen2 as raw JSON files.
2. **Data Transformation:** Data is processed in Azure Databricks using PySpark. Transformation includes:
   - Null value handling
   - Data type conversions
   - Filtering and deduplication
   - Data aggregation (e.g., commit count per author, daily commit trends)
3. **Data Storage:** Transformed data is stored in the Silver layer (cleaned data) and Gold layer (aggregated data) in Azure Data Lake Gen2.
4. **Data Analytics:** Data from the Gold layer is queried in Azure Synapse Analytics to extract insights such as top commit authors, commit frequencies, and daily commit trends.

## 🔨 Implementation Steps
### Step 1: Data Ingestion
- Access the GitHub API to extract commit data using Python requests.
- Store raw JSON files in the Bronze layer in Azure Data Lake Gen2 under the path `bronze/commits/YYYY/MM/DD/`. 
- Example Data Fields:
  - Commit SHA
  - Author Name
  - Commit Date
  - Commit Message

### Step 2: Data Transformation (Azure Databricks)
- Create a Databricks notebook for data processing.
- Implement the following transformations using PySpark:

#### Data Loading:
- Data is read from the Bronze layer as CSV files using PySpark.
- Datasets include Calendar, Customers, Product Categories, Products, Returns, Sales, Territories, and Subcategories.

#### Data Transformation:
- **Calendar Data:** Extract month and year from the date column and write as Parquet to the Silver layer.
  ```python
  df_cal = df_cal.withColumn('Month', month(col('Date')))
                 .withColumn('Year', year(col('Date')))
  df_cal.write.format('parquet')
              .mode('append')
              .option("path", "abfss://silver@storagedatalakede.dfs.core.windows.net/AdventureWorks_Calendar")
              .save()
  ```

- **Customers Data:** Concatenate first name, last name, and prefix to create a full name column. Data is reordered and saved as Parquet.
  ```python
  df_cus = df_cus.withColumn('FullName', concat_ws(' ', col('Prefix'), col('FirstName'), col('LastName')))
  desired_order = ['CustomerKey', 'Prefix', 'FirstName','FullName', 'LastName', 'BirthDate', 'MaritalStatus', 'Gender', 'EmailAddress', 'AnnualIncome', 'TotalChildren', 'EducationLevel', 'Occupation', 'HomeOwner']
  df_cus = df_cus.select(*desired_order)
  df_cus.write.format('parquet')
                .mode('append')
                .save('abfss://silver@storagedatalakede.dfs.core.windows.net/AdventureWorks_Customerrs')
  ```

- **Product Data:** Split ProductSKU and ProductName columns and write to Silver layer.
  ```python
  df_pro = df_pro.withColumn('ProductSKU', split(col('ProductSKU'), '-')[0])
                  .withColumn('ProductName', split(col('ProductName'), ' ')[0])
  df_pro.write.format('parquet')
                .mode('append')
                .save('abfss://silver@storagedatalakede.dfs.core.windows.net/AdventureWorks_Products')
  ```

- **Sales Data:**
  - Convert StockDate to timestamp
  - Replace 'S' with 'T' in OrderNumber
  - Add a calculated column 'Multiply'
  ```python
  df_sales = df_sales.withColumn('StockDate', to_timestamp('StockDate'))
  df_sales = df_sales.withColumn('OrderNumber', regexp_replace(col('OrderNumber'), 'S', 'T'))
  df_sales = df_sales.withColumn('Multiply', col('OrderLineItem') * col('OrderQuantity'))
  df_sales.write.format('parquet')
                .mode('append')
                .save('abfss://silver@storagedatalakede.dfs.core.windows.net/AdventureWorks_Sales')
  ```

- **Territories and Returns Data:** Directly written to Silver layer as Parquet.

- Create a Databricks notebook for data processing.
- Implement the following transformations using PySpark:
  - Schema definition and enforcement
  - Null handling using `dropna()` and `fillna()`
  - Data deduplication using `dropDuplicates()`
  - Data type conversions (e.g., timestamp parsing)
  - Aggregation to calculate daily commit counts and top commit authors
- Save cleaned data to the Silver layer and aggregated data to the Gold layer under paths `silver/commits` and `gold/commit_summary`.

### Step 3: Data Storage & Partitioning
- Data is partitioned by date (`YYYY/MM/DD`) for easy querying and cost optimization.
- Data Lake Structure:
  - `/bronze/commits/YYYY/MM/DD/` - Raw JSON data
  - `/silver/commits/` - Cleaned data in Parquet format
  - `/gold/commit_summary/` - Aggregated data in Parquet format

### Step 4: Data Querying (Azure Synapse Analytics)
- Synapse was utilized to create external tables and views based on the transformed data stored in the Silver and Gold layers of Azure Data Lake Gen2. This enables querying large datasets without physically moving data.

#### Credential and Data Source Setup:
- A **Managed Identity** credential was created to securely access the data in Azure Data Lake:
  ```sql
  CREATE DATABASE SCOPED CREDENTIAL cred_sachin
  WITH IDENTITY = 'Managed Identity';
  ```

- External Data Sources for Silver and Gold layers:
  ```sql
  CREATE EXTERNAL DATA SOURCE source_silver
  WITH (
      LOCATION = 'https://storagedatalakede.blob.core.windows.net/silver',
      CREDENTIAL = cred_sachin
  );

  CREATE EXTERNAL DATA SOURCE source_gold
  WITH (
      LOCATION = 'https://storagedatalakede.blob.core.windows.net/gold',
      CREDENTIAL = cred_sachin
  );
  ```

#### External File Format Setup:
- Parquet format with Snappy compression was defined to ensure efficient querying:
  ```sql
  CREATE EXTERNAL FILE FORMAT format_parquet
  WITH (
      FORMAT_TYPE = PARQUET,
      DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'
  );
  ```

#### External Table Creation:
- Example: External Table for `Sales` data:
  ```sql
  CREATE EXTERNAL TABLE gold.extsales
  WITH (
      LOCATION = 'extsales',
      DATA_SOURCE = source_gold,
      FILE_FORMAT = format_parquet
  ) AS
  SELECT * FROM gold.sales;

  SELECT * FROM gold.extsales;
  ```

#### Creating Views from Silver Layer:
- Views were created to simplify data access and facilitate analytics:
  - **Calendar Data:**
  ```sql
  CREATE VIEW gold.calendar AS
  SELECT * FROM
  OPENROWSET(
      BULK 'https://storagedatalakede.blob.core.windows.net/silver/AdventureWorks_Calendar/',
      FORMAT = 'PARQUET'
  ) AS QUERY1;
  ```

  - **Customers Data:**
  ```sql
  CREATE VIEW gold.customers AS
  SELECT * FROM
  OPENROWSET(
      BULK 'https://storagedatalakede.blob.core.windows.net/silver/AdventureWorks_Customers/',
      FORMAT = 'PARQUET'
  ) AS QUERY1;
  ```

  - **Products Data:**
  ```sql
  CREATE VIEW gold.products AS
  SELECT * FROM
  OPENROWSET(
      BULK 'https://storagedatalakede.blob.core.windows.net/silver/AdventureWorks_Products/',
      FORMAT = 'PARQUET'
  ) AS QUERY1;
  ```

#### Creating the Gold Schema:
- A `gold` schema was created to logically group the external tables and views:
  ```sql
  CREATE SCHEMA gold;
  ```
- Connect Synapse to the Gold layer in Azure Data Lake Gen2.
- Create external tables to query data using T-SQL.
- Example Queries:
  - Retrieve top 5 commit authors by commit count:
    ```sql
    SELECT author_name, COUNT(commit_sha) AS commit_count 
    FROM gold.commit_summary
    GROUP BY author_name
    ORDER BY commit_count DESC LIMIT 5;
    ```
  - Daily commit trend analysis:
    ```sql
    SELECT commit_date, COUNT(commit_sha) AS daily_commits
    FROM gold.commit_summary
    GROUP BY commit_date
    ORDER BY commit_date;
    ```

### Step 5: Data Validation & Testing
- Verify data consistency between Bronze, Silver, and Gold layers using sample data checks.
- Validate the data schema in Databricks.
- Test SQL queries in Synapse to ensure accuracy of aggregation and filtering logic.

## 🌱 Key Learnings
- Implementing the Medallion Architecture in Azure to handle structured data processing
- Creating robust PySpark scripts for data cleaning, transformation, and aggregation
- Managing data partitioning and storage structure in Azure Data Lake Gen2
- Querying large datasets in Azure Synapse Analytics to extract actionable insights
- Monitoring data flow and storage costs to optimize resource usage

## 🖼️ Screenshots
1. **Data Lake Folder Structure (Bronze, Silver, Gold)**
2. **Databricks PySpark Notebook (Data Transformation)**
3. **Synapse Query Results (Top Authors, Daily Commits)**
4. **Data Ingestion Sample JSON Files**

## 📦 Next Steps
- Implement real-time data streaming using Kafka and Spark Streaming
- Develop a Power BI dashboard to visualize commit trends and top authors

## 🤝 Connect
Feel free to connect with me on [LinkedIn](https://www.linkedin.com) to discuss this project or potential collaborations!
