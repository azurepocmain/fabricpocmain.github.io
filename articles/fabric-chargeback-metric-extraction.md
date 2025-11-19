# Fabric Chargeback Metrics Extraction
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

As organizations deploy Fabric workspace capacity to multiple departments, it becomes imperative to implement tracking mechanisms for accurate inter-departmental billing. 
This optimal tool for this purpose is Fabric Chargeback: <a href="https://learn.microsoft.com/en-gb/fabric/release-plan/admin-governance#capacity-metrics-chargeback-public-preview" target="_blank">Fabric Chargeback</a>
The below will provide insights regarding long term extraction and retention of that data.

**Steps: Fabric Chargeback Metrics Extraction**

# *Step1:*
Firstly, download the Fabric Chargeback Reporting App by navigating to AppSource > Microsoft Fabric Chargeback Reporting and selecting "Get it now".

# *Step2:*
Ensure that a Fabric Data Warehouse has been established, as it will serve as the repository for the initial data load and subsequent delta updates.

# *Step3:*
If you want to automate this job, please review the following link, which contains details on how to add the script to a pipeline job and automate the process. The instructions below provide only a single notebook with multiple windows to extract the necessary data. <a href="https://azurepocmain.github.io/fabricpocmain.github.io/articles/fabric-capacity-metrics-extraction.html" target="_blank">Fabric Capacity Metrics Extraction</a>
# **Notebook 1**

``` 
import sempy.fabric as fabric

# GET THE WORKSPACE ID THAT HAS THE FABRIC CAPACITY METRIC
# First verify your Capacity workspace ID from the below. We should have the ability to access remote workspaces
capacity_workspace_check=fabric.list_workspaces()

display(capacity_workspace_check)
```


```
import sempy.fabric as fabric

# GET THE DATASET
# Next, using the FabricWorkspaceId from the above, add it to the FabricWorkspaceId variable to get the fabric capacity dataset 
# The UUID function should not be needed only there incase the workspace_id throws an exception ensure that you invoke the import as well
#from uuid import UUID
# Fabric_Capacity_WorkspaceId = UUID("work_space_id_here_if_the_below_fails_but_should_not_be_needed)  
Fabric_Capacity_WorkspaceId="capacity_workspace_id_here"              ###<----ADD
# No need to change in most cases unless you updated prior
DatasetName="Fabric Chargeback Reporting"
DatasetId="Dataset Id"

get_dataset=fabric.list_datasets(workspace=Fabric_Capacity_WorkspaceId, mode="rest")
get_dataset = get_dataset.loc[get_dataset['Dataset Name'] == DatasetName]
get_dataset_id=get_dataset['Dataset Id'].values[0] if not get_dataset.empty else None
# Get the column value without the column name
get_dataset = get_dataset['Dataset Name'].values[0]
display(get_dataset)
display(get_dataset_id)
```


```
import sempy.fabric as fabric

# Next get the Fabric Data Warehouse Name & ID from the Id column from the results below that you want to use. 
# Be sure to replace Add_Fabric_Workspace_Id_Where_Farbic_Data_Warehouse_Resides with the above workspace ID that has the Fabric DW.
get_fabric_dw_to_use=fabric.list_items(workspace="workspace_id_where_DW_lives"); ###<----ADD
# Only return DW so semantic model name wont confuse anyone as they have the same name 
get_fabric_dw_to_use=get_fabric_dw_to_use.loc[get_fabric_dw_to_use["Type"] == "Warehouse"]
display(get_fabric_dw_to_use)
```


```
#Fabric DW ID Assign 
# Assign the Fabric DW ID that will store the data
FabircWarehouse_WorkSpace_ID="workspace_id_where_DW_lives"           ###<----ADD
FabricWarehouseID="fabric_DW_ID"       ###<----ADD
FabricWarehouseName="fabric_dw_name_here"                  ###<----ADD
```



```
#Chargeback data
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException
import pandas as pd

client = fabric.FabricRestClient()
workspace_id =  Fabric_Capacity_WorkspaceId
dataset_id = get_dataset_id


url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/executeQueries"

dax_query = {
    "queries": [{
        "query": "EVALUATE 'Chargeback'"
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

df_chargeback_table = pd.DataFrame(rows)

```



```
#Workspace Data
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

df_capacities_table = pd.DataFrame(rows)

```



```
# Items Data
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
# Workspace Data
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

df_workspaces_table = pd.DataFrame(rows)
```

```
# Check if the pandas DataFrame is empty
if df_chargeback_table.empty:
    # Create an empty Spark DataFrame with schema
    df_chargeback_table_spark = spark.createDataFrame([], df_items_table_schema)
else:
    # Create Spark DataFrame from Pandas
    df_chargeback_table_spark = spark.createDataFrame(df_chargeback_table)
```


```
# Check if the pandas DataFrame is empty
if df_capacities_table.empty:
    # Create an empty Spark DataFrame with schema
    df_capacities_table_spark = spark.createDataFrame([], df_capacities_table_schema)
else:
    # Create Spark DataFrame from Pandas
    df_capacities_table_spark = spark.createDataFrame(df_capacities_table)
```

