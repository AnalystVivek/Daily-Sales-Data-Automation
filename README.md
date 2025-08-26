# 📊 ETL Sales Data Analytics Project 🚀

Welcome to my Sales Data ETL and Analytics Pipeline!  
Dive in to discover how I built an automated, industry-grade solution for cleaning, analyzing, and visualizing sales trends with complete integration to Google Drive and automated email reporting! 🌟

---

## 🗂 Project Overview

This project is an end-to-end ETL (Extract, Transform, Load) pipeline for sales data analytics. It automates data extraction from a MySQL database, applies robust cleaning and outlier detection, transforms sales data, generates insightful sales trend charts for each product, and delivers daily reports via email and Google Drive. The pipeline exemplifies best practices for reproducible, scalable, and secure data analytics workflows, making it an ideal showcase of technical proficiency and business impact.

---

## 🛠️ Libraries Used & Their Purpose

Here's a concise list of the libraries I used and why:

```ruby
import pandas as pd                                    # Data loading, cleaning, manipulation
import numpy as np                                     # Efficient array operations, data cleaning
import mysql.connector                                 # MySQL DB connectivity and data extraction
from pydrive.auth import GoogleAuth                    # Google Drive authentication
from pydrive.drive import GoogleDrive                  # Uploading files to cloud
import logging                                         # Structured logging of pipeline steps
import time                                            # Timestamp and process scheduling
import schedule                                        # Automated daily process scheduling
import os                                              # File operations, system interactions
import matplotlib.pyplot as plt                        # Chart & visualization creation
import seaborn as sns                                  # Enhanced data visualization (not used in final chart)
import smtplib, from email.message import EmailMessage # Automated email reporting
```

**Each library was hand-picked to build a robust, automated, and production-ready pipeline.**

---

## 👨‍💻 Function Breakdown & Code Walkthrough

Below I've showcased my coding approach for each function, paired with annotated code snippets:

### 1. **Data Extraction**

```ruby
def extract_data():                                                       
    
    logger.info("Starting data extraction from MySQL database.....")

    try:
        conn = mysql.connector.connect(**mysql_config)               # Connect to MySQL database
        df = pd.read_sql("select * from aug_sales;", conn)            # To save data from the table into a DataFrame
        conn.close()                                                   # Close the connection
        logger.info(f"Data extraction completed. Extracted {len(df)} records.")
        return df
    
    except Exception as e:
        logger.error("Failed fetch data from database.", exc_info = e)
        return None
```
**What & Why:**  
I connected securely to MySQL, precisely fetched all sales records, and closed connections rapidly to avoid leaks. Efficient extraction is the backbone for subsequent transformations.

---

### 2. 🧹**Data Cleaning & Transformation**

```ruby
def clean_transform_data(df):

# Checking if the DataFrame is empty

    if df is None or df.empty:
        logger.warning("There is no data to transform.")
        return None

    logger.info("Starting data cleaning and transformation .....")

    try:
        df['date'] = pd.to_datetime(df['date'], format='%d-%m-%Y', errors = 'coerce') # Convert 'date' column to datetime format

        before = len(df)  # Record count before cleaning
        df = df.drop_duplicates()  # Remove duplicate records
        df = df.dropna(subset= ['date', 'product_name', 'unit_price', 'quantity'])  # Remove records with nulls in critical columns
        after = len(df)  # Record count after cleaning

        logger.info(f"After removing duplicates and nulls, {before - after} records were removed. {after} records remain.")

        df['discount'] = df['discount'].fillna(0)  # Fill missing discount values with 0
        
        df['unit_price'] = df['unit_price'].astype(float)  # Ensure unit_price is float

        df = df.sort_values(by='date', ascending=True)  # Sort data by date

        mean_unit_price = df['unit_price'].mean()  # Calculate mean unit price
        std_unit_price = df['unit_price'].std()    # Calculate std deviation of unit price
        is_inlier = df['unit_price'].between(mean_unit_price - 3 * std_unit_price, mean_unit_price + 3 * std_unit_price)
        is_outlier = (~is_inlier).sum() # Count outliers
        
        df = df[is_inlier]

        logger.info(f"Removed {is_outlier} outlier records based on unit_price.")  

        df['quantity'] = df['quantity'].astype(int)  # Ensure quantity is integer
        df['discount'] = df['discount'].astype(int)  # Ensure discount is integer

        df['total_sales'] = df['unit_price'] * df['quantity'] * (1 - df['discount'] / 100)  # Calculate total sales

        logger.info("Data cleaning and transformation completed Sucessfully.")

        return df

    except Exception as e:
        logger.error("Data transformation failed.", exc_info = e)
        return None
```

