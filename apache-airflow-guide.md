# Apache Airflow Installation and Usage Guide

## Troubleshooting

Before beginning, ensure your system is up to date and you have the latest versions of essential tools:

```bash
sudo apt update
python -m pip install --upgrade pip
pip install --upgrade setuptools
```

### Setting up Python

Ensure you're using Python 3.6:

```bash
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.6 2
sudo update-alternatives --config python
# Select python3.6
sudo update-alternatives --set python /usr/bin/python3.6
python --version
```

## Creating a Virtual Environment

It's recommended to use a virtual environment for Apache Airflow:

```bash
sudo apt install python3-venv
python3 -m venv myenv
source myenv/bin/activate
```

## Installing Apache Airflow

Install Apache Airflow and its dependencies:

```bash
pip install apache-airflow
pip install pandas
pip install s3fs
```

## Starting Apache Airflow

Start Apache Airflow in standalone mode:

```bash
airflow standalone
```

## Creating an Admin User

Create an admin user for the Airflow UI:

```bash
airflow users create --role Admin --username admin --email admin --firstname admin --lastname admin --password admin
```

Or, for an alternative user:

```bash
airflow users create --role Admin --username admin2 --email admin --firstname admin --lastname admin --password admin
```

## Accessing Apache Airflow UI

Open a web browser and navigate to:

```
localhost:8080
```

Use the following credentials:
- Username: admin
- Password: admin

## Alpha Vantage API

For stock data, we'll use the Alpha Vantage API:

- API: https://www.alphavantage.co/support/#api-key
- Link: https://www.alphavantage.co/documentation/
- API Key: ODGUTFT1TU5LHI4O

Example API call:

```
https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=IBM&interval=5min&apikey=ODGUTF1TU5LHI4O
```

## Setting Up Airflow DAGs

1. Open a new terminal tab
2. Activate the virtual environment:

```bash
python3 -m venv myenv
source myenv/bin/activate
```

3. Create a new directory for DAGs:

```bash
cd airflow
mkdir apsit
cd ..
```

4. Edit the Airflow configuration:

```bash
gedit airflow.cfg
```

Add the new directory for keeping DAGs, then save and exit.

5. Create a new DAG file:

```bash
gedit alpha.py
```

6. Add the following code to `alpha.py`:

```python
import os
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import requests
import json

# Set your AlphaVantage API key here
API_KEY = 'ODGUTF1TU5LHI4O'
STOCK_SYMBOL = 'IBM'

# Get the Airflow home directory
HOME_DIR = os.path.expanduser('~')
DATA_DIR = os.path.join(HOME_DIR, 'airflow_data')
os.makedirs(DATA_DIR, exist_ok=True)

# Paths to store the JSON files
STOCK_DATA_PATH = os.path.join(DATA_DIR, 'stock_data.json')
TRANSFORMED_DATA_PATH = os.path.join(DATA_DIR, 'transformed_stock_data.json')

# Default arguments for the DAG
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

# Initialize the DAG
dag = DAG(
    'alphavantage_stock_data',
    default_args=default_args,
    description='Download, transform, and load stock data from AlphaVantage',
    schedule='@hourly',
    start_date=datetime(2024, 9, 30),
)

# Task 1: Download stock data from AlphaVantage
def download_stock_data():
    url = f'https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol={STOCK_SYMBOL}&interval=5min&apikey={API_KEY}'
    response = requests.get(url)
    data = response.json()
    # Save stock data to the Airflow data directory
    with open(STOCK_DATA_PATH, 'w') as f:
        json.dump(data, f)

# Task 2: Transform the stock data
def transform_stock_data():
    with open(STOCK_DATA_PATH, 'r') as f:
        data = json.load(f)
    time_series = data.get('Time Series (5min)', {})
    transformed_data = []
    for timestamp, values in time_series.items():
        transformed_data.append({
            'timestamp': timestamp,
            'open': values['1. open'],
            'high': values['2. high'],
            'low': values['3. low'],
            'close': values['4. close'],
            'volume': values['5. volume'],
        })
    # Save transformed data to the Airflow data directory
    with open(TRANSFORMED_DATA_PATH, 'w') as f:
        json.dump(transformed_data, f)

# Task 3: Output transformed data
def output_transformed_data():
    with open(TRANSFORMED_DATA_PATH, 'r') as f:
        transformed_data = json.load(f)
    print("Transformed Stock Data:")
    for entry in transformed_data:
        print(entry)

# Define the tasks using PythonOperator
download_task = PythonOperator(
    task_id='download_stock_data',
    python_callable=download_stock_data,
    dag=dag,
)

transform_task = PythonOperator(
    task_id='transform_stock_data',
    python_callable=transform_stock_data,
    dag=dag,
)

output_task = PythonOperator(
    task_id='output_transformed_data',
    python_callable=output_transformed_data,
    dag=dag,
)

# Define task dependencies
download_task >> transform_task >> output_task
```

## Running the DAG

1. Stop the Apache Airflow standalone server (Ctrl-C)
2. Restart Apache Airflow:

```bash
airflow standalone
```

3. In the Airflow UI, search for "alpha" in DAGs
4. Run the DAG

## Accessing the Data

To view the generated data:

```bash
cd ~/airflow_data
gedit stock_data.json
gedit transformed_stock_data.json
```

## Scheduling Options in Airflow DAGs

| Preset | Description | Example Use Case |
|--------|-------------|------------------|
| @once | Run the DAG only once | For one-time jobs |
| @hourly | Run every hour | Regular hourly data processing |
| @daily | Run once a day at midnight | Daily ETL jobs |
| @weekly | Run once a week, midnight Sunday | Weekly reports |
| @monthly | Run once a month, midnight of 1st | Monthly billing jobs |
| @yearly or @annually | Run once a year, midnight of Jan 1st | Yearly archiving jobs |

This guide provides a comprehensive overview of setting up Apache Airflow, creating a DAG for stock data processing, and various scheduling options. Students can follow these steps to get started with Airflow and understand its basic concepts.
