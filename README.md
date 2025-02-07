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

## Step 1: Data Factory
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

In order to execute this, I first created 

```ruby
@item().SchemaName
@item().TableName
```

```ruby
@{concat('SELECT * FROM ', item().SchemaName,'.',item().TableName )}
```
