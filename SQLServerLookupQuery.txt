SELECT
s.name AS SchemaName,
t.Name AS TableName
FROM sys.tables t
INNER JOIN sys.schemas s
ON t.schema_id = s.schema_id
WHERE s.Name = 'SalesLT'