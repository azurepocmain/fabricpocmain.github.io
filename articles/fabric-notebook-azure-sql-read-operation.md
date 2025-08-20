# Fabric Spark Notebook Reading Data from Azure SQL
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

This section outlines the process for retrieving data from Azure SQL using a Spark notebook in Microsoft Fabric, detailing the integration methods and key configuration steps required for seamless connectivity and efficient data access.

```
servername = "jdbc:sqlserver://servername.database.windows.net:1433"
dbname = "db_name"
url = servername + ";" + "databaseName=" + dbname + ";"
user = "username" 
password = mssparkutils.credentials.getSecret('https://keyvaultURL.vault.azure.net/','secretname')
dbtable="schema.table"
print("read data from SQL Server table ")
jdbcDF = spark.read \
    .format("jdbc") \
    .option("url", url) \
    .option("dbtable", dbtable) \
    .option("user", user) \
    .option("password", password) \
    .option("driver", "com.microsoft.sqlserver.jdbc.SQLServerDriver") \
    .load()

display(jdbcDF)

```


