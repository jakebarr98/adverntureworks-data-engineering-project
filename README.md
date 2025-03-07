# End to End Azure Data Engineering Project

This project's purpose is to demonstrate my ability to create an end to end data engineering solution using Microsoft Azure to build pipelines automating data extraction, transformation and loading into PowerBI for reporting.


## Architecture
![Project Architecture](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/main/DataEngProjectDA.jpeg)


## Technology Used
- SQL
- Python
- Azure
- PowerBI

## Dataset Used
I used the AdventureWorks dataset available on the Microsoft website. I restored the .bak file in my SQL Server Management Studio 

## Step 1: Data Factory - Extract Data
I started by building a pipeline in data factory that looks up the tables in my on-prem SQL server
I used a lookup activity, linked it to my on-prem SQL Server and used the following query to return all tables in my database where the schema was 'SalesLT'

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

My source was the on-prem SQL server, from which I ran the following query:

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

## Step 2: Databricks - Transform Data Scripting
In databricks I started by creating a single node cluster compute to run the notebooks. As I am extracting from data factory, transforming in databricks and loading in synapse, I need a compute that is able to process all of the information from the 3 resources. I only need it to be single node as all of the resources are located on a single machine.

Following this, I started with my first notebook which was to mount the data lake storage containers I was going to be reading data from and writing data to. My storage mount script is below:

![Storage mount python script](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/main/storagemount.ipynb)

Once my storage containers were mounted, I was ready to begin transforming the data

The first transformation I underwent was changing the formatting for all of the date fields in each of the tables. I then wrote the updated data into the silver storage mount. Python script below

![Bronze to silver python script](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/e8c2535498f087b180027c6f69a021737a21aa93/bronze%20to%20silver%20-%20extended.ipynb)

Secondly I wanted to transform all of the field names into snake_case so they were all consistent. I then wrote the updated data into the gold storage mount. Python script below

![Silver to gold python script](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/e8c2535498f087b180027c6f69a021737a21aa93/silver%20to%20gold%20-%20extended.ipynb)

## Step 3: Data Factory - Transform Data Pipeline
In datafactory, I created a following action on the success of the copy tables activity to run the databricks notebook bronze to silver, and then on success of this activity to run the silver to gold notebook.

This concluded my datafactory pipeline, as can be seen below:

![Data Factpry Pipeline](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/main/DataFactoryPipeline.png)

## Step 4: Create views in Synapse

Next, I launched Synapse Studio and created a new serverless SQL database (gold_db). As my datalake is already linked to Synapse, I had access to all of the data in my storage containers. 

I wanted to create a view for each of my tables in the gold storage container. In order to do this dynamically I created a pipeline within Synapse that runs a stored procedure, using @viewname as a variable to create a view for each table in the gold storage container. 

Firstly, I created the stored procedure that would create the view, using the script below:
```ruby
USE gold_db
GO

CREATE OR ALTER PROC CreateSQLServerlessView_gold @ViewName nvarchar(100)
AS
BEGIN
    DECLARE @statement VARCHAR(MAX)
    SET @statement = N'CREATE OR ALTER VIEW ' + @ViewName + ' AS
    SELECT
        *
    FROM
        OPENROWSET(
                BULK''https://adventureworkssa1.dfs.core.windows.net/gold/SalesLT/' + @ViewName + '/'',
                FORMAT = ''DELTA''
            ) AS [result]'
        
        EXEC (@statement)
    END
    GO
```
I created a linked service to to my gold_db SQL database so I can access the stored procedure from the pipeline.

I then began creating my pipeline, which began with an ativity of getting the metadata from each of the child items from the gold storage container in my datalake. This gives me the names of all of the folders (tables) in the storage container.

Next I added a For Each loop, within which I added a dynamic expression to fetch the name of each of the folders from the previous steps output

```ruby
@activity('Get Table Names').output.childItems
```
Within the for each, I added a stored procedure activity. In order to make this dynamic and use the @ViewName variable, I created a Stored Procedure parameter within the activity settings as can be seen below - this uses the table name as the @ViewName variable

![Stored Procedure Parameter](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/main/StoredProcParameter.png)

After triggering once, this pipeline doesn't have to be run again unless there are any schema changes since it's a view

![Synapse Pipeline](https://github.com/jakebarr98/adverntureworks-data-engineering-project/blob/main/SynapsePipeline.png)

## Step 5: Load into PowerBI & Create Data Model

In order to use the data to build BI reports, I created a connection within PowerBI to my gold_db database within Synapse and imported all of the tables into my data model. 

Once I had laoded the tables, I had to build the data model by identifying relationships between the tables. I created all of the relationships resulting in my complete data model which is ready to be used to develop dashboard and reports within PowerBI.
