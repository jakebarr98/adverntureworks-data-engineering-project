# End to End Azure Data Engineering Project

This project is a basic beginners project to help me understand the fundamentals of data engineering using the microsoft stack of Azure.

My dataset for this project is the Adventureworks database from Microsoft, which I had restored in SSMS

## Architecture
![Project Architecture](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/main/DataEngProjectDA.jpeg)


## Technology Used
- SQL
- Python
- Azure
- PowerBI

Within Azure I created the following resources:
  Resource Group
    Subscription
    Key Vault
    Data Factory
    Databricks
    Storage Account (Data Lake)

## Dataset Used
I used the AdventureWorks dataset available on the microsoft website. I restored the .bak file in my SQL Server Management Studio 

## Step 1: Data Factory - Extract Data
I started by building a pipeline in data factory that looks up the tables in my on prem SQL server
I used a lookup activity, linked it to my on prem SQL Server and used the following query to return all tables in my database where the schema was 'SalesLT'

```ruby
SELECT
s.name AS SchemaName,
t.Name AS TableName
FROM sys.tables t
INNER JOIN sys.schemas s
ON t.schema_id = s.schema_id
WHERE s.Name = 'SalesLT'
```
I then created a for each activity, within which there was a copy data activity that would copy the data from each of the tables the lookup returned

My source was the on prem SQL server, from which I ran the following query:

```ruby
@{concat('SELECT * FROM ', item().SchemaName,'.',item().TableName )}
```

My sink was a parquet dataset within my azure data lake, for which I created a linked service to the storage account (data lake) I created , I made the file path bronze (storage container within my data lake) and then made the following paths dynamic with the expression:
```ruby
@{concat(dataset().schemaname,'/',dataset().tablename)} / @{concat(dataset().tablename, '.parquet')}
```
I set up the paramaters within the sink to use in my dynamic expressions as below
```ruby
schemaname = @item().SchemaName
tablename = @item().TableName
```
This would create a new folder within the bronze container for the schema, and then within this folder, created a folder for each table, which contains the parquet file of that tables data.

## Step 2: Databricks - Transform Data Scipting
In databricks I started by creating a single node cluster compute. [EXPLAIN WHY]

Following this, I started with my first notebook which was to mount the data lake storage containers I was going to be reading data from and writing data to. My storage mount script is below:

![Storage mount python script](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/main/storagemount.ipynb)

Once my storage containers were mounted, I was ready to begin transofrming the data

The first transformation I underwent was changingthe formatting for all of the date fields in each of the tables. I then wrote the updated data into the silver storage mount. Python script below

![Bronze to silver python script](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/e8c2535498f087b180027c6f69a021737a21aa93/bronze%20to%20silver%20-%20extended.ipynb)

Secondly I wanted to transform all of the field names into snake_case so they were all consistent. I then wrote the updated data into the gold storage mount. Python script below

![Silver to gold python script](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/e8c2535498f087b180027c6f69a021737a21aa93/silver%20to%20gold%20-%20extended.ipynb)

## Step 3: Data Factory - Transform Data Pipeline
