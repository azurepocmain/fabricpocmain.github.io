# Fabric Data Warehouse Performance Optimization
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >


System performance is a vital consideration in enterprise environments. This document outlines strategies and methodologies for optimizing Fabric Data Warehouse to maintain a high performance operational environment.


# **Options**

# **Option 1: Use Data Clustering **

Data clustering efficiently organizes records with similar values, enabling faster queries and minimizing compute and storage costs. 
By grouping related data and reducing scanned files, clustering significantly enhances data warehouse performance. This approach is especially effective when your system frequently queries specific subsets of data. 
By selecting the column or columns most commonly used in WHERE predicates as the clustering key, you can optimize query performance and efficiency.

**For example**
```
CREATE TABLE dbo.Sales
(SalesId        bigint        NOT NULL,
SaleDate       date          NOT NULL,
CustomerId     int           NOT NULL,
ProductId      int           NOT NULL,
StoreId        int           NOT NULL,
Quantity       int           NOT NULL,
NetAmount      decimal(18,2) NOT NULL,
TaxAmount      decimal(18,2) NOT NULL,
TotalAmount    decimal(18,2) NOT NULL,
CreatedAtUtc   datetime2(0)  NOT NULL
)WITH (CLUSTER BY (SaleDate, CustomerId));
```

In our test scenarios, we observed significant improvements in performance. Please refer to the execution plan below to see the marked differences in optimization and query execution for the same table and query, with one clustered and the other not.

**Non Clustering Key Column**
<img width="1148" height="293" alt="image" src="https://github.com/user-attachments/assets/6f9d2965-f5d0-423e-a061-5a254537cb75" />

**Clustering Key Column**
<img width="764" height="133" alt="image" src="https://github.com/user-attachments/assets/7d9cbbc0-5385-4448-ae17-bbd40ac93e27" />





