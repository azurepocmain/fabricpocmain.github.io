# Fabric Capacity Metrics Extraction
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

***Update Log:***
- ***8-13-2025*** The recent update to the Fixed Capacity Metric application (version 40) introduced an issue affecting the read table function for required tables.

- ***8-6-2025***

- ***7-11-2025***

- ***7-8-2025***

- ***6-30-2025***

- ***6-24-2025***
  
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


# **PySpark Code Steps: Fabric Capacity Metrics Extraction:**

# **PySpark Code Steps: Modify Capacity from Pro to Fabric:**

# **Notebook 1**

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
DatasetId="Dataset Id"    # Getting Dataset ID info 

get_dataset=fabric.list_datasets(workspace=Fabric_Capacity_WorkspaceId, mode="rest")   # Added mode argument in light of new app change error
get_dataset = get_dataset.loc[get_dataset['Dataset Name'] == DatasetName]
get_dataset_id=get_dataset['Dataset Id'].values[0] if not get_dataset.empty else None     # Getting dataset id for the API call now
# Get the column value without the column name
get_dataset = get_dataset['Dataset Name'].values[0]
display(get_dataset)
display(get_dataset_id)
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

We are extracting the actual data from the dataset to store the values in the corresponding variables. This section was broken with the new Capacity method app update when using the python sempy read_table function and has been updated to use the API instead in light of the app change. 
```
# Get Metrics
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException
import pandas as pd

client = fabric.FabricRestClient()
workspace_id =  Fabric_Capacity_WorkspaceId
dataset_id = get_dataset_id


url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/executeQueries"

dax_query = {
    "queries": [{
        "query": "EVALUATE 'Metrics By Item and Day'"
    }],
    "serializerSettings": {"includeNulls": True}
}

headers = {
    "Content-Type": "application/json",

}

rows = []
continuation_token = None

while True:
    query_payload = dax_query.copy()
    if continuation_token:
        query_payload["continuationToken"] = continuation_token

    resp = client.post(url, headers=headers, json=query_payload)
    result = resp.json()

    # Parse rows
    new_rows = result['results'][0]['tables'][0]['rows']
    rows.extend(new_rows)

    # Check for next page
    continuation_token = result['results'][0].get('continuationToken')
    if not continuation_token:
        break

df_metrics_by_item_day_table = pd.DataFrame(rows)

```

```
#Get Items
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException
import pandas as pd

client = fabric.FabricRestClient()
workspace_id =  Fabric_Capacity_WorkspaceId
dataset_id = get_dataset_id


url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/executeQueries"

dax_query = {
    "queries": [{
        "query": "EVALUATE 'Items'"
    }],
    "serializerSettings": {"includeNulls": True}
}

headers = {
    "Content-Type": "application/json",

}

rows = []
continuation_token = None

while True:
    query_payload = dax_query.copy()
    if continuation_token:
        query_payload["continuationToken"] = continuation_token

    resp = client.post(url, headers=headers, json=query_payload)
    result = resp.json()

    # Parse rows
    new_rows = result['results'][0]['tables'][0]['rows']
    rows.extend(new_rows)

    # Check for next page
    continuation_token = result['results'][0].get('continuationToken')
    if not continuation_token:
        break

df_items_table = pd.DataFrame(rows)

```

```
# Get workspaces
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException
import pandas as pd

client = fabric.FabricRestClient()
workspace_id =  Fabric_Capacity_WorkspaceId
dataset_id = get_dataset_id


url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/executeQueries"

dax_query = {
    "queries": [{
        "query": "EVALUATE 'Workspaces'"
    }],
    "serializerSettings": {"includeNulls": True}
}

headers = {
    "Content-Type": "application/json",

}

rows = []
continuation_token = None

while True:
    query_payload = dax_query.copy()
    if continuation_token:
        query_payload["continuationToken"] = continuation_token

    resp = client.post(url, headers=headers, json=query_payload)
    result = resp.json()

    # Parse rows
    new_rows = result['results'][0]['tables'][0]['rows']
    rows.extend(new_rows)

    # Check for next page
    continuation_token = result['results'][0].get('continuationToken')
    if not continuation_token:
        break

df_workspace_data = pd.DataFrame(rows)
```

