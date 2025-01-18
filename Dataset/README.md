# Dataset Overview
Here's a visual representation of our dataset:

<div align="center">
  <img src="/Images/Salary Data.png" width="70%">
  <p>Salary Dataset</p>
</div>



<br/>

# Table of Contents
- [Purpose of This README](#purpose-of-this-readme)
- [Dataset Details](#dataset-details)
- [Accessing the Dataset](#accessing-the-dataset)
- [Data Transformation Process](#data-transformation-process)

    - [Query 1](#query-1)
    - [Explanation](#explanation)
    - [Query 2](#query-2)
    - [Explanation](#explanation-1)

- [Replication Guide](#replication-guide)
- [ETL Implementation](#etl-implementation)

<br/>

# Purpose of This README

This documentation specifically focuses on the dataset's technical details, including its source, transformations, and preprocessing steps. For information about the main project and implementation details, please refer to our main [README](/README.md).

<br/>

# Dataset Details
You can obtain the dataset from [Kaggle](https://www.kaggle.com/datasets/lukebarousse/data-analyst-job-postings-google-search). This dataset is dynamic in nature, compiling data analyst job postings from the United States. The collection began on November 4, 2022, with approximately 100 new job postings added daily. Special thanks to Luke Barousse for his efforts in creating and maintaining this valuable resource.

<br/>

# Accessing the Dataset
You can access the dataset in two ways:
- **Manual Download**: Download the dataset directly from Kaggle.
- **Remote Source**: Access the dataset through a provided [remote link](https://storage.googleapis.com/gsearch_share/gsearch_jobs).


The first dashboard will utilize the manually downloaded dataset, while the second dashboard will draw data from the remote source.

<br/>

# Data Transformation Process


The dataset underwent preprocessing using Power Query. We've simplified the transformation process into just two queries, making it efficient and easily reproducible.

<br/>

### Query 1

```powerquery
let
    Source = Csv.Document(File.Contents("C:\Users\astro\Desktop\gsearch_jobs.csv"),[Delimiter=",", Columns=27, Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",{{"", Int64.Type}, {"index", Int64.Type}, {"title", type text}, {"company_name", type text}, {"location", type text}, {"via", type text}, {"description", type text}, {"extensions", type text}, {"job_id", type text}, {"thumbnail", type text}, {"posted_at", type text}, {"schedule_type", type text}, {"work_from_home", type logical}, {"salary", type text}, {"search_term", type text}, {"date_time", type datetime}, {"search_location", type text}, {"commute_time", type text}, {"salary_pay", type text}, {"salary_rate", type text}, {"salary_avg", type number}, {"salary_min", type number}, {"salary_max", type number}, {"salary_hourly", type number}, {"salary_yearly", Int64.Type}, {"salary_standardized", type number}, {"description_tokens", type text}}),
    #"Removed """", index, description, job_id, thumbnail, search_term, search_location, commute_time, salary_pay, salary & posted_at" = Table.RemoveColumns(#"Changed Type",{"", "index", "description", "job_id", "thumbnail", "search_term", "search_location", "commute_time", "salary_pay", "salary", "posted_at"}),
    #"Renamed via to jobs_via" = Table.RenameColumns(#"Removed """", index, description, job_id, thumbnail, search_term, search_location, commute_time, salary_pay, salary & posted_at",{{"via", "jobs_via"}}),
    #"Removed via from jobs_via" = Table.ReplaceValue(#"Renamed via to jobs_via","via","",Replacer.ReplaceText,{"jobs_via"}),
    #"Trimmed location & jobs_via" = Table.TransformColumns(#"Removed via from jobs_via",{{"location", Text.Trim, type text}, {"jobs_via", Text.Trim, type text}}),
    #"Replaced an hour to hourly" = Table.ReplaceValue(#"Trimmed location & jobs_via","an hour","hourly",Replacer.ReplaceText,{"salary_rate"}),
    #"Replaced a year to yearly" = Table.ReplaceValue(#"Replaced an hour to hourly","a year","yearly",Replacer.ReplaceText,{"salary_rate"}),
    #"Removed rows before 1/1/2024" = Table.SelectRows(#"Replaced a year to yearly", each [date_time] > #datetime(2023, 12, 31, 0, 0, 0)),
    #"Added no_degree_mentioned" = Table.AddColumn(#"Removed rows before 1/1/2024", "no_degree_mentioned", each if Text.Contains([extensions], "No degree mentioned") then true else false),
    #"Added health_insurance" = Table.AddColumn(#"Added no_degree_mentioned", "health_insurance", each if Text.Contains([extensions], "Health insurance") then true else false),
    #"Added dental_insurance" = Table.AddColumn(#"Added health_insurance", "dental_insurance", each if Text.Contains([extensions], "Dental insurance") then true else false),
    #"Added paid_time_off" = Table.AddColumn(#"Added dental_insurance", "paid_time_off", each if Text.Contains([extensions], "Paid time off") then true else false),
    #"Replaced Value" = Table.ReplaceValue(#"Added paid_time_off","Anywhere","Remote",Replacer.ReplaceText,{"location"}),
    #"Removed brackets from location" = Table.AddColumn(#"Replaced Value", "location_cleaned", each Text.BeforeDelimiter([location], "("), type text),
    #"Trimmed location_cleaned" = Table.TransformColumns(#"Removed brackets from location",{{"location_cleaned", Text.Trim, type text}}),
    #"Added seniority" = Table.AddColumn(#"Trimmed location_cleaned", "seniority", each if Text.Contains(Text.Lower([title]), "sr") or Text.Contains(Text.Lower([title]), "senior") 
then "Senior"
else "Non-Senior"),
    #"Replaced null with FALSE in work_from_home" = Table.ReplaceValue(#"Added seniority",null,false,Replacer.ReplaceValue,{"work_from_home"}),
    #"Added job_title_short" = Table.AddColumn(#"Replaced null with FALSE in work_from_home", "job_title", each if Text.Contains(Text.Lower([title]), "business analyst") or Text.Contains(Text.Lower([title]), "business analytics") or Text.Contains(Text.Lower([title]), "business intelligence") then "Business Analyst"
else if Text.Contains(Text.Lower([title]), "cloud") and Text.Contains(Text.Lower([title]), "engineer") then "Cloud Engineer"
else if Text.Contains(Text.Lower([title]), "data analysis") or Text.Contains(Text.Lower([title]), "data analytics") or Text.Contains(Text.Lower([title]), "data analyst") then "Data Analyst"
else if Text.Contains(Text.Lower([title]), "data engineer") then "Data Engineer"
else if Text.Contains(Text.Lower([title]), "data science") or Text.Contains(Text.Lower([title]), "data scientist") then "Data Scientist"
else null),
    #"Filtered null from job_title_short" = Table.SelectRows(#"Added job_title_short", each ([job_title] <> null)),
    #"Added job_title_final with seniority" = Table.AddColumn(#"Filtered null from job_title_short", "job_title_final", each if [seniority] = "Senior" then "Senior " & [job_title]
else if [seniority] = "Non-Senior" then [job_title]
else null),
    #"Inserted schedule_type_final" = Table.AddColumn(#"Added job_title_final with seniority", "schedule_type_final", each Text.Start([schedule_type], 9), type text),
    #"Replaced Contracto to Contractor" = Table.ReplaceValue(#"Inserted schedule_type_final","Contracto","Contractor",Replacer.ReplaceText,{"schedule_type_final"}),
    #"Replaced Internshi to Internship" = Table.ReplaceValue(#"Replaced Contracto to Contractor","Internshi","Internship",Replacer.ReplaceText,{"schedule_type_final"}),
    #"Added Index" = Table.AddIndexColumn(#"Replaced Internshi to Internship", "index", 1, 1, Int64.Type),
    #"Reordered Columns" = Table.ReorderColumns(#"Added Index",{"index", "title", "job_title_final", "job_title", "seniority", "company_name", "location", "location_cleaned", "jobs_via", "extensions", "work_from_home", "no_degree_mentioned", "health_insurance", "dental_insurance", "paid_time_off", "schedule_type", "schedule_type_final", "date_time", "salary_rate", "salary_avg", "salary_min", "salary_max", "salary_hourly", "salary_yearly", "salary_standardized", "description_tokens"})
in
    #"Reordered Columns"
```

## Explanation

#### Initial Setup
- Reads CSV file with 27 columns using UTF-8 encoding
- Promotes headers and sets appropriate data types for all columns
- Removes unnecessary columns like index, description, job_id, thumbnail, etc.
#### Data Cleaning Steps
- Renames 'via' column to 'jobs_via' and removes "via" text
- Trims whitespace from location and jobs_via columns
- Standardizes salary rate terms ('an hour' → 'hourly', 'a year' → 'yearly')
- Filters data to include only entries after December 31, 2023
#### Feature Engineering
- Adds boolean columns for benefits:
    - *no_degree_mentioned*
    - *health_insurance*
    - *dental_insurance*
    - *paid_time_off*
- Creates cleaned location column by removing text in brackets
- Adds seniority classification (Senior/Non-Senior)
- Standardizes 'Remote' work locations
- Creates job title categories (Data Analyst, Business Analyst, etc.)
- Standardizes schedule types (Contractor, Internship)
#### Final Touches
- Adds new index column
- Reorders columns for better organization
- Filters out null job titles
- Creates final job titles with seniority prefixes

<br/>

### Query 2

```plaintext
let
    Source = #"Data Jobs Cleaned",
    #"Removed Columns" = Table.RemoveColumns(Source,{"job_title", "seniority", "location", "extensions", "title", "schedule_type"})
in
    #"Removed Columns"
```

  
### Explanation: 

- Takes "Data Jobs Cleaned" as source table
- Removes 6 redundant columns:
    - *job_title*
    - *seniority*
    - *location*
    - *extensions*
    - *title*
    - *schedule_type*

<br/>

# Replication Guide
- Open Power Query Editor
- Access the Advanced Editor (Alt + F12)
- Paste the provided M-Language queries
- Apply transformations in sequence:

<br/>

# ETL Implementation
The dashboard connected to the remote source enables automated data updates through a simple refresh mechanism. This implementation demonstrates Excel's ETL (Extract, Transform, Load) capabilities:
- **Extract**: Data pulled from remote source
- **Transform**: Automated application of our Power Query transformations
- **Load**: One-click refresh for updated insights