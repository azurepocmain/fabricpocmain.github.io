# Querying Fabric OneLake Fast
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Microsoft Fabric OneLake's Lakehouse SQL analytics endpoint offers a unified location for disparate systems to store and query integrated data directly within OneLake. This solution addresses a critical need for nearly every organization, particularly in reporting and processes that require correlation of data from various systems. SQL, as the universal language, along with the SQL analytics endpoint or a Spark engine, empowers organizations to democratize their ETL processes when leveraging OneLake mirroring features.

However, it is essential to note that the SQL analytics endpoint depends on background processes to synchronize the metadata of delta parquet files. For big data processes that do not use the endpoint and read directly from OneLake, this synchronization is not an issue. Conversely, for systems that rely on the Lakehouse SQL analytics endpoint and necessitate near real-time responses (e.g., 5-15 seconds), this can present a unique challenge.

Reference:  <a href="https://learn.microsoft.com/en-us/fabric/data-warehouse/sql-analytics-endpoint-performance" target="_blank">SQL Analytics Endpoint Performance </a>


In response to the growing demand for HTAP systems, a hybrid systems combining OLTP and OLAP functionalities this document offers an alternative to the Lakehouse SQL analytics endpoint for scenarios requiring near real-time data querying within the OneLake Lakehouse.
_______________________________________________________________________________________

**Solution: Fabric Eventhouse**

Fabric Eventhouse offers a unique solution with the capability to query large data volumes in near real time. This is achieved by bypassing the SQL analytics endpoint and directly querying the data from OneLake storage, similar to Spark.
How is this accomplished, you may ask? Through the use of external tables. Below is a DDL example illustrating how to create an external table in Eventhouse to directly query data from the OneLake Lakehouse storage. This approach eliminates the dependency on the SQL analytics endpoint and the synchronization process.

