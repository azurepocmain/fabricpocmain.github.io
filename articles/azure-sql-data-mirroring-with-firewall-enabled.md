# Azure SQL Data Fabric Mirroring with Firewall Enabled
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Ensuring security while integrating disparate data stores is of utmost importance for organizations. Azure SQL provides robust security measures to prevent unauthorized public access to data stores. 
Additionally, the need to aggregate data from diverse systems has become essential to obtain comprehensive insights and deliver near real-time data to users. 
This post outlines a solution for enabling Fabric Database Mirroring to connect securely to Azure SQL, facilitating the seamless integration of data sources while having a Azure SQL firewall enabled.

Based on the current documentation, Fabric Mirroring for Azure SQL databases does not support private endpoints, Fabric gateways, or virtual network data gateways. 
Therefore, to configure Fabric Mirroring for Azure SQL, it is essential to enable the firewall and allow the exception: "Allow Azure services and resources to access this server" as illustrated below.

![image](https://github.com/user-attachments/assets/a7b7a501-aa58-4926-992b-7ac0efa990a7)

This configuration will facilitate the resolution of any potential errors as outlined below, enabling seamless data mirroring to Fabric.

![image](https://github.com/user-attachments/assets/dc7240b4-4e76-4ee9-84f5-770691efb7b0)


Reference: <a href="https://learn.microsoft.com/en-us/fabric/database/mirrored-database/azure-sql-database" target="_blank">Fabric Database Mirroring</a>

