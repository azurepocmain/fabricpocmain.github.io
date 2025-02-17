# Querying Fabric OneLake Fast
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Microsoft Fabric OneLake's Lakehouse SQL analytics endpoint offers a unified location for disparate systems to store and query integrated data directly within OneLake. This solution addresses a critical need for nearly every organization, particularly in reporting and processes that require correlation of data from various systems. SQL, as the universal language, along with the SQL analytics endpoint or a Spark engine, empowers organizations to democratize their ETL processes.

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
