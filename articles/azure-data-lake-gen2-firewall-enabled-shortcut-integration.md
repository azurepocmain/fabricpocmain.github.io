# Azure Data Lake Gen2 Firewall Enabled Shortcut Integration 
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Shortcuts enable disparate systems to interface with a unified OneLake storage location within Fabric, thereby presenting a virtual data lake for organizational use. 
Azure Data Lake Gen2 can seamlessly integrate with these shortcuts. 
However, when the firewall of an Azure Data Lake Gen2 storage account is enabled, additional configurations are required to permit access to the secure storage. 
Detailed below are the setup steps necessary to accomplish this integration.

It is crucial to understand the security considerations for shortcut integration with Azure Data Lake Gen2 storage accounts. 
Currently, shortcuts to Azure Data Lake Gen2 storage accounts do not support private endpoints, virtual network gateways, or Fabric on-premises data gateway (OPDG). 
As a result, this document will focus on using trusted workspace access. Note that "Trusted workspace is limited to F SKU capacities."
Reference:  <a href="https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access#arm-template-sample" target="_blank">Trusted workspace access</a>


**Trusted Workspace Access Configuration Steps:**

First, we will need to create and deploy a custom template as the below. 
![image](https://github.com/user-attachments/assets/9c005bdd-9be5-497f-96cf-fd2394231c36)

The ARM template sample is located at the bottom page of the link below, fill out all the input values it will look like the below. You will use the ADLS Gen 2 destination and the Fabric workspace information. 
![image](https://github.com/user-attachments/assets/c3ddf582-8040-441d-aa17-2d1750b39e06)

Once saved and deployed, the ADLS Gen 2 under “Security + networking” will have the fabric workspace and instance information listed as the below. 
![image](https://github.com/user-attachments/assets/e1ef7609-4fa5-45b2-ba86-904312cd876d)

Verify that the workspace identity is registered for the Fabric workspace. 
Provide the Fabric workspace storage blog data contributor on the ADLS Gen 2 account as well. 

![image](https://github.com/user-attachments/assets/4439faee-9d8d-4dbc-901f-d50e1cab8145)

In the Entra AD, locate the App registrations and select the Fabric workspace ID. 
Proceed to create a client secret: 
![image](https://github.com/user-attachments/assets/275fa9ef-09ee-4a06-a390-e229e2b61e95)

Finally recreate the shortcut but this time select service principal since the storage account is secured by a firewall and use the Fabric credentials from the above. 

Reference:  <a href="https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access#arm-template-sample" target="_blank">Trusted workspace access</a>


***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***
