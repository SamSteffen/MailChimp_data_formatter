# MailChimp_Data_Formatter
A step-by-step guide to building a python function that cleans and transforms multiple raw MailChimp data downloads into a single analysis-ready csv.

## Overview
MailChimp is a useful marketing tool utilized by many businesses and organizations for a variety of reasons. Depending on one's subscription tier and workplace authorizations, harvesting data from MailChimp can vary in difficulty and complexity. For those who may use MailChimp for multiple campaigns and are not authorized to access the data from campaigns save as individual csv exports, you're probably familiar with, and frustrated by, the following output format:

[pic]

While this report certainly provides some useful metrics, it's of little use to us if our intention is to compile this data with more like it for further, long-term analysis.

This repo is intended to provide a step-by-step guide explaining how to build a python function that will allow you to automate the otherwise painstaking process of manually reformatting multiple datasets like the one shown above, into a single, compiled, user-friendly format, like this:

[pic_final]

## Directory Set-Up
This function utilizes the following file structure:

> directory
    > input
    > output

To begin, download as many csvs as you'd like from your MailChimp email campaigns and store them in the input directory. The output directory should remain empty.

## The Code
This python function was written using a jupyter notebook.

### Step 1: Import Dependencies
```python
import pandas as pd
import os
import numpy as np
from datetime import datetime as dt
import matplotlib
import csv
```
### Step 2: Establish Your Filepath
```python
#'..\\directory\\input\\'
data_path_root = config.py
```

### Step 3: Put all the .csv files in the directory in a list
```python
raw_csvs = os.listdir(data_path_root)
raw_csvs
```

### Step 4: Create a variable that concatenates the full file location with the csv name; call the first csv for cleaning
```python
data = str(data_path_root + raw_csvs[0])
```

### Step 5: Read in the data as a df
```python
df = pd.read_csv(data, sep=";", skip_blank_lines=False)

# view the data
df
```
[pic_df]


### STOPPED HERE.
```python
#STEP 8: While the urls are still data in a single row, make them all shorter
url_data = list(df['Email Campaign Report'])
url_data = url_data[22:]

for urls in url_data:
    df = df.replace(urls, urls.split('?')[0])

df
```

```python
#STEP 9: Transform the Data: split the data into multiple columns by the delimiter ","
split_df = df['Email Campaign Report'].str.split(',', expand=True)
split_df
```


```python
#STEP 10: Transpose the column data into headers and rows
trans_df = split_df.transpose()
trans_df
```


```python
#STEP 11: Make row 0 into a header column
new_header = trans_df.iloc[0]
```


```python
#STEP 12: Make the df the df minus the first row of data
trans_df = trans_df[1:]

#STEP 13: Set header row as the df header
trans_df.columns = new_header

trans_df
```


```python
#STEP 14: Merge the data in the Delivery Date/Time column into a single cell
date_time_list = trans_df['Delivery Date/Time:']
date_time_list = date_time_list.tolist()
date_time = str(date_time_list[0] + "," + date_time_list[1] + date_time_list[2])
trans_df.insert(2, 'Delivery Datetime', date_time)
trans_df
```


```python
#STEP 15: Check rows 2 and 3 for other useful data
# list(trans_df.loc[[1][0]])
# list(trans_df.loc[[2][0]])
# list(trans_df.loc[[3][0]])
```


```python
#STEP 16: Drop rows 2-3
data_row = trans_df.drop([2,3])
data_row
```


```python
#STEP 17: Call only the desired columns in the desired order
# list(data_row)

data_row = data_row[['Title:',
                     'Subject Line:',
                     'Delivery Datetime',
                     'Total Recipients:',
                     'Successful Deliveries:',
                     'Bounces:',
                     'Times Forwarded:',
                     'Forwarded Opens:',
                     'Recipients Who Opened:',
                     'Total Opens:',
                     'Recipients Who Clicked:',
                     'Total Clicks:',
                     'Total Unsubs:',
                     'Times Liked on Facebook:'
                    ]]

data_row
```


```python
#STEP 18: Remove the "" from the data
data_row = data_row.applymap(lambda x: x.replace('"', ''))
data_row
```