**What & Why:**  
I applied advanced data cleaning by parsing dates, removing duplicates, filling in missing discounts, and filtering critical nulls, then performed statistical outlier removal to ensure only valid data is analyzed. Calculating total sales per record integrates business logic for actionable analytics.

---

### 3. **Saving Cleaned Data**

```ruby
def save_to_csv(df, filename='cleaned_data.csv'):

    try:
        df.to_csv(filename, index=False)                                  # Save DataFrame to CSV
        logger.info(f"Saved cleaned data to {filename} successfully.")
    except Exception as e:
        logger.error(f"Failed to save data {filename} .", exc_info = e)
```

**What & Why:**  
Saved the processed DataFrame to a clean CSV file for transparency and reproducibility, facilitating sharing and downstream analytics.

---

### 4. **Google Drive Upload Integration**

```ruby
def upload_to_google_drive(filename='cleaned_data.csv', folder_id='1wU8s54nSqp9WDJUeBn-uzGws48DQexVJ'):
    logger.info("Starting upload to Google Drive.....")

    try:
        gauth = GoogleAuth()                   # Create GoogleAuth object
        gauth.LocalWebserverAuth()             # Authenticate using local webserver
        drive = GoogleDrive(gauth)             # Create GoogleDrive object

        # We need to create a file in Google Drive

        drive_file = drive.CreateFile({
            'title': filename,
            'parents': [{'id': folder_id}]  # To save in a specific folder
        })
        drive_file.SetContentFile(filename)    # Set the content of the file from local file
        drive_file.Upload()                     # Upload the file to Google Drive
        logger.info(f"Uploaded {filename} to Google Drive successfully.")

    except Exception as e:
        logger.error("Failed to upload file to Google Drive.", exc_info = e)
```

**What & Why:**  
Automated secure file uploads directly to Google Drive ensuring safe storage and enabling collaboration without manual intervention.

---

### 5. **Sales Trend Chart Generation**

```ruby
def create_sales_trend_chart(df, chart_filename='sales_trend.png'):
    logger.info("Starting sales trend chart creation.....")
    plt.figure(figsize=(15, 10))
    products = df['product_name'].unique()
    n = len(products)
    fig, axs = plt.subplots(n, 1, figsize=(15, 5 * n), sharex=True)  # Share x-axis for all subplots
    if n == 1:
        axs = [axs]  # Ensuring axs is iterable when there's only one product

    for i, product in enumerate(products):
        df_product = df[df['product_name'] == product].copy()
        df_product = df_product.sort_values('date')
        df_product.set_index('date', inplace=True)

        # Resample total_sales weekly and sum
        weekly_sales = df_product['total_sales'].resample('W').sum()

        ax = axs[i]

        # Plot weekly sales trend line
        ax.plot(weekly_sales.index, weekly_sales.values, color='blue', linewidth=1, label='Total Sales')

        ax.set_title(f'Sales Trend for {product}')
        ax.set_xlabel('Date')
        ax.set_ylabel('Total Sales')
        ax.legend()
        ax.grid(True)

        # Format x-axis to show week start date or week number
        ax.xaxis.set_major_formatter(plt.matplotlib.dates.DateFormatter('%d-%b'))

    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.savefig(chart_filename)
    plt.close()
    logger.info(f"Sales trend chart saved as {chart_filename} successfully.")
```

**What & Why:**  
Developed clear, multi-faceted charts for each product to visually compare sales patterns and spot outliers and trends, vital for real-world decision making.

---

### 6. **Automated Email Reporting**

```ruby
def send_email(subject, body, to_emails, attachments):
    logger.info("Starting email sending process.....")

    SMTP_SERVER = 'smtp.gmail.com'
    SMTP_PORT = 587
    EMAIL_ADDRESS = '***********@gmail.com'
    EMAIL_PASSWORD = '*************'

    msg = EmailMessage()
    msg['Subject'] = subject
    msg['From'] = EMAIL_ADDRESS
    msg['To'] = ', '.join(to_emails)
    msg.set_content(body)

    for filepath in attachments:
        with open(filepath, 'rb') as f:
                file_data = f.read()
                file_name = os.path.basename(filepath)
        msg.add_attachment(file_data, maintype='application', subtype='octet-stream', filename=file_name)

    try:
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()  # Upgrade the connection to a secure encrypted SSL/TLS connection
            server.login(EMAIL_ADDRESS, EMAIL_PASSWORD)
            server.send_message(msg)
        logger.info("Email sent successfully.")

    except Exception as e:
        logger.error("Failed to send email.", exc_info = e)
```
**What & Why:**  
Automated outbound email with multi-file attachment, sending daily sales reports and trend charts directly to stakeholders—ensuring they stay informed and empowered.

