# Microsoft Fabric Data Transformation and Validation
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

The Microsoft Fabric platform offers a comprehensive array of solutions for data transformation and validation. 
This document aims to outline several potential methodologies tailored to specific use cases within this domain, though it may not encompass every possible option. 
As leveraging integration of Spark within the Fabric ecosystem, this list can be potentially limitless. 

_______________________________________________________________________________________
**Potential Solutions:**
***Real-Time/Near Real-Time***

No discussion of real-time to near real-time solutions would be complete without addressing Eventstreams within Microsoft Fabric. 
This option not only encompasses various data ingestion methods via streaming but also offers a robust framework for validation and transformation of Azure SQL stream data as well as other data repositories with low to no code options.
![image](https://github.com/user-attachments/assets/bd4e5123-5ab3-4893-bc29-43382119d68a)

Once the Azure SQL database source has been selected, the "Transform events" dropdown menu offers several transformation and validation options, including the "Filter" function, which facilitates advanced data processing workflows.
![image](https://github.com/user-attachments/assets/810d419f-4ecf-46dd-9d0c-8a9ac52e4fa1)

Utilizing the "Table" key in this scenario, I can ensure that the streamed data undergoes specific table source prior to its complete storage. This enables the establishment of various conditional checks that can be executed before the final data set is committed to storage.
![image](https://github.com/user-attachments/assets/1369a5c3-6b18-4951-88f1-ad362a3d20d6)
![image](https://github.com/user-attachments/assets/557be556-b3f1-4482-9df5-10f772a79a5e)

Running a test on a subset of inserts from Azure SQL, the filter validation demonstrated a latency of only 1-2 seconds with Eventstream. This is significantly lower than the 10-15 seconds of latency observed with database mirroring.

It is essential to consider that when handling data from Eventstreams via Azure SQL, the data manifests as a JSON object from the source. In our experimental scenarios, the data was directed to the Eventhouse database, wherein it is incumbent upon us to formulate transformation logic 
to track the deletes and updates as a payload of changes is sent to the stream weather insert, update or delete. This ensures that the gold copy of the data adheres to the slowly changing dimensions anticipated by both the business and developers.

The metadata can be meticulously tracked through the 'before' and 'after' keys within the array, allowing for precise updating of data changes for the dimension table or view. 

![image](https://github.com/user-attachments/assets/895c67c5-3594-4f28-818a-4398ef22aa0a)

Furthermore, when handling JSON objects, if decomposing the data from Eventstream options proves too complex, the parse_json KQL function can be utilized to conform the data to a tabular format as illustrated below: 
```
let rawData = journals_all
| project payload;

let parsedData = rawData
| extend payloadJson = parse_json(payload)
| extend afterJson = payloadJson.after
| project 
    BKPF_BUKRS = tostring(afterJson.BKPF_BUKRS),
    BKPF_BELNR = tolong(afterJson.BKPF_BELNR),
    BKPF_GJAHR = toint(afterJson.BKPF_GJAHR),
    BKPF_MONAT = toint(afterJson.BKPF_MONAT),
    BKPF_BUDAT = todatetime(afterJson.BKPF_BUDAT),
    BKPF_CPUDT = todatetime(afterJson.BKPF_CPUDT),
    BSEG_SHKZG = tostring(afterJson.BSEG_SHKZG),
    BSEG_DMBTR = toreal(afterJson.BSEG_DMBTR);

parsedData
```
![image](https://github.com/user-attachments/assets/f1bca52b-aafe-43b2-905c-7c2274dd7adc)

Additionally, when utilizing KQL internal tables, it is possible to implement update policies that automatically insert data into a table when the source table undergoes data insert.
Our initial tests demonstrated that the update policy processing took approximately one minute during the first run. This suggests that, at present, this approach may not be ideal for solutions requiring near real-time updates.


Reference:  <a href="https://learn.microsoft.com/en-us/kusto/management/update-policy?view=microsoft-fabric" target="_blank">KQL Eventhouse Update Policy Overview</a>

Overall: 

Pros:

•	Minimal latency and inherent data validation during the streaming process.

•	Additional transformation capabilities during data streaming.

•	Low-code to no-code implementation.
Cons:

•	Data deletions and updates must be manually processed from the JSON object to ensure the dimension table or view aligns with business requirements.

•	Table retention needs to be verified to ensure the data purge policy complies with business objectives.

_______________________________________________________________________________________
***Near Real-Time***

In Eventhouse, KQL provides the capability to orchestrate data transformation processes after data has been ingested into the OneLake LakeHouse. 
This section outlines various KQL syntax and functions compatible with external tables, enabling advanced data transformation and validation workflows.

The following function can be created and invoked to execute various data transformations and validations:

```
.create-or-alter function TransformAddressData() {
    external_table("Address")
    | extend ConcatenatedAddress = strcat(AddressLine1, " ", PostalCode)
    | extend NewAddressID = AddressID + 10
    | project ConcatenatedAddress, NewAddressID, AddressLine1
}
```

The function on the external table can be invoked by executing the following command:
```
TransformAddressData()
```

Furthermore, if a medallion architecture with an internal table is required, the transformed or validated data can be seamlessly loaded into the "Gold" zone from the the external table by via the below steps.

Create the table in the Gold zone:
```
.create table TransformAddressDataTab (ConcatenatedAddress: string, NewAddressID: long, AddressLine1: string) with (folder="Gold")
```

Then subsequently invoking the function and appending the processed data into the newly established table in the Gold zone.
There are numerous built-in functions available that can be utilized to execute various data transformations and validations, including joins that verify specific primary key and foreign key references.
For additional reference please visit the link below: 
Reference:  <a href="https://learn.microsoft.com/en-us/fabric/data-warehouse/sql-analytics-endpoint-performance" target="_blank">Kusto documentation</a>
```
.append TransformAddressDataTab <| TransformAddressData() 
```

Additionally, SQL can be employed to execute data transformations and validations within Eventhouse, ensuring precise manipulation and verification of datasets.
```
SELECT 
    a1.AddressID AS AddressID1,
    CONCAT(a1.AddressLine1, ' ', a1.AddressLine2) AS FullAddress1,
    LEN(CONCAT(a1.AddressLine1, ' ', a1.AddressLine2)) AS AddressLength1,
    a2.AddressID AS AddressID2,
    CONCAT(a2.AddressLine1, ' ', a2.AddressLine2) AS FullAddress2,
    LEN(CONCAT(a2.AddressLine1, ' ', a2.AddressLine2)) AS AddressLength2
FROM 
    Address a1
INNER JOIN 
    Workers a2
ON 
    a1.City = a2.City AND a1.PostalCode = a2.PostalCode;
```

Additionally, if the SQL statement is prefixed with the keyword 'EXPLAIN', the output will include the KQL equivalent, providing a detailed insight into the query KQL version:
![image](https://github.com/user-attachments/assets/7fdbc291-fef8-4fbf-b5e2-ea2442b694ec)

Overall: 

Pros:

•	Enables near real-time KQL transformations and validations.

•	Functions can be applied to external tables and loaded into the Gold zone internal tables.

•	Allows SQL query language to be utilized for data transformation and validation.

•	Has the ability to directly read delta files from OneLake, enabling the mitigation of potential additional latency.
Cons:

•	Minimum resource consumption must be verified to ensure adequate resource allocation and concurrency.

•	Retention policies for internal tables need to be confirmed.

•	Data limit parameters should be configured to remove query limitations.



_______________________________________________________________________________________
***Near Real-Time/Batch Warehouse***

The next solution will be observed once data is ingested into OneLake LakeHouse or Warehouse, utilizing the SQL analytics endpoint. It has been noted that data loaded via database mirroring exhibits randomized latency due to the metadata synchronization process that must be conducted.


The advantageous aspect here is that the identical query syntax and functions, already familiar to us, can be employed for executing data transformation and validation tasks, as demonstrated in the subsequent sections. Notice how the query in Eventhouse KQL is similar to the Lakehouse and Data Warehouse.

![image](https://github.com/user-attachments/assets/d45022fe-e37c-4cf5-8b89-dc6e0fcf014d)


Overall: 

Pros:

•	Familiar SQL syntax functions and stored procedures.

•	Capability to perform data joins between data warehouses and Lakehouse.

•	SQL endpoint compatibility with various applications.

Cons:

•	Potential latency issues with SQL analytics endpoint.

•	Requirement for meticulous design to mitigate network bottlenecks in SQL analytics endpoint.

•	Lack of primary key and foreign key enforcement.

•	Currently lacks the ability to read delta parquet files directly from OneLake for the SQL analytics endpoint.



_______________________________________________________________________________________
***Batch Spark***

The next solution is Apache Spark, which offers similar capabilities such as support for multiple languages, including SQL, as well as advanced transformation and validation functionalities.
Due to its design for big data processing, Spark operates with a distributed architecture that involves multiple compute nodes, akin to Eventhouse.

As demonstrated below, the same query utilized via Lakehouse can be invoked through the Spark SQL API. Note the elapsed time in comparison to Lakehouse—it is approximately five seconds longer to retrieve the data.
In summary, Spark provides a robust array of options for advanced big data transformation and validation processes.

![image](https://github.com/user-attachments/assets/9359aa12-3b8b-406d-a091-3e0b5502f315)

Pros:

• Optimized for high-volume data processing.

• Supports multiple programming languages including SQL.

• Capable of directly reading delta files from OneLake, thereby eliminating potential additional latency inherent in other metadata synchronization processes.

Cons:

• Cluster initialization period can be significant.

• Requires utilization of the Spark DataFrame API for table write operations.

• Concurrency assessments are essential to ascertain maximum workload capacity.


_______________________________________________________________________________________
***Batch Prepare Data***

In addition to the above points, Fabric offers advanced capabilities for data transformation and validation, with some features utilizing the same compute as mentioned earlier. These functionalities address specific use cases and requirements, providing comprehensive solutions for data management. For detailed documentation and further information, please refer to the official resources.

![image](https://github.com/user-attachments/assets/de9ee63f-d874-40e6-ac7a-b40bc46eed14)

