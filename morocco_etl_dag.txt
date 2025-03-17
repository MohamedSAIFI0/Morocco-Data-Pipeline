# Mohamed SAIFI CLOUD DATA ENGINEER
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
import pandas as pd
import io
import boto3
import mysql.connector
import os
from sqlalchemy import create_engine


default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}


def extract_data():
    """Extract data from S3 and save to the local filesystem"""
    s3 = boto3.client(
        's3',
        #aws_access_key_id=os.getenv("AWS_ACCESS_KEY_ID"),
        #aws_secret_access_key=os.getenv("AWS_SECRET_ACCESS_KEY"),                       #dont forget to change this
       # region_name=os.getenv("AWS_REGION")
    )
    
    bucket_name = "raw-data-morocco-saifi"
    file_name = "morocco_data.csv"
    
    response = s3.get_object(Bucket=bucket_name, Key=file_name)
    csv_data = response['Body'].read()
    csv_file = io.StringIO(csv_data.decode('utf-8'))
    df = pd.read_csv(csv_file)
    

    df.to_csv('/opt/airflow/data/raw_data.csv', index=False)
    print("Extraction complete. Data saved to /opt/airflow/data/raw_data.csv")
    return '/opt/airflow/data/raw_data.csv'

def transform_data(**context):
    """Clean and transform the data"""

    data_path = context['ti'].xcom_pull(task_ids='extract_task')
    data = pd.read_csv(data_path)
    
    print(f"Data shape: {data.shape}")
    print(f"Columns: {data.columns}")
    print(f"Missing values: {data.isnull().sum().sum()}")
    
    data = data.rename(columns={
        'GDP (Million USD)': 'GDP_USD',
        'Tourists per Year': 'Tourists_per_Year',
        'Main Industry': 'Main_Industry',
        'Average Temperature (°C)': 'Average_Temperature'
    })
    

    data.dropna(inplace=True)
    data.drop_duplicates(inplace=True)

    data['Average_Temperature'] = pd.to_numeric(data['Average_Temperature'], errors='coerce')
    
  
    data['GDP_DHS'] = data['GDP_USD'] * 10
    data['Tourists_per_Month'] = data['Tourists_per_Year'] / 12

    output_path = '/opt/airflow/data/cleaned_data.csv'
    data.to_csv(output_path, index=False)
    print(f"Transformation complete. Data saved to {output_path}")
    return output_path

def load_data(**context):
    """Load data to MySQL database"""
    data_path = context['ti'].xcom_pull(task_ids='transform_task')
    

    engine = create_engine("mysql+pymysql://root:password@mysql:3306/Morocco") 
    conn = engine.raw_connection()
    cursor = conn.cursor()

    create_table_query = """
    CREATE TABLE IF NOT EXISTS morocco_table(
        ID INT AUTO_INCREMENT PRIMARY KEY,
        City VARCHAR(20),
        Population INT,
        GDP_USD INT,
        Tourists_per_Year INT,
        Main_Industry VARCHAR(50),
        Average_Temperature FLOAT,
        GDP_DHS FLOAT,
        Tourists_per_Month FLOAT
    );
    """
    cursor.execute(create_table_query)
    conn.commit()
    
    df = pd.read_csv(data_path)
    
    if 'ID' in df.columns:
        df = df.drop('ID', axis=1)

    df.to_sql("morocco_table", con=engine, if_exists="replace", index=False)
    print("Data loaded successfully to MySQL database")
    
    conn.close()

dag = DAG(
    'morocco_etl_pipeline',
    default_args=default_args,
    description='ETL pipeline for Morocco data',
    schedule_interval=timedelta(minutes=5),
    start_date=datetime(2025, 3, 17),
    catchup=False,
    tags=['morocco', 'etl'],
)

extract_task = PythonOperator(
    task_id='extract_task',
    python_callable=extract_data,
    dag=dag,
)

transform_task = PythonOperator(
    task_id='transform_task',
    python_callable=transform_data,
    provide_context=True,
    dag=dag,
)

load_task = PythonOperator(
    task_id='load_task',
    python_callable=load_data,
    provide_context=True,
    dag=dag,
)

extract_task >> transform_task >> load_task