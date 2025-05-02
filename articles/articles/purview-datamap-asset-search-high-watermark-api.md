# Purview DataMap Asset Search High Watermark API
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Organizations utilizing Microsoft Purview for data governance and asset management often require mechanisms to monitor and audit artifacts generated within Microsoft Fabric or other data services scanned by Purview. 
This document provides a comprehensive guide on leveraging Microsoft Fabric Notebooks to query assets within Purview DataMaps and automate the delivery of the results via email to designated stakeholders.

***Steps***

***Step1:***
Establish a service principal and configure Purview API permissions, specifically granting access to the Purview Application API. (Further testing is required for complete validation we will update.)


***Step2:***
Create an Azure Key Vault to store the service principal metadata. 

***Step3:***
Create a secret for your service principal and store the tenant_id, client_id, and client_secret in your key vault. 
Keep track of the names of the metadata and the key vault name. 


***Step4:***
Initialize a Microsoft Fabric Notebook and insert the following code into a designated cell for each script block. 
Ensure that you input the corresponding KeyVault name and reference the precise secret names where the service principal metadata is stored.


```
import requests
import time
import pandas as pd
from azure.identity import ClientSecretCredential
from notebookutils.credentials import getSecret

# URL of your Key Vault
key_vault_url = 'https://you_keyvault_url_here.vault.azure.net/'


# Retrieve the secrets
tenant_id = getSecret(key_vault_url, "tenant-id")
client_id = getSecret(key_vault_url, "pruview-service-princiapl-id")
client_secret = getSecret(key_vault_url, "pruview-service-principal-key")
PURVIEW_ACCOUNT_NAME=getSecret(key_vault_url, "purview-account-name")

# --- Auth Setup ---
credential = ClientSecretCredential(
    tenant_id=tenant_id,
    client_id=client_id,
    client_secret=client_secret
)

token = credential.get_token("https://purview.azure.net/.default").token

headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json"
}

search_url = f"https://{PURVIEW_ACCOUNT_NAME}.purview.azure.com/datamap/api/search/query?api-version=2023-09-01"


# --- Pagination Loop --- Or you wiil miss records using the continuation token key to verify additional data before stopping 
all_assets = []
continuation_token = None
limit = 30  # Max is 30 from my testing

while True:
    query = {
        "keywords": "",
        "limit": limit
    }

    if continuation_token:
        query["continuationToken"] = continuation_token

    response = requests.post(search_url, headers=headers, json=query)
    if response.status_code != 200:
        print("Error:", response.status_code, response.text)
        break

    results = response.json()
    batch = results.get("value", [])
    all_assets.extend(batch)

    continuation_token = results.get("continuationToken")
    if not continuation_token:
        break  # No more pages

# ---Convert to DataFrame ---
df_purview_assets = pd.DataFrame(all_assets)
print(f"Retrieved {len(df_purview_assets)} assets")
df_purview_assets

```

```
#Fabric DW ID Assign 
# Assign the Fabric DW ID that will store the data
FabircWarehouse_WorkSpace_ID="Worksspace_ID_Where_Fabric_DW_Will_Reside"
FabricWarehouseID="Fabric_DW_ID_Where_Data_Will_Be_Stored"
FabricWarehouseName="Fabric_Warehouse_Name"
```

```
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, max as spark_max
from pyspark.sql.utils import AnalysisException
import py4j.protocol
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

# Get the max max_createTime from the DW to compare teh deltas of the API call
# Initialize Spark session (if not already)
spark = SparkSession.builder.appName("PurviewProcess").getOrCreate()

# This will prevent the entire code from failing if the table has yet to be created
try:
    max_createTime_fabric_dw = spark.read.option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).option(Constants.DatawarehouseId, FabricWarehouseID).synapsesql(f"{FabricWarehouseName}.dbo.Purview_Asset_Max_UpdateTime")


    # Get the max from the createTime column
    max_row = max_createTime_fabric_dw.selectExpr("MAX(max_create_time) AS max_create_time").collect()[0]
    max_create_time_dw = max_row["max_create_time"]

    print(f"Max create time: {max_create_time_dw}")

except py4j.protocol.Py4JJavaError as e:
    # Inspect error message for known failure pattern
    if "tds.read.error.FabricSparkTDSReadError" in str(e) or "doesn't have read access" in str(e):
        print("Table doesn't exist or access denied — skipping.")
        max_create_time_dw = None
    else:
        raise  # re-raise other exceptions


```


