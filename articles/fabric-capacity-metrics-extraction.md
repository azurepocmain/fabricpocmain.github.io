# Fabric Capacity Metrics Extraction
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

As organizations deploy Fabric workspace capacity to multiple departments, it becomes imperative to implement precise tracking mechanisms for accurate inter-departmental billing. 
The optimal tool for this purpose is Fabric Chargeback: <a href="https://learn.microsoft.com/en-gb/fabric/release-plan/admin-governance#capacity-metrics-chargeback-public-preview" target="_blank">Fabric Chargeback</a>
However, since Fabric Chargeback is currently in public preview, this documentation will outline alternative methodologies utilizing Fabric Capacity Metrics for data extraction and storage to facilitate precise chargeback calculations.

**Steps: Fabric Capacity Metrics Extraction**

*Step1:*
Firstly, download the Fabric Capacity Metrics App by navigating to AppSource > Microsoft Fabric Capacity Metrics and selecting "Get it now". The corresponding link is accessible here:  <a href="learn.microsoft.com/en-us/fabric/enterprise/metrics-app-install?tabs=1st" target="_blank">Install the Microsoft Fabric capacity metrics app</a>

*Step2:*
Subsequently, configure the Capacity Metrics App by adhering to the instructions outlined in the aforementioned documentation. Navigate to the Microsoft Fabric Capacity Metrics workspace via the left pane link, proceed to "Workspace settings" located on the top right, and then select "License info" followed by the edit option. 
Choose "Fabric capacity" and allocate the necessary capacity. 
The allocated capacity is essential for executing the notebook provided below against the semantic model for the Microsoft Fabric Capacity Metrics workspace.
![image](https://github.com/user-attachments/assets/8ed9ba2a-e6a2-49e1-8e59-97e248c383e5)

*Step3:*
Finally, ensure that a Fabric Workspace has been established, as it will serve as the repository for the initial data load and subsequent delta updates.

*Step4:*
Initialize a Spark notebook utilizing the PySpark language. Execute the code steps as follows:

**Spark Code Steps: Fabric Capacity Metrics Extraction:**

To obtain the FabricWarehouseID and FabricWarehouseID execute the following command utilizing PySpark within the Spark notebook. Ensure you have identified the WorkspaceID and the FabricWarehouse ItemID relevant to your operations.
```
import sempy.fabric as fabric

# Get FabricWarehouseID and FabricWarehouseID
df_workspace_warehouse_ids= fabric.read_table("Fabric Capacity Metrics", "Items")
display(df_workspace_warehouse_ids)
```

Define and assign the following variables utilizing the retrieved IDs in your Spark session:
```
FabricWorkspaceId="Add-Fabric-Workspace-ID-HERE"
FabricWarehouseID="Add-Fabric-Warehouse-ID-HERE"
```

Please note that some of the below steps will not be needed and w

```
import sempy.fabric as fabric

# Execute the export of data from the two semantic model objects. This operation should be scheduled either on a daily basis or at an interval of every 13 days.
# This procedure will ensure that the 14-day data retention window is rigorously maintained.
# Primary Table for CU Consumption
df_metrics_by_item_day_table = fabric.read_table("Fabric Capacity Metrics", "MetricsByItemandDay")

# Primary Table for Worksapce Items
df_items_table = fabric.read_table("Fabric Capacity Metrics", "Items")

```



```
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("FabricMetrics").getOrCreate()

#Convert a Pandas DataFrame to a PySpark DataFrame, as the dataframes retrieved from sempy.fabric are initially in Pandas format.
df_metrics_by_item_spark = spark.createDataFrame(df_metrics_by_item_day_table)
df_items_table_spark = spark.createDataFrame(df_items_table)
```


```
# Process Capacity Metrics Inserts
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  


#INITIAL EXECUTION: Uncomment this section for the initial run and comment out all subsequent code blocks.
# The table will be auto-created; adjust the table name as necessary, add your Fabric DW name in the FabricwarehouseName.
#df_spark.write.mode("append").option(Constants.WorkspaceId, FabricWorkspaceId).synapsesql("FabricwarehouseName.dbo.FabricCapacityMetrics")

#Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
comparison_columns = ["DateTime", "PremiumCapacityId", "ItemId", "sum_CU", "sum_duration", "WorkspaceId", "UniqueKey" ]  #Using following columns as a unique key for  join

#Step 1: Read existing data from the Fabric Warehouse
df_current_metric_table = spark.read.option(Constants.WorkspaceId, FabricWorkspaceId).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql("FabricwarehouseName.dbo.FabricCapacityMetrics")


#Step 2: Identify new records using left_anti on multiple columns above
df_new_metric_insert = df_metrics_by_item_spark.join(df_current_metric_table, comparison_columns, "left_anti")

#Step 3: Append only new records to Fabric Warehouse for each invocation
df_new_metric_insert.write.mode("append").option(Constants.WorkspaceId, FabricWorkspaceId).synapsesql("Fabricwarehouse1.dbo.FabricCapacityMetrics")

```

```
# Process Item Table 
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

#INITIAL EXECUTION: Uncomment this section for the initial run and comment out all subsequent code blocks.
# The table will be auto-created; adjust the table name as necessary, add your Fabric DW name in the FabricwarehouseName.
#df_spark.write.mode("append").option(Constants.WorkspaceId, FabricWorkspaceId).synapsesql("FabricwarehouseName.dbo.FabricItems")

#Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
comparison_columns = ["capacityId", "ItemId", "ItemKind", "ItemName", "Timestamp", "WorkspaceId", "UniqueKey" ]  #Using following columns as a unique key for  join

#Step 1: Read existing data from the Fabric Warehouse
df_current_metric_table = spark.read.option(Constants.WorkspaceId, FabricWorkspaceId).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql("FabricwarehouseName.dbo.FabricItems")


#Step 2: Identify new records using left_anti on multiple columns above
df_new_metric_insert = df_items_table_spark.join(df_current_metric_table, comparison_columns, "left_anti")

#Step 3: Append only new records to Fabric Warehouse for each invocation
df_new_metric_insert.write.mode("append").option(Constants.WorkspaceId, FabricWorkspaceId).synapsesql("Fabricwarehouse1.dbo.FabricItems")
```

**SQL Code Steps: Fabric Capacity Metrics Percentabe Query:**
A percentage output for each day and related workspace will be the output. You can potentially utilize this percentage and divide it by the daily cost via the Azure Portal to implement the chargeback process.
Update the below table names to reflect your environment.
```
WITH DailyTotal AS (
    --Calculate total CU usage per day across all workspaces potentially use the percentage to multiple the overall bill cost per owner of workplace
    SELECT 
        CAST([Date] AS DATE) AS UsageDate, 
        SUM(sum_CU) AS TotalDailyCU --Total CU for the day no storage 
    FROM 
        dbo.FabricCapacityMetrics metics -- Update table name as needed
	JOIN FabricItems item on metics.ItemId=item.ItemId  -- Update table name as needed
	AND metics.WorkspaceId=item.WorkspaceId
	AND  metics.UniqueKey=item.UniqueKey
	WHERE item.[Billable type] IN ('Billable', 'Both') -- Only getting the actual cost
    GROUP BY 
        CAST([Date] AS DATE)
)

SELECT 
    metics.WorkspaceId, 
	item.WorkspaceName,
    CAST(metics.[Date] AS DATE) AS UsageDate, 
    SUM(metics.sum_CU) AS WorkspaceTotalCU,  -- Total CU for the workspace
    d.TotalDailyCU,  -- Total CU across all workspaces for the day
    (SUM(metics.sum_CU) * 100.0 / NULLIF(d.TotalDailyCU, 0)) AS UsagePercentage --Stop and Prevents division by zero errors that I had
FROM 
    dbo.FabricCapacityMetrics metics  -- Update table name as needed
	JOIN FabricItems item on metics.ItemId=item.ItemId  -- Update table name as needed
	AND metics.WorkspaceId=item.WorkspaceId
	AND  metics.UniqueKey=item.UniqueKey
JOIN 
    DailyTotal d
ON 
    CAST(metics.[Date] AS DATE) = d.UsageDate
	WHERE item.[Billable type] IN ('Billable', 'Both') --O nly getting the actual cost
GROUP BY 
    metics.WorkspaceId, item.WorkspaceName, CAST(metics.[Date] AS DATE), d.TotalDailyCU
ORDER BY 
    UsageDate, UsagePercentage DESC;
```


***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***


