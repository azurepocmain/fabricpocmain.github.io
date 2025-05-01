# Microsoft Fabric Virtual Connection With Azure Data Explorer 
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Having the ability to connect to desperate data systems is one of the robust features in Fabric. This documentation will go over steps to connnect security to Azure Data Explorer. 
This will allow users to query data into Fabric from thier Data Exploer with the firewall enabled. 

**Steps**

***Step1:*** 
Verify that the subscription intended for creating the virtual network has the "Microsoft.PowerPlatform" resource provider registered. This can be checked and configured under "Subscription" -> "Settings" -> "Resource Providers."

![image](https://github.com/user-attachments/assets/806fd558-ddde-48e4-8bfc-68ad80b7f604)

***Step2:*** 
Set up or utilize an existing Azure Virtual Network and provision two subnets: one designated for Azure Data Explorer and the other for Microsoft Fabric. 
Ensure that the subnet allocated for Microsoft Fabric has "Subnet Delegation" enabled with the appropriate delegation configurations as depicted in the below image.

![image](https://github.com/user-attachments/assets/3f46cfa5-6187-422a-b23f-d91eafc99f61)


***Step3:*** 
Proceed to the Azure Data Explorer Cluster, navigate to "Security + Networking," and under "Private endpoint connections," create a new private endpoint connection.
Ensure that public access is disabled.

![image](https://github.com/user-attachments/assets/41c50a40-e7f8-4220-adcf-4bef040b32cf)


![image](https://github.com/user-attachments/assets/03c89793-9a2e-4ea3-9e5b-bd6701466a6f)


![image](https://github.com/user-attachments/assets/80fc48cb-6db6-4ef5-ae82-12a4f210adb6)


![image](https://github.com/user-attachments/assets/9bf666c9-7f5d-49b8-a7d8-c9a543d55b6a)



***Step4:*** 
Create a virtual network gateway under settings -> "Manage connections and gateways"


![image](https://github.com/user-attachments/assets/34beca38-50fe-4675-83ed-b2c9f81f9f88)

![image](https://github.com/user-attachments/assets/ca729a23-be6f-4fb4-afc0-d4c9c951dec9)

![image](https://github.com/user-attachments/assets/89873376-32a2-4f43-bb12-e4b868ab9be8)

***Step5:*** 
Navigate to the "Connections" tab and create a new connection. Select the virtual network created earlier and choose "Azure Data Explorer."

![image](https://github.com/user-attachments/assets/81990621-91ad-48ce-aa7c-6510271144fc)


![image](https://github.com/user-attachments/assets/07adc879-c119-4f72-9727-d5bf07c9a5f6)

***Step6:*** 
Test the connection: 

![image](https://github.com/user-attachments/assets/dcc460f0-ecb7-4718-a453-cc7108a2c11e)