```
# Get capacity Units Info
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException
import pandas as pd

client = fabric.FabricRestClient()
workspace_id =  Fabric_Capacity_WorkspaceId
dataset_id = get_dataset_id


url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/executeQueries"

dax_query = {
    "queries": [{
        "query": "EVALUATE 'CU Detail'"
    }],
    "serializerSettings": {"includeNulls": True}
}

headers = {
    "Content-Type": "application/json",

}

rows = []
continuation_token = None

while True:
    query_payload = dax_query.copy()
    if continuation_token:
        query_payload["continuationToken"] = continuation_token

    resp = client.post(url, headers=headers, json=query_payload)
    result = resp.json()

    # Parse rows
    new_rows = result['results'][0]['tables'][0]['rows']
    rows.extend(new_rows)

    # Check for next page
    continuation_token = result['results'][0].get('continuationToken')
    if not continuation_token:
        break

df_capacity_units_details = pd.DataFrame(rows)

```


```
# Get Capacities Info
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException
import pandas as pd

client = fabric.FabricRestClient()
workspace_id =  Fabric_Capacity_WorkspaceId
dataset_id = get_dataset_id


url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/executeQueries"

dax_query = {
    "queries": [{
        "query": "EVALUATE 'Capacities'"
    }],
    "serializerSettings": {"includeNulls": True}
}

headers = {
    "Content-Type": "application/json",

}

rows = []
continuation_token = None

while True:
    query_payload = dax_query.copy()
    if continuation_token:
        query_payload["continuationToken"] = continuation_token

    resp = client.post(url, headers=headers, json=query_payload)
    result = resp.json()

    # Parse rows
    new_rows = result['results'][0]['tables'][0]['rows']
    rows.extend(new_rows)

    # Check for next page
    continuation_token = result['results'][0].get('continuationToken')
    if not continuation_token:
        break

df_capacities = pd.DataFrame(rows)

```


