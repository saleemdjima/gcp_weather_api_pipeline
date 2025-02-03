# Weather Data Pipeline on Google Cloud Platform

This project sets up an automated weather data pipeline using various GCP services to collect, store, and visualize weather data.

## Prerequisites

1. Google Cloud Platform account
2. Weather API token (from weatherapi.com)
3. Basic familiarity with GCP services

## Architecture

![Architecture Diagram](/image/Architecture.png)

## Setup Instructions

### 1. Initial Setup

1. Create a new GCP Project

   - Project name: `weather-api-project`

2. Enable Required APIs

```bash
gcloud services enable cloudfunctions.googleapis.com \
    sqladmin.googleapis.com \
    run.googleapis.com \
    cloudbuild.googleapis.com \
    artifactregistry.googleapis.com \
    eventarc.googleapis.com \
    compute.googleapis.com \
    servicenetworking.googleapis.com \
    pubsub.googleapis.com \
    logging.googleapis.com
```

### 2. Cloud Scheduler Configuration

1. Create a Cron Job
   - Name: `weather_call`
   - Region: `us-central1`
   - Frequency: `0 * * * *` (hourly)
   - Timezone: UTC

2. Create Pub/Sub Topic
   - Topic ID: `weather_calls`
   - Message body: `update`

### 3. Database Setup

#### A. Infrastructure Setup

1. Create Compute Engine VM
   - Name: `weather-vm`
   - Series: N1
   - Machine type: f1-micro
   - Network settings: Allow HTTP/HTTPS traffic

2. Setup Cloud SQL

   - Engine: MySQL 8.0
   - Instance ID: `weather-db`
   - Preset: Development
   - Machine: Lightweight (1vCPU, 3.75GB)
   - Storage: 10GB SSD (auto-increase enabled)
   - Network connection name: `connection-db-vm`

#### B. Database Configuration

1. Install MySQL on VM

```bash
sudo apt-get update
sudo apt-get install default-mysql-server
```

2. Connect to MySQL

```bash
mysql -h <INSTANCE_IP> -u root -p
```

3. Create Database and Table

```sql
CREATE DATABASE weather_db;

CREATE TABLE IF NOT EXISTS weather_data (
    id INT PRIMARY KEY AUTO_INCREMENT,
    lat FLOAT NOT NULL,
    lon FLOAT NOT NULL,
    temperature_c FLOAT NOT NULL,
    feelslike_c FLOAT NOT NULL,
    humidity FLOAT NOT NULL,
    last_updated timestamp,
    wind_kph FLOAT,
    name varchar(255)
);
```

### 4. Pub/Sub Configuration

1. Create Topic
   - Topic ID: `apiweather-extract`

2. Create Subscription
   - Subscription ID: `apiweather-extract-subscription`

### 5. Cloud Functions Setup

#### A. Weather Data Puller

- Name: `pull-weather-data`
- Region: `us-central1`
- Generation: 1st gen
- Memory: 512 MiB
- Environment Variables:

  ```
  api_token=<YOUR_API_TOKEN>
  base_url=http://api.weatherapi.com/v1/current.json
  location=<YOUR_LOCATION>
  project_id=<YOUR_PROJECT_ID>
  region=us-central1
  topic_id=apiweather-extract
  ```
- Source: [pull_from_api.py](https://github.com/team-data-science/course-gcp/blob/main/code/pull_from_api.py)
- Requirements: [requirements.txt](https://github.com/team-data-science/course-gcp/blob/main/code/pull-weather-data_requirements.txt)

#### B. SQL Writer

- Name: `write-to-sql`
- Region: `us-central1`
- Generation: 2nd gen
- Memory: 256 MiB
- Environment Variables:
  ```
  project_id=<YOUR_PROJECT_ID>
  region=us-central1
  db_user=root
  db_pass=admin1234
  db_name=weather_db
  instance_name=weather-db
  ```

#### C. Database Pusher

- Name: `weather-data-to-db`
- Region: `us-central1`
- Generation: 1st gen
- Memory: 512 MiB
- Environment Variables:
  ```
  project_id=<YOUR_PROJECT_ID>
  region=us-central1
  db_user=root
  db_pass=admin1234
  db_name=weather_db
  instance_name=weather-db
  subscription_id=apiweather-extract-subscription
  ```


1. Connect to Looker Studio
   - URL: https://lookerstudio.google.com/
   - Use Instance Connection Name
   - Database: `weather_db`
   - Credentials: root/admin1234

2. Dashboard Tips

   - For location concatenation:
     ```sql
     CONCAT(lat,",",lon)
     ```

### Notes
The setup above is only for tutorial purposes. In a real-world scenario, you would need to setup a proper VPC network and do a better configuration of the Cloud instance.
