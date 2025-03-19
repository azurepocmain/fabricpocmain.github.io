# Fabric Notebook Default Lakehouse for Current Session
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >


In certain scenarios, it may be necessary to load a Lakehouse into a notebook programmatically to process datasets from a designated Lakehouse. The following configuration command provides the functionality to achieve this:

```
%%configure
{
    "defaultLakehouse": { 
        "name": "<Lake_House_Name_Here>",
        "id": "<Lake_House_ID_HERE",
        "workspaceId": "<Workspace-ID-That-Contains-The-Lakehouse>" 
    }
}
```

![image](https://github.com/user-attachments/assets/6ba178eb-2414-47cd-9ecf-94afddbd1434)



Upon successful execution of the above command, the full qualifier can then be used to access resources within the specified Lakehouse.

![image](https://github.com/user-attachments/assets/4a0c75e0-2d66-4246-8591-b619f75f3762)


Reference: <a href="https://learn.microsoft.com/en-us/fabric/data-engineering/author-execute-notebook" target="_blank">Author Execute Notebook</a>


