# Fabric Capacity Metrics Extraction
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

***Update Log:***
  
- ***6-20-2025***

- ***6-6-2025***

- ***3-12-2025***

As organizations deploy Fabric workspace capacity to multiple departments, it becomes imperative to implement tracking mechanisms for accurate inter-departmental billing. 
The optimal tool for this purpose is Fabric Chargeback: <a href="https://learn.microsoft.com/en-gb/fabric/release-plan/admin-governance#capacity-metrics-chargeback-public-preview" target="_blank">Fabric Chargeback</a>
However, since Fabric Chargeback is currently in public preview, this documentation will outline alternative methodologies utilizing Fabric Capacity Metrics for data extraction and storage to facilitate chargeback calculations.

**Steps: Fabric Capacity Metrics Extraction**

# *Step1:*
Firstly, download the Fabric Capacity Metrics App by navigating to AppSource > Microsoft Fabric Capacity Metrics and selecting "Get it now". The corresponding link is accessible here:  <a href="learn.microsoft.com/en-us/fabric/enterprise/metrics-app-install?tabs=1st" target="_blank">Install the Microsoft Fabric capacity metrics app</a>

# *Step2:*
Subsequently, configure the Capacity Metrics App by adhering to the instructions outlined in the aforementioned documentation. 
Navigate to the Microsoft Fabric Capacity Metrics workspace via the left pane link, proceed to “Workspace settings” located on the top right, and then select “License info” followed by the edit option. Ensure that  “Pro” is selected which should be the default. The assigned capacity (Pro) remains until the extraction process initiates. This procedure has been automated, ensuring a stable and reliable extraction of Capacity Metrics App data from the semantic model via altering the capacity from Pro to Fabric. This configuration is essential, as failing to use Fabric capacity during extraction may result in permission exceptions. This approach differs from the initial solution, which kept the Capacity Metrics App capacity permanently set to Fabric, a configuration that is no longer necessary. In addition, an iterative loop has been integrated to collect telemetry data from each capacity within the environment, allowing for comprehensive resource utilization analysis.
![image](https://github.com/user-attachments/assets/f320cdcf-275c-4598-8d11-be8be7b2a67a)


# *Step3:*
Finally, ensure that a Fabric Data Warehouse has been established, as it will serve as the repository for the initial data load and subsequent delta updates.

# *Step4:*
To ensure the API call accurately reflects updated capacity settings, initialize three distinct Spark notebooks utilizing PySpark. Each notebook should execute unique logic below, providing isolated session contexts necessary for the API to recognize capacity modifications. Attempting to combine the initial and final logic within a single script may prevent the session from detecting the newly applied API changes.
Execute the code steps as follows for the three notebooks:


**PySpark Code Steps: Fabric Capacity Metrics Extraction:**

**PySpark Code Steps: Modify Capacity from Pro to Fabric:**

**Notebook 1**

For the next step, create a new notebook and consider naming it something like “Capacity Metric Job Capacity Adjustment Notebook1.” Once created, add the script provided below. Be sure to modify the script variables as needed to fit your requirements.

```
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException

client = fabric.FabricRestClient()

# It is recommended to assign the workspace to a Fabric capacity at runtime, as alternative methods are susceptible to increased variability in operational behavior. 
# This configuration ensures that capacity remains in its default state until an explicit query or reallocation is required. 



# Get the workspace_id if needed and the capacity_id, assuming a workspace is already allocated to it. 
import sempy.fabric as fabric

# GET THE WORKSPACE ID THAT HAS THE FABRIC CAPACITY METRIC
# First verify your Capacity workspace ID from the below. We should have the ability to access remote workspaces
capacity_workspace_check=fabric.list_workspaces()

display(capacity_workspace_check)


# Assign the workspace_id of the Microsoft Fabric Capacity Metrics and the capacity_id that is part of the Fabric capacity it can be any size we will just use it for this process before switching back.
workspace_id="add_your_workspace_ID_here_for_the_capacity_metric_app"
capacity_id="assign_a_capacity_that_is_not_busy_to_the_workspace"


# Create the function to call the API to alter the capacity from Pro to Fabric

def assign_workspace_to_capacity(workspace_id, capacity_id):
   
    url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/AssignToCapacity"
    
    payload = {
        "capacityId": capacity_id
    }

    r = client.post(url, json=payload)
    if r.status_code == 200:
        print(f"Workspace {workspace_id} successfully moved to capacity {capacity_id}")
    else:
        print(f"Failed to assign workspace: {r.status_code} - {r.text}")
		
		
# Call the function with the parameters
assign_workspace_to_capacity(workspace_id, capacity_id)



# Get al the capcity IDs for environment to loop through
import sempy.fabric as fabric

# GET ALL THE CAPACITY_IDS
CapacityID_to_Process=fabric.list_workspaces()

display(CapacityID_to_Process)


# Remove Null 
CapacityID_to_Process=CapacityID_to_Process['Capacity Id'].dropna().unique()

# Validation
print(CapacityID_to_Process)



import json
import numpy as np


# Convert NumPy array to regular Python list
clean_list = CapacityID_to_Process.tolist()

# # Clean and Exit value for notebook for Loop
mssparkutils.notebook.exit(json.dumps(clean_list))		

```



# **PySpark Code Steps: Capacity ID Loop Task:**

# **Notebook 2**

This section of Notebook 3 implements logic to update the semantic model's assigned capacity ID, a critical dependency required to retrieve and display metadata for the selected capacity.
You will need the dataset name for the fabric metric app dataset in addition to the workdspace ID for the capacity metric app. 
You can use the below to get the dataset ID: 
```
import sempy.fabric as fabric

# GET THE DATASET
# Next, using the FabricWorkspaceId from the above, add it to the FabricWorkspaceId variable to get the fabric capacity dataset 
Fabric_Capacity_WorkspaceId="fabirc_capacity_workspace_id_here"
# No need to change in most cases unless you updated prior
DatasetName="Fabric Capacity Metrics"

get_dataset=fabric.list_datasets(workspace=Fabric_Capacity_WorkspaceId)

display(get_dataset) ## Add the dataset ID to the below code
```

```
# Modify the CapacityID to the next capacity on the capacity list
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException

client = fabric.FabricRestClient()

fabirc_capacity_worspace_id="capacity_app_workspace_id_here"
fabric_capacity_metric_app_data_set_id="capacity_app_data_set_id_here"

url = f"https://api.powerbi.com/v1.0/myorg/groups/{fabirc_capacity_worspace_id}/datasets/{fabric_capacity_metric_app_data_set_id}/Default.UpdateParameters"

payload = {
"updateDetails": [
    {
        "name": "CapacityID",
        "newValue": f"{capacity_id_to_extract_data_from}"
    }
]
}

r = client.post(url, json=payload)
if r.status_code == 200:
    print(r.status_code, r.text)
else:
    print(f"Failed to assign workspace: {r.status_code} - {r.text}")
    
    
```




# **PySpark Code Steps: Capacity Metric Specific Capacity Data Extraction ETL to Data Warehouse:**

# **Notebook 3**

Each code below will be a different cell in the same Notebook 2
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
# Primary Table for CU Consumption
df_metrics_by_item_day_table = fabric.read_table(workspace=Fabric_Capacity_WorkspaceId, dataset=get_dataset, table="MetricsByItemandDay")

# Primary Table for Worksapce Items
df_items_table = fabric.read_table(workspace=Fabric_Capacity_WorkspaceId, dataset=get_dataset, table="Items")
```


We will convert the pandas DataFrame to a Spark DataFrame to ensure the data is properly inserted using Spark APIs for the Fabric Data Warehouse.
```
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("FabricMetrics").getOrCreate()

#Convert a Pandas DataFrame to a PySpark DataFrame, as the dataframes retrieved from sempy.fabric are initially in Pandas format.
df_metrics_by_item_spark = spark.createDataFrame(df_metrics_by_item_day_table)
df_items_table_spark = spark.createDataFrame(df_items_table)
capacity_workspace_check_spark= spark.createDataFrame(capacity_workspace_check)   ## Added for FabricWorkspacesList table to only show latest workspace name in the event workspace name has been altered
```

Capacity Metrics Usage Inserts Section
This section is very important, as the section without comments will only be executed the first time.
Afterwards, the section below marked with ⬇️ should be executed for all subsequent runs.
This approach ensures that there are no duplicate records and only new inserts are added to the table.
Please adjust the table name if needed, but all other parameters should maintain their respective variables.
```
# Process Capacity Metrics Inserts
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

#⚠️ Warning:** THIS IS IMPORTANT.
#⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
# The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
df_metrics_by_item_spark.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityMetrics")

# #⚠️ Warning:** THIS IS IMPORTANT.
# #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack not the above!⬇️
# # Delta Section 
# #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
# comparison_columns = ["DateTime", "PremiumCapacityId", "ItemId", "sum_CU", "sum_duration", "WorkspaceId", "UniqueKey" ]  #Using following columns as a unique key for  join

# #Step 1: Read existing data from the Fabric Warehouse
# df_current_metric_table = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityMetrics")


# #Step 2: Identify new records using left_anti on multiple columns above
# df_new_metric_insert = df_metrics_by_item_spark.join(df_current_metric_table, comparison_columns, "left_anti")

# #Step 3: Append only new records to Fabric Warehouse for each invocation
# df_new_metric_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityMetrics")
```


Process Item Table Inserts Section
This section is very important, as the section without comments will only be executed the first time.
Afterwards, the section below marked with ⬇️ should be executed for all subsequent runs.
This approach ensures that there are no duplicate records and only new inserts are added to the table.
Please adjust the table name if needed, but all other parameters should maintain their respective variables.
```
# Process Item Table Inserts
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

#⚠️ Warning:** THIS IS IMPORTANT.
#⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
# The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
df_items_table_spark.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItems")

# #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
# #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
# comparison_columns = ["capacityId", "ItemId", "ItemKind", "ItemName", "Timestamp", "WorkspaceId", "UniqueKey" ]  #Using following columns as a unique key for  join

# #Step 1: Read existing data from the Fabric Warehouse
# df_current_metric_table = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItems") # Update Table Name as needed


# #Step 2: Identify new records using left_anti on multiple columns above
# df_new_metric_insert = df_items_table_spark.join(df_current_metric_table, comparison_columns, "left_anti")

# #Step 3: Append only new records to Fabric Warehouse for each invocation
# df_new_metric_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItems") # Update Table Name as needed
```


Leveraging the Fabric workspace list dataset ensures that the environment consistently reflects the most up-to-date workspace nomenclature, thereby maintaining data integrity and alignment with the latest execution cycle.
```
# Process Workspace Table FabricWorkspacesList Inserts
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

#⚠️Please note: Always use the initial execution below, as it will be overwritten with each run to ensure that the latest workspace names are reflected!
# The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
capacity_workspace_check_spark.write.mode("overwrite").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkspacesList")



```


# **PySpark Code Steps: Capacity ID Loop Task:**

# **Notebook 4**

```
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException

client = fabric.FabricRestClient()

# It is recommended to assign the workspace to a Fabric capacity at runtime, as alternative methods are susceptible to increased variability in operational behavior. 
# This configuration ensures that capacity remains in its default state until an explicit query or reallocation is required. 



# Get the workspace_id if needed and the capacity_id, assuming a workspace is already allocated to it. 
import sempy.fabric as fabric

# GET THE WORKSPACE ID THAT HAS THE FABRIC CAPACITY METRIC
# First verify your workspace ID from the below. 
capacity_workspace_check=fabric.list_workspaces()

display(capacity_workspace_check)


# Assign the workspace_id back to the default
workspace_id="capacity_metric_workspace_id_here"
capacity_id="00000000-0000-0000-0000-000000000000"   # Dont alter as this will set back to Pro capacity


# Create the function to call the API to alter the capacity from Fabric to Pro

def assign_workspace_to_capacity(workspace_id, capacity_id):
   
    url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/AssignToCapacity"
    
    payload = {
        "capacityId": capacity_id
    }

    r = client.post(url, json=payload)
    if r.status_code == 200:
        print(f"Workspace {workspace_id} successfully moved to capacity {capacity_id}")
    else:
        print(f"Failed to assign workspace: {r.status_code} - {r.text}")
		
		
# Call the function with the parameters
assign_workspace_to_capacity(workspace_id, capacity_id)

```

# **Pipeline Integration**
# **Put it all together**


To ensure seamless execution, the solution should be structured as follows: each section described above must be implemented in a separate notebook instance, as initiating a new session is necessary to accurately reflect the updated state resulting from API invocations.

![image](https://github.com/user-attachments/assets/d7aa750e-639d-4ca4-9453-aa4dca62e4a1)


The following screens depict the pipeline workflow and detail the necessary parameter configurations. Omitting any required parameters may result in job execution failures.

# **Activity 1 Notebook:** 

No parameters required besides Notebook 1 selected:

![image](https://github.com/user-attachments/assets/4f289ccc-570f-44ed-84a5-6366d4a3e9d1)


# **Activity 2 For Each Loop:**

To capture the output value from Notebook 1, utilize the following expression. Ensure that the activity name precisely matches the name of your initial activity, as defined within your environment configuration.
Ensure that the loop is set to sequential. 

```
@json(activity('NotebookAdjustCapacitytoFabric').output.result.exitValue)
```

![image](https://github.com/user-attachments/assets/3bc67529-1780-4ecb-9a13-26049105b323)


**Activity 3 Inside For Each Loop Notebook 3:**
For Notebook 3, which is responsible for updating the dataset capacity ID to ensure comprehensive capacity coverage, incorporate the specified base parameter expression name  `capacity_id_to_extract_data_from`  and value `@item()` within the loop activity. This configuration guarantees that the necessary value is programmatically passed from the Foreach activity into the Notebook, supporting robust and sequential execution across all capacity instances.

![image](https://github.com/user-attachments/assets/4c660758-2d48-43f7-8cd9-bd7ca845327c)


# **Activity 4 Inside For Each Loop Notebook 4:**

No parameters required besides Notebook 4 selected:

![image](https://github.com/user-attachments/assets/db265eee-d75a-4e83-87c8-2dbe54762696)


# **Activity 5 Outside For Each Loop Notebook 5:** 

Once processing is complete, the capacity metric application should be programmatically reverted to the Pro capacity setting. It is important to ensure that both "On Fail" and "On Success" dependencies from the ForEach loop are properly linked to the final notebook activity. This approach guarantees that, regardless of execution outcome, the Fabric capacity metric application consistently returns to its default configuration, thereby maintaining operational stability even in the event of capacity changes.

![image](https://github.com/user-attachments/assets/d1b01ea8-1a7d-48d7-874d-84faa4339f74)



ℹ️ Make sure to schedule a job that runs the above Python notebook daily.  
This is important because after 14 days, the older data will be purged, as the metrics have a 14-day retention period.


# **SQL Code Steps: Fabric Capacity Metrics Percentabe Query:**
Duplicate workspace names are systematically eliminated by executing a join with the FabricWorkspacesList table, which reliably retrieves the most current workspace names during each execution cycle.
A percentage output for each day and related workspace will be the output. You can potentially utilize this percentage and divide it by the daily cost via the Azure Portal to implement the chargeback process.
Update the below table names to reflect your environment.
```
WITH DailyTotal AS (
    --Calculate total CU usage per day across all workspaces potentially use the percentage to multiple the overall bill cost per owner of workspace
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
	wklst.Name as WorkspaceName,
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
JOIN FabricWorkspacesList wklst
ON upper(wklst.Id)=upper(metics.WorkspaceId)
	WHERE item.[Billable type] IN ('Billable', 'Both') --Only getting the actual cost
GROUP BY 
    metics.WorkspaceId, wklst.Name, CAST(metics.[Date] AS DATE), d.TotalDailyCU
ORDER BY 
    UsageDate, UsagePercentage DESC;
```
![image](https://github.com/user-attachments/assets/6b40ab34-33fd-4ff7-a867-097bce2d73a2)


Lastly, consider utilizing the following formula for calculating departmental costs: ***AZURE COST FOR DAY / UsagePercentage = DAY COST PER DEPARTMENT***

***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***


