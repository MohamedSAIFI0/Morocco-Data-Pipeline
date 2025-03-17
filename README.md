# Morocco ETL Pipeline with Airflow and Docker

This project implements an ETL (Extract, Transform, Load) pipeline for Morocco data using Apache Airflow orchestration and Docker containerization.

## Project Structure

```
morocco-etl-project/
├── dags/               # Airflow DAG definitions
├── plugins/            # Airflow plugins
├── data/               # Mounted volume for data files
├── logs/               # Mounted volume for logs
├── Dockerfile          # Custom Airflow image
├── docker-compose.yml  # Docker Compose configuration
├── requirements.txt    # Python dependencies
└── README.md           # This file
```

## Prerequisites

- Docker and Docker Compose installed
- AWS credentials (for S3 access)

## Setup Instructions

1. Clone this repository:
   ```
   git clone <repository-url>
   cd morocco-etl-project
   ```

2. Create the required directories:
   ```
   mkdir -p dags plugins data logs
   ```

3. Update AWS credentials in the `dags/morocco_etl_dag.py` file:
   ```python
   s3 = boto3.client(
       's3',
       aws_access_key_id='YOUR_ACCESS_KEY',
       aws_secret_access_key='YOUR_SECRET_KEY',
       region_name='YOUR_REGION'
   )
   ```

4. Start the containers:
   ```
   docker-compose up -d
   ```

5. Access the Airflow web interface:
   - Open a browser and navigate to `http://localhost:8080`
   - Login with username `admin` and password `admin`

6. Trigger the DAG from the Airflow UI or wait for it to run at the scheduled time.

## Pipeline Overview

1. **Extract**: Downloads data from AWS S3 bucket and saves it to a local file.
2. **Transform**: Cleans and transforms the data, creating new features.
3. **Load**: Loads the transformed data into a MySQL database.

## Customization

- Modify the DAG schedule in `dags/morocco_etl_dag.py` by changing the `schedule_interval` parameter.
- Adjust MySQL connection settings in the `docker-compose.yml` file and in the DAG code.
- Add more transformation steps in the `transform_data` function as needed.

## Troubleshooting

- Check container logs:
  ```
  docker-compose logs -f
  ```

- Check Airflow task logs through the web interface or in the `logs/` directory.

- Ensure all containers are running:
  ```
  docker-compose ps
  ```