---

### 7. **Main ETL Pipeline**

```ruby
def etl_process():
    logger.info("ETL process started.")

    df = extract_data()                      # Extract data from MySQL
    cleaned_df = clean_transform_data(df)   # Clean and transform data

    if cleaned_df is not None:
        save_to_csv(cleaned_df)              # Save cleaned data to CSV
        create_sales_trend_chart(cleaned_df)  # Creating sales trend chart
        upload_to_google_drive()              # Upload CSV to Google Drive

        # Send email with the chart attached
        receivers = ['example1@gmail.com', 'example2@gmail.com']
        send_email(
            subject="Daily Sales Report & Sales trend Chart",
            body="Please find attached Sales data with Sales trend chart.",
            to_emails=receivers,
            attachments=['cleaned_data.csv', 'sales_trend.png']
        )
        logger.info("ETL process and Email sending completed.")

    else:
        logger.warning("ETL process aborted due to no data after cleaning.")
```
**What & Why:**  
Orchestrates all critical steps—ensuring smooth, automated, scalable data pipelines for daily analytics and reporting.

---

### 8. **Automated Scheduling**

```ruby
if __name__ == "__main__":

    etl_process()

    schedule.every().day.at("07:00").do(etl_process)  # Schedule ETL to run daily at 7 AM
    logger.info("Scheduled ETL process to run daily at 7 AM. is completed...")

    while True:
        schedule.run_pending()
        time.sleep(60)  # Check every minute
```
**What & Why:**  
Implements industry-standard job scheduling to ensure the pipeline runs consistently every morning, maximizing operational efficiency.

---
## Log Data & Executed Output

```ruby
2025-08-26 18:52:42,946 - INFO - ETL process started.
2025-08-26 18:52:42,947 - INFO - Starting data extraction from MySQL database.....
2025-08-26 18:52:43,010 - INFO - Data extraction completed. Extracted 968 records.
2025-08-26 18:52:43,010 - INFO - Starting data cleaning and transformation .....
2025-08-26 18:52:43,016 - INFO - After removing duplicates and nulls, 19 records were removed. 949 records remain.
2025-08-26 18:52:43,018 - INFO - Removed 5 outlier records based on unit_price.
2025-08-26 18:52:43,021 - INFO - Data cleaning and transformation completed Sucessfully.
2025-08-26 18:52:43,030 - INFO - Saved cleaned data to cleaned_data.csv successfully.
2025-08-26 18:52:43,030 - INFO - Starting sales trend chart creation.....
2025-08-26 18:52:44,406 - INFO - Sales trend chart saved as sales_trend.png successfully.
2025-08-26 18:52:44,406 - INFO - Starting upload to Google Drive.....
2025-08-26 18:52:51,164 - INFO - Successfully retrieved access token
2025-08-26 18:52:51,174 - INFO - file_cache is only supported with oauth2client<4.0.0
2025-08-26 18:52:53,966 - INFO - Uploaded cleaned_data.csv to Google Drive successfully.
2025-08-26 18:52:53,966 - INFO - Starting email sending process.....
2025-08-26 18:52:59,922 - INFO - Email sent successfully.
2025-08-26 18:52:59,922 - INFO - ETL process and Email sending completed.
2025-08-26 18:52:59,923 - INFO - Scheduled ETL process to run daily at 7 AM. is completed...
```
- Drive Screenshot:
<img width="2539" height="781" alt="drive ss" src="https://github.com/user-attachments/assets/dea691a1-7a60-4828-a6d0-b3b7841c736a" />

- Mail Screenshot:
<img width="1267" height="620" alt="mail ss" src="https://github.com/user-attachments/assets/462d0d2f-77b3-452c-88ac-226e1f5a72e2" />

---
## 📈 Project Impact in Real-World Environments

This solution delivers automated daily reporting and visualization, enabling business teams to make informed, rapid decisions by accessing high-quality, cleaned data and actionable product sales trends with zero manual effort. Automation of data cleaning, outlier detection, chart creation, secure sharing, and professional communication demonstrates mastery of the full analytics pipeline—crucial for driving business value in fast-paced industries.

---

## 🖼️ Example Output

- Cleaned sales data: [cleaned_data.csv](https://github.com/AnalystVivek/Daily-Sales-Data-Automation/blob/main/cleaned_data.csv)  
- Visual sales trend multi-chart: [Sales Trend Chart](***)

---

### 💬 Connect
<p align="left"> <a href="https://www.linkedin.com/in/vivekpattnaik" target="_blank"> <img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"> </a>  </a> <a href="mailto:vivek.pattnaik@gmail.com"> <img src="https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"> </a> </p>  