```
# Check if the pandas DataFrame is empty
if df_items_table.empty:
    # Create an empty Spark DataFrame with schema
    df_items_table_spark = spark.createDataFrame([], df_items_table_schema)
else:
    # Create Spark DataFrame from Pandas
    df_items_table_spark = spark.createDataFrame(df_items_table)
```

```
# Check if the pandas DataFrame is empty
if df_workspaces_table.empty:
    # Create an empty Spark DataFrame with schema
    df_workspaces_table_spark = spark.createDataFrame([], df_workspaces_table_schema)
else:
    # Create Spark DataFrame from Pandas
    df_workspaces_table_spark = spark.createDataFrame(df_workspaces_table)
```


```
# Process Chargeback Capacities Table Inserts
from pyspark.sql.functions import col
from pyspark.sql import functions as F
import com.microsoft.spark.fabric
from pyspark.sql.types import NullType, StringType
from com.microsoft.spark.fabric.Constants import Constants  
from pyspark.sql.functions import col

if not df_capacities_table_spark.isEmpty():

    df_capacities_table_spark_null = [f.name for f in df_capacities_table_spark.schema.fields if isinstance(f.dataType, NullType)]
    df_capacities_table_spark = df_capacities_table_spark.select(*[
    F.col(c).cast(StringType()) if c in df_capacities_table_spark_null else F.col(c)
    for c in df_capacities_table_spark.columns
    ])

    

    rename_map = {
        "Capacities[Capacity Id]": "CapacityId",
        "Capacities[Core count]": "CapacityCoreCount",
        "Capacities[Region]": "CapacityRegion",
        "Capacities[Capacity name]": "CapacityName",
        "Capacities[SKU]": "CapacitiesSKU"
    }

    df_capacities_table_spark_rename = df_capacities_table_spark

    for old, new in rename_map.items():
        if old in df_capacities_table_spark_rename.columns:
            df_capacities_table_spark_rename = df_capacities_table_spark_rename.withColumnRenamed(old, new)




    #⚠️ Warning:** THIS IS IMPORTANT.
    #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    df_capacities_table_spark_rename.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacitiesCB")
 
    # #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
    # #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
    # comparison_columns = ["CapacityId", "CapacityCoreCount", "CapacityRegion", "CapacityName", "CapacitiesSKU"]  #Using following columns as a unique key for  join

    # #Step 1: Read existing data from the Fabric Warehouse
    # df_chargebackcapacitiestable = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacitiesCB") # Update Table Name as needed


    # #Step 2: Identify new records using left_anti on multiple columns above
    # df_new_chargebackcapacities_insert = df_capacities_table_spark_rename.join(df_chargebackcapacitiestable, comparison_columns, "left_anti")

    # #Step 3: Append only new records to Fabric Warehouse for each invocation
    # df_new_chargebackcapacities_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricCapacitiesCB") # Update Table Name as needed
```

```
# Process Chargeback Items Table Inserts
from pyspark.sql.functions import col
from pyspark.sql import functions as F
import com.microsoft.spark.fabric
from pyspark.sql.types import NullType, StringType
from com.microsoft.spark.fabric.Constants import Constants  
from pyspark.sql.functions import col

if not df_items_table_spark.isEmpty():

    df_items_table_spark_null = [f.name for f in df_items_table_spark.schema.fields if isinstance(f.dataType, NullType)]
    df_items_table_spark = df_items_table_spark.select(*[
    F.col(c).cast(StringType()) if c in df_items_table_spark_null else F.col(c)
    for c in df_items_table_spark.columns
    ])

    

    rename_map = {
        "Items[Item Id]": "ItemId",
        "Items[Item kind]": "ItemKind",
        "Items[Item name]": "ItemName",
        "Items[Virtualised item]": "VirtualisedItem",
        "Items[Virtualised workspace]": "VirtualisedWorkspace"
    }

    df_items_table_spark_rename = df_items_table_spark

    for old, new in rename_map.items():
        if old in df_items_table_spark_rename.columns:
            df_items_table_spark_rename = df_items_table_spark_rename.withColumnRenamed(old, new)




    #⚠️ Warning:** THIS IS IMPORTANT.
    #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    df_items_table_spark_rename.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItemsCB")
 
    # #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
    # #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
    # comparison_columns = ["ItemId", "ItemKind", "ItemName", "VirtualisedItem", "VirtualisedWorkspace"]  #Using following columns as a unique key for  join

    # #Step 1: Read existing data from the Fabric Warehouse
    # df_chargebackitemstable = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItemsCB") # Update Table Name as needed


    # #Step 2: Identify new records using left_anti on multiple columns above
    # df_new_chargeback_items_insert = df_items_table_spark_rename.join(df_chargebackitemstable, comparison_columns, "left_anti")

    # #Step 3: Append only new records to Fabric Warehouse for each invocation
    # df_new_chargeback_items_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricItemsCB") # Update Table Name as needed

```

