# Fabric Business Continuity and Disaster Recovery (BCDR) 
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Business continuity and disaster recovery (BCDR) are critical components of a robust enterprise risk management framework. This documentation outlines the essential procedures for implementing and validating your BCDR strategy for Fabric.
Please note that this guide is not exhaustive, additional sections and detailed best practices will be incorporated soon in future updates.
Please be aware that once this configuration is enabled, it will remain active for a minimum of 30 days and cannot be reverted during that period. This ensures proper synchronization and seamless migration to the designated paired region. Additionally, enabling this option will result in increased operational costs.


# **Steps**

# **Step 1: Enable BCDR**
To enable disaster recovery capabilities in Microsoft Fabric, navigate to the Admin portal and select Capacity settings. From there, choose the appropriate Fabric Capacity, then access the Disaster Recovery configuration.

![image](https://github.com/user-attachments/assets/6cd45f87-b68e-458e-b304-0b4305ba73ba)

Activating this setting ensures that workspace items stored in OneLake are replicated to the designated secondary region for redundancy. 
However, it is important to note that not all objects, such as those managed by EventHouse, store their data in OneLake. 
For these services, alternative disaster recovery procedures must be implemented to ensure data resiliency.

# **Step 2: Create DR Capacity Workspace and Item Services**
The subsequent step involves provisioning a capacity and workspace within the designated paired region and reestablishing corresponding Lakehouses, Warehouses, and job configurations. 
During disaster recovery testing, it is essential to adhere closely to established naming conventions to maintain consistency and facilitate seamless failover operations. 
For illustration, recommended practice may involve using identical service names as the primary environment, with the addition of a "_DR" suffix to distinguish disaster recovery instances. 
Nonetheless, organizations should align with their internal DR naming standards as appropriate.

**Step 3: Restore Lakehouse Tables**
Once disaster recovery has been enabled in Fabric, and both the new capacity and workspace have been provisioned with failover completed, the service restoration process can commence. 
The initial focus should be on recovering Lakehouse Tables, ensuring that data structures and associated configurations are accurately restored in the secondary environment.

To facilitate the restoration of Lakehouse tables, utilize the following PySpark Notebook script to initiate a copy operation. 
This process will efficiently replicate the tables from the OneLake primary source environment to the designated disaster recovery destination.


```
# Source
source_paths=mssparkutils.fs.ls(f'abfss://{source}@msit-onelake.dfs.fabric.microsoft.com/')

sorurce_filtered_paths = [
    path.path
    for path in source_paths
    if path.path.endswith(".Lakehouse") and "Staging" not in path.path
]

print("Filtered Lakehouse Paths:")
for p in sorurce_filtered_paths:
    print(p)
```


```
# Destination
destination_paths=mssparkutils.fs.ls(f'abfss://{destination}@msit-onelake.dfs.fabric.microsoft.com/')

destination_filtered_paths = [
    path.path
    for path in destination_paths
    if path.path.endswith(".Lakehouse") and "Staging" not in path.path
]

print("Filtered Lakehouse Paths:")
for p in destination_filtered_paths:
    print(p)
```


```
paths_pairs = list(zip(sorurce_filtered_paths, destination_filtered_paths))
```


```
paths_pairs
```



```
from datetime import datetime

for source_lakehouse, dest_lakehouse in paths_pairs:
    tables_dir = f"{source_lakehouse}/Tables"

    try:
        schema_folders = mssparkutils.fs.ls(tables_dir)
    except Exception as e:
        print(f"No 'Tables' directory in {source_lakehouse}, skipping.")
        continue

    for schema_folder in schema_folders:
        schema_path = schema_folder.path.rstrip("/")
        try:
            table_folders = mssparkutils.fs.ls(schema_path)
        except Exception:
            print(f"No tables in schema {schema_path}, skipping.")
            continue

        for table in table_folders:
            table_path = table.path.rstrip("/")
            dest_table_path = f"{dest_lakehouse}/Tables/{schema_folder.name}/{table.name}"

            print(f"Recursively copying from:\n  {table_path}\n  -> {dest_table_path}")

            try:
                # Recursively copy the full table directory including all delta logs and data
                mssparkutils.fs.cp(table_path, dest_table_path, recurse=True)
                print(f"Done copying full table directory: {schema_folder.name}.{table.name}\n")
            except Exception as e:
                print(f"Failed to copy {table_path}: {e}\n")
```

It is important to note that the provided script assumes all corresponding Lakehouse instances in the source environment have already been established in the disaster recovery destination. 
If this alignment is not in place, inconsistencies may arise, potentially resulting in data being copied to incorrect Lakehouse targets. In such cases, the script may require enhancements, such as incorporating logic to match Lakehouse names or dynamically referencing the full path via a path.path.endswith value, to ensure accurate data mapping and prevent operational anomalies.

**Step 3a: Restore Lakehouse Tables Pipeline**

To streamline operations, consider implementing a pipeline that automatically triggers the provided script. While this pipeline is not strictly required, it facilitates reuse of the script without the need for hardcoded values, thereby enhancing flexibility and maintainability. 
Alternatively, the script can be executed manually to transfer objects from the primary Lakehouse to the disaster recovery (DR) Lakehouse Tables folder as needed.
For pipeline configuration, navigate to the "Parameters" section and define two parameters: `source` and `destination`. 
These parameters will be passed to the notebook, allowing dynamic specification of the primary and DR Lakehouse environments during execution. All Lakehosues and their respective tables will be copied in the process. 


![image](https://github.com/user-attachments/assets/2988cffd-6810-41b4-8130-1e5e3232ff78)

To configure the notebook for dynamic execution, define two base parameters corresponding to `source` and `destination` within the notebook settings. Ensure these parameters are properly referenced throughout your workflow, as this enables seamless parameter passing from the pipeline and allows for flexible specification of both primary and disaster recovery Lakehouse environments during runtime.

![image](https://github.com/user-attachments/assets/b3232305-4697-489f-bbf3-33e348260dca)


# Notice

Please note that after restoring your Lakehouse tables, you may observe certain artifacts or discrepancies within your environment. This behavior is currently okay in our testing, and the SQL endpoint queries for the Lakehouse should remain fully operational. We will provide more guidance to resolve this as we get them. 

![image](https://github.com/user-attachments/assets/b70f375a-ba8f-4438-81c0-7f0c607efab8)




Pelase note that after restoring your lakehosue tables you may see the folloiwng in your environmetn, 

# **Step 4: Restore Lakehouse Files Notebook**

Now we will process Lakehouse Files: 


```
# Source
source_paths=mssparkutils.fs.ls(f'abfss://{source}@msit-onelake.dfs.fabric.microsoft.com/')

sorurce_filtered_paths = [
    path.path
    for path in source_paths
    if path.path.endswith(".Lakehouse") and "Staging" not in path.path
]

print("Filtered Lakehouse Paths:")
for p in sorurce_filtered_paths:
    print(p)
```


```
# Destination
destination_paths=mssparkutils.fs.ls(f'abfss://{destination}@msit-onelake.dfs.fabric.microsoft.com/')

destination_filtered_paths = [
    path.path
    for path in destination_paths
    if path.path.endswith(".Lakehouse") and "Staging" not in path.path
]

print("Filtered Lakehouse Paths:")
for p in destination_filtered_paths:
    print(p)
```


```
paths_pairs = list(zip(sorurce_filtered_paths, destination_filtered_paths))
```


```
paths_pairs
```

```
from datetime import datetime

for source_lakehouse, dest_lakehouse in paths_pairs:
    files_dir = f"{source_lakehouse}/Files"

    try:
        file_folders = mssparkutils.fs.ls(files_dir)
    except Exception as e:
        print(f"No 'Files' directory in {source_lakehouse}, skipping.")
        continue

    for item in file_folders:
        item_path = item.path.rstrip("/")
        item_name = item.name
        dest_item_path = f"{dest_lakehouse}/Files/{item_name}"

        print(f"Recursively copying from:\n  {item_path}\n  → {dest_item_path}")

        try:
            mssparkutils.fs.cp(item_path, dest_item_path, recurse=True)
            print(f"Done copying file/folder: {item_name}\n")
        except Exception as e:
            print(f"Failed to copy {item_path}: {e}\n")
```


# **Step 4a: Restore Lakehouse Files Pipeline**

To implement this approach, construct a pipeline analogous to the one described in section 3a. Configure the pipeline and associated notebook parameters in alignment with those outlined in 3a, ensuring both `source` and `destination` parameters are systematically defined and propagated. This setup enables parameterized, automated execution and maintains consistency across environments.

![image](https://github.com/user-attachments/assets/b3232305-4697-489f-bbf3-33e348260dca)



# **Step 5: Restore Data Warehouse Notebook**

The process of restoring a Fabric Data Warehouse involves a specialized workflow, as the underlying data resides in OneLake in Delta Parquet format. Given the presence of multiple files, the data must first be transferred to a Lakehouse environment. From there, a CREATE TABLE AS SELECT (CTAS) command is executed to reconstitute the data within the Data Warehouse. The provided scripts facilitate this operation by systematically handling the data migration one Data Warehouse at a time. This approach requires specification of both the source and target workspaces, as well as the Fabric source Data Warehouse and the target Fabric Lakehouse. Data is migrated from the source Warehouse to the Lakehouse, and subsequently restored into the destination Warehouse, ensuring integrity and consistency throughout the process. Accordingly, the process necessitates provisioning a new Lakehouse instance to serve as the initial landing zone for the migrated data, followed by the deployment of a Disaster Recovery (DR) Data Warehouse to facilitate restoration from the primary environment.


```
import sempy.fabric as fabric
# Get the workspace ID

workspace_df = fabric.list_workspaces()
workspace_df = workspace_df[workspace_df['Name'].str.lower() == destination.lower()]
workspace_id = workspace_df['Id'].values[0]

workspace_id

```



```
# Source
source_paths=mssparkutils.fs.ls(f'abfss://{source}@msit-onelake.dfs.fabric.microsoft.com/')

sorurce_filtered_paths = [
    path.path
    for path in source_paths
    if path.path.endswith(f"{fabric_source_dw_name}.Warehouse") and "Staging" not in path.path
]

print("Filtered Lakehouse Paths:")
for p in sorurce_filtered_paths:
    print(p)

```



```
# Destination
destination_paths=mssparkutils.fs.ls(f'abfss://{destination}@msit-onelake.dfs.fabric.microsoft.com/')

destination_filtered_paths = [
    path.path
    for path in destination_paths
    if path.path.endswith(f"{fabric_destination_lakehouse_name}.Lakehouse") and "Staging" not in path.path
]

print("Filtered Lakehouse Paths:")
for p in destination_filtered_paths:
    print(p)

```



```
paths_pairs = list(zip(sorurce_filtered_paths, destination_filtered_paths))

```



```
paths_pairs
```


```
from datetime import datetime

# Output for CTAS
warehouse_create_table_list = []

for source_lakehouse, dest_lakehouse in paths_pairs:
    tables_dir = f"{source_lakehouse}/Tables"

    try:
        schema_folders = mssparkutils.fs.ls(tables_dir)
    except Exception as e:
        print(f"No 'Tables' directory in {source_lakehouse}, skipping.")
        continue

    for schema_folder in schema_folders:
        schema_path = schema_folder.path.rstrip("/")
        try:
            table_folders = mssparkutils.fs.ls(schema_path)
        except Exception:
            print(f"No tables in schema {schema_path}, skipping.")
            continue

        for table in table_folders:
            table_path = table.path.rstrip("/")
            dest_table_path = f"{dest_lakehouse}/Tables/{schema_folder.name}/{table.name}"

            print(f"Recursively copying from:\n  {table_path}\n  -> {dest_table_path}")
            warehouse_create_table_list.append({
            "schema_table_name": schema_folder.name + '.' + table.name,
            "source_lakehouse_table_info": fabric_destination_lakehouse_name +'.' + schema_folder.name + '.' + table.name,
            "warehouse_name":fabric_source_dw_name,
            "workspace_id" :workspace_id,
            "schema_name": schema_folder.name,
            "table_name" : table.name,
            "fabric_destination_lakehouse_name": fabric_destination_lakehouse_name,
            "warehouse_name":fabric_final_destination_dw_name,
            "datawarehouse_connection_string" :datawarehouse_connection_string
            })

            try:
                # Recursively copy the full table directory including all delta logs and data
                mssparkutils.fs.cp(table_path, dest_table_path, recurse=True)
                print(f"Done copying full table directory: {schema_folder.name}.{table.name}\n")
            except Exception as e:
                print(f"Failed to copy {table_path}: {e}\n")


```



```
# Exit value for notebook for Loop
mssparkutils.notebook.exit(warehouse_create_table_list)

```



# **Step 5a: Restore Data Warehouse Pipeline**

The next phase involves designing a pipeline comprising three core activities, each parameterized for runtime execution. This pipeline will feature a notebook activity, a foreach loop, and a script activity, orchestrated as outlined below to ensure efficient and automated data migration.

![image](https://github.com/user-attachments/assets/8ebe77e3-5f68-4fa6-a6c9-eee35cc33766)

Accurate configuration of these parameters will help ensure a smooth and efficient data migration process.
# Data Migration Parameters
Ensure the following parameters are configured correctly in the parameter table of the pipeline to facilitate seamless data staging and ingestion:
- **source**: The name of your source workspace
- **destination**: The name of your destination workspace for both the DR Data Warehouse and staging Lakehouse
- **fabric_source_dw_name**: The name of the primary site's Data Warehouse from which data will be extracted
- **fabric_destination_lakehouse_name**: The name of the destination Lakehouse where the data will be staged prior to loading into the Data Warehouse
- **datawarehouse_connection_string**: The connection string for your DR Data Warehouse
- **fabric_final_destination_dw_name**: The name of your DR Data Warehouse
Accurate configuration of these parameters will help ensure a smooth and efficient data migration process.
![image](https://github.com/user-attachments/assets/331e787f-9a16-44c3-84a5-a3695681295c)



# Add the notebook from step 5 and the base parameters and their values: 
- **source**: `@pipeline().parameters.source`
- **destination**: `@pipeline().parameters.destination`
- **fabric_source_dw_name**: `@pipeline().parameters.fabric_source_dw_name`
- **fabric_destination_lakehouse_name**: `@pipeline().parameters.fabric_destination_lakehouse_name`
- **datawarehouse_connection_string**: `@pipeline().parameters.datawarehouse_connection_string`
- **fabric_final_destination_dw_name**: `@pipeline().parameters.fabric_final_destination_dw_name`

![image](https://github.com/user-attachments/assets/ee5dee17-9429-42bd-a1b3-b86acf20a7e3)



Next add the foreach loop and the following expression in the item window: `@json(activity('Warehouse to LakeHouse DR Notebook1').output.result.exitValue)`

![image](https://github.com/user-attachments/assets/8f8d2af1-9bf7-4130-ab58-ead7088236a7)

Proceed by configuring the foreach loop to include a script activity. On the settings tab, verify that all parameter values are accurately aligned with the reference example below to ensure consistency and correctness throughout the integration process. Utilize the CTAS (Create Table As Select) syntax as demonstrated in the reference image below to ensure proper implementation in your workflow and ensure that they are no new line spaces in the string: `@concat('CREATE TABLE ', item().schema_table_name, ' AS SELECT * FROM ', item().source_lakehouse_table_info  )`

![image](https://github.com/user-attachments/assets/f38c4229-25d9-4478-8801-6ebb70699ebf)


![image](https://github.com/user-attachments/assets/e7458332-4138-41b3-8c5e-0841f59806c1)


***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. 
THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS 
FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) 
to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is 
embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the 
Sample Code.***






