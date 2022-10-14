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

[pic]

To begin, download as many csvs as you'd like from your MailChimp email campaigns and store them in the input directory. Once finished, the input folder should resemble the following:

[csvs_pic]

The output directory should remain empty.

## The Code
This python function was written piecemeal using a jupyter notebook to test the steps before being compiled into the function.

### Step 1: Import Dependencies
```python
import pandas as pd
import os
import numpy as np
from datetime import datetime as dt
import matplotlib
import csv
```
### Step 2: Create the ```clean_csvs()``` function that will take the contents of the input directory for its only parameter.

```python
def clean_csvs(csvs_data_file):
   
    # define the file path to the input folder where the raw csvs are stored
    csvs_data_path_root = '..\\directory\\input\\'
    raw_csvs = os.listdir(csvs_data_path_root)

    #create an output print statement for the program
    dataset_count = len(raw_csvs)
   
    print(f'This program will clean {dataset_count} csv files and prepare them for analysis.')
   
    #create an empty list to store cleaned csvs as data rows
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
        # add try and except blocks to account for potential errors in other csvs
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

        # codify the series data to appropriate abbreviations
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

        # codify the email campaign types
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

        # add an output print statement to demonstrate program's progress
        print(f"Dataset {i} cleaned and added to list.")

        #increase iterator by 1
        i += 1
           
    #add all the cleaned dfs to a single df
    cleaned_data = pd.concat(cleaned_data_list)
               
    #export the df to a .csv file
    return cleaned_data.to_csv('..directory\\output\\Mail_Chimp_data_cleaned.csv')
```

### Step 3: Call the function
```python
#STEP 31: Call the function
csvs_data_file = '..directory\\input'

clean_csvs(csvs_data_file)
```
The output of this will yield a csv file called 'Mail_Chimp_data_cleaned.csv' and should resemble the following:

[compiled_data_pic]
