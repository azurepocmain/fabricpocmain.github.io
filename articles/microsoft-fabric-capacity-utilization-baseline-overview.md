# Auto Start DB Mirrors After Capacity Pause
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Establishing an accurate baseline for Microsoft Fabric resource utilization is essential for ensuring optimal sizing and effective allocation of workspace resources. 
This guide provides detailed instructions on utilizing the Fabric Capacity Metric App to assess and validate the baseline metrics for the assigned capacity.
The documentation presumes that the Fabric Capacity Metric App is already deployed and operational within your Microsoft Fabric environment.


**Steps**

***Step1:*** 
In the Capacity Metric App select the capacity name from the top left of the screen. 

Identify the date with the highest aggregated capacity usage using the Fabric Capacity Metric App, noting that the application retains data for only the past 14 days. 
Once the peak utilization date has been determined, drill down into the specific metrics by selecting the lower section of that date. 
This approach enables a precise and comprehensive analysis of the daily capacity utilization.

![image](https://github.com/user-attachments/assets/f537c49e-63ca-4fc1-8cfc-4c71369ca8f5)


***Step2:*** 
After selecting the date, the initial metric to analyze is the CU (Capacity Unit) utilization. Navigate to the CU utilization graph located on the right panel. 
Ensure that the "Utilization" option is activated, as shown in the accompanying illustration, and that the view mode is set to "Logarithmic."
The dotted line represents the maximum capacity for the designated SKU. Focus on the utilized CU percentage, highlighted by the red indicator. 
If the percentage of used CU approaches or exceeds 100% of the allocated capacity, it signifies that the current capacity SKU may be insufficient, necessitating a potential scale-up to accommodate the workload.
If the utilized capacity remains consistently below 80%, it indicates that the allocated resources for the fabric capacity are sufficient to meet the operational requirements.

![image](https://github.com/user-attachments/assets/f98930ea-f64e-460c-8236-3652af4fe234)


***Step3:*** 
The subsequent metric to evaluate is throttling, which provides detailed insights into throttling delays, interactive rejections, and background rejections. 
Each of these metrics operates on smoothed usage aggregations over three intervals: 10 minutes, 60 minutes, and 24 hours respectively. 
Throttling is triggered when capacity usage exceeds the smoothed usage thresholds for the respective aggregation period.
As demonstrated below, the current interactive delays reflect a utilization rate of only 10%, significantly below the 100% threshold that would result in delays. 
This indicates that the allocated capacity SKU is operating well within acceptable parameters. A similar analysis can be conducted for interactive rejections and background rejections. 
However, interactive delays serve as the initial layer in the throttling hierarchy, making them a critical checkpoint. 
If interactive delays are observed, it is likely that interactive rejections will also occur within the system.


![image](https://github.com/user-attachments/assets/e0d1a36a-8ce3-4d8c-b265-4ff975720dd3)


***Step4:*** 
To identify the specific compute workloads within the workspace consuming the majority of capacity units, navigate to the throttling chart and select the desired area of interest. 
Once selected, utilize the "Explore" option located at the bottom-right of the chart. 
This action will provide a detailed breakdown of all background operations along with their respective capacity unit usage metrics.

![image](https://github.com/user-attachments/assets/b3558031-48a2-4e0c-af9a-5f1acbf9ae92)

![image](https://github.com/user-attachments/assets/8551702b-db1c-4f83-8970-cf0871775a4d)







***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. 
THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS 
FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) 
to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is 
embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the 
Sample Code.***
