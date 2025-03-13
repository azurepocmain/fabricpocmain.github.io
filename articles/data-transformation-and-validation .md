# Microsoft Fabric Data Transformation and Validation
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

The Microsoft Fabric platform offers a comprehensive array of solutions for data transformation and validation. 
This document aims to outline several potential methodologies tailored to specific use cases within this domain, though it may not encompass every possible option. 
As leveraging integration of Spark within the Fabric ecosystem, this list can be potentially limitless. 

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