We will convert the pandas DataFrame to a Spark DataFrame to ensure the data is properly inserted using Spark APIs for the Fabric Data Warehouse.
Updated this to prevent empty dataframes from throwing an exception.
```
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("FabricMetrics").getOrCreate()

#Convert a Pandas DataFrame to a PySpark DataFrame, as the dataframes retrieved from sempy.fabric are initially in Pandas format.

# Define df_metrics_by_item_day_table to prevent exceptions
from pyspark.sql.types import StructType, StructField, StringType, FloatType, LongType, TimestampType

df_metrics_by_item_day_table_schema = StructType([
    StructField("DateTime", TimestampType(), True),
    StructField("Date", TimestampType(), True),
    StructField("PremiumCapacityId", StringType(), True),
    StructField("ItemId", StringType(), True),
    StructField("sum_CU", FloatType(), True),
    StructField("sum_duration", FloatType(), True),
    StructField("count_operations", LongType(), True),
    StructField("count_users", LongType(), True),
    StructField("percentile_DurationMs_50", FloatType(), True),
    StructField("percentile_DurationMs_90", FloatType(), True),
    StructField("avg_DurationMS", FloatType(), True),
    StructField("Throttling (min)", FloatType(), True),
    StructField("count_failure_operations", LongType(), True),
    StructField("count_rejected_operations", LongType(), True),
    StructField("count_successful_operations", LongType(), True),
    StructField("count_InProgress_operations", LongType(), True),
    StructField("count_cancelled_operations", LongType(), True),
    StructField("count_Invalid_operations", LongType(), True),
    StructField("WorkspaceId", StringType(), True),
    StructField("UniqueKey", StringType(), True)
])

# Check if the pandas DataFrame is empty
if df_metrics_by_item_day_table.empty:
    # Create an empty Spark DataFrame with schema
    df_metrics_by_item_spark = spark.createDataFrame([], df_metrics_by_item_day_table_schema)
else:
    # Create Spark DataFrame from Pandas
    df_metrics_by_item_spark = spark.createDataFrame(df_metrics_by_item_day_table)


# Define df_items_table to prevent exceptions
from pyspark.sql.types import StructType, StructField, StringType, TimestampType

df_items_table_schema = StructType([
    StructField("capacityId", StringType(), True),
    StructField("ItemId", StringType(), True),
    StructField("ItemKind", StringType(), True),
    StructField("ItemName", StringType(), True),
    StructField("dcount_Identity", StringType(), True),
    StructField("Timestamp", TimestampType(), True),
    StructField("WorkspaceId", StringType(), True),
    StructField("WorkspaceName", StringType(), True),
    StructField("Billable type", StringType(), True),
    StructField("IsVirtualArtifactName", StringType(), True),
    StructField("IsVirtualWorkspaceName", StringType(), True),
    StructField("IsVirtualArtifactStatus", StringType(), True),
    StructField("IsVirtualWorkspaceStatus", StringType(), True),
    StructField("UniqueKey", StringType(), True),
    StructField("ItemKey", StringType(), True)
])



# Check if the pandas DataFrame is empty
if df_items_table.empty:
    # Create an empty Spark DataFrame with schema
    df_items_table_spark = spark.createDataFrame([], df_items_table_schema)
else:
    # Create Spark DataFrame from Pandas
    df_items_table_spark = spark.createDataFrame(df_items_table)


# Define df_workspace_data to prevent exceptions 
from pyspark.sql.types import StructType, StructField, StringType, BooleanType

df_workspace_data_schema = StructType([
    StructField("WorkspaceId", StringType(), True),
    StructField("WorkspaceKey", StringType(), True),
    StructField("WorkspaceName", StringType(), True),
    StructField("PremiumCapacityId", StringType(), True),
    StructField("WorkspaceProvisionState", StringType(), True)

])


# Check if the pandas DataFrame is empty
if df_workspace_data.empty:
    # Create an empty Spark DataFrame with schema
    df_workspace_data_spark = spark.createDataFrame([], df_workspace_data_schema)   
else:
    # Create Spark DataFrame from Pandas
    df_workspace_data_spark= spark.createDataFrame(df_workspace_data)  ## Added for FabricWorkspaces table to show active workspace flags in the event workspace name has been altered




# Define capacity_units_details to prevent exceptions
from pyspark.sql.types import StructType, StructField, StringType, FloatType, LongType, TimestampType
capacity_units_details_schema = StructType([
    StructField("WindowStartTime", TimestampType(), True),
    StructField("Interactive", FloatType(), True),
    StructField("Background", FloatType(), True),
    StructField("InteractivePreview", FloatType(), True),
    StructField("BackgroundPreview", FloatType(), True),
    StructField("Interactive Delay %", FloatType(), True),
    StructField("Interactive Rejection %", FloatType(), True),
    StructField("Background Rejection %", FloatType(), True),
    StructField("AutoScaleCapacityUnits", LongType(), True),
    StructField("StartOfHour", TimestampType(), True),
    StructField("CUs", FloatType(), True),
    StructField("BaseCapacityUnits", LongType(), True),
    StructField("WindowEndTime", TimestampType(), True),
    StructField("Threshold", FloatType(), True),
    StructField("StartOf6min", TimestampType(), True),
    StructField("Peak6minInteractive", FloatType(), True),
    StructField("Peak6minBackground", FloatType(), True),
    StructField("Peak6minInteractivePreview", FloatType(), True),
    StructField("Peak6minBackgroundPreview", FloatType(), True),
    StructField("Peak6min Interactive Delay %", FloatType(), True),
    StructField("Peak6min Interactive Rejection %", FloatType(), True),
    StructField("Peak6min Background Rejection %", FloatType(), True),
    StructField("Start of Hour", TimestampType(), True),
    StructField("CU Limit", FloatType(), True),
    StructField("SKU", StringType(), True)
])

# Check if the pandas DataFrame is empty
if df_capacity_units_details.empty:
    # Create an empty Spark DataFrame with schema
    df_capacity_units_details_spark = spark.createDataFrame([], capacity_units_details_schema)
else:
    # Create Spark DataFrame from Pandas
    df_capacity_units_details_spark = spark.createDataFrame(df_capacity_units_details) # Added to track capacity size

```


