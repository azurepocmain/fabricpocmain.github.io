# Microsoft Fabric Spark Job Livy and Notebook Runtime Evaluation 

The utilization of Apache Spark has become a pivotal component in organizational workflows, enabling efficient data processing and analytics. 
Microsoft Fabric facilitates the deployment of Spark clusters through various mechanisms, including Spark jobs, interactive notebooks, and Livy batch jobs. 
These integrations also enable the invocation of Spark processes within Fabric pipelines as notebooks and Spark job, allowing for the orchestration of diverse tasks as part of a comprehensive solution. 
This documentation aims to detail the runtime metrics, configuration procedures, and API invocation methodologies for each of the aforementioned functionalities. 
By incorporating these capabilities into application-level APIs or deployment tools, organizations can achieve enhanced automation and monitoring, thereby optimizing operational flexibility and scalability.

It is important to highlight that all runtime instances below utilize the same Spark transformation via API, where the parameters are transmitted within the JSON payload of the request body.

_______________________________________________________________________________________

**Livy Job Runtime Metrics:**

The total execution duration of the Livy job was recorded at an efficient `2 minutes and 46 seconds` as we can see below. 
![image](https://github.com/user-attachments/assets/b302d484-b98f-456a-aa5d-2fe05912cb58)

The Livy job was stored in a Fabric Lakehouse and incorporated the following code snippet. Note how the `argparse` library and the `argparse.ArgumentParser()` function are utilized to capture the input 
parameters from the API call.
_______________________________________________________________________________________
![image](https://github.com/user-attachments/assets/50fc2e7c-0671-4652-9bcc-d73fc5353524)

```
from pyspark.sql import SparkSession
import sys
import argparse
from datetime import datetime

# Initialize Spark session with Delta support
spark = SparkSession.builder \
    .appName("Spark_Process_Speed_POC") \
    .getOrCreate()


# Print all received arguments for debugging
print("All arguments:", sys.argv)

# Use argparse to parse named arguments
parser = argparse.ArgumentParser()

# Add expected arguments
parser.add_argument("--param1", type=str, help="First parameter LakeHouseContainer")
parser.add_argument("--param2", type=str, help="Second parameter LakeHouseName")
parser.add_argument("--param3", type=str, help="Third parameter DeltaSchema")
parser.add_argument("--param4", type=str, help="Forth parameter ParquetFileName")


# Parse the arguments
args = parser.parse_args()

# Print the extracted arguments for validation
print(f"LakeHouseContainer: {args.param1}")
print(f"LakeHouseName: {args.param2}")
print(f"DeltaSchema: {args.param3}")

# 2. Assign arguments to variables to use below 
LakeHouseContainer = args.param1
LakeHouseName = args.param2
DeltaSchema = args.param3
ParquetFileName = args.param4


#Check that the variables have been assigned properly
print(LakeHouseContainer)
print(LakeHouseName)
print(DeltaSchema)
print(ParquetFileName)



# Path to the Delta table
delta_table_path = f"abfss://{LakeHouseContainer}@onelake.dfs.fabric.microsoft.com/{LakeHouseName}.Lakehouse/Tables/{DeltaSchema}/{ParquetFileName}"

# Read the Delta table
df = spark.read.format("delta").load(delta_table_path)

# Transformation filter by City
transformed_df = df.select("AddressID", "AddressLine1", "City", "PostalCode") \
                   .filter(df.City == "Nashville")


# Generate a timestamp string as i want to run this process a few times and keep track
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

#Define the output path with timestamp
output_path = f"Files/{DeltaSchema}/parquet_{timestamp}"

#Write the DataFrame to Parquet format
transformed_df.write.mode("overwrite").parquet(output_path)

print(f"Data written to {output_path}")
```

To call the above Livy job via API the below code was leveraged: 

```
# Livy With parameters passing
import requests
import os
import requests
from dotenv import load_dotenv

load_dotenv()




tenant_id = os.getenv("TENANT_ID")
client_id = os.getenv("CLIENT_ID_FABRIC")
LakehouseName= os.getenv("LakeHouseName")

workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
lakehouse_id = os.getenv("FabricLakeHouseItemId")

load_dotenv()

def get_fabric_token() -> str:
    """Obtain an Azure AD access token for Microsoft Fabric API."""
    tenant_id = os.getenv("TENANT_ID")
    client_id = os.getenv("CLIENT_ID_FABRIC")
    client_secret = os.getenv("CLIENT_SECRET_FABRIC")

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



def start_livy_job(livy_parameters: dict = None):
    access_token = get_fabric_token()
    api_base_url ='https://api.fabric.microsoft.com/v1'
    livy_base_url = api_base_url + "/workspaces/"+workspace_id+"/lakehouses/"+lakehouse_id +"/livyApi/versions/2023-12-01/batches"
    headers = {"Authorization": "Bearer " + access_token}
    # call get batch API
    get_livy_get_batch = livy_base_url
    get_batch_response = requests.get(get_livy_get_batch, headers = headers)
    if get_batch_response.status_code == 200:
        print("API call successful")
        print(get_batch_response.json())
    else:
        print(f"API call failed with status code: {get_batch_response.status_code}")
        print(get_batch_response.text)


    # submit payload to existing batch session

    print('Submit a spark job via the livy batch API to ') 

    # Generate arguments as a list for Livy
    command_line_args = []
    for key, value in livy_parameters.items():
        if value is not None:
            command_line_args.append(f"--{key}")
            command_line_args.append(str(value))  # Separate key and value or I would get an error 

    print("Generated commandLineArguments:", command_line_args)

    payload_data = {
        "name":f"livy_batch_poc_with_{LakehouseName}",
        "args": command_line_args,  # Use "args" for Livy instead of "commandLineArguments"
        "file":"abfss://fabricland@onelake.dfs.fabric.microsoft.com/FabricLake.Lakehouse/Files/livy_jobs/transformation_poc.py", # Store in variables instead
        "conf": {
            "spark.targetLakehouse": "sparklow"   # Tested with both sizes and similar times
            }
        }

    print(payload_data)
    get_batch_response = requests.post(get_livy_get_batch, headers = headers, json = payload_data)

    print("The Livy batch job submitted successful")
    print(get_batch_response.json())


if __name__ == "__main__":
    # Parameters For Livy job
    livy_parameters = {
        "param1": os.getenv("LakeHouseContainer"),
        "param2": os.getenv("LakeHouseName"),
        "param3": os.getenv("DeltaSchema"),
        "param4": os.getenv("ParquetFileName")
    }

    start_livy_job(livy_parameters)


```


_______________________________________________________________________________________

**Spark Job Definition Runtime Metrics:**

The total execution duration of the Spark Job Definition via API call was recorded at  `2 minutes and 45 seconds` as we can see below. 

![image](https://github.com/user-attachments/assets/9c6a41a7-8838-4337-afcb-13ee279ca1e0)


The Spark Job Definition code used the same code as the above Spark Livy. 


To call the Spark Job Definition via API the below code was leveraged: 


```
# Spark job invocation with parameters 
import os
import requests
from dotenv import load_dotenv
import time

load_dotenv()

def get_fabric_token() -> str:
    """Obtain an Azure AD access token for Microsoft Fabric API."""
    tenant_id = os.getenv("TENANT_ID")
    client_id = os.getenv("CLIENT_ID_FABRIC")
    client_secret = os.getenv("CLIENT_SECRET_FABRIC")

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

def start_job(workspace_id: str, sparkJobId: str, run_name: str = "Run via Python", job_parameters: dict = None):
    token = get_fabric_token()

    url = (
        f"https://api.fabric.microsoft.com/v1/workspaces/"
        f"{workspace_id}/items/{sparkJobId}/jobs/instances?jobType=sparkjob"
    )

    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

# Convert job_parameters to a command line argument string
    command_line_args = " ".join(
        [f"--{key} {value}" for key, value in job_parameters.items() if value is not None]
    )

    print("Generated commandLineArguments:", command_line_args)

    payload = {
  "executionData": {
     "commandLineArguments": command_line_args
  }
}

    print("DEBUG: Payload being sent:")
    print(payload)

    try:
        response = requests.post(url, headers=headers, json=payload)
        response.raise_for_status()

        job_instance_url = response.headers.get("Location")
        retry_after = int(response.headers.get("Retry-After", 60))

        print(f"Job triggered successfully: {run_name}")
        print(f"Job instance URL: {job_instance_url}")

        time.sleep(retry_after)

        status_response = requests.get(job_instance_url, headers=headers)
        status_response.raise_for_status()
        job_status = status_response.json()

        print(f"Job status: {job_status}")
        return job_status

    except requests.RequestException as e:
        raise Exception(f"Failed to start job: {e}")


if __name__ == "__main__":
    workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
    sparkJobDefinitionId = os.getenv("SPARK_JOB_DEFINITION_ID")
    run_name = "Triggered from Python SDK service principal invoking job"

    job_parameters = {
        "param1": os.getenv("LakeHouseContainer"),
        "param2": os.getenv("LakeHouseName"),
        "param3": os.getenv("DeltaSchema"),
        "param4": os.getenv("ParquetFileName")
    }

    if not workspace_id or not sparkJobDefinitionId:
        raise Exception("Workspace ID and JOB ID must be set in .env")

    result = start_job(workspace_id, sparkJobDefinitionId, run_name, job_parameters)
    print("Run ID:", result.get("id", "No run ID returned; pipeline may be running asynchronously."))





```

_______________________________________________________________________________________

**Notebook Runtime Metrics:**

The total execution duration of the interactive Notebook via API call was recorded at  `3 minutes and 47 seconds` as we can see below. 

![image](https://github.com/user-attachments/assets/4af50553-5021-4300-b3d1-ab46607fced8)

The interactive Notebook invoked the exact same code as above, however, the use of the `argparse` library is not required. 
This is because the interactive Notebook automatically maps the parameters to the correct key variables during runtime in the first cell.


![image](https://github.com/user-attachments/assets/447274e5-5352-4eb0-b7da-632d5a9d48d3)


To call the interactive Notebook via API the below code was leveraged: 

```
#Spark job notebook test run
import os
import requests
from dotenv import load_dotenv
import time

load_dotenv()

def get_fabric_token() -> str:
    """Obtain an Azure AD access token for Microsoft Fabric API."""
    tenant_id = os.getenv("TENANT_ID")
    client_id = os.getenv("CLIENT_ID_FABRIC")
    client_secret = os.getenv("CLIENT_SECRET_FABRIC")

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

def start_notebook(workspace_id: str, notebook_id: str, environment_id: str, environment_name: str, defaultLakehouse_name: str, defaultLakehouse_id: str, defaultLakehouse_workspaceId: str, useWorkspacePool: str,  run_name: str = "Run via Python", notebook_parameters: dict = None):
    token = get_fabric_token()

    url = (
        f"https://api.fabric.microsoft.com/v1/workspaces/"
        f"{workspace_id}/items/{notebook_id}/jobs/instances?jobType=RunNotebook"
    )

    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    payload = {
        "executionData": {
            "parameters": {
                key: {
                    "value": value,
                    "type": "string"
                } for key, value in notebook_parameters.items()
            },
            "configuration": {
                "conf": {
                    
                },
                "environment": {
                    "id": os.getenv("environment_id"),
                    "name": os.getenv("environment_name")
                },
                "defaultLakehouse": {
                    "name": os.getenv("defaultLakehouse_name"),
                    "id": os.getenv("defaultLakehouse_id"),
                    "workspaceId": os.getenv("defaultLakehouse_workspaceId")
                },
                "useStarterPool": False,
                "useWorkspacePool": os.getenv("useWorkspacePool") 
            }
        }
    }
    print("DEBUG: Payload being sent:", payload)

    try:
        response = requests.post(url, headers=headers, json=payload)
        response.raise_for_status()

        # Extract the job instance URL from the response headers
        job_instance_url = response.headers.get("Location")
        retry_after = int(response.headers.get("Retry-After", 60))

        print(f"Pipeline triggered successfully: {run_name}")
        print(f"Job instance URL: {job_instance_url}")

        # Wait for the specified retry interval before checking the status
        time.sleep(retry_after)

        # Fetch the job status
        status_response = requests.get(job_instance_url, headers=headers)
        status_response.raise_for_status()
        job_status = status_response.json()

        print(f"Job status: {job_status}")
        return job_status

    except requests.RequestException as e:
        raise Exception(f"Failed to start notebook: {e}")

if __name__ == "__main__":
    # Environment parameters for the notebook
    workspace_id = os.getenv("FABRIC_WORKSPACE_ID")
    notebook_id = os.getenv("NoteBookID_Speed_Test")
    run_name = "Triggered from Python SDK service principal invoking notebook"

    # Add your parameters here
    notebook_parameters = {
        "LakeHouseContainer": os.getenv("LakeHouseContainer"),
        "LakeHouseName": os.getenv("LakeHouseName"),
        "DeltaSchema": os.getenv("DeltaSchema"),
        "ParquetFileName": os.getenv("ParquetFileName")
    }

    if not workspace_id or not notebook_id:
        raise Exception("Workspace ID and Notebook ID must be set in .env")

    result = start_notebook(workspace_id, notebook_id, run_name, notebook_parameters)
    # Print run ID if available, or note that no content was returned.
    print("Run ID:", result.get("id", "No run ID returned; pipeline may be running asynchronously."))

```


_______________________________________________________________________________________

**Notebook In Pipeline Runtime Metrics:**

The total execution duration of the interactive Notebook in a Fabric Pipeline via API call was recorded at  `4 minutes and 19 seconds` as we can see below. 

![image](https://github.com/user-attachments/assets/d8f209f8-7bc1-4307-a26a-32f6ebbc2396)

The same code stack notebook was used to execute the performance test. 
To pass the parameters from the API call to the notebook, the parameters must first be declared within the pipeline.
![image](https://github.com/user-attachments/assets/cf310e0c-c4e1-4b3e-a99b-873f322394ce)

Finally, assign those parameters to the notebook's base parameters using the following format for each parameter:  `@string(pipeline().parameters.LakeHouseContainer)`


![image](https://github.com/user-attachments/assets/6764050f-d546-4060-8a27-31da51ef55a7)

To call the Pipeine interactive Notebook via API the below code was leveraged: 

```
#Notebook run in pipeline
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


def start_pipeline(workspace_id: str, pipeline_id: str, run_name: str = "Run via Python", pipeline_parameters: dict = None):
    # You can pass parameter below if needed. Have two code stacks to share with client one with parameters and another without
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
    pipeline_id = os.getenv("FABRIC_PIPELINE_ID_PERFORMANCE_TEST")
    run_name = "Triggered from Python SKD service principal invoking notebook created by user"

    # Add your parameters here we can store them in Azure SQL or Fabric DW to allow more automated approach and update a column when completed. 
    pipeline_parameters = {
        "LakeHouseContainer": os.getenv("LakeHouseContainer"),
        "LakeHouseName": os.getenv("LakeHouseName"),
        "DeltaSchema": os.getenv("DeltaSchema"),
        "ParquetFileName": os.getenv("ParquetFileName")
    }

    if not workspace_id or not pipeline_id:
        raise Exception("Workspace ID and Pipeline ID must be set in .env")

    result = start_pipeline(workspace_id, pipeline_id, run_name, pipeline_parameters)
    # Print run ID if available, or note that no content was returned.
    print("Run ID:", result.get("id", "No run ID returned; pipeline may be running asynchronously."))

```



**Spark Job In Pipeline Runtime Metrics:**

The total execution duration of the Spark Job in a Fabric Pipeline via API call was recorded at  `3 minutes and 37 seconds`  as we can see below. 

![image](https://github.com/user-attachments/assets/f2269d60-4e74-49e2-85e8-3f59fff21c0a)

The same code stack Spark job was used to execute the performance test. 

Remember, to pass the parameters from the API call to the Spark job, the parameters must first be declared within the pipeline.
![image](https://github.com/user-attachments/assets/cf310e0c-c4e1-4b3e-a99b-873f322394ce)

Finally, we need to assign these parameters to the Spark job's `Command line arguments`, which can be found under Spark job settings -> Advanced settings -> Command line arguments, as shown in the image below.

![image](https://github.com/user-attachments/assets/305ccce9-f01a-4c02-9bb6-df960f3ea1cd)

![image](https://github.com/user-attachments/assets/c14da978-4313-48f4-9fcb-e837a57d532a)


To call the Pipeine Spark Job via API the below code was leveraged: 

```
#Notebook run with Spark Job
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


def start_pipeline(workspace_id: str, pipeline_id: str, run_name: str = "Run via Python", job_parameters: dict = None):
    # You can pass parameter below if needed. Have two code stacks to share with client one with parameters and another without
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
    if job_parameters:
        payload["executionData"] = {"parameters": job_parameters}
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
    pipeline_id = os.getenv("FABRIC_PIPELINE_ID_PERFORMANCE_TEST2")
    run_name = "Triggered from Python SKD service principal invoking notebook created by user"

    # Add your parameters here we can store them in Azure SQL or Fabric DW to allow more automated approach and update a column when completed. 
    pipeline_parameters = {
        "LakeHouseContainer": os.getenv("LakeHouseContainer"),
        "LakeHouseName": os.getenv("LakeHouseName"),
        "DeltaSchema": os.getenv("DeltaSchema"),
        "ParquetFileName": os.getenv("ParquetFileName")
    }

    if not workspace_id or not pipeline_id:
        raise Exception("Workspace ID and Pipeline ID must be set in .env")

    result = start_pipeline(workspace_id, pipeline_id, run_name, pipeline_parameters)
    # Print run ID if available, or note that no content was returned.
    print("Run ID:", result.get("id", "No run ID returned; pipeline may be running asynchronously."))

```


_______________________________________________________________________________________

Among all execution methods evaluated within Fabric Spark engine, the Spark job demonstrated superior performance, achieving the fastest runtime of just `2 minutes and 45 seconds`. 
This makes it significantly more efficient compared to the potential other options. 
From a usability perspective, the Spark interactive notebook demonstrates superior ease of use, making it the preferred choice for time to market. 



_______________________________________________________________________________________
DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.