```
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, LongType
# Get the max createTime to store in the DW from the above API call
# Initialize Spark session (if not already)
spark = SparkSession.builder.appName("PurviewProcess").getOrCreate()

# Get the max_createTime from pandas API above this is not the DW BUT API CALL 
max_createTime = df_purview_assets["createTime"].max()

# Create a DataFrame with one row and one column
data = [(max_createTime,)]
schema = StructType([StructField("max_create_time", LongType(), False)])

max_createTime_spark = spark.createDataFrame(data, schema)

max_createTime_spark.show()

```


```
# Store the max createTime to a Fabric DW
from pyspark.sql.functions import col
import com.microsoft.spark.fabric
from com.microsoft.spark.fabric.Constants import Constants  

# The table will be auto-created; adjust the table name as necessary, the variable above will be used for the below.
max_createTime_spark.write.mode("append").option(Constants.WorkspaceId, FabircWarehouse_WorkSpace_ID).synapsesql(f"{FabricWarehouseName}.dbo.Purview_Asset_Max_UpdateTime")


```


```
# Only return the deltas from the high watermark
# Filter records where createTime are newer 
df_purview_assets_deltas = df_purview_assets[df_purview_assets["createTime"] > max_create_time_dw]

print(f"Filtered {len(df_purview_assets_deltas)} new records.")

```


```
# We will convert the Unix aka epoch Timestamp h into a normal timestamp so that its easy on the eyes 
df_purview_assets_deltas["createTime_dt"] = pd.to_datetime(df_purview_assets_deltas["createTime"], unit='ms')

```

```
# We will convert the dataframe to an HTML to show up nicely in the email
md_string = df_purview_assets.to_html(index=False, escape=False)
```

```
# # Exit notebook value output for email input
mssparkutils.notebook.exit(md_string)
```

***Step5:***
Create a pipeline and drag the three activities below the mail activity will go into the IF condition: 

![image](https://github.com/user-attachments/assets/c4f48510-aa16-4e05-99d6-0c191ef0ca2c)

Add the following expression into the IF condition: 

![image](https://github.com/user-attachments/assets/da35dc65-0fbf-4e39-b74e-246a7106cf3e)

![image](https://github.com/user-attachments/assets/4c18f717-7df0-4afc-b249-46f2577993c9)

This will ensure that no blank emails are sent if artifacts are not created for that job run. 
Please note you can add an and condition with the same condition but different string to check for other strings like Synapse if you want this job and API to consolidate the process of both artifacts.

```
@contains(activity('NotebookPurviewAPI').output.result.exitValue, 'Fabric')

```

***Step6:***
In the notebook, select the "Settings" and the notebook that was created in Step3. 

![image](https://github.com/user-attachments/assets/50a096da-3194-4911-81d6-5e0b7bf5fcc8)



***Step7:***
In the Email activity, login, and select the recipients and subject. 
![image](https://github.com/user-attachments/assets/6f66d44b-a231-45f2-9236-08490d96b95f)

***Step8:***
For the body, go to the bottom of the page and select "View in expression builder" and add the below expression and ensure no other characters are added: 

![image](https://github.com/user-attachments/assets/c469bea9-0cc3-4e16-a9cf-000c4f2d02bd)

```
@activity('NotebookPurviewAPI').output.result.exitValue
```

![image](https://github.com/user-attachments/assets/5064921b-0969-4169-b673-6f76f427110f)

![image](https://github.com/user-attachments/assets/5ea8c26b-919b-45ae-9b54-9c1d5916e198)


***Step9:***
Run the pipeline to confirm the behavior, then setup a run schedule.
The output should look like the below: 

![image](https://github.com/user-attachments/assets/771efe7d-0ded-4fc4-a200-3776628d40d7)



***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED “AS IS” WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***






