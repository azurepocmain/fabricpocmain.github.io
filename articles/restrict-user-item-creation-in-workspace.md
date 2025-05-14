# Restrict User Item Creation In Workspace
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >


In certain scenarios, it may be necessary to restrict users from creating items in a workspace while still allowing them the capability to generate reports. 
This document provides a detailed procedure for implementing restrictions on item creation within a workspace environment.

_______________________________________________________________________________________

**Solution**

In the `Admin Portal` -> `Capacity Settings` -> select the capacity -> `Delegated tenant settings` -> `Users can create Fabric items` -> `Except specific security group` and provide the security group users are assigned to.

![image](https://github.com/user-attachments/assets/72d1a442-02ec-46e6-8ffd-72fbe98bb20e)

Configuring the specified settings will effectively restrict users from generating Fabric items while still maintaining their ability to produce reports within the workspace allocated to the corresponding capacity.

![image](https://github.com/user-attachments/assets/e7a5981d-8c6c-42e0-b05c-9ce31f3b96ce)