```python
#STEP 19: Separate columns with 2 datasets into seperate columns and remove parenthesis from data
data_row['Bounce Count:'] = data_row['Bounces:'].str.split(" ", expand=True)[0]
data_row['Bounces (%):'] = data_row['Bounces:'].str.split(" ", expand=True)[1]
data_row['Bounces (%):'] = data_row['Bounces (%):'].str.replace("(", "").str.replace(")", "")

data_row['Open Count:'] = data_row['Recipients Who Opened:'].str.split(" ", expand=True)[0]
try:
    data_row['Opens (%):'] = data_row['Recipients Who Opened:'].str.split(" ", expand=True)[1]
except KeyError:
    data_row['Opens (%):'] = data_row['Recipients Who Opened:']
    
try:
    data_row['Opens (%):'] = data_row['Opens (%):'].str.replace("(","").str.replace(")","")
except KeyError:
    data_row['Opens (%):'] = data_row['Opens (%):']
    
data_row['Clicks Count:'] = data_row['Recipients Who Clicked:'].str.split(" ", expand=True)[0]
data_row['Clicks (%):'] = data_row['Recipients Who Clicked:'].str.split(" ", expand=True)[1]
data_row['Clicks (%):'] = data_row['Clicks (%):'].str.replace("(","").str.replace(")","")

#STEP 20: Keep the split columns and remove the origin columns to avoid unncessary duplication
data_row = data_row.drop(columns=['Bounces:', 'Recipients Who Opened:', 'Recipients Who Clicked:'])

data_row
```


```python
# STEP 20: Add a 'Series' column for 'Series' data
data_row['Series:'] = data_row['Title:'].str.split("-", expand=True)[2]
data_row['Series:'] = data_row['Series:'].str.split(" ", expand=True)[1]

data_row
```


```python
#STEP 21: Add a 'Type' column for email Type
data_row['Type:'] = data_row['Title:'].str.split("-", expand=True)[2]

try:
    data_row['Type:'] = data_row['Type:'].str.split(" ", expand=True)[2]  
except KeyError:
    email_type = "Daily"
    data_row['Type'] = email_type

data_row
```


```python
# STEP 22: Convert the datetime columns into datetime datatypes
data_row['Delivery Datetime'] = pd.to_datetime(data_row['Delivery Datetime'])

data_row
```


```python
#STEP 23: Split the 'Delivery Datetime' column into 4 columns and convert the datatypes
data_row['Send Date:'] = data_row['Delivery Datetime'].dt.date
data_row['Send Time (PT):'] = data_row['Delivery Datetime'].dt.time
data_row['Send Weekday:'] = data_row['Delivery Datetime'].dt.strftime('%A')

# STEP 24: Drop the Delivery Datetime column from the data_row
data_row.drop(columns=['Delivery Datetime'])

data_row
```


