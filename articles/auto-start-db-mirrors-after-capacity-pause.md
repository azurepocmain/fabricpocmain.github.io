# Auto Start DB Mirrors After Capacity Pause
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

In certain operational scenarios, it may be necessary to temporarily pause a Fabric Capacity. During this pause, the database mirroring process halts, preventing the capture of changes from the source system. 
To resume functionality, the mirroring process must be re-initiated by stopping and restarting it. This can become a cumbersome task for organizations managing multiple capacities or extensive database mirroring setups. 
This document outlines a solution to automate this process using Azure Monitor in conjunction with Azure Automation, thereby streamlining operations and reducing manual intervention.
It is important to note that the signal utilized here differs from when a Fabric Capacity is scaled to an alternate capacity size. In that case, it is imperative to apply the database mirroring deltas seamlessly without necessitating the cessation or reinitialization of the process.

_______________________________________________________________________________________

**Steps**

***Step1:*** 
Within the same tenant as the Azure Fabric capacity, ensure the deployment of an Azure Automation account if one is not already provisioned.

![image](https://github.com/user-attachments/assets/5289cd37-79c8-4ac3-aa89-5fe2db63a24b)

Next, create a Python runbook.

![image](https://github.com/user-attachments/assets/235e02eb-6b9a-4958-bd27-0cd46453323f)
![image](https://github.com/user-attachments/assets/a56de04f-80db-48df-824c-066c7c83ca9a)




***Step2:***  
Go to Azure Monitor in the same tenant as the Azure Fabric Capacity, select "Alerts" -> "Create" -> "Alert Rule"

![image](https://github.com/user-attachments/assets/0c51554d-fbbf-4656-9e71-5d1578a785a5)

In "Scope" select all the Fabric Capacities that you want to monitor for this event. 

![image](https://github.com/user-attachments/assets/d1f62ce7-0fa3-4a0f-8b29-96cb792a41e5)

For "Condition", select "Resume the specified Fabric capacity" and ensure that the below "Alert Logic" is selected.

![image](https://github.com/user-attachments/assets/532caa8c-cdd2-438e-a124-526526cdb139)
![image](https://github.com/user-attachments/assets/ad714eea-31b3-43a7-8cfb-3a723b438cb4)

Under the "Actions" section, configure a new "Action Group" to enable email notifications whenever the specified alert condition is triggered. 
Additionally, utilize the "Action Type" labeled "Automation Runbook" and ensure the runbook created in Step 1 is selected for execution.

![image](https://github.com/user-attachments/assets/12095c36-2a41-4a92-8fa8-1ce8c18d8fff)

**Critical Configuration**: Within the "Details" tab, navigate to the "Advanced options" section and locate "Custom properties" 
Ensure that each capacity name selected in the "Scope" is mapped to its respective "Capacity ID," which can be obtained from the Fabric Workspace. 
This configuration is essential as it enables the webhook payload to reference the correct capacity name paired with its corresponding ID, establishing a mandatory dependency for the operational code.

![image](https://github.com/user-attachments/assets/48e92cd2-aa61-45aa-8a28-000d1439ed13)



***Step3:*** 
To facilitate this process, a service principal will be utilized.
The service principal has been granted the "Contributor" role within the workspace for initial setup and testing purposes. (Other permission configurations are currently under evaluation.)
Within the Azure Automation account created in Step 1, navigate to the "Variables" section located under "Shared Resources" and proceed to define the following three secure variables:

![image](https://github.com/user-attachments/assets/13394c4e-249f-4952-8f27-1ffc78c3414a)

Next go to "Runbooks" under "Process Automation" select the runbook that was created in Step1. 
Select "Edit" -> "Edit in portal"

![image](https://github.com/user-attachments/assets/f17001e5-de53-48bf-89ec-8078bfe465cc)

Proceed to paste the below code: 
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

# All the functions we need to make the API calls are here:

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

def list_workspaces_by_capacity(capacity_id, token):
    url = f"{FABRIC_API_BASE}/workspaces?capacityId={capacity_id}"
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json().get("value", [])

def list_mirrored_databases(workspace_id, token):
    url = f"{FABRIC_API_BASE}/workspaces/{workspace_id}/mirroredDatabases"
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json().get("value", [])

def control_mirroring(workspace_id, mirrored_db_id, action, token):
    assert action in ["start", "stop"]
    url = f"{FABRIC_API_BASE}/workspaces/{workspace_id}/mirroredDatabases/{mirrored_db_id}/{action}Mirroring"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    response = requests.post(url, headers=headers)
    if response.status_code in [200, 202]:
        print(f"{action.title()}ed mirroring: {mirrored_db_id}")
    else:
        print(f"Failed to {action} mirroring for {mirrored_db_id}: {response.status_code} {response.text}")

def restart_mirroring_for_capacity(subscription_id, resource_group, capacity_name, capacity_id ):
    fabric_token = get_access_token("https://api.fabric.microsoft.com/.default")

    #capacity_id, capacity_display_name = find_capacity_by_custom_property(fabric_token, capacity_name)
    print(f"Using capacity: {capacity_name} ({capacity_id})")

    workspaces = list_workspaces_by_capacity(capacity_id, fabric_token)
    print(f"Processing Workspaces: {workspaces} ")

    for workspace in workspaces:
        workspace_id = workspace.get("id")
        print(f"\nChecking workspace: {workspace_id}")
        mirrored_dbs = list_mirrored_databases(workspace_id, fabric_token)
        for db in mirrored_dbs:
            db_id = db.get("id")
            print(f"Restarting mirroring for DB: {db_id}")
            control_mirroring(workspace_id, db_id, "stop", fabric_token)
            time.sleep(45)
            control_mirroring(workspace_id, db_id, "start", fabric_token)


def parse_scope(scope: str):
    pattern = (
        r"^/subscriptions/(?P<subscription_id>[^/]+)"
        r"/resourceGroups/(?P<resource_group>[^/]+)"
        r"/providers/Microsoft\.Fabric/capacities/(?P<capacity_name>[^/]+)$"
    )
    m = re.match(pattern, scope)
    if not m:
        raise ValueError(f"Scope did not match expected format: {scope!r}")
    return m.group("subscription_id"), m.group("resource_group"), m.group("capacity_name")



# Main entry of the code the invoke the above functions 
if __name__ == "__main__":
    if len(sys.argv) < 4:
        raise Exception("Missing parameters. Usage: <subscription_id> <resource_group> <capacity_name>")
    
    # This is how python get the payload it can have several parts, we just need the first three for this process
    fabric_meta_data1 = sys.argv[1]
    fabric_meta_data2 = sys.argv[2]
    fabric_meta_data3 = sys.argv[3]


    # We will use a basic parcer 
    raw = fabric_meta_data1
    m = re.search(r'"scope"\s*:\s*"([^"]+)"', raw)
    if not m:
        raise ValueError("Couldn't find scope in payload")
    scope = m.group(1)



    fabric_output_data=parse_scope(scope)
    # Debug output
    print(f"The output is: {fabric_output_data[0]}, {fabric_output_data[1]},  {fabric_output_data[2]}")

    # Getting Matched capacity name from the potential list of names and IDs
    capacity_aggregrated= f"{fabric_meta_data1}{fabric_meta_data2}{fabric_meta_data3}"
    print(capacity_aggregrated)
    # 1) pull out the scope so we know what capacity_name to look for
    m = re.search(r'"scope"\s*:\s*"([^"]+)"', capacity_aggregrated)
    if not m:
        raise ValueError("Couldn't find scope in payload")
    scope = m.group(1)

    # 2) parse out the ARM‐style capacity_name
    scope_pattern = (
        r"^/subscriptions/[^/]+"
        r"/resourceGroups/[^/]+"
        r"/providers/Microsoft\.Fabric/capacities/(?P<capacity_name>[^/]+)$"
    )
    m2 = re.match(scope_pattern, scope)
    if not m2:
        raise ValueError(f"Scope did not match format: {scope!r}")
    capacity_name = m2.group("capacity_name")

    # 3) grab *all* "properties":{…} blocks and pick the last one as this may show up a few times in the payload
    blocks = re.findall(r'"properties"\s*:\s*(\{[^}]+\})', capacity_aggregrated)
    if not blocks:
        raise ValueError("No properties blocks found in payload")
    fab_props = json.loads(blocks[-1])  

    # DEBUG: see what I actually have if needed
    print("capacity_name =", capacity_name)
    print("fab_props.keys() =", list(fab_props.keys()))

    # 4) try to match keys that start with capacity_name
    matches = [k for k in fab_props if k.startswith(capacity_name)]

    if matches:
        # pick the *first* matching one
        chosen_key = matches[0]
        capacity_id = fab_props[chosen_key]
    else:
        # fallback: if at least two entries, pick the SECOND one
        items = list(fab_props.items())
        if len(items) >= 2:
            chosen_key, capacity_id = items[1]
        else:
            # only one entry, use it
            chosen_key, capacity_id = items[0]

    print(f"chosen_key = {chosen_key}")
    print(f"capacity_id = {capacity_id}")

    subscription_id=fabric_output_data[0]
    resource_group=fabric_output_data[1]
    capacity_name=fabric_output_data[2]

    restart_mirroring_for_capacity(subscription_id, resource_group, capacity_name, capacity_id )
```



***Step4:***
Save and publish the code. Ensure that the runbook is republished following each modification to guarantee proper functionality.

![image](https://github.com/user-attachments/assets/e8166a74-3483-4a0c-88a8-ba449bf28093)

Test to confirm the behavior. 





***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. 
THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS 
FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) 
to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is 
embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the 
Sample Code.***





