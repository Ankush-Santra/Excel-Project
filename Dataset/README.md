Here's a sneak preview of what our data looks like:

![](https://33333.cdn.cke-cs.com/kSW7V9NHUXugvhoQeFaf/images/d24f0514a6f3a172482cfca7b399119015ad65707ea29ccf.png)

I'm excited to share details about an interesting data analysis project that leverages a comprehensive job postings dataset. The data, courtesy of Luke Barrouse on [Kaggle](https://www.kaggle.com/datasets/lukebarousse/data-analyst-job-postings-google-search), focuses on Data Analyst positions across the United States, with daily updates of approximately 100 new job listings since November 2022. 
<br/>
<br/>

## Dataset Overview
The dataset captures real-world job posting information pulled directly from Google search results. What makes it particularly valuable is its dual availability:

*   A static CSV file for traditional analysis
*   A [remote source](https://storage.googleapis.com/gsearch_share/gsearch_jobs.csv) for real-time data updates
<br/>

## Dashboard Implementation

I've created two interactive dashboards that offer unique perspectives on the salary data:

1.  **CSV-Based Dashboard**
    *   Uses the static dataset
    *   Implements Power Query transformations
    *   Provides foundational salary insights
2.  **Real-Time Dashboard**
    *   Connects to the remote data source
    *   Applies identical Power Query transformations
    *   Offers up-to-date market insights

Both dashboards utilize two carefully crafted queries to transform and analyze the data effectively. The transformation logic remains consistent across both implementations, ensuring reliable comparisons and insights.

The detailed skills required for each dashboard can be found in the project's [README](https://github.com/Ankush-Santra/Excel-Project/blob/main/README.md) file, making it easy for others to replicate or build upon this work.
<br/>
<br/>

## Building the Data Pipeline: A Closer Look

Let me walk you through our data transformation process, which is both efficient and elegantly simple.

#### QUERY 1

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

#### QUERY 2

```plaintext
let
    Source = #"Data Jobs Cleaned",
    #"Removed Columns" = Table.RemoveColumns(Source,{"job_title", "seniority", "location", "extensions", "title", "schedule_type"})
in
    #"Removed Columns"
```

### Query Structure

We implemented two streamlined queries to handle our data processing needs:

*   The first query performs the core transformations (code provided above)
*   The second query focuses on column optimization by removing unnecessary fields

The transformation logic is straightforward and follows standard data cleaning practices, making it easy to understand and modify if needed.

## Easy Replication Steps

Want to recreate these dashboards? Here's how:

1.  Open Power Query Editor
2.  Access the Advanced Editor
3.  Paste the provided code
4.  Watch the magic happen!

## Automated ETL Process

The real beauty lies in the second dashboard's automation. By utilizing a remote URL as the data source, we've created a true ETL (Extract, Transform, Load) pipeline that:

*   Updates with a single click on "Refresh All" under the Data Tab
*   Automatically pulls fresh data from the source
*   Applies transformations instantly
*   Loads the results into your dashboard

This automation eliminates manual updates and ensures you're always working with the latest data. It's data analysis made simple, yet powerful!