```python
# STEP 25: Write a conditional that will transform whatever series is printed to the appropriate code
series_codes = ['BH in PC', 'OPSUD', 'CTSUDs', 'XWT', 'PBH', 'PSUD', 'COVID', 'PALTC', 'VHLC', 'Syphilis', 'PASD', 'MOUD', 'General']

for value in data_row['Series:']:
    if value == 'BH':
        data_row.loc[data_row['Series:'] == 'BH', 'Series:'] = series_codes[0]
    elif value == 'COVID Long Term':
        data_row.loc[data_row['Series:'] == 'COVID Long Term', 'Series:'] = series_codes[7]
    elif value == 'COVID-19':
        data_row.loc[data_row['Series:'] == 'COVID-19', 'Series:'] = series_codes[6]
    elif value == 'CT-SUD':
        data_row.loc[data_row['Series:'] == 'CT-SUD', 'Series:'] = series_codes[2]
    elif value == 'CTSUDs':
        data_row.loc[data_row['Series:'] == 'CTSUDs', 'Series:'] = series_codes[2]
    elif value == 'HCV':
        data_row.loc[data_row['Series:'] == 'HCV', 'Series:'] = series_codes[8]
    elif value == 'Hep-C':
        data_row.loc[data_row['Series:'] == 'Hep-C', 'Series:'] = series_codes[8]
    elif value == 'MOUD Consultation Hours':
        data_row.loc[data_row['Series:'] == 'MOUD Consultation Hours', 'Series:'] = series_codes[11]
    elif value == 'MOUD Office Hours (Solo)':
        data_row.loc[data_row['Series:'] == 'MOUD Office Hours (Solo)', 'Series:'] = series_codes[11]
    elif value == 'OP':
        data_row.loc[data_row['Series:'] == 'OP', 'Series:'] = series_codes[1]
    elif value == 'PBH':
        data_row.loc[data_row['Series:'] == 'PBH', 'Series:'] = series_codes[4]
    elif value == 'Pediatric Autism':
        data_row.loc[data_row['Series:'] == 'Pediatric Autism', 'Series:'] = series_codes[10]
    elif value == 'PedsAutism':
        data_row.loc[data_row['Series:'] == 'PedsAutism', 'Series:'] = series_codes[10]
    elif value == 'Podcast':
        data_row.loc[data_row['Series:'] == 'Podcast', 'Series:'] = series_codes[10]
    elif value == 'VH&LC':
        data_row.loc[data_row['Series:'] == 'VH&LC', 'Series:'] = series_codes[8]
    elif value == 'VHLC':
        data_row.loc[data_row['Series:'] == 'VHLC', 'Series:'] = series_codes[8]
    elif value == 'Viral Hep':
        data_row.loc[data_row['Series:'] == 'Viral Hep', 'Series:'] = series_codes[8]
    elif value == 'X-Waiver Training':
        data_row.loc[data_row['Series:'] == 'X-Waiver Training', 'Series:'] = series_codes[3]
    else:
        data_row.loc[data_row['Series:'] == value, 'Series:'] = series_codes[12]

# data_row

# For reference the codes to use for each series are as follows:
# BH in PC for 'Behavioral Health in Primary Care'
# OPSUD for 'Opioids, Pain and Substance Use Disorders'
# CTSUDs for 'Counseling Techniques for Substance Use Disorders'
# XWT for 'X-Waiver Trainings'
# PBH for 'Pediatric Behavioral Health'
# PSUD for 'Perinatal Substance Use Disorder'
# COVID for 'COVID-19 or Ambulatory/Acute Care COVID or COVID-19 and Other Current Infections'
# PALTC for 'Nursing Home COVID or COVID-19 Safety for Post Acute and Long Term Care'
# VHLC for 'Hepatitis C or Viral Hepatitis and Liver Care'
# Syphilis for 'Syphilis'
# PASD for 'Pediatric Autism'
# MOUD for 'Medications for Opioid Use Disorders Consultation/Office Hours'
# General for 'Other'
```


```python
# STEP 26: Write a conditional that will transform whatever email type is printed to the appropriate code
type_codes = ['Daily','Weekly','Newsletter','XWT Promo','Special']

for value in data_row['Type:']:
    if value == 'Day':
        data_row.loc[data_row['Type:'] == 'Day', 'Type:'] = type_codes[0]
    if value == 'Weekly':
        data_row.loc[data_row['Type:'] == 'Weekly', 'Type:'] = type_codes[1]
    if value == 'Newsletter':
        data_row.loc[data_row['Type:'] == 'Newsletter', 'Type:'] = type_codes[2]
    if value == 'XWT Promo':
        data_row.loc[data_row['Type:'] == 'XWT Promo', 'Type:'] = type_codes[3]
    else:
        data_row.loc[data_row['Type:'] == value, 'Type:'] = type_codes[4]
    
data_row

#For reference the codes to use for each email 'type' are as follows
# Daily for 'Day of'
# Weekly for 'Weekly'
# Newsletter for 'Newsletter'
# XWT Promo for 'XWT'
# Special for 'Other'
```


```python
data_row.columns
```


```python
#STEP 27: Reorder the columns to their desired export arrangement
# data_row.columns
data_row = data_row[[
    'Send Date:',
    'Send Weekday:',
    'Send Time (PT):',
    'Series:',
    'Type:',
    'Title:',
    'Subject Line:',
    'Total Recipients:',
    'Successful Deliveries:',
    'Bounce Count:', 
    'Times Forwarded:',
    'Forwarded Opens:', 
    'Open Count:',
    'Opens (%):',
    'Total Opens:', 
    'Clicks Count:',
    'Clicks (%):',
    'Total Clicks:',
    'Total Unsubs:',
    'Times Liked on Facebook:',
    'Bounces (%):'
]]

data_row
```


