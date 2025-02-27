# Entity Framework Eventhouse SQL to KQL Options
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

While Entity Framework supports a wide array of data sources, it currently lacks a direct provider for KQL (Kusto Query Language) or Fabric Eventhouse/Azure Data Explorer (ADX). Consequently, Entity Framework cannot natively translate LINQ queries into KQL as it does with SQL databases. 
Nevertheless, by utilizing the .NET SDK and the query language set option, SQL syntax can be leveraged, and the KQL engine will seamlessly translate it into the corresponding KQL.

**.NET SDK: SQL to KQL  C# Examples**

Below is the code example illustrating how SQL syntax can be utilized, with automatic translation to the corresponding KQL by leveraging the query_language SQL set option. 
This approach facilitates a more intuitive development experience for SQL developers, aligning closely with their existing skill set.

```
using System;
using System.Threading.Tasks;
using System.Collections.Generic;
using System.IO;
using Kusto.Data;
using Kusto.Data.Net.Client;
using Kusto.Data.Common;
using Microsoft.Extensions.Configuration;

class Program
{
    static async Task Main()
    {
        //Load settings from appsettings.json I stored i in the bin\Debug\net8.0 locatoin of the C# project
        var config = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("./appsettings.json", optional: false, reloadOnChange: true)
            .Build();

        // Retrieve authentication details from appsettings.json
        string tenantId = config["AzureSettings:TenantId"] ?? throw new ArgumentNullException("AzureSettings:TenantId is missing in appsettings.json");
        string clientId = config["AzureSettings:ClientId"] ?? throw new ArgumentNullException("AzureSettings:ClientId is missing in appsettings.json");
        string clientSecret = config["AzureSettings:ClientSecret"] ?? throw new ArgumentNullException("AzureSettings:ClientSecret is missing in appsettings.json");
        string eventhouseCluster = config["AzureSettings:EventhouseCluster"] ?? throw new ArgumentNullException("AzureSettings:EventhouseCluster is missing in appsettings.json");
        string database = config["AzureSettings:Database"] ?? throw new ArgumentNullException("AzureSettings:Database is missing in appsettings.json");

        ///Build Kusto connection string with Azure AD application key authentication
        var kcsb = new KustoConnectionStringBuilder(eventhouseCluster)
            .WithAadApplicationKeyAuthentication(clientId, clientSecret, tenantId);

        // Initialize Kusto client
        using var client = KustoClientFactory.CreateCslQueryProvider(kcsb);

        //SQL Query ADX will auto-convert it to KQL we need to dobuble check the set notruncation; as I did not need it while invoking SQL 
        string sqlQuery = "SELECT top(10) * FROM Address2";  //SQL Statement goes here instead of KQL 

        //Set SQL query mode in request properties
        var clientRequestProperties = new ClientRequestProperties();
        clientRequestProperties.SetOption("query_language", "SQL");  //Forces SQL mode

        try
        {
            //Execute the SQL query
            var reader = await client.ExecuteQueryAsync(database, sqlQuery, clientRequestProperties);

            //Display the results
            while (reader.Read())
            {
                var values = new List<string>();
                for (int i = 0; i < reader.FieldCount; i++)
                {
                    values.Add(reader[i]?.ToString() ?? "NULL");
                }
                Console.WriteLine(string.Join(" | ", values));
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error executing query: {ex.Message}");
        }
    }
}
```

Example of the output: 
![image](https://github.com/user-attachments/assets/1a61d148-f73b-4c36-bb2c-dbf20b3893cc)


***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***
