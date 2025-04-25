# Purview DataMap Asset Search API
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
# We will convert the Unix Timesamp into a normal timestamp, this assumes the job will be invoked every 24 hours first run can omit all of the below
df_purview_assets["createTime_dt"] = pd.to_datetime(df_purview_assets["createTime"], unit='ms')
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
Create a pipeline and drag the two activities below: 

![image](https://github.com/user-attachments/assets/2c8bafe3-1af8-481a-a0d8-f363e4f394b4)


***Step6:***
In the notebook, select the "Settings" and the notebook that was created in Step3. 

![image](https://github.com/user-attachments/assets/e22cdfe0-929e-497c-885f-446a9665e556)


***Step7:***
In the Email activity, login, and select the recipients and subject. 
![image](https://github.com/user-attachments/assets/6f66d44b-a231-45f2-9236-08490d96b95f)

***Step8:***
For the body, go to the bottom of the page and select "View in expression builder" and add the below expression and ensure no other characters are added: 

![image](https://github.com/user-attachments/assets/c469bea9-0cc3-4e16-a9cf-000c4f2d02bd)

```
@activity('NotebookPruviewAPI').output.result.exitValue
```

![image](https://github.com/user-attachments/assets/5740311f-c501-4174-bfb4-24eb8b8a34df)

***Step9:***
Run the pipeline to confirm the behavior, then setup a run schedule.
The output should look like the below: 

![image](https://github.com/user-attachments/assets/771efe7d-0ded-4fc4-a200-3776628d40d7)



***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED “AS IS” WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***