Function that will be used to remove extra brackets and table name from the names: 
```
import re
from functools import reduce

def strip_brackets(colname):
    # Extract everything inside [ ]
    match = re.search(r"\[(.*)\]", colname)
    return match.group(1) if match else colname

```




Capacity Metrics Usage Inserts Section
This section is very important, as the section without comments will only be executed the first time.
Afterwards, the section below marked with ⬇️ should be executed for all subsequent runs.
This approach ensures that there are no duplicate records and only new inserts are added to the table.
Please adjust the table name if needed, but all other parameters should maintain their respective variables.
```
# Process Capacity Metrics Inserts
from pyspark.sql.functions import col
from pyspark.sql import functions as F
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

if not df_metrics_by_item_spark.isEmpty():
    old_cols = df_metrics_by_item_spark.columns
    new_cols = [strip_brackets(c) for c in old_cols]

    df_metrics_by_item_spark_df_renamed = reduce(
        lambda temp_df, idx: temp_df.withColumnRenamed(old_cols[idx], new_cols[idx]),
        range(len(old_cols)),
        df_metrics_by_item_spark
    )
    
    overrides = {
    "Datetime": "DateTime",
    "Date": "Date",
    "Capacity Id": "PremiumCapacityId",
    "Item Id": "ItemId",
    "CU (s)": "sum_CU",
    "Duration (s)": "sum_duration",
    "Operations": "count_operations",
    "Users": "count_users",
    "Percentile duration (ms) 50": "percentile_DurationMs_50",
    "Percentile duration (ms) 90": "percentile_DurationMs_90",
    "Avg duration (ms)": "avg_DurationMS",
    "Throttling (min)": "Throttling (min)",
    "Failed operations": "count_failure_operations",
    "Rejected operations": "count_rejected_operations",
    "Successful operations": "count_successful_operations",
    "Inprogress operations": "count_InProgress_operations",
    "Cancelled operations": "count_cancelled_operations",
    "Invalid operations": "count_invalid_operations",
    "Workspace Id": "WorkspaceId",
    "Unique key": "UniqueKey",
    }

    #  Rename 
    df_metrics_by_item_spark_df_renamed = df_metrics_by_item_spark_df_renamed.select([F.col(c).alias(overrides[c]) for c in df_metrics_by_item_spark_df_renamed.columns])
    
    #⚠️ Warning:** THIS IS IMPORTANT.
    #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    df_metrics_by_item_spark_df_renamed.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityMetrics")

    # #⚠️ Warning:** THIS IS IMPORTANT.
    # #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack not the above!⬇️
    # # Delta Section 
    # #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
    # comparison_columns = ["DateTime", "PremiumCapacityId", "ItemId", "sum_CU", "sum_duration", "WorkspaceId", "UniqueKey" ]  #Using following columns as a unique key for  join

    # #Step 1: Read existing data from the Fabric Warehouse
    # df_current_metric_table = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityMetrics")


    # #Step 2: Identify new records using left_anti on multiple columns above
    # df_new_metric_insert = df_metrics_by_item_spark_df_renamed.join(df_current_metric_table, comparison_columns, "left_anti")

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

if not df_items_table_spark.isEmpty():
    old_cols = df_items_table_spark.columns
    new_cols = [strip_brackets(c) for c in old_cols]

    df_items_table_spark_df_renamed = reduce(
        lambda temp_df, idx: temp_df.withColumnRenamed(old_cols[idx], new_cols[idx]),
        range(len(old_cols)),
        df_items_table_spark
    )


    overrides = {
    "Capacity Id":                 "capacityId",
    "Item Id":                     "ItemId",
    "Item kind":                   "ItemKind",
    "Item name":                   "ItemName",
    "Users":                       "dcount_Identity",
    "Timestamp":                   "Timestamp",
    "Workspace Id":                "WorkspaceId",
    "Workspace name":              "WorkspaceName",
    "Billable type":               "Billable type",
    "Virtualised item":            "IsVirtualArtifactName",
    "Virtualised workspace":       "IsVirtualWorkspaceName",
    "Is virtual item status":      "IsVirtualArtifactStatus",
    "Is virtual workspace status": "IsVirtualWorkspaceStatus",
    "Unique key":                  "UniqueKey",
    "Item key":                    "ItemKey",
    }

    #  Rename 
    df_items_table_spark_df_renamed = df_items_table_spark_df_renamed.select([F.col(c).alias(overrides[c]) for c in df_items_table_spark_df_renamed.columns])

    #⚠️ Warning:** THIS IS IMPORTANT.
    #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    # df_items_table_spark_df_renamed.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItems")

    #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
    #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
    comparison_columns = ["capacityId", "ItemId", "ItemKind", "ItemName", "Timestamp", "WorkspaceId", "UniqueKey" ]  #Using following columns as a unique key for  join

    #Step 1: Read existing data from the Fabric Warehouse
    df_items_table = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItems") # Update Table Name as needed


    #Step 2: Identify new records using left_anti on multiple columns above
    df_new_items_insert = df_items_table_spark_df_renamed.join(df_items_table, comparison_columns, "left_anti")

    #Step 3: Append only new records to Fabric Warehouse for each invocation
    df_new_items_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItems") # Update Table Name as needed
```

