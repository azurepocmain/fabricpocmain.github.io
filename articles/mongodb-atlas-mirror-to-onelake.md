# MongoDB Mirroring To Fabric
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

As disparate systems continue to proliferate rapidly within organizations, the necessity to integrate these systems into a unified repository has become paramount. 
This document provides a comprehensive analysis of how Microsoft Fabric offers an effective solution for synchronizing data from MongoDB Atlas into Microsoft Fabric Onelake. 
Adherence to the standards outlined in 
<a href="https://www.mongodb.com/developer/products/atlas/near-real-time-analytics-powered-mirroring-microsoft-fabric-mongodb-atlas/?msockid=05101b1fe8f2682f37010e6fe93b697c" target="_blank">near-real-time-analytics-powered-mirroring-microsoft-fabric-mongodb</a>
and 
<a href="https://learn.microsoft.com/en-us/fabric/database/mirrored-database/open-mirroring-partners-ecosystem" target="_blank">open-mirroring-partners-ecosystem</a>, 
which both ultimately reference
<a href="https://github.com/mongodb-partners/MongoDB_Fabric_Mirroring" target="_blank">MongoDB_Fabric_Mirroring</a>, is maintained throughout the document.

A few things to note when setting up the python script for the MongoDB_Fabric_Mirroring, there is a dependency on the pandas, pyarrow, fastparquet, and flask libaries. You can easily install them in your environment by invoking the pip install pandas pyarrow fastparquet flask command. 

When running either the application on a VM or Azure App Service, it is crucial to adjust the parameter based on latency and near real-time requirements. This can be configured in the .env file as "TIME_THRESHOLD_IN_SEC," which governs the incremental synchronization waits before replicating accumulated changes when the next event occurs. By default, this is set to 180 seconds. 
During testing, incremental changes took approximately 5 minutes to replicate. Adjusting the parameter to 5 seconds reduced latency to around 15 to 20 seconds, similar to Azure SQL mirroring performance. 
Note that in the Azure deployment, this parameter is labeled "IncrementalSyncMaxTimeInterval" within the Azure Resource Manager template. 
Also, I was concurrently inserting multiple records. When inserting records sequentially, latency increased exponentially. 
Additionally, specific collections can be designated using the parameter: MONGO_COLLECTION = ["collection1", "collection2"].


If it is necessary to convert the specified MongoDB collection into an external Lake in Eventhouse, the analogous location utilized for other tables should be used: abfss://vicfabricpoc@onelake.dfs.fabric.microsoft.com/FabricLakeHouse.Lakehouse/Tables/SalesLT/Address. However, it is imperative to verify the path using Microsoft Azure Storage Explorer to confirm the accurate absolute path. The path will consistently conclude with the table name as illustrated in the accompanying image:
![image](https://github.com/user-attachments/assets/84f2d471-6142-4041-ad20-b049026b2154)

It is important to note that when establishing an external table, if the specified path is invalid, no exception will be raised during the table creation process. However, an exception will be triggered upon executing a query against the table with an invalid path.

To generate the external table the below DDL was leveraged: 
```
.create external table embeddedmovies
(
    _id: string,
    plot: string,
    genres: string,
    runtime: string,
    cast: string,
    poster: string,
    title: string,
	fullplot: string,
    languages: string,
    released: string,
    directors: string,
    rated: string,
    awards: string,
    lastupdated: string,
	year: string,
    imdb: string,
    countries: string,
    type: string,
    tomatoes: string,
    num_mflix_comments: string,
	plot_embedding: string,
    writers: string,
    metacritic: string
)
kind=delta
('abfss://vicfabricpoc@onelake.dfs.fabric.microsoft.com/MirroredDatabase_MongoAtlas.MountedRelationalDatabase/Tables/dbo/embedded_movies;impersonate')

```
