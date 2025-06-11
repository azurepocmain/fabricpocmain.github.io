# Fabric Business Continuity and Disaster Recovery (BCDR) 
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Business continuity and disaster recovery (BCDR) are critical components of a robust enterprise risk management framework. This documentation outlines the essential procedures for implementing and validating your BCDR strategy for Fabric.
Please note that this guide is not exhaustive, additional sections and detailed best practices will be incorporated in future updates.


**Steps**

**Step 1: Enable BCDR**
To enable disaster recovery capabilities in Microsoft Fabric, navigate to the Admin portal and select Capacity settings. From there, choose the appropriate Fabric Capacity, then access the Disaster Recovery configuration.

![image](https://github.com/user-attachments/assets/6cd45f87-b68e-458e-b304-0b4305ba73ba)

Activating this setting ensures that workspace items stored in OneLake are replicated to the designated secondary region for redundancy. 
However, it is important to note that not all objects, such as those managed by EventHouse, store their data in OneLake. 
For these services, alternative disaster recovery procedures must be implemented to ensure data resiliency.

**Step 2: Create DR Workspace and Item Services**
The subsequent step involves provisioning a workspace within the designated paired region and reestablishing corresponding Lakehouses, Warehouses, and job configurations. 
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



**Step 4: Restore Lakehouse Files Notebook**

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


**Step 4a: Restore Lakehouse Files Pipeline**

To implement this approach, construct a pipeline analogous to the one described in section 3a. Configure the pipeline and associated notebook parameters in alignment with those outlined in 3a, ensuring both `source` and `destination` parameters are systematically defined and propagated. This setup enables parameterized, automated execution and maintains consistency across environments.

![image](https://github.com/user-attachments/assets/b3232305-4697-489f-bbf3-33e348260dca)



**Step 5: Restore Data Warehouse Notebook**