⚠️ Warning ONLY RUN THIS ONCE IF YOU ALREADY HAVE DATA IN THE TABLE IN THE DATA WAREHOUSE. THIS WILL NEVER BE RUN AGAIN AS THE PYSPARK LOGIC WILL TAKE CARE OF THIS! 
⚠️ Please note that if you already have data stored in the table, you should run this update script to ensure that the new workspace data conforms to the correct active naming convention.

```
--Fabric DW table update IF needed one time script to update all records: 
--update FabricWorkspaces set WorkspaceProvisionState = 'Inactive' where WorkspaceProvisionState ='Active'

```


Leveraging the Fabric workspace table semantic model ensures that the environment consistently reflects the most up-to-date workspace nomenclature, thereby maintaining data integrity and alignment with the latest execution cycle.
⚠️ Please note that the first section has been commented out. 
```
# Process Workspace Table Inserts
from pyspark.sql.functions import col, lit, current_timestamp
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

if not df_workspace_data_spark.isEmpty():
    old_cols = df_workspace_data_spark.columns
    new_cols = [strip_brackets(c) for c in old_cols]

    df_workspace_data_spark_df_renamed = reduce(
        lambda temp_df, idx: temp_df.withColumnRenamed(old_cols[idx], new_cols[idx]),
        range(len(old_cols)),
        df_workspace_data_spark
    )

    overrides = {
    "Workspace Id":               "WorkspaceId",
    "Workspace key":              "WorkspaceKey",
    "Workspace name":             "WorkspaceName",
    "Capacity Id":                "PremiumCapacityId",
    "Workspace provision state":  "WorkspaceProvisionState",
    }

    #  Rename 
    df_workspace_data_spark_df_renamed = df_workspace_data_spark_df_renamed.select([F.col(c).alias(overrides[c]) for c in df_workspace_data_spark_df_renamed.columns])


    # #⚠️ Warning:** THIS IS IMPORTANT.
    # #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    #df_workspace_data_spark_df_renamed.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkspaces")


    # #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
    all_cols = ["WorkspaceId","WorkspaceKey","WorkspaceName","PremiumCapacityId","WorkspaceProvisionState"]


    # 1) Read current FabricWorkspaces table
    df_current = (spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID)
            .synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkspaces")
            .select(*all_cols)
    )

    # 2) Select new incoming data
    df_new = df_workspace_data_spark_df_renamed.select(*all_cols)

    # 3) Identify "already existing" exact matches (no change)
    df_existing_exact = df_current.join(df_new, on=all_cols, how="inner")

    # 4) Identify rows needing deactivation (ID+Key match but Name changed)
    df_to_deactivate = (
        df_current.alias("curr")
            .join(df_new.alias("new"), on=["WorkspaceId", "WorkspaceKey"], how="inner")
            .filter(
                (col("curr.WorkspaceName") != col("new.WorkspaceName")) &
                (col("curr.WorkspaceProvisionState") != lit("Inactive"))
            )
            .select("curr.*")
            .withColumn("WorkspaceProvisionState", lit("Inactive"))

    )


    # 5) Identify *truly new* rows (not already in full table)
    df_new_only = (
        df_new.alias("new")
            .join(df_current.alias("curr"), on=all_cols, how="left_anti")
    )


    # 6) Combine: rows to deactivate + truly new rows
    df_to_insert = df_to_deactivate.unionByName(df_new_only)

    # 7) Only write if there’s anything to insert
    if df_to_insert.count() > 0:
        df_to_insert.write \
            .mode("append") \
            .option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID) \
            .synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkspaces")
    else:
        print("✅ No changes needed — nothing inserted.")


```




