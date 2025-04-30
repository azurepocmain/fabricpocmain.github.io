# Extract Images From Excel Example
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

This example demonstrates the process of extracting images from an Excel file and saving them into a PDF document. 
The approach relies on the xlwings library for opening and interacting with Excel files, while ensuring that the images are correctly embedded into the generated PDF. 
Note that the functionality may depend on the operating system's compatibility with xlwings modules.



```
import xlwings as xw
import os
import time
# Unfortunately this needs to be run on a windows machine with excel as xlwings needs to open the excel file
# Saves all the iamges to a PDF file for each page
wb = xw.Book("path/excel_file/file.xlsx")

output_dir = "path/output"
os.makedirs(output_dir, exist_ok=True)

for sheet in wb.sheets:
    print(f"Processing sheet: {sheet.name}")

    if not sheet.shapes:
        print("No shapes found in this sheet.")

    #Save the active sheet as PDF temporarily
    temp_pdf = os.path.join(output_dir, f"{sheet.name}.pdf")

    try:
        sheet.api.ExportAsFixedFormat(0, temp_pdf)  # 0 = PDF format
        print(f"Exported {sheet.name} as PDF: {temp_pdf}")

    except Exception as e:
        print(f"Error exporting {sheet.name} as PDF: {e}")

wb.close()

```
