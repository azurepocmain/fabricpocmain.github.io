# Python Eventhouse SQL to KQL Options
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

The Python Kusto module can effectively translate standard SQL queries into the equivalent KQL syntax. To achieve this, simply add the set_option query_language property to SQL value. Detailed example is provided below.
Notice the execution of slightly complex SQL query below.

```
from azure.kusto.data import KustoClient, KustoConnectionStringBuilder, ClientRequestProperties
from azure.kusto.data.exceptions import KustoApiError
from azure.identity import ClientSecretCredential
import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Retrieve credentials and configuration
TENANT_ID = os.getenv("TENANT_ID")
CLIENT_ID = os.getenv("CLIENT_ID")
CLIENT_SECRET = os.getenv("CLIENT_SECRET")
EVENTHOUSE_CLUSTER = os.getenv("EVENTHOUSE_CLUSTER")
DATABASE = os.getenv("DATABASE")

#validate environment variables if needed
required_vars = {
    "TENANT_ID": TENANT_ID,
    "CLIENT_ID": CLIENT_ID,
    "CLIENT_SECRET": CLIENT_SECRET,
    "EVENTHOUSE_CLUSTER": EVENTHOUSE_CLUSTER,
    "DATABASE": DATABASE
}
for var_name, var_value in required_vars.items():
    if not var_value:
        raise ValueError(f"Environment variable '{var_name}' is not set or empty.")

#service principal authenticaion
credential = ClientSecretCredential(
    tenant_id=TENANT_ID,
    client_id=CLIENT_ID,
    client_secret=CLIENT_SECRET
)

# Build Kusto Connection String
kcsb = KustoConnectionStringBuilder.with_aad_application_key_authentication(
    EVENTHOUSE_CLUSTER, CLIENT_ID, CLIENT_SECRET, TENANT_ID
)

# Initialize Kusto Client
client = KustoClient(kcsb)

# Define SQL query for conversion to KQL
sql_query = 'SELECT TOP 10 * FROM (SELECT DISTINCT plot, cast   FROM embeddedmovies) AS subquery;'

#Set request properties to use SQL mode
request_properties = ClientRequestProperties()
request_properties.set_option("query_language", "sql")  #Use SQL "sql" for SQL mode
request_properties.set_option("notruncation", True)   #Requried for large data sets or it will be turncated 


#Invoke the query
response = client.execute(DATABASE, sql_query, request_properties)
#  show the results 
for row in response.primary_results[0]:
    print(row)
```

Output: 
![image](https://github.com/user-attachments/assets/2265d3e2-50e7-4037-8ba3-bd8c7c07f182)


***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***
