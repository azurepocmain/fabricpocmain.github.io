# Fabric Notebook Imported Values
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

Leveraging pipelines to invoke notebooks enables a streamlined and efficient data processing workflow, facilitating seamless integration between successive pipeline executions. By utilizing the outputs of previous pipeline runs as inputs for subsequent executions, this approach ensures optimal resource utilization and modularity in orchestration. 
The documentation below outlines the technical steps involved in implementing this process.

# *Step1:*

Create a dictionary of the variables you would like to use as outputs, as shown below:

```
output_payload = {
    "notebookname": notebookname,
    "currentdate": currentdate,
    "lakehouse_path": lakehouse_path,

}
```

# *Step2:*

Use the Spark exit function below to output the values from the dictionary you created earlier.

```
import json

# Exit with JSON string
mssparkutils.notebook.exit(json.dumps(output_payload))

```


# *Step3:*

In your pipeline, for the notebook to which you are passing parameters, ensure that the expression in the "Base parameters" variable is formatted as shown below and the "NotebookParameter1" is the pipeline Notebook name which you are getting the parameters from.

```
@activity('NotebookParameter1').output.result.exitValue
```

<img width="819" height="375" alt="image" src="https://github.com/user-attachments/assets/fedab637-5d1b-43a0-b4ff-fd2201cbef72" />


# *Step4:*

In that particular notebook, the cell will generate those runtime parameters.
Next, parse the values so you can use the variables in your Spark calls, either as shown below or by using similar methods.

```
import json

# Parse the JSON string into a dictionary
parsed_values = json.loads(notebookvalues)

# Assign to individual variables
notebookname = parsed_values["notebookname"]
currentdate = parsed_values["currentdate"]
lakehouse_path = parsed_values["lakehouse_path"]

```

# *Step5:*
You can test the variable output if required as well: 

```
print(f'The imported values are {notebookname} and {currentdate} and {lakehouse_path}')
```












