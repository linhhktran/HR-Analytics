# HR Analytics: Job Change of Data Scientists
![](https://blog.darwinbox.com/hubfs/MicrosoftTeams-image%20(23).png)
## Introduction
A company which is active in Big Data and Data Science wants to hire data scientists among people who successfully pass some courses which conduct by the company.

Many people signup for their training. Company wants to know which of these candidates are really wants to work for the company after training or looking for a new employment because it helps to reduce the cost and time as well as the quality of training or planning the courses and categorization of candidates.

Information related to demographics, education, experience are in hands from candidates signup and enrollment.

## Data sources
Data sources will be taken from different sources, such as: databases, websites, excel, csv, google sheet... Specifically as follows:
### 1. Enrollies' data
As enrollies are submitting their request to join the course via Google Forms, we have the Google Sheet that stores data about enrolled students, containing the following columns:

- enrollee_id: unique ID of an enrollee
- full_name: full name of an enrollee
- city: the name of an enrollie's city
- gender: gender of an enrollee
The source: https://docs.google.com/spreadsheets/d/1VCkHwBjJGRJ21asd9pxW4_0z2PWuKhbLR3gUHm-p4GI/edit?usp=sharing
### 2. Enrollies' education
After enrollment everyone should fill the form about their education level. This form is being digitalized manually. Educational department stores it in the Excel format here: https://assets.swisscoding.edu.vn/company_course/enrollies_education.xlsx

This table contains the following columns:

- enrollee_id: A unique identifier for each enrollee. This integer value uniquely distinguishes each participant in the dataset.

- enrolled_university: Indicates the enrollee's university enrollment status. Possible values include no_enrollment, Part time course, and Full time course.

- education_level: Represents the highest level of education attained by the enrollee. Examples include Graduate, Masters, etc.

- major_discipline: Specifies the primary field of study for the enrollee. Examples include STEM, Business Degree, etc.

### 3. Enrollies' working experience
Another survey that is being collected manually by educational department is about working experience.

Educational department stores it in the CSV format here: https://assets.swisscoding.edu.vn/company_course/work_experience.csv

This table contains the following columns:

- enrollee_id: A unique identifier for each enrollee. This integer value uniquely distinguishes each participant in the dataset.

- relevent_experience: Indicates whether the enrollee has relevant work experience related to the field they are currently studying or working in. Possible values include has relevent experience and No relevent experience.

- experience: Represents the number of years of work experience the enrollee has. This can be a specific number or a range (e.g., >20, <1).

- company_size: Specifies the size of the company where the enrollee has worked, based on the number of employees. Examples include 50−99, 100−500, etc.

- company_type: Indicates the type of company where the enrollee has worked. Examples include Pvt Ltd, Funded Startup, etc.

- last_new_job: Represents the number of years since the enrollee's last job change. Examples include never, >4, 1, etc.

### 4. Training hours
From LMS system's database you can retrieve a number of training hours for each student that they have completed.

Database credentials:

- Database type: MySQL
- Host: 112.213.86.31
- Port: 3360
- Login: etl_practice
- Password: 550814
- Database name: company_course
- Table name: training_hours

### 5. City development index
Another source that can be usefull is the table of City development index.

The City Development Index (CDI) is a measure designed to capture the level of development in cities. It may be significant for the resulting prediction of student's employment motivation.

It is stored here: https://sca-programming-school.github.io/city_development_index/index.html

### 6. Employment
From LMS database you can also retrieve the fact of employment. If student is marked as employed, it means that this student started to work in our company after finishing the course.

Database credentials:

- Database type: MySQL
- Host: 112.213.86.31
- Port: 3360
- Login: etl_practice
- Password: 550814
- Database name: company_course
- Table name: employment

## ETL (Extract - Transform - Load)
There are many separate datasets that need to be gathered into one dataset. Therefore, the ETL process needs to be done. The following codes are written on an IDE named Visual Studio Code (VSCode), including steps such as: extracting data from sources, transforming data and loading data to an existing data warehouse

### Step 1. Extract the data
Firstly, we need to import some packages to operate the code
```python
import os
import pandas as pd
import requests
from sqlalchemy import create_engine
import pymysql
```
By coding on VSCode, I encounter most problems relating to missing libraries or packages. The syntax to install a library is:
```python
pip install (name of library)
```

After installing all the necessary libraries and packages, we can begin writing code to extract data from sources. The data should be automatically downloaded each time the script is executed, ensuring that we always work with the most up-to-date information.

```python
# Extract data from the google sheets
gg_sheet_id = '1VCkHwBjJGRJ21asd9pxW4_0z2PWuKhbLR3gUHm-p4GI'
gg_sheet_url = 'https://docs.google.com/spreadsheets/d/' + gg_sheet_id + '/export?format=xlsx'
df = pd.read_excel(gg_sheet_url, sheet_name='enrollies')

# Extract data from the xlsx file
excel_url = 'https://assets.swisscoding.edu.vn/company_course/enrollies_education.xlsx'
# Check if the files exists before downloading them
if not os.path.exists('enrollies_education.xlsx'):
  excel_response = requests.get(excel_url.strip())
  with open('enrollies_education.xlsx', 'wb') as file:
    file.write(excel_response.content)
enrollies_education = pd.read_excel('enrollies_education.xlsx')

# Extract data from the csv file
csv_url = 'https://assets.swisscoding.edu.vn/company_course/work_experience.csv'
if not os.path.exists('work_experience,csv'):
  csv_response = requests.get(csv_url)
  with open('work_experience.csv', 'wb') as file:
    file.write(csv_response.content)
work_experience = pd.read_csv('work_experience.csv')

# Extract data from database
engine = create_engine('mysql+pymysql://etl_practice:550814@112.213.86.31:3360/company_course')
training_hours = pd.read_sql_table('training_hours', con=engine)

engine_1 = create_engine('mysql+pymysql://etl_practice:550814@112.213.86.31:3360/company_course')
employment = pd.read_sql_table('employment', con=engine_1)

# Extract data from a website
table = pd.read_html('https://sca-programming-school.github.io/city_development_index/index.html')
city = table[0]

```

### Step 2. Transform the data

In each dataset, missing values (N/A) and appropriate data types must be properly handled. However, in the previous Google Colab Notebook, redundant code was observed across multiple datasets. To eliminate repetition and improve efficiency, a function was created to standardize the data transformation process. The function applies the following steps:

* Check all columns in every dataset to detect missing (N/A) values.
* For columns containing missing values, determine the data type and handle them accordingly:
  * If the column is numeric, fill N/A values with the median.
  * If the column is categorical (object or category), fill N/A values with 'Unknown'.

```python
# Create a function to transform data
def handling_missing_value(df):
  for col in df.columns:
    if df[col].isna().sum() > 0:
      if pd.api.types.is_numeric_dtype(df[col]):
        df[col].fillna(df[col].median(), inplace=True)
      elif pd.api.types.is_object_dtype(df[col]) or pd.api.types.is_categorical_dtype(df[col]):
        df[col].fillna('Unknown', inplace=True)
        df[col] = df[col].str.lower()
  return df

# Create a dictionary to store all of the tables in the dataset
data_tables = {
  'enrollies': df,
  'enrollies_education': enrollies_education,
  'work_experience': work_experience,
  'training_hours': training_hours,
  'employment': employment,
  'city': city
}
# Run a loop for handling missing values in those six data tables
for table_name, data_frame in data_tables.items():
  data_tables[table_name] = handling_missing_value(data_frame)
```
### Step 3. Load the data to data warehouse
After transforming the data, we load it into a data warehouse for storage and future analysis. To keep things simple, we use an SQLite database in DBeaver as our data warehouse, ensuring efficient storage and easy access to all datasets

```python
# Path to the SQLite database
db_path = 'data_warehouse.db'
# Create an SQL Alchemy engine
engine_2 = create_engine(f'sqlite:///{db_path}')

df.to_sql('Dim_enrollies', con=engine_2, if_exists='replace', index=False)
enrollies_education.to_sql('Dim_education', con=engine_2, if_exists='replace', index=False)
work_experience.to_sql('Dim_experience', con=engine_2, if_exists='replace', index=False)
training_hours.to_sql('Dim_training', con=engine_2, if_exists='replace', index=False)
city.to_sql('Dim_city', con=engine_2, if_exists='replace', index=False)
employment.to_sql('Fact_employment', con=engine_2, if_exists='replace', index=False)
```
### Step 4. Task scheduling
So we have written a cycle to automatically extract data from many different data sources. And every day, according to the set time, the system will automatically access the above links to automatically retrieve and update new data (if any), without having to manually edit when there are changes to the original data.

So we have completed a piece of code to run the ETL cycle. The last thing is to set up a schedule so that the system can automatically get data from sources at a predetermined time. I am using Windows so I chose Task Scheduler to perform this task, specifically as follows:
- Open Task Scheduler: search for "Task Scheduler" in Start menu and open it.
- Create a New Task:
  - In the right-hand pane, click on "Create Basic Task"
  - Give your task a name and description, then click "Next"
- Trigger the Task:
  - Select "Daily" and click "Next"
  - Set the start date and time, and specify that the task should recur every day.
- Action:
  - Select "Start a Program" and click "Next"
  - Click "Browse" and select the Python executable
  - In the "Add arguments (optional)" field, enter the path to your ETL script.
- Finish the Task: Review your settings and click "Finish".