This section provides detailed tracking of the capacity allocation, enabling more precise calculations and management of capacity size.

```
# Process Capacity Metrics Inserts
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

if not df_capacity_units_details_spark.isEmpty():
    old_cols = df_capacity_units_details_spark.columns
    new_cols = [strip_brackets(c) for c in old_cols]

    df_capacity_units_details_spark_df_renamed = reduce(
        lambda temp_df, idx: temp_df.withColumnRenamed(old_cols[idx], new_cols[idx]),
        range(len(old_cols)),
        df_capacity_units_details_spark
    )

    overrides = {
    "Window start time":                   "WindowStartTime",
    "Interactive":                         "Interactive",
    "Background":                          "Background",
    "Interactive non billable":            "InteractivePreview",
    "Background non billable":             "BackgroundPreview",
    "Interactive delay %":                 "Interactive Delay %",
    "Interactive rejection %":             "Interactive Rejection %",
    "Background rejection %":              "Background Rejection %",
    "Auto scale capacity units":           "AutoScaleCapacityUnits",
    "StartOfHour":                         "StartOfHour",
    "CU (s)":                              "CUs",
    "Base capacity units":                 "BaseCapacityUnits",
    "Window end time":                     "WindowEndTime",
    "Threshold":                           "Threshold",
    "Start of 6min":                       "StartOf6min",
    "Peak 6min interactive":               "Peak6minInteractive",
    "Peak 6min background":                "Peak6minBackground",
    "Peak 6min interactive non billable":  "Peak6minInteractivePreview",
    "Peak 6min background non billable":   "Peak6minBackgroundPreview",
    "Peak 6min interactive delay %":       "Peak6min Interactive Delay %",
    "Peak 6min interactive rejection %":   "Peak6min Interactive Rejection %",
    "Peak 6min background rejection %":    "Peak6min Background Rejection %",
    "Start of Hour":                       "Start of Hour",  # keep distinct if both exist
    "CU limit":                            "CU Limit",
    "SKU":                                 "SKU",
    }


    #  Rename 
    df_capacity_units_details_spark_df_renamed = df_capacity_units_details_spark_df_renamed.select([F.col(c).alias(overrides[c]) for c in df_capacity_units_details_spark_df_renamed.columns])

    #⚠️ Warning:** THIS IS IMPORTANT.
    #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    df_capacity_units_details_spark_df_renamed.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityUnitsDetails")

    # #⚠️ Warning:** THIS IS IMPORTANT.
    # #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack not the above!⬇️
    # # Delta Section 
    # #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
    # comparison_columns = ["WindowStartTime", "StartOfHour", "WindowEndTime", "Start of Hour", "SKU" ]  #Using following columns as a unique key for  join

    # #Step 1: Read existing data from the Fabric Warehouse
    # df_current_capacity_units_details_table = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityUnitsDetails")


    # #Step 2: Identify new records using left_anti on multiple columns above
    # df_new_capacity_units_details_insert = df_capacity_units_details_spark_df_renamed.join(df_current_capacity_units_details_table, comparison_columns, "left_anti")

    # #Step 3: Append only new records to Fabric Warehouse for each invocation
    # df_new_capacity_units_details_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacityUnitsDetails")

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
	wklst.WorkspaceName,
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
JOIN FabricWorkspaces wklst
ON upper(wklst.WorkspaceId)=upper(metics.WorkspaceId)
	WHERE item.[Billable type] IN ('Billable', 'Both') --Only getting the actual cost
    AND wklst.WorkspaceProvisionState='Active'  --Only Active Workspaces to remove duplicate entries 
GROUP BY 
    metics.WorkspaceId, wklst.WorkspaceName, CAST(metics.[Date] AS DATE), d.TotalDailyCU
ORDER BY 
    UsageDate, UsagePercentage DESC;
```
![image](https://github.com/user-attachments/assets/6b40ab34-33fd-4ff7-a867-097bce2d73a2)


In addition, consider utilizing the following formula for calculating departmental costs: ***AZURE COST FOR DAY / UsagePercentage = DAY COST PER DEPARTMENT***


# Pipeline API Call
To initiate the above pipeline via an API using a service principal, you can utilize code structured similarly to the following example:

```
import os
import requests
from dotenv import load_dotenv

load_dotenv()


def get_fabric_token() -> str:
    """Obtain an Azure AD access token for Microsoft Fabric API."""
    tenant_id = os.getenv("TENANT_ID")
    client_id = os.getenv("CLIENT_ID")
    client_secret = os.getenv("CLIENT_SECRET")

    token_url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"

    payload = {
        "client_id": client_id,
        "client_secret": client_secret,
        "grant_type": "client_credentials",
        "scope": "https://api.fabric.microsoft.com/.default"
    }

    try:
        response = requests.post(token_url, data=payload)
        response.raise_for_status()
        return response.json()["access_token"]
    except requests.RequestException as e:
        raise Exception(f"Failed to get token: {e}")


def start_pipeline(workspace_id: str, pipeline_id: str, run_name: str = "Run via Python"):
    """Start a Fabric pipeline by workspace and pipeline item ID."""
    token = get_fabric_token()

    url = (
        f"https://api.fabric.microsoft.com/v1/workspaces/"
        f"{workspace_id}/items/{pipeline_id}/jobs/instances?jobType=Pipeline"
    )

    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    payload = {
        "runName": run_name
    }

    try:
        response = requests.post(url, headers=headers, json=payload)
        response.raise_for_status()

        if response.status_code in [200, 201, 202]:
            print(f"Pipeline triggered successfully: {run_name}")
            #return response.json()
        else:
            raise Exception(f"Unexpected status code: {response.status_code}")

    except requests.RequestException as e:
        raise Exception(f"Failed to start pipeline: {e}")


if __name__ == "__main__":
    #My env parameters for the pipeline
    workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
    pipeline_id = os.getenv("FABRIC_PIPELINE_ID")    ## Pipeline to run from the above code. 
    run_name = "Triggered from most recent working script"

    if not workspace_id or not pipeline_id:
        raise Exception("Workspace ID and Pipeline ID must be set in .env")

    result = start_pipeline(workspace_id, pipeline_id, run_name)
    #print("Run ID:", result.get("id"))

```


***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***


