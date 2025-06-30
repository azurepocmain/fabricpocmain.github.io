# Fabric item ownership options 

In certain environments, it is best practice to avoid assigning production level object ownership to individual users. If a user departs from the team, group, or organization, this can lead to permission complications and potential instability when running critical jobs.
This documentation outlines alternative strategies for establishing a universal owner for items within Microsoft Fabric, ensuring consistent access and operational reliability.

The overall ownership documentation is located here: <a href="https://learn.microsoft.com/en-us/fabric/fundamentals/item-ownership-take-over" target="_blank">Take ownership of Fabric items</a>


As illustrated below, service principals are unable to assume ownership of workspace items. 
Consequently, it is necessary to implement a strategy that designates a dedicated user account responsible solely for owning and operating production level jobs within the environment. 

![image](https://github.com/user-attachments/assets/d3f5a93c-715c-4370-81b4-636bfb405eea)


Once this account has been established and accessed, you can follow the outlined steps to transfer item ownership appropriately. 
Additionally, ensure that any new connections are either owned by the designated production user or that the correct permissions are granted. 
Please ensure that the “Repair connections after Fabric item ownership change” section of the documentation is reviewed as well for additional details. 
Reference:  <a href="https://learn.microsoft.com/en-us/fabric/fundamentals/item-ownership-take-over#repair-connections-after-fabric-item-ownership-change" target="_blank">Repair Connections After Fabric Item Ownership Change</a>

Pelase note that as of this writing, there is no option to perform this via API. 

![image](https://github.com/user-attachments/assets/37966639-dbde-41d8-bdb6-4ae2d50972c6)

![image](https://github.com/user-attachments/assets/fb7f152d-9a0b-4e25-aff8-e7918750bbeb)




It is also important to reassign ownership of the semantic model for the metric capacity report to this account, as demonstrated in the accompanying image:

![image](https://github.com/user-attachments/assets/98b7c345-1723-41fd-8fdf-6f5ea1012598)



***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***


