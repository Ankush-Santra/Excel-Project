# Overview of the Dataset

Below is a visual representation of the dataset:

<div align="center">
  <img src="https://github.com/Ankush-Santra/Excel-Project/blob/main/Images/Salary%20Data.png" width="80%">
</div>  

<br/>

# Purpose of This README

This README file is designed to inform users about the source of the dataset and the transformations applied to it. For more information about the project and access to related dashboards, please refer to the main [README](https://github.com/Ankush-Santra/Excel-Project/blob/main/README.md) file.

<br/>

# Dataset Details

You can access the Kaggle dataset [here](https://www.kaggle.com/datasets/lukebarousse/data-analyst-job-postings-google-search). This dataset compiles job postings for Data Analyst positions across the United States. The collection began on November 4, 2022, with approximately 100 new entries added daily. We extend our gratitude to Luke Barousse for his efforts in creating and maintaining this dataset.

<br/>

# Accessing the Dataset

There are two methods to obtain this dataset:

1.  **Download the Dataset**: You can simply download the dataset for personal use.
2.  **Access the Remote Source**: Alternatively, you can utilize the [remote source](https://storage.googleapis.com/gsearch_share/gsearch_jobs.csv) for real-time data insights at the click of a button.

The first dashboard will utilize the downloaded dataset, while the second dashboard will leverage the remote source for the most current information.

<br/>

# Data Transformation Overview

Both dashboards employ identical transformation processes, implemented through Power Query. I have developed two distinct queries, each with its specific set of transformations.

<br/>

## Query Details and Documentation

Below, you'll find the detailed code implementation along with comprehensive explanations for each query:

### Query 1

```plaintext
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

### Explanation

Here's a breakdown of the key transformations in the code:

#### Initial Data Loading and Basic Transformations

*   Loads a CSV file from the specified path with 27 columns
*   Promotes the first row to headers
*   Changes data types for all columns appropriately (text, numbers, datetime, etc.)

#### Data Cleaning Operations

*   Removes several unnecessary columns (index, description, job\_id, thumbnail, etc.)
*   Renames the 'via' column to 'jobs\_via' and removes the text "via" from values
*   Trims whitespace from 'location' and 'jobs\_via' columns
*   Standardizes salary rate terminology ("an hour" → "hourly", "a year" → "yearly")
*   Filters data to only include entries after December 31, 2023

#### Benefits and Requirements Analysis

*   Adds boolean columns for:
    *   No degree requirement
    *   Health insurance
    *   Dental insurance
    *   Paid time off
*   These are determined by checking the 'extensions' column for specific text

#### Location Processing

*   Replaces "Anywhere" with "Remote"
*   Creates a cleaned location column by removing text in parentheses
*   Trims the cleaned location

#### Job Title Processing

*   Adds a 'seniority' column (Senior/Non-Senior) based on title keywords
*   Creates standardized job titles based on keywords:
    *   Business Analyst
    *   Cloud Engineer
    *   Data Analyst
    *   Data Engineer
    *   Data Scientist
*   Combines seniority with job title for final job title classification

#### Final Formatting

*   Standardizes schedule types
*   Fixes typos in contract and internship terms
*   Adds a new index column
*   Reorders columns in a logical sequence

This transformation pipeline creates a clean, standardized dataset suitable for analysis and visualization.

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

This is a simple follow-up query that builds upon the previously cleaned data ("Data Jobs Cleaned"). Here's what it does:

#### Source Selection

*   Uses the output from the first query ("Data Jobs Cleaned") as its source

#### Column Removal

Removes redundant or unnecessary columns:

*   job\_title
*   seniority
*   location
*   extensions
*   title
*   schedule\_type

#### Purpose

This transformation creates a more streamlined version of the dataset by:

*   Eliminating duplicate information
*   Removing intermediate columns used in the previous transformations
*   Keeping only the final, processed columns needed for analysis

This simplified dataset likely serves as the foundation for specific visualizations or analyses where the removed columns aren't required.

<br/>

# How to Replicate This Work
## Step-by-Step Implementation Guide

1.  **Launch Power Query Editor**
    *   Open your Power BI Desktop
    *   Access the Power Query Editor through the 'Transform Data' button
2.  **Access the Advanced Editor**
    *   Navigate to the 'View' tab
    *   Click on 'Advanced Editor'
3.  **Implement the Code**
    *   Copy the provided code
    *   Paste it into the Advanced Editor window
    *   Click 'Done' to apply the transformations

## Data Source Options

You have two flexible options for data implementation:

1.  **Static Dataset**
    *   Download and use the provided dataset
    *   Perfect for one-time or periodic analysis
2.  **Remote Source Connection**
    *   Connect to the remote data source
    *   Enables automatic daily updates
    *   Implements real-time ETL (Extract, Transform, Load) processing
    *   Transformations are automatically applied to new data

## Key Advantage

By setting up this automated ETL pipeline, you can maintain an up-to-date analysis with minimal manual intervention. The transformations will automatically process any new job listings as they're added to the dataset.
