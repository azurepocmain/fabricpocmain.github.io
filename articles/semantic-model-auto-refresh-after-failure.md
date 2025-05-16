# Semantic Model Auto Refresh After Failure

This document outlines a structured approach for implementing advanced retry logic and notification mechanisms for semantic model refresh operations, specifically tailored for the 
Microsoft Fabric Capacity Metrics semantic model. The outlined solution is adaptable for other semantic models and workspaces as well. The implementation will provide comprehensive logs to 
diagnose job failures and enable automated retry attempts.
Key prerequisites include assigning a capacity to the Microsoft Fabric Capacity Metrics app. Additionally, this solution requires the integration of an Azure Automation account for executing 
Python scripts, Azure Log Analytics for running remote KQL queries from Eventhouse, and Azure Monitor for setting up alert rules and web hook for the retry logic. 
These components collectively ensure a robust and reliable refresh process.

It is important to reiterate that this solution is contingent upon the Microsoft Fabric Capacity Metrics application having an assigned capacity, as the functionality may not operate effectively without this prerequisite.
_______________________________________________________________________________________

## Steps ##

## Step1: ##
Go the Microsoft Fabric Capacity Metrics App Workspace, and select `Workspace Settings` -> `Monitoring` -> enable `Eventhouse`. 
This implementation facilitates the creation of an EventHouse equipped with detailed logs, enabling comprehensive failure tracking and triggering specific events in response to detected failures.

![image](https://github.com/user-attachments/assets/aabb4e11-837b-4ec7-9d78-af465030e5ee)


![image](https://github.com/user-attachments/assets/6f8bdfc1-1e61-4e88-b6e9-f7721901b39e)


![image](https://github.com/user-attachments/assets/c4614597-0ccc-428e-baf7-dd1478a1c57d)

_______________________________________________________________________________________
## Step2: ##
Next, go to the `Monitoring Eventhouse` that was just created and on the far right copy the `Query URI`. This will be used in Azure Log Analytics to query this Eventhouse telemetry data remotely.

![image](https://github.com/user-attachments/assets/a3ace015-abf2-4022-a371-860b3094a6b4)


_______________________________________________________________________________________
## Step3: ##
Proceed to Azure and configure an Azure Automation Account. Within this account, create the designated runbook with the specified settings outlined below.

![image](https://github.com/user-attachments/assets/9f34fc4d-58f2-4278-ae71-6a8a5ad07a80)


_______________________________________________________________________________________
## Step4: ##
Edit the above runbook in the portal and copy and paste the below Python code. 
Ensure that each `automationassets.get_automation_variable()` call is appropriately configured by adding the corresponding variable under the `Shared Resources` → `Variables` section. This step is crucial for the proper execution of the code.

![image](https://github.com/user-attachments/assets/8289ec4e-a4ac-4141-a762-2e5018137cef)


```

import sys
import json
import requests
import time
import re
import os
import automationassets
from automationassets import AutomationAssetNotFound

FABRIC_API_BASE = "https://api.fabric.microsoft.com/v1"




def get_access_token(scope):
    tenant_id = automationassets.get_automation_variable("TENANT_ID")
    print(tenant_id)
    client_id = automationassets.get_automation_variable("CLIENT_ID")
    print(client_id)
    client_secret = automationassets.get_automation_variable("CLIENT_SECRET")
    print(client_secret)


    token_url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"
    payload = {
        "grant_type": "client_credentials",
        "client_id": client_id,
        "client_secret": client_secret,
        "scope": scope
    }

    response = requests.post(token_url, data=payload)
    response.raise_for_status()
    return response.json()["access_token"]





def start_pipeline(workspace_id: str, pipeline_id: str, run_name: str = "Run via Python", pipeline_parameters: dict = None):
    # You can pass parameter below if needed. Have two code stacks to share with client one with parameters and another without
    token = get_access_token("https://api.fabric.microsoft.com/.default")

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
    # Environment parameters for the notebook
    workspace_id = automationassets.get_automation_variable("ENVIRONMENT_ID")
    pipeline_id = automationassets.get_automation_variable("PIPELINE_ID")
    run_name = "Triggered from Azure Runbook Python SDK service principal invoking pipeline job"

    # Save for later 
    pipeline_parameters = {
    
    }
    if not workspace_id or not pipeline_id:
        raise Exception("Workspace ID and Notebook ID must be set in .env")

    result = start_pipeline(workspace_id, pipeline_id, run_name, pipeline_parameters)
    # Print run ID if available, or note that no content was returned.
    print("Run ID:", result.get("id", "No run ID returned; pipeline may be running asynchronously."))

```



**Please** ensure that you `Publish` the runbook or it will not run the latest version of the code: 

![image](https://github.com/user-attachments/assets/b743f47b-6dce-4a3f-852a-93593efe20a2)

_______________________________________________________________________________________
## Step5: ##
Access your Azure Log Analytics account, navigate to the `Logs` section, and input the KQL provided below. Ensure that the `Query URI` obtained in `Step 2` is incorporated correctly. 
This KQL query will execute remotely, enabling the EventHouse to detect and aggregate semantic model job failures effectively, which can then be utilized to establish alert thresholds.
Please disregard any errors shown in the KQL pane, as the query should still execute successfully despite these warnings.

adx('https://your_eventhouse_uri_goes_here.fabric.microsoft.com/Monitoring Eventhouse').SemanticModelLogs | where Status in  ("Failed" ) and ItemName =="Fabric Capacity Metrics" | count 

![image](https://github.com/user-attachments/assets/233412cd-cf2d-465c-9b25-e14b501f275c)


_______________________________________________________________________________________
## Step6: ##

Once the KQL has been invoked, select new alert rule. 

![image](https://github.com/user-attachments/assets/12d6eeed-8e69-416b-9e63-69f2e2a66266)


## The last steps will be implmented tomorrow TO BE CONTINUED! ## 







