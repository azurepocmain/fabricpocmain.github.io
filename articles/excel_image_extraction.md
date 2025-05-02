# Extract Images From Excel Example End To End
<link rel="icon" href="articles/fabric_16_color.svg" type="image/x-icon" >

This example demonstrates the process of extracting images from an Excel file and saving them into a PDF document. 
Subsequently, the process iterates through each page of the PDF, converting it into one or more image files based on the number of distinct images identified per page in the Excel file. 
It is important to note that if multiple images are in physical contact, the program will treat them as a single composite image.
The approach relies on the xlwings library for opening and interacting with Excel files, while ensuring that the images are correctly embedded into the generated PDF. 
Note that the functionality may depend on the operating system's compatibility with xlwings modules.


**Prerequisites:**
```
pip install xlwings 
pip install pdf2image
pip install opencv-python
pip install pillow

Install Poppler the zip file on Windows:
Download from:
https://github.com/oschwartz10612/poppler-windows/releases

Extract to: C:\poppler
You will just need to create the  C:\poppler path and copy the Library folder from the extracted file. 

Your folder structure should look like:
C:\poppler\Library\bin\pdftoppm.exe
C:\poppler\Library\bin\pdfinfo.exe


```

**Create First Function For Excel To PDF Extraction:**
```
import xlwings as xw
import os
import time

def excel_image_extract_to_pdf(input_file, output_folder):
# Unfortunately this needs to be run on a windows machine with excel as xlwings needs to open the excel file
# Saves all the images to a PDF file for each page
    wb = xw.Book(input_file)

    output_dir = output_folder
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


**Create Second Function For PDF To Image Conversion:**
```
import os
import cv2
import numpy as np
from pathlib import Path
from PIL import Image
from pdf2image import convert_from_path

# Image to PDF conversion using png and 600 Dots Per Inch (dip) value
def detect_and_crop_multiple_images(pil_img, output_folder, prefix="page1", min_area=500):
    img_cv = cv2.cvtColor(np.array(pil_img), cv2.COLOR_RGB2BGR)
    gray = cv2.cvtColor(img_cv, cv2.COLOR_BGR2GRAY)

    # Step 1: Threshold to isolate non-white content
    _, thresh = cv2.threshold(gray, 240, 255, cv2.THRESH_BINARY_INV)

    # Step 2: Morphology to separate touching shapes
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (10, 10))  # Tune as needed
    separated = cv2.morphologyEx(thresh, cv2.MORPH_OPEN, kernel)

    # Step 3: Find external contours
    contours, _ = cv2.findContours(separated, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

    count = 0
    for cnt in contours:
        x, y, w, h = cv2.boundingRect(cnt)
        if w * h < min_area:
            continue

        margin = 10
        x1 = max(x - margin, 0)
        y1 = max(y - margin, 0)
        x2 = min(x + w + margin, pil_img.width)
        y2 = min(y + h + margin, pil_img.height)

        cropped = pil_img.crop((x1, y1, x2, y2))
        output_path = os.path.join(output_folder, f"{prefix}_img{count+1}.png")
        cropped.save(output_path)
        print(f"Saved: {output_path}")
        count += 1

def process_pdf(pdf_path, output_dir, dpi=600, poppler_path=None):
    pdf_name = Path(pdf_path).stem
    pages = convert_from_path(pdf_path, dpi=dpi, poppler_path=poppler_path)

    for i, page in enumerate(pages, start=1):
        page_folder = os.path.join(output_dir, f"{pdf_name}_page{i}")
        os.makedirs(page_folder, exist_ok=True)
        detect_and_crop_multiple_images(page, page_folder, prefix=f"{pdf_name}_p{i}")

def batch_process_folder(pdf_folder, output_dir, dpi=600, poppler_path=None):
    pdf_folder = Path(pdf_folder)
    for pdf_file in pdf_folder.glob("*.pdf"):
        print(f"\nProcessing PDF: {pdf_file.name}")
        process_pdf(str(pdf_file), output_dir, dpi=dpi, poppler_path=poppler_path)
```


**Invoking The Above Two Functions:**
```
# First Invoking the excel to image PDF extraction process
input_file="path/excel_file/file.xlsx"
output_folder="path/extracted_image_pdf/output"
excel_image_extract_to_pdf(input_file,output_folder )

# Second Invoking The PDF To Image Conversion Process
batch_process_folder(
    pdf_folder="path/pdf_folder/output",
    output_dir="path/image_folder/output",
    dpi=600,
    poppler_path=r"C:\poppler\Library\bin"  ## or None if installed system-wide this needs to be installed prior to the run, and the path \ are correct 
)

```





***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. 
THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS 
FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) 
to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is 
embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the 
Sample Code.***