```python
#STEP 28: Retitle the columns to desired export
data_row = data_row.rename(columns={    
    'Send Date:': 'Date',
    'Send Weekday:': 'Weekday',
    'Send Time (PT):': 'Time',
    'Series:': 'Series',
    'Type:': 'Type',
    'Title:': 'Title',
    'Subject Line:' : 'Subject',
    'Total Recipients:': 'Total Recipients',
    'Successful Deliveries:': 'Successful Deliveries',
    'Bounce Count:': 'Total Bounces', 
    'Times Forwarded:': 'Times Forwarded',
    'Forwarded Opens:': 'Forwarded Opens', 
    'Open Count:': 'Unique Opens',
    'Opens (%):': 'Open Rate',
    'Total Opens:': 'Total Opens', 
    'Clicks Count:': 'Unique Clicks',
    'Clicks (%):': 'Click Rate',
    'Total Clicks:': 'Total Clicks',
    'Total Unsubs:': 'Unsubscribes',
    'Times Liked on Facebook:': 'FB Likes',
    'Bounces (%):': 'Bounce Rate'
})

data_row
```


```python
data_row.dtypes
```


```python
#STEP 29: Convert the datatypes
data_row['Date'] = pd.to_datetime(data_row['Date'], format='%Y-%m-%d')
data_row['Total Recipients'] = pd.to_numeric(data_row['Total Recipients'])
data_row['Successful Deliveries'] = pd.to_numeric(data_row['Successful Deliveries'])
data_row['Total Bounces'] = pd.to_numeric(data_row['Total Bounces'])
data_row['Times Forwarded'] = pd.to_numeric(data_row['Times Forwarded'])
data_row['Forwarded Opens'] = pd.to_numeric(data_row['Forwarded Opens'])

# data_row.dtypes
```