Here is the DDL and path from the Fabric LakeHouse that is used to create the external table:
![image](https://github.com/user-attachments/assets/bf001ec6-ba5a-45e5-a587-a4e360426eb6)
![image](https://github.com/user-attachments/assets/1614a305-5348-45c3-b2a1-53c319786db8)

External Table creation DDL: 
<pre>
.create external table Address2
(
    AddressID: int,
    AddressLine1: string,
    AddressLine2: string,
    City: string,
    PostalCode: string,
    rowguid : string,
    modifieddate: datetime
)
kind=delta
('abfss://vicfabricpoc@onelake.dfs.fabric.microsoft.com/FabricLakeHouse.Lakehouse/Tables/SalesLT/Address;impersonate')
</pre>

Key points to note:
-	Delta Object: The kind is set to delta, ensuring that the actual delta object in OneLake is read.
-	Impersonate Authentication: This method allows the system to forward the user's identity to the data store.

In addition to querying OneLake and other datalake storages via external tables in Fabric Eventhouse, you can also read SQL tables from Azure SQL, MySQL, PostgreSQL, and Cosmos DB.
Lastly it's crucial to understand the following limitations:
-	The maximum limit of external tables per database is 1,000.
-	External table names are case-sensitive and must not overlap with Kusto table names. For more information, see the Identifier Naming Rules.
-	Azure Data Explorer supports export and continuous export to an external table.
-	Data purge does not apply to external tables; records are never deleted.
-	Row-level security policies cannot be configured on external tables.<br>

Reference:  <a href="https://learn.microsoft.com/en-us/kusto/query/schema-entities/external-tables?view=microsoft-fabric" target="_blank">Fabric Eventhouse External Tables </a>

**Performance: Fabric Eventhouse Performance Overview**

While Fabric Eventhouse offers a robust solution for reading data in the OneLake, developers should be aware of its performance limits to ensure that the concurrency aligns with their business requirements.

One of the critical performance aspects to consider is the result set size of queried data. By default, Kusto restricts the number of records returned to the client to 500,000, with an overall data size limit of 64 MB for those records. This limitation can be circumvented by utilizing the “set notruncation;” parameter before executing the query, as demonstrated in the C# code below.

Concurrency is another vital metric for developers. In Eventhouse, concurrency is contingent on the SKU of the database, calculated as: Cores-Per-Node multiplied by 10. Therefore, the formula is: number_of_cores (x) 10 = concurrency total. The default and maximum request rate limit is set to 10,000.

Additionally, the execution timeout value defaults to 4 minutes, which can be extended up to 1 hour by employing parameters such as norequesttimeout set to false. For further information regarding Eventhouse query limits and additional details, please refer to the provided link.

Reference:  <a href="https:/learn.microsoft.com/en-us/kusto/concepts/query-limits?view=microsoft-fabric" target="_blank">Fabric Eventhouse Query limits</a>

**.NET SDK: Fabric Eventhouse C# Examples**

The following example demonstrates how to invoke an Eventhouse KQL query to retrieve the necessary dataset. The referenced libraries were sourced from the NuGet repository.
![image](https://github.com/user-attachments/assets/d0bd7963-b445-4cd9-a31f-2e1b8699bc71)

Observe that the "set notruncation;" parameter is prefixed before the actual query. This enables the handling of large data volumes exceeding 500,000 records or 64MB without triggering an exception.

<pre>
using System;
using System.Threading.Tasks;
using System.Collections.Generic;
using Kusto.Data;
using Kusto.Data.Net.Client;
using Azure.Identity;
using Kusto.Data.Common;
using Microsoft.Extensions.Configuration;

class Program
{
    static async Task Main()
    {
        //Load settings from appsettings.json I stored i in the bin\Debug\net8.0 locatoin of the C# project
        var config = new ConfigurationBuilder()
        .SetBasePath(Directory.GetCurrentDirectory())
        .AddJsonFile("./appsettings.json", optional: false, reloadOnChange: true)
        .Build();
        // Retrieve authentication details from appsettings.json
        string tenantId = config["AzureSettings:TenantId"] ?? throw new ArgumentNullException("AzureSettings:TenantId is missing in appsettings.json");
        string clientId = config["AzureSettings:ClientId"] ?? throw new ArgumentNullException("AzureSettings:ClientId is missing in appsettings.json");
        string clientSecret = config["AzureSettings:ClientSecret"] ?? throw new ArgumentNullException("AzureSettings:ClientSecret is missing in appsettings.json");
        string eventhouseCluster = config["AzureSettings:EventhouseCluster"] ?? throw new ArgumentNullException("AzureSettings:EventhouseCluster is missing in appsettings.json");
        string database = config["AzureSettings:Database"] ?? throw new ArgumentNullException("AzureSettings:Database is missing in appsettings.json");


        ///Build Kusto connection string with Azure AD application key authentication
        var kcsb = new KustoConnectionStringBuilder(eventhouseCluster)
            .WithAadApplicationKeyAuthentication(clientId, clientSecret, tenantId);

        //Initialize Kusto client
        using var client = KustoClientFactory.CreateCslQueryProvider(kcsb);

        // KQL query (I added the notruncation to ensure results are returned that are a larger payload. you can add others)
        string query = "set notruncation; external_table('Address')";

        try
        {
            // Execute the query
            var reader = await client.ExecuteQueryAsync(database, query, new ClientRequestProperties());

            // Display the results
            while (reader.Read())
            {
                var values = new List<string>();
                for (int i = 0; i < reader.FieldCount; i++)
                {
                    values.Add(reader[i]?.ToString() ?? "NULL");
                }
                Console.WriteLine(string.Join(" | ", values));
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error executing query: {ex.Message}");
        }
    }
}
</pre>    

The above results are as follows: 
![image](https://github.com/user-attachments/assets/865f5bba-554a-45ef-bedb-0658804a8468)

**Performance: Fabric Eventhouse Concurrency Test**
This test evaluates the concurrency dynamics of the Eventhouse instance relative to the allocated capacity units.
For this particular demonstration, the lowest capacity F2 is employed. It is recommended to conduct tests with a higher capacity for more robust results that are realistic to your business needs. 
![image](https://github.com/user-attachments/assets/876ee308-c82c-4060-9ef8-2aa02ec76850)


As you can see below, I am able to sustain 10 concurrent sessions with a relatively large payload of over 500,000 records before I get the 429 too many request exception.


