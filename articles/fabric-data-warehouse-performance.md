# Fabric Data Warehouse Performance Optimization
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >


System performance is a vital consideration in enterprise environments. This document outlines strategies and methodologies for optimizing Fabric Data Warehouse to maintain a high performance operational environment.


# **Options**

# **Option 1: Use Data Clustering **

Data clustering efficiently organizes records with similar values, enabling faster queries and minimizing compute and storage costs. 
By grouping related data and reducing scanned files, clustering significantly enhances data warehouse performance. This approach is especially effective when your system frequently queries specific subsets of data. 
By selecting the column or columns most commonly used in WHERE predicates as the clustering key, you can optimize query performance and efficiency.

**For example**
```
CREATE TABLE dbo.Sales
(SalesId        bigint        NOT NULL,
SaleDate       date          NOT NULL,
CustomerId     int           NOT NULL,
ProductId      int           NOT NULL,
StoreId        int           NOT NULL,
Quantity       int           NOT NULL,
NetAmount      decimal(18,2) NOT NULL,
TaxAmount      decimal(18,2) NOT NULL,
TotalAmount    decimal(18,2) NOT NULL,
CreatedAtUtc   datetime2(0)  NOT NULL
)WITH (CLUSTER BY (SaleDate, CustomerId));
```

In our test scenarios, we observed significant improvements in performance. Please refer to the execution plan below to see the marked differences in optimization and query execution for the same table and query, with one clustered and the other not.

**Non Clustering Key Column:**

<img width="1148" height="293" alt="image" src="https://github.com/user-attachments/assets/6f9d2965-f5d0-423e-a061-5a254537cb75" />

**Clustering Key Column:**

<img width="764" height="133" alt="image" src="https://github.com/user-attachments/assets/7d9cbbc0-5385-4448-ae17-bbd40ac93e27" />


# **Option 2: Update Statistics **
Another important, yet often overlooked, consideration is updating statistics. Keeping statistics current supplies the optimizer with valuable metadata about data distribution and density. Given its significance, scheduling a statistics update job is recommended to ensure this process occurs as part of large data loads.
Statistics jobs can be executed by multiple compute engines within Fabric. One of the most dynamic and efficient approaches is to implement a Fabric pipeline, as outlined in the steps below.

# **Steps **

***Step: 1*** 
Create a pipeline and provide it with an appropriate name as the below:

<img width="522" height="328" alt="image" src="https://github.com/user-attachments/assets/ab82fb26-bdd8-4cdc-b496-603443f807b4" />

***Step: 2*** 
Select the "Activities" tab and select "Script" and provide it with an appropriate name:

<img width="671" height="965" alt="image" src="https://github.com/user-attachments/assets/ea89d016-45ef-4eda-8509-3467d3e5d268" />

***Step: 3*** 

Select settings and select your data warehouse connection to connect the activity to and after select "Edit in code editor": 


<img width="505" height="483" alt="image" src="https://github.com/user-attachments/assets/0799ee71-52ca-40fd-8f87-1f90be2e7e3e" />


<img width="658" height="330" alt="image" src="https://github.com/user-attachments/assets/eb27f8fe-74ac-4255-8d94-b5b9d4c25c2e" />


***Step: 4*** 
Add the following expression to retrieve all statistics for user tables, these will be used as our cursor.

```
SELECT CONCAT(QUOTENAME(s.name), N'.', QUOTENAME(t.name)) AS full_table
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id;
```
<img width="860" height="474" alt="image" src="https://github.com/user-attachments/assets/1faf68a7-7c52-42da-ab24-61234b98293f" />



***Step: 5*** 
In "Activities," select the "ForEach" loop, connect the previous "Script" activity to it, then select "Settings" on the "ForEach" loop. Modify the "Batch count" to 5, select the items input, and add the expression below. Please note that "IndexRebuildCursor" should be the same name as the first script activity.

```
@activity('IndexRebuildCursor').output.resultSets[0].rows
```

<img width="740" height="416" alt="image" src="https://github.com/user-attachments/assets/9115f31d-8630-4f95-a8fb-433eb4d2cd99" />

<img width="1038" height="188" alt="image" src="https://github.com/user-attachments/assets/da7a05d4-215d-4781-94df-21836855f787" />

<img width="899" height="369" alt="image" src="https://github.com/user-attachments/assets/44411729-58f0-4ac9-bc95-51b7f23bc236" />

<img width="878" height="367" alt="image" src="https://github.com/user-attachments/assets/9c54af6c-74de-4afa-b283-19bf49d717b8" />


***Step: 6*** 
Enter the "ForEach" loop, select the "Activities" tab, and add a "Script" activity to update all statistics in the database. Select the same connection as the outer script, then, select the "expression builder" to add the expression below.
```
@concat('UPDATE STATISTICS ', item().full_table, ' WITH FULLSCAN;')

```
<img width="1051" height="175" alt="image" src="https://github.com/user-attachments/assets/d9567ff6-bb12-4718-b10a-74c34291f5a7" />


<img width="998" height="849" alt="image" src="https://github.com/user-attachments/assets/1155542b-411e-4c27-a1cc-eadfe4869aca" />

<img width="689" height="302" alt="image" src="https://github.com/user-attachments/assets/60a3e94c-2013-47fc-89b0-78ac4eaa26ca" />

<img width="851" height="575" alt="image" src="https://github.com/user-attachments/assets/2de192d4-4a58-4d4c-bab6-4c8c165fb241" />

<img width="698" height="600" alt="image" src="https://github.com/user-attachments/assets/a7393db0-154f-4eb4-9fda-040112b95f61" />


***Step: 7*** 
Save and test and add to you process. 


# **Option 3: Create Statistics Manually **
Another commonly overlooked step is manually creating statistics in the data warehouse. Doing so provides the necessary metadata to optimize system performance, especially when handling large workloads. Focus on columns used in WHERE clauses, JOIN conditions, and compound columns, as these are strong candidates for statistics creation. Refer to the following syntax for guidance.

```
CREATE STATISTICS Unique_Name_Here_FullScan
ON schema.table (key) WITH FULLSCAN;
```




***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. 
THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS 
FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) 
to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is 
embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the 
Sample Code.***

