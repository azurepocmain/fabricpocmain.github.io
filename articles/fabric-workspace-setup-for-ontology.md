# Microsoft Fabric Workspace setup for Fabric Ontology 
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

## Steps:

# **Step1:**

Create a Microsoft Fabric capacity, ensuring that it is deployed in the appropriate region to meet any applicable data privacy and residency requirements.

<img width="811" height="229" alt="image" src="https://github.com/user-attachments/assets/d82d372e-9891-41d5-8d8d-1af36cae54f9" />

<img width="807" height="667" alt="image" src="https://github.com/user-attachments/assets/7dbc7e23-5dcb-4ed3-bbdb-21f6a5ea6c07" />


# **Step2:**

Next, sign in to Microsoft Fabric, select "Workspaces" from the left navigation pane, and then choose "+ New workspace" at the bottom of the page.
Ensure that "Fabric" is selected as the capacity. If you plan to use a semantic model larger than 1 GB, select "Large semantic model storage format" as the semantic model storage format.


<img width="349" height="221" alt="image" src="https://github.com/user-attachments/assets/d0edabd2-e2f6-4385-9e16-12c18a5edfa8" />

<img width="633" height="579" alt="image" src="https://github.com/user-attachments/assets/bf79268c-ec4c-4808-8bf9-b98b3c25a5e6" />

<img width="644" height="851" alt="image" src="https://github.com/user-attachments/assets/bbf96464-2ab7-4e9a-8343-96a6a3e633b3" />

# **Step3:**

After the workspace has been created, select "New item," and then create a Lakehouse to load or mirror your data in Microsoft Fabric.

<img width="808" height="98" alt="image" src="https://github.com/user-attachments/assets/792c9cde-299d-4265-8a1f-4c598c7f2409" />

<img width="975" height="392" alt="image" src="https://github.com/user-attachments/assets/a09e9c3a-9415-4ace-9d39-2990b37d8185" />

<img width="651" height="359" alt="image" src="https://github.com/user-attachments/assets/4b288e5a-d5d0-45dd-bc83-a4acb4a18906" />



# **Step4:**

For this example, we will load the data into Microsoft Fabric by using the "New copy job" activity. For more advanced data transformations, select Dataflow. Alternatively, use a shortcut to mirror or reference data, or use Eventstream to ingest and process streaming data.
After selecting the required tables, choose the appropriate copy method based on the connection type and data-update pattern: use an incremental copy for data that will be updated regularly, or a full copy for data that is expected to remain unchanged.

<img width="480" height="413" alt="image" src="https://github.com/user-attachments/assets/e82761c6-9b21-4282-b798-0d0eda6c30f3" />

<img width="975" height="456" alt="image" src="https://github.com/user-attachments/assets/f9a63bc0-fdfb-4881-a2c3-120ad263afa6" />


# **Step5:**

One efficient way to join your data is to use the semantic model directly from the Lakehouse. Select "New semantic model," and record both the model ID and workspace ID.
You can then use the Microsoft Fabric semantic model APIs to create the required joins and relationships. This is a critical step because the graph relies on these definitions to establish meaningful connections across your data.
Take care to define the relationships and cardinality accurately, as they are essential to the integrity and effectiveness of the model.

<img width="975" height="150" alt="image" src="https://github.com/user-attachments/assets/36a41eac-d8e8-4f71-8eb4-bae680084bb4" />

<img width="513" height="625" alt="image" src="https://github.com/user-attachments/assets/b8d59079-763f-419a-80e7-5186db8c1148" />


# **Step6:**

After defining the relationships, you can create the ontology directly from the semantic model.

<img width="975" height="138" alt="image" src="https://github.com/user-attachments/assets/ee9ff109-0f54-48bd-a14a-59378a76d9b8" />


# **Step7:**

Check each entity to ensure that the binding look correct and add any additional metadata if initially omitted.
<img width="975" height="258" alt="image" src="https://github.com/user-attachments/assets/a3c24e26-bcab-476e-b8a5-75868e866f7c" />


# **Step8:**
Permissions, ensure that the user has at least viewer role on the workspace or read permission on the ontology. 
In addition, the user account must be assigned to the Lakehouse OneLake Security role with the ReadAll permission, or specific tables you would like the user to have for the bind data. 
Without this permission, the ontology search function will be unable to access or search the data, and you will get *404* Not Found errors. 

<img width="975" height="217" alt="image" src="https://github.com/user-attachments/assets/afea4353-a005-4df6-b94b-41f2a56628d8" />
