# Service Principal Shortcut on a DataLake with Firewall Enabled via API and Object Level Security Overview
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Service principals and managed identities are essential components in streamlining automation processes within an organization. 
In Microsoft Fabric OneLake, shortcuts play a pivotal role in enabling disparate systems to interact without duplicating data. 
However, incorporating service principals for remote source interactions and facilitating communication with Azure Data Lake firewalls introduces certain complexities, 
as firewall restrictions typically block API communication. This documentation outlines an effective solution utilizing Fabric Data Factory pipelines to trigger a Spark 
Notebook via the SemPy Fabric library and the FabricRestClient constructor, ensuring seamless integration while maintaining security.

Furthermore, this documentation will elaborate on the process of granting object-level permissions to users for the newly created Delta Table within the LakeHouse.

_______________________________________________________________________________________

**Steps**


Step1: 
Begin by ensuring that the service principal is properly configured with the necessary permissions on the Fabric workspace. 
Define all required parameters by assigning them to the corresponding variables. 
This setup can be automated through a CI/CD pipeline or managed by storing the configuration data in an Azure SQL table.

**API Code to call Fabric DataFactory pipeline with notebook activity**
Please note that this operation is asynchronous, and as a result, a run ID may not be returned during invocation.

```
import os
import requests
from dotenv import load_dotenv

load_dotenv()


def get_fabric_token() -> str:
    """Obtain an Azure AD access token for Service Principal Microsoft Fabric API."""
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


def start_pipeline(workspace_id: str, pipeline_id: str, run_name: str = "Run via Python", pipeline_parameters: dict = None):
    # You can pass parameter below if needed. You can store the values in Azure SQL if you like as well
    # Remember the below is calling a pipeline you will the pipeline ID as well as the workspace ID.
    # This data can be acquired via import sempy.fabric as fabric; fabric.list_items(workspace="workspaceID")
    #You will also need the connection ID from the “Manage Connections and Gateways” settings page of the Datalake.
    #This assumes that you have your DataLake connection created and Trusted workspace access configured.
    token = get_fabric_token()

    url = (
        f"https://api.fabric.microsoft.com/v1/workspaces/"
        f"{workspace_id}/items/{pipeline_id}/jobs/instances?jobType=Pipeline"
    )

    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    # Log the payload for debugging purposes
    payload = {"runName": run_name}
    if pipeline_parameters:
        payload["executionData"] = {"parameters": pipeline_parameters}
    print("DEBUG: Payload being sent:", payload)

    try:
        response = requests.post(url, headers=headers, json=payload)
        response.raise_for_status()

        # If there's no content, avoid calling .json()
        if response.content:
            result = response.json()
        else:
            result = {}

        print(f"Pipeline triggered successfully: {run_name}")
        return result

    except requests.RequestException as e:
        raise Exception(f"Failed to start pipeline: {e}")


if __name__ == "__main__":
    # Environment parameters for the pipeline
    workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
    pipeline_id = os.getenv("FABRIC_PIPELINE_ID")
    run_name = "Triggered from Python SKD service principal invoking notebook created by user"

    # Add your parameters here we can store them in Azure SQL or Fabric DW to allow more automated approach and update a column when completed. 
    pipeline_parameters = {
        "WorkspaceId": os.getenv("WorkspaceId"),
        "FabricLakeHouseItemId": os.getenv("FabricLakeHouseItemId"),
        "OneLakeSchemaName": os.getenv("OneLakeSchemaName"),
        "Onelake_Folder_Path": os.getenv("Onelake_Folder_Path"),
        "Location": os.getenv("Location"),
        "SubPath": os.getenv("SubPath"),
        "ConnectionId": os.getenv("ConnectionId"),
        "ShortcutConflictPolicy": os.getenv("ShortcutConflictPolicy"),
        "LakeHouseContainer": os.getenv("LakeHouseContainer"),
        "LakeHouseName": os.getenv("LakeHouseName"),
        "DeltaSchema": os.getenv("DeltaSchema"),
        "DeltaTableName": os.getenv("DeltaTableName")
    }

    if not workspace_id or not pipeline_id:
        raise Exception("Workspace ID and Pipeline ID must be set in .env")

    result = start_pipeline(workspace_id, pipeline_id, run_name, pipeline_parameters)
    #Print run ID if available, or note that no content was returned.
    print("Run ID:", result.get("id", "No run ID returned; pipeline may be running asynchronously."))
```


Step 2: 
Ensure that the notebook configuration is set up under a user account with sufficient permissions. 
The pipeline will leverage the owner's credentials to execute the API call effectively.

```
# Shortcut creation steps all parameters will pass via the API invocation payload from pipeline no need to modify unless new process is required
# Import the required libraries
import sempy.fabric as fabric
from sempy.fabric.exceptions import FabricHTTPException

client = fabric.FabricRestClient()

# The shortcut API URL from the API payload, these parameters will be passed in via the pipeline to the FabricRestClient constructor
create_shortcut=f"https://api.fabric.microsoft.com/v1/workspaces/{WorkspaceId}/items/{FabricLakeHouseItemId}/shortcuts?shortcutConflictPolicy={ShortcutConflictPolicy}"


# The request body that will pass via the FabricRestClient constructor
request_body ={
  "path": Onelake_Folder_Path,
  "name": OneLakeSchemaName,
  "target": {
    "adlsGen2": {
      "location": Location,
      "subpath": SubPath,
      "connectionId": ConnectionId
    }
  }
}

# Create the shortcut
create_shortcut_process=client.post(create_shortcut, json=request_body)

# Check the response
create_shortcut_process

# Delta table creation process here
# Reading a Parquet file, as a client-side operation via an API call is not supported for Lakehouse environments with schema enforcement enabled
df = spark.read.parquet(f"abfss://{LakeHouseContainer}@onelake.dfs.fabric.microsoft.com/{LakeHouseName}.Lakehouse/{Onelake_Folder_Path}/{OneLakeSchemaName}") 

# Persist the DataFrame to a Delta table corresponding to the specified table
df.write.format("delta").mode("overwrite").save(f"abfss://{LakeHouseContainer}@onelake.dfs.fabric.microsoft.com/FabricLakeHouse.Lakehouse/Tables/{DeltaSchema}/{DeltaTableName}")
```


Step 3: 
To Provide find grain permission to the specific delta table. 
Go to the Lakehouse and select settings and select "share"

![image](https://github.com/user-attachments/assets/bd4ae523-64ea-42f2-9ea3-ba71644bc48b)


Search for the user's email and "DO NOT PROVIDE THE USER ANY PERMISSIONS" just ensure that their respective email is added then select “GRANT”. 
This will act as the connect permission for that database.

![image](https://github.com/user-attachments/assets/a2d2113d-538d-4a15-bdb9-9c885cfb7df0)


Next go to the SQL endpoint and grant the users read permission on that specific objects similar to the below: 
```
GRANT SELECT ON OBJECT::schema.table TO  [useremail@domain.com];
```

Within Power BI, navigate to the "OneLake Catalog" and select "Warehouses." Users should then be able to locate the corresponding table within the interface.
Additionally, when utilizing client tools for database access, ensure that the database name is explicitly specified during login to establish a successful connection.



***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***
