# Fabric Notebook Default Lakehouse for Current Session
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >


In certain scenarios, it may be necessary to load a Lakehouse into a notebook to process datasets from a designated Lakehouse. The following configuration command provides the functionality to achieve this:

```
%%configure
{
    "defaultLakehouse": { 
        "name": "<Lake_House_Name_Here>",
        "id": "<Lake_House_ID_HERE",
        "workspaceId": "<Workspace-ID-That-Contains-The-Lakehouse" 
    }
}
```

![image](https://github.com/user-attachments/assets/1715e177-ce81-4a50-a1de-35f6540feda4)


Upon successful execution of the above command, the full qualifier can then be used to access resources within the specified Lakehouse.

![image](https://github.com/user-attachments/assets/4a0c75e0-2d66-4246-8591-b619f75f3762)


Reference: https://learn.microsoft.com/en-us/fabric/data-engineering/author-execute-notebook