```python
# STEP 30: Create a function out of steps 1-29 that takes a folder full of .csvs for a parameter

def clean_csvs(csvs_data_file):
    csvs_data_path_root = 'C:\\Users\\ssteffen\\University of Idaho\\Storage-Boise - ECHO\\Staff\\Sam\\Data\\Spreadsheets\\raw_MailChimp\\input\\'
    raw_csvs = os.listdir(data_path_root)
    dataset_count = len(raw_csvs)
    
    print(f'This program will clean {dataset_count} csv files and prepare them for analysis.')
    
    cleaned_data_list = []
    
    #create a for loop to loop through and clean all the downloaded csv files
    i = 0

    for csvs in raw_csvs:
        data = str(data_path_root + raw_csvs[i])
        df = pd.read_csv(data, sep=";", skip_blank_lines=False)

        # create a df and clean the urls
        url_data = list(df['Email Campaign Report'])
        url_data = url_data[22:]

        for urls in url_data:
            df = df.replace(urls, urls.split('?')[0])

        # split the data into columns
        split_df = df['Email Campaign Report'].str.split(',', expand=True)

        # transpose the data
        trans_df = split_df.transpose()

        # make row[0] into a header
        new_header = trans_df.iloc[0]

        # Make the df the df minus the first row of data
        trans_df = trans_df[1:]

        # Set header row as the df header
        trans_df.columns = new_header

        # Merge the data in the Delivery Date/Time column into a single cell
        date_time_list = trans_df['Delivery Date/Time:']
        date_time_list = date_time_list.tolist()
        date_time = str(date_time_list[0] + "," + date_time_list[1] + date_time_list[2])
        trans_df.insert(2, 'Delivery Datetime', date_time)

        # Drop rows 2-3
        data_row = trans_df.drop([2,3])

        # create a new df of desired columns    
        data_row = data_row[['Title:',
                     'Subject Line:',
                     'Delivery Datetime',
                     'Total Recipients:',
                     'Successful Deliveries:',
                     'Bounces:',
                     'Times Forwarded:',
                     'Forwarded Opens:',
                     'Recipients Who Opened:',
                     'Total Opens:',
                     'Recipients Who Clicked:',
                     'Total Clicks:',
                     'Total Unsubs:',
                     'Times Liked on Facebook:'
                    ]]

        # remove "" from the data
        data_row = data_row.applymap(lambda x: x.replace('"', ''))

        # separate values into separate columns, remove parentheses
        data_row['Bounce Count:'] = data_row['Bounces:'].str.split(" ", expand=True)[0]
        data_row['Bounces (%):'] = data_row['Bounces:'].str.split(" ", expand=True)[1]
        data_row['Bounces (%):'] = data_row['Bounces (%):'].str.replace("(", "").str.replace(")", "")
        
        data_row['Open Count:'] = data_row['Recipients Who Opened:'].str.split(" ", expand=True)[0]
        try:
            data_row['Opens (%):'] = data_row['Recipients Who Opened:'].str.split(" ", expand=True)[1]
        except KeyError:
            data_row['Opens (%):'] = data_row['Recipients Who Opened:']
    
        try:
            data_row['Opens (%):'] = data_row['Opens (%):'].str.replace("(","").str.replace(")","")
        except KeyError:
            data_row['Opens (%):'] = data_row['Opens (%):']
        
        data_row['Clicks Count:'] = data_row['Recipients Who Clicked:'].str.split(" ", expand=True)[0]
        data_row['Clicks (%):'] = data_row['Recipients Who Clicked:'].str.split(" ", expand=True)[1]
        data_row['Clicks (%):'] = data_row['Clicks (%):'].str.replace("(","").str.replace(")","")

        # Keep the split columns and remove the origin columns to avoid unncessary duplication
        data_row = data_row.drop(columns=['Bounces:', 'Recipients Who Opened:', 'Recipients Who Clicked:'])

        # Add 'Series' column for series data
        data_row['Series:'] = data_row['Title:'].str.split("-", expand=True)[2]
        data_row['Series:'] = data_row['Series:'].str.split(" ", expand=True)[1]

        # Add a 'Type' column for email Type
        data_row['Type:'] = data_row['Title:'].str.split("-", expand=True)[2]

        try:
            data_row['Type:'] = data_row['Type:'].str.split(" ", expand=True)[2]  

        except KeyError:
            email_type = "Daily"
            data_row['Type'] = email_type

        # convert datetime columns into appropriate datatype
        data_row['Delivery Datetime'] = pd.to_datetime(data_row['Delivery Datetime'])

        # split the datetime data into seperate columns 
        data_row['Send Date:'] = data_row['Delivery Datetime'].dt.date
        data_row['Send Time (PT):'] = data_row['Delivery Datetime'].dt.time
        data_row['Send Weekday:'] = data_row['Delivery Datetime'].dt.strftime('%A')

        # Drop the Delivery Datetime column from the data_row
        data_row.drop(columns=['Delivery Datetime'])

        series_codes = ['BH in PC', 'OPSUD', 'CTSUDs', 'XWT', 'PBH', 'PSUD', 'COVID', 'PALTC', 'VHLC', 'Syphilis', 'PASD', 'MOUD', 'General']

        # codify the series
        for value in data_row['Series:']:
            if value == 'BH':
                data_row.loc[data_row['Series:'] == 'BH', 'Series:'] = series_codes[0]
            elif value == 'COVID Long Term':
                data_row.loc[data_row['Series:'] == 'COVID Long Term', 'Series:'] = series_codes[7]
            elif value == 'COVID-19':
                data_row.loc[data_row['Series:'] == 'COVID-19', 'Series:'] = series_codes[6]
            elif value == 'CT-SUD':
                data_row.loc[data_row['Series:'] == 'CT-SUD', 'Series:'] = series_codes[2]
            elif value == 'CTSUDs':
                data_row.loc[data_row['Series:'] == 'CTSUDs', 'Series:'] = series_codes[2]
            elif value == 'HCV':
                data_row.loc[data_row['Series:'] == 'HCV', 'Series:'] = series_codes[8]
            elif value == 'Hep-C':
                data_row.loc[data_row['Series:'] == 'Hep-C', 'Series:'] = series_codes[8]
            elif value == 'MOUD Consultation Hours':
                data_row.loc[data_row['Series:'] == 'MOUD Consultation Hours', 'Series:'] = series_codes[11]
            elif value == 'MOUD Office Hours (Solo)':
                data_row.loc[data_row['Series:'] == 'MOUD Office Hours (Solo)', 'Series:'] = series_codes[11]
            elif value == 'OP':
                data_row.loc[data_row['Series:'] == 'OP', 'Series:'] = series_codes[1]
            elif value == 'PBH':
                data_row.loc[data_row['Series:'] == 'PBH', 'Series:'] = series_codes[4]
            elif value == 'Pediatric Autism':
                data_row.loc[data_row['Series:'] == 'Pediatric Autism', 'Series:'] = series_codes[10]
            elif value == 'PedsAutism':
                data_row.loc[data_row['Series:'] == 'PedsAutism', 'Series:'] = series_codes[10]
            elif value == 'Podcast':
                data_row.loc[data_row['Series:'] == 'Podcast', 'Series:'] = series_codes[10]
            elif value == 'VH&LC':
                data_row.loc[data_row['Series:'] == 'VH&LC', 'Series:'] = series_codes[8]
            elif value == 'VHLC':
                data_row.loc[data_row['Series:'] == 'VHLC', 'Series:'] = series_codes[8]
            elif value == 'Viral Hep':
                data_row.loc[data_row['Series:'] == 'Viral Hep', 'Series:'] = series_codes[8]
            elif value == 'X-Waiver Training':
                data_row.loc[data_row['Series:'] == 'X-Waiver Training', 'Series:'] = series_codes[3]
            else:
                data_row.loc[data_row['Series:'] == value, 'Series:'] = series_codes[12]

        # codify the email types
        type_codes = ['Daily','Weekly','Newsletter','XWT Promo','Special']

        for value in data_row['Type:']:
            if value == 'Day':
                data_row.loc[data_row['Type:'] == 'Day', 'Type:'] = type_codes[0]
            if value == 'Weekly':
                data_row.loc[data_row['Type:'] == 'Weekly', 'Type:'] = type_codes[1]
            if value == 'Newsletter':
                data_row.loc[data_row['Type:'] == 'Newsletter', 'Type:'] = type_codes[2]
            if value == 'XWT Promo':
                data_row.loc[data_row['Type:'] == 'XWT Promo', 'Type:'] = type_codes[3]
            else:
                data_row.loc[data_row['Type:'] == value, 'Type:'] = type_codes[4]

        # reorder the columns
        data_row = data_row[[
            'Send Date:',
            'Send Weekday:',
            'Send Time (PT):',
            'Series:',
            'Type:',
            'Title:',
            'Subject Line:',
            'Total Recipients:',
            'Successful Deliveries:',
            'Bounce Count:', 
            'Times Forwarded:',
            'Forwarded Opens:', 
            'Open Count:',
            'Opens (%):',
            'Total Opens:', 
            'Clicks Count:',
            'Clicks (%):',
            'Total Clicks:',
            'Total Unsubs:',
            'Times Liked on Facebook:',
            'Bounces (%):'
        ]]

        # retitle the columns
        data_row = data_row.rename(columns={    
            'Send Date:': 'Date',
            'Send Weekday:': 'Weekday',
            'Send Time (PT):': 'Time',
            'Series:': 'Series',
            'Type:': 'Type',
            'Title:': 'Title',
            'Subject Line:' : 'Subject',
            'Total Recipients:': 'Total Recipients',
            'Successful Deliveries:': 'Successful Deliveries',
            'Bounce Count:': 'Total Bounces', 
            'Times Forwarded:': 'Times Forwarded',
            'Forwarded Opens:': 'Forwarded Opens', 
            'Open Count:': 'Unique Opens',
            'Opens (%):': 'Open Rate',
            'Total Opens:': 'Total Opens', 
            'Clicks Count:': 'Unique Clicks',
            'Clicks (%):': 'Click Rate',
            'Total Clicks:': 'Total Clicks',
            'Total Unsubs:': 'Unsubscribes',
            'Times Liked on Facebook:': 'FB Likes',
            'Bounces (%):': 'Bounce Rate'
        })

        # convert the data types
        data_row['Date'] = pd.to_datetime(data_row['Date'], format='%Y-%m-%d')
        data_row['Total Recipients'] = pd.to_numeric(data_row['Total Recipients'])
        data_row['Successful Deliveries'] = pd.to_numeric(data_row['Successful Deliveries'])
        data_row['Total Bounces'] = pd.to_numeric(data_row['Total Bounces'])
        data_row['Times Forwarded'] = pd.to_numeric(data_row['Times Forwarded'])
        data_row['Forwarded Opens'] = pd.to_numeric(data_row['Forwarded Opens'])    

        #add the cleaned df to the list
        cleaned_data_list.append(data_row)

        print(f"Dataset {i} cleaned and added to list.")

        i += 1
            
    #add all the cleaned dfs to a single df
    cleaned_data = pd.concat(cleaned_data_list)
                
    #export the df to a .csv file
    return cleaned_data.to_csv('C:\\Users\\ssteffen\\University of Idaho\\Storage-Boise - ECHO\\Staff\\Sam\\Data\\Projects\\MailChimp_formatter\\raw_MailChimp\\output\\Mail_Chimp_data_cleaned.csv')
```