```
# Process Chargeback Items Table Inserts
from pyspark.sql.functions import col
from pyspark.sql import functions as F
import com.microsoft.spark.fabric
from pyspark.sql.types import NullType, StringType
from com.microsoft.spark.fabric.Constants import Constants  
from pyspark.sql.functions import col

if not df_chargeback_table_spark.isEmpty():

    df_chargeback_table_spark_null = [f.name for f in df_chargeback_table_spark.schema.fields if isinstance(f.dataType, NullType)]
    df_chargeback_table_spark = df_chargeback_table_spark.select(*[
    F.col(c).cast(StringType()) if c in df_chargeback_table_spark_null else F.col(c)
    for c in df_chargeback_table_spark.columns
    ])

    

    rename_map = {
        "Chargeback[Capacity Id]": "CapacityId",
        "Chargeback[Date]": "Date",
        "Chargeback[Item Id]": "ItemId",
        "Chargeback[Billing type]": "BillingType",
        "Chargeback[Operation name]": "OperationName",
        "Chargeback[User]": "User",
        "Chargeback[Workspace Id]": "WorkspaceId",
        "Chargeback[CU (s)]": "CUs",
        "Chargeback[Duration (s)]": "Duration",
        "Chargeback[Operations]": "Operations",
        "Chargeback[Experience]": "Experience",
        "Chargeback[Domain unique key]": "DomainUniqueKey"
    }

    df_chargeback_table_spark_rename = df_chargeback_table_spark

    for old, new in rename_map.items():
        if old in df_chargeback_table_spark_rename.columns:
            df_chargeback_table_spark_rename = df_chargeback_table_spark_rename.withColumnRenamed(old, new)




    #⚠️ Warning:** THIS IS IMPORTANT.
    #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    df_chargeback_table_spark_rename.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricChargeBackCB")
 
    # #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
    # #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
    # comparison_columns = ["CapacityId", "Date", "ItemId", "BillingType", "OperationName", "User", "WorkspaceId", "CUs", "Duration", "DomainUniqueKey"]  #Using following columns as a unique key for  join

    # #Step 1: Read existing data from the Fabric Warehouse
    # df_chargebackchargetable = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricChargeBackCB") # Update Table Name as needed


    # #Step 2: Identify new records using left_anti on multiple columns above
    # df_new_chargebackcharge_insert = df_chargeback_table_spark_rename.join(df_chargebackchargetable, comparison_columns, "left_anti")

    # #Step 3: Append only new records to Fabric Warehouse for each invocation
    # df_new_chargebackcharge_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricChargeBackCB") # Update Table Name as needed

```

```
# Process Chargeback Items Table Inserts
from pyspark.sql.functions import col
from pyspark.sql import functions as F
import com.microsoft.spark.fabric
from pyspark.sql.types import NullType, StringType
from com.microsoft.spark.fabric.Constants import Constants  
from pyspark.sql.functions import col

if not df_workspaces_table_spark.isEmpty():

    df_workspaces_table_spark_null = [f.name for f in df_workspaces_table_spark.schema.fields if isinstance(f.dataType, NullType)]
    df_workspaces_table_spark = df_workspaces_table_spark.select(*[
    F.col(c).cast(StringType()) if c in df_workspaces_table_spark_null else F.col(c)
    for c in df_workspaces_table_spark.columns
    ])

    

    rename_map = {
        "Workspaces[Workspace Id]": "WorkspaceId",
        "Workspaces[Workspace name]": "WorkspaceName"
    }

    df_workspaces_table_spark_rename = df_workspaces_table_spark

    for old, new in rename_map.items():
        if old in df_workspaces_table_spark_rename.columns:
            df_workspaces_table_spark_rename = df_workspaces_table_spark_rename.withColumnRenamed(old, new)




    #⚠️ Warning:** THIS IS IMPORTANT.
    #⚠️INITIAL EXECUTION: Ensure this section is commented out after the initial run. This is VERY IMPORTANT or your data will have duplicate values!
    # The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
    df_workspaces_table_spark_rename.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkSpacesCB")
 
    # #⚠️⬇️Uncomment the below section after the first above run, all subsequent runs moving forward should use the below code stack⬇️ 
    # #Columns utilized for comparison to ensure that only new delta records are inserted, using unique keys
    # comparison_columns = ["WorkspaceId", "WorkspaceName"]  #Using following columns as a unique key for  join

    # #Step 1: Read existing data from the Fabric Warehouse
    # df_chargebackworkspacestable = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkSpacesCB") # Update Table Name as needed


    # #Step 2: Identify new records using left_anti on multiple columns above
    # df_new_workspaces_insert = df_workspaces_table_spark_rename.join(df_chargebackworkspacestable, comparison_columns, "left_anti")

    # #Step 3: Append only new records to Fabric Warehouse for each invocation
    # df_new_workspaces_insert.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.FabricWorkSpacesCB") # Update Table Name as needed
```



***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***




















