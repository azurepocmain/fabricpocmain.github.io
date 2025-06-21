# Fabric Storage Usage Extraction
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >


As organizations deploy Fabric workspace capacity to multiple departments, it becomes imperative to implement tracking mechanisms for accurate inter-departmental billing. 
The optimal tool for this purpose is Fabric Chargeback: <a href="https://learn.microsoft.com/en-gb/fabric/release-plan/admin-governance#capacity-metrics-chargeback-public-preview" target="_blank">Fabric Chargeback</a>
However, since Fabric Chargeback is currently in public preview, this documentation will outline alternative methodologies utilizing Fabric Capacity Metrics for data extraction and storage to facilitate chargeback calculations.

**Steps: Fabric Capacity Metrics Extraction**

*Step1:*
Firstly, download the Fabric Capacity Metrics App by navigating to AppSource > Microsoft Fabric Capacity Metrics and selecting "Get it now". The corresponding link is accessible here:  <a href="learn.microsoft.com/en-us/fabric/enterprise/metrics-app-install?tabs=1st" target="_blank">Install the Microsoft Fabric capacity metrics app</a>

*Step2:*
Subsequently, configure the Capacity Metrics App by adhering to the instructions outlined in the aforementioned documentation. 
Navigate to the Microsoft Fabric Capacity Metrics workspace via the left pane link, proceed to “Workspace settings” located on the top right, and then select “License info” followed by the edit option. Ensure that  “Pro” is selected which should be the default. The assigned capacity (Pro) remains until the extraction process initiates. This procedure has been automated, ensuring a stable and reliable extraction of Capacity Metrics App data from the semantic model via altering the capacity from Pro to Fabric. This configuration is essential, as failing to use Fabric capacity during extraction may result in permission exceptions. This approach differs from the initial solution, which kept the Capacity Metrics App capacity permanently set to Fabric, a configuration that is no longer necessary. In addition, an iterative loop has been integrated to collect telemetry data from each capacity within the environment, allowing for comprehensive resource utilization analysis.
![image](https://github.com/user-attachments/assets/e431c2ae-3623-49bc-85e3-ac85abf27aa2)



*Step3:*
Finally, ensure that a Fabric Data Warehouse has been established, as it will serve as the repository for the initial data load and subsequent delta updates.

*Step4:*
Initialize a Spark notebook utilizing the PySpark language. Execute the code steps as follows:

**PySpark Code Steps: Fabric Capacity Metrics Extraction:**

**Please note that the following step represents the third phase in the redesigned pipeline for comprehensive capacity retrieval within the environment. Refer to the "Fabric Capacity Metrics Extraction" documentation for the complete workflow, however, substitute the section titled "PySpark Code Steps: Capacity Metric Specific Capacity Data Extraction ETL to Data Warehouse: Notebook 3" with the instructions detailed below in a new pipeline of course. You can just make a copy of the metric pipeline, rename it to storage and add the below. In the pipeline activity selection, integrate this into "Activity 4 Inside For Each Loop Notebook 4" within your pipeline configuration. Finally, it is critical to avoid executing the metric and storage extraction processes concurrently, as simultaneous operations may introduce conflicts or inconsistencies in the capacity selected data.**

<a href="https://azurepocmain.github.io/fabricpocmain.github.io/articles/fabric-capacity-metrics-extraction.html" target="_blank">Fabric Capacity Metrics Extraction</a>


Getting the Fabric Capacity Workspace ID
To obtain the Workspace ID for the Fabric Capacity Metrics workspace execute the following command utilizing PySpark within the Spark notebook. Ensure you have identified the Workspace ID of the remote Capacity Metrics App. We are using the workspace ID as there can be some errors if we solely use the workspace name as it can be altered etc.. 
```
import sempy.fabric as fabric

# GET THE WORKSPACE ID THAT HAS THE FABRIC CAPACITY METRIC
# First verify your Capacity workspace ID from the below. We should have the ability to access remote workspaces
capacity_workspace_check=fabric.list_workspaces()

display(capacity_workspace_check)
```
Getting the DataSet. 
Using the above Workspace ID, add it to the below variable Fabric_Capacity_WorkspaceId replacing the value ADD_THE_ABOVE_FABRIC_CAPACITY_WORKSPACE_ID_HERE with your Fabric Capacity workspace ID from the above.
In testing the Workspace ID did not require a conversion with the UUID function but it is added in case. 
No other variables should require altering unless the default dataset name has been altered.
```
import sempy.fabric as fabric

# GET THE DATASET
# Next, using the FabricWorkspaceId from the above, add it to the FabricWorkspaceId variable to get the fabric capacity dataset 
# The UUID function should not be needed only there incase the workspace_id throws an exception ensure that you invoke the import as well
#from uuid import UUID
# Fabric_Capacity_WorkspaceId = UUID("work_space_id_here_if_the_below_fails_but_should_not_be_needed)  
Fabric_Capacity_WorkspaceId="ADD_THE_ABOVE_FABRIC_CAPACITY_WORKSPACE_ID_HERE"
# No need to change in most cases unless you updated prior
DatasetName="Fabric Capacity Metrics"

get_dataset=fabric.list_datasets(workspace=Fabric_Capacity_WorkspaceId)
get_dataset = get_dataset.loc[get_dataset['Dataset Name'] == DatasetName]
# Get the column value without the column name
get_dataset = get_dataset['Dataset Name'].values[0]
display(get_dataset)
```

We need to get the Fabric Data Warehouse Name,  ID and the Workspace ID where the Warehouse resides, as we will store the data in that table.
The initial command assumes that you have already invoked the capacity metric app and report, and data has been collected.
If not, the second command can be leveraged. It expects you to be in the workspace context of the Fabric Data Warehouse.
```
import sempy.fabric as fabric

# Next get the Fabric Data Warehouse Name & ID from the Id column from the results below that you want to use. 
# Be sure to replace Add_Fabric_Workspace_Id_Where_Farbic_Data_Warehouse_Resides with the above workspace ID that has the Fabric DW.
get_fabric_dw_to_use=fabric.list_items(workspace="Add_Fabric_Workspace_Id_Where_Farbic_Data_Warehouse_Resides");
# Only return DW so semantic model name wont confuse anyone as they have the same name 
get_fabric_dw_to_use=get_fabric_dw_to_use.loc[get_fabric_dw_to_use["Type"] == "Warehouse"]
display(get_fabric_dw_to_use)


```

Assign the three above variables to the three below variables.
```
#Fabric DW ID Assign 
# Assign the Fabric DW ID that will store the data
FabircWarehouse_WorkSpace_ID="FABRIC_WORKSPACE_ID_WHERE_FABIRC_DW_LIVES"
FabricWarehouseID="FABRIC_DW_RESOURCE_ID"
FabricWarehouseName="FARBIC_WAREHOUSE_NAME"
```

We are extracting the actual data from the dataset to store the values in the corresponding variables. 
```
import sempy.fabric as fabric

# Execute the export of data from the two semantic model objects. This operation should be scheduled either on a daily basis or at an interval of every 13 days.
# This procedure will ensure that the 14-day data retention window is rigorously maintained.
# Primary Table for Storage Consumption for Day
df_storage_usage_data = fabric.read_table(workspace=Fabric_Capacity_WorkspaceId, dataset=get_dataset, table="StorageByWorkspacesandDay")

# Primary Table for Worksapce List
df_workspace_data = fabric.read_table(workspace=Fabric_Capacity_WorkspaceId, dataset=get_dataset, table="Workspaces")
```


We will convert the pandas DataFrame to a Spark DataFrame to ensure the data is properly inserted using Spark APIs for the Fabric Data Warehouse.
```
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("FabricMetrics").getOrCreate()

#Convert a Pandas DataFrame to a PySpark DataFrame, as the dataframes retrieved from sempy.fabric are initially in Pandas format.
df_storage_usage_data_spark = spark.createDataFrame(df_storage_usage_data)
df_workspace_data_spark = spark.createDataFrame(df_workspace_data)
```

Storage Usage Inserts Section
This section is very important, as the section without comments will only be executed the first time.
Afterwards, the section below marked with ⬇️ should be executed for all subsequent runs.
This approach ensures that there are no duplicate records and only new inserts are added to the table.
Please adjust the table name if needed, but all other parameters should maintain their respective variables.
```
# Process Storage Usage Inserts
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

#⚠️ Warning:** THIS IS IMPORTANT.
#⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
# The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
df_storage_usage_data_spark.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricStorageUage")

# # #⚠️ Warning:** THIS IS IMPORTANT.
# #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack not the above!⬇️
# # Delta Section 
# #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
# comparison_columns = ["Date", "PremiumCapacityId", "StaticStorageInGb", "OperationName","WorkloadKind" ]  #Using following columns as a unique key for  join

# #Step 1: Read existing data from the Fabric Warehouse
# df_current_storage_table = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricStorageUage")


# #Step 2: Identify new records using left_anti on multiple columns above
# df_new_storage_insert = df_storage_usage_data_spark.join(df_current_storage_table, comparison_columns, "left_anti")

# #Step 3: Append only new records to Fabric Warehouse for each invocation
# df_new_storage_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricStorageUage")
```


Process Workspace Table Inserts Section
This section is very important, as the section without comments will only be executed the first time.
Afterwards, the section below marked with ⬇️ should be executed for all subsequent runs.
This approach ensures that there are no duplicate records and only new inserts are added to the table.
Please adjust the table name if needed, but all other parameters should maintain their respective variables.
This procedure has been further optimized through the integration of the FabricWorkspacesList process, as detailed in the capacity metrics extraction documentation. For scenarios requiring traceability of Workspace name changes, a join with the table provided below enables comprehensive visibility into these modifications.
```
# Process Workspace Table Inserts
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

#⚠️ Warning:** THIS IS IMPORTANT.
#⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
# The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
df_workspace_data_spark.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkspaces")

# #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
# #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
# comparison_columns = ["WorkspaceId", "WorkspaceKey", "WorkspaceName"]  #Using following columns as a unique key for  join

# #Step 1: Read existing data from the Fabric Warehouse
# df_current_workspace_table = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkspaces") # Update Table Name as needed


# #Step 2: Identify new records using left_anti on multiple columns above
# df_new_workspace_insert = df_workspace_data_spark.join(df_current_workspace_table, comparison_columns, "left_anti")

# #Step 3: Append only new records to Fabric Warehouse for each invocation
# df_new_workspace_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkspaces") # Update Table Name as needed
```

ℹ️ Make sure to schedule a job that runs the above Python notebook daily. 
This is important because after 14 days, the older data will be purged, as the metrics have a 14-day retention period.


**SQL Code Steps: Fabric Storage Usage Percentabe Query:**
A percentage output for each day and related workspace will be the output. You can potentially utilize this percentage and divide it by the daily cost via the Azure Portal to implement the chargeback process.
Update the below table names to reflect your environment. Please note that, depending on your billing configuration, you may need to segment the query by capacity usage rather than aggregating it for the entire tenant. Please be advised that this query has been optimized to exclusively return the most recent workspace name, ensuring accurate identification even in cases where name changes have occurred. This optimization is implemented by leveraging the FabricWorkspacesList table, as sourced from the capacity metrics extraction documentation.
```
WITH DailyStorage AS (
    SELECT 
        Date,
        SUM([Utilization (GB)]) AS TotalTenantStorageForDay
    FROM 
        [dbo].[FabricStorageUage]
    GROUP BY 
        Date
),
WorkspaceDailyUsage AS (
    SELECT 
        WorkspaceId,
        Date,
        SUM([Utilization (GB)]) AS WorkspaceStorageForDay
    FROM 
        [dbo].[FabricStorageUage]
    GROUP BY 
        WorkspaceId, 
        Date
)
SELECT 
	  wks.Name,
    w.Date,
    w.WorkspaceId,
    w.WorkspaceStorageForDay,
    d.TotalTenantStorageForDay,
    (w.WorkspaceStorageForDay / NULLIF(d.TotalTenantStorageForDay, 0)) * 100 AS PercentageUsage
FROM 
    WorkspaceDailyUsage w  -- New Table removes duplicates
JOIN 
    DailyStorage d
ON 
    w.Date = d.Date
JOIN [dbo].[FabricWorkspacesList] wks
ON 
	upper(wks.Id)=upper(w.WorkspaceId)
ORDER BY 
    w.Date, 
    w.WorkspaceId;


```

Alternatively, if multiple workspace names are associated with the same workspace ID, it likely indicates that the workspace name has been updated. This change can impact the accuracy of the SQL statement referenced above.
The following SQL query is designed to aggregate all workspace names corresponding to a single workspace ID into one column, along with percentage-based aggregations to provide a quantifiable view of these occurrences.


```
WITH DailyStorage AS (
    SELECT 
        [Date],
        SUM([Utilization (GB)]) AS TotalTenantStorageForDay
    FROM dbo.FabricStorageUage
    GROUP BY [Date]
),
WorkspaceDailyUsage AS (
    SELECT 
        WorkspaceId,
        [Date],
        SUM([Utilization (GB)]) AS WorkspaceStorageForDay
    FROM dbo.FabricStorageUage
    GROUP BY WorkspaceId, [Date]
),
WorkspaceNames AS (
    -- 1) de-duplicate workspace names in the inner query as someone in your org may have updated them
    SELECT
      WorkspaceId,
      STRING_AGG(WorkspaceName, ', ') AS AllWorkspaceNames
    FROM (
      SELECT DISTINCT
        WorkspaceId,
        WorkspaceName
      FROM dbo.FabricWorkspaces
    ) AS dedup
    GROUP BY WorkspaceId
)
SELECT
    n.AllWorkspaceNames       AS WorkspaceName,
    w.Date,
    w.WorkspaceId,
    w.WorkspaceStorageForDay,
    d.TotalTenantStorageForDay,
    (w.WorkspaceStorageForDay  
       / NULLIF(d.TotalTenantStorageForDay, 0)
    ) * 100                  AS PercentageUsage
FROM WorkspaceDailyUsage AS w
JOIN DailyStorage        AS d ON w.Date        = d.Date
JOIN WorkspaceNames      AS n ON w.WorkspaceId = n.WorkspaceId
ORDER BY w.Date, w.WorkspaceId;

```




![image](https://github.com/user-attachments/assets/eabb6b71-8d52-4a85-a916-a4c983f6130b)


Lastly, consider utilizing the following formula for calculating departmental costs: ***AZURE COST FOR DAY / PercentageUsage = DAY COST PER DEPARTMENT***

***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***