```python
#STEP 31: Call the function
csvs_data_file = 'C:\\Users\\ssteffen\\University of Idaho\\Storage-Boise - ECHO\\Staff\\Sam\\Data\\Projects\\MailChimp_formatter\\raw_MailChimp\\input'

clean_csvs(csvs_data_file)
```

    This program will clean 52 csv files and prepare them for analysis.
    Dataset 0 cleaned and added to list.
    Dataset 1 cleaned and added to list.
    Dataset 2 cleaned and added to list.
    Dataset 3 cleaned and added to list.
    Dataset 4 cleaned and added to list.
    Dataset 5 cleaned and added to list.
    Dataset 6 cleaned and added to list.
    Dataset 7 cleaned and added to list.
    Dataset 8 cleaned and added to list.
    Dataset 9 cleaned and added to list.
    Dataset 10 cleaned and added to list.
    Dataset 11 cleaned and added to list.
    Dataset 12 cleaned and added to list.
    Dataset 13 cleaned and added to list.
    Dataset 14 cleaned and added to list.
    Dataset 15 cleaned and added to list.
    Dataset 16 cleaned and added to list.
    Dataset 17 cleaned and added to list.
    Dataset 18 cleaned and added to list.
    Dataset 19 cleaned and added to list.
    Dataset 20 cleaned and added to list.
    Dataset 21 cleaned and added to list.
    Dataset 22 cleaned and added to list.
    Dataset 23 cleaned and added to list.
    Dataset 24 cleaned and added to list.
    Dataset 25 cleaned and added to list.
    Dataset 26 cleaned and added to list.
    Dataset 27 cleaned and added to list.
    Dataset 28 cleaned and added to list.
    Dataset 29 cleaned and added to list.
    Dataset 30 cleaned and added to list.
    Dataset 31 cleaned and added to list.
    Dataset 32 cleaned and added to list.
    Dataset 33 cleaned and added to list.
    Dataset 34 cleaned and added to list.
    Dataset 35 cleaned and added to list.
    Dataset 36 cleaned and added to list.
    Dataset 37 cleaned and added to list.
    Dataset 38 cleaned and added to list.
    Dataset 39 cleaned and added to list.
    Dataset 40 cleaned and added to list.
    Dataset 41 cleaned and added to list.
    Dataset 42 cleaned and added to list.
    Dataset 43 cleaned and added to list.
    Dataset 44 cleaned and added to list.
    Dataset 45 cleaned and added to list.
    Dataset 46 cleaned and added to list.
    Dataset 47 cleaned and added to list.
    Dataset 48 cleaned and added to list.
    Dataset 49 cleaned and added to list.
    Dataset 50 cleaned and added to list.
    Dataset 51 cleaned and added to list.
    


```python
# LAST STEP: Delete all the downloaded raw .csv files from the raw_MailChimp/input folder.
```









