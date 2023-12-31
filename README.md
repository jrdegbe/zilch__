# zilch__

# Data Engineering on AWS
This project shows one way to perform basic Data engineering using mainly AWS and a few open-source tools.

## Table of contents
- [Overview](#overview)
- [The Goal](#the-goal)
- [The dataset](#the-dataset)
- [Data modeling](#data-modeling)
- [Tools](#tools)
- [Scalability](#scalability)
- [Running the project](#running-the-project)
  * [1. Requirements](#1-requirements)
  * [2. Clone the repository](#2-clone-the-repository)
  * [3. Run Terraform](#3-run-terraform)
  * [4. Create the Data Warehouse](#4-create-the-data-warehouse)
  * [5. Run Airflow](#5-run-airflow)
  * [6. Run the Airflow DAGs](#6-run-the-airflow-dags)
  * [7. Visualise data on Metabase](#7-visualise-data-on-metabase)
- [Project limitations](#project-limitations)
  * [Manual DAGs triggering](#manual-dags-triggering)


## Overview

The current work aims to give answers to business questions concerning bicycle rentals in the city of London from 2021 to January 2022. To do so, we are going to build a data pipeline that collects data from multiple sources, applies transformations, and displays the preprocessed data in a dashboard. 

The following diagram illustrates a high-level structure of the pipeline where data flows from different sources to the final visualization tool.

![The ELT](/images/architecture.png)

## The Goal
The end goal of the current project is to preprocess the data on the AWS platform and get useful insights from it. We can learn more from the data by responding to some of the following business questions on the final dashboard.

- At what time or which hour of the day has the most active rental on average? 

- Which area has the most active bike rentals in London?

- Which day of the week is the most active in general?

- What is the global trend for daily rentals over the year?

## The dataset
We are going to process 3 datasets in this project.

1. __Cycling journey__ dataset from January 2021 to January 2022. It is spread into multiple files in the [Transport for London (TFL)](https://cycling.data.tfl.gov.uk/) website. We will scrap the web page to extract all the relevant links. Then download each file afterward. This dataset contains the main features for every cycling journey, including the locations of the start/end point of each journey, the timestamps for both departure and arrival, etc. 

2. __Stations__ dataset encompasses the details of every station involved in a journey. This dataset is quite outdated as it does not include stations that were added after 2016. To solve this issue, We will add in this old dataset, all the new stations we encounter in each journey. The stations were found in a forum [What do they know](www.whatdotheyknow.com) and can be downloaded directly from [here](https://www.whatdotheyknow.com/request/664717/response/1572474/attach/3/Cycle%20hire%20docking%20stations.csv.txt).

3. __Weather__ dataset includes daily weather data in the city of London from January 2021 to January 2022. It was originally retrieved from [Visual Crossing](https://www.visualcrossing.com/) website and made available to download from [this link](https://drive.google.com/file/d/13LWAH93xxEvOukCnPhrfXH7rZZq_-mss/view?usp=sharing).

In total, the cycling journey data contains 10925928 entries, stations: 808, and weather: 396.

## Data modeling
We are going to build a **Star Schema** which comprises one fact and multiple dimension tables for our Data Warehouse.

The Entity Relational Diagram (ERD) for the final Data Warehouse is represented in the following image:
![The ERD](/images/ERD.png "ERD edited from dbdiagram.io")

In the transformation phase, several columns from both weather and journey data will be removed. Also, we will add dimension table dim_datetime which will contain the reference for all datetime-related columns.

The given schema will facilitate the exploration of the whole data in order to answer relevant business questions about them.

## Tools
1. **Terraform**: an open-source tool that provides `Infrastructure as Code (IaC)`. It allows us to build and maintain our AWS infrastructure including: 'Redshift', 'S3', and 'EC2 instances'. We will not include our 'EMR clusters' in  Terraform as they will be manually added and terminated from `Airflow` when we need them.


2. **Apache Airflow**: an open-source tool to programmatically author, schedule and monitor workflows. The majority of data tasks in the project will be monitored on Airflow.


3. **Selenium** and **BeautifulSoup** are packages that help us to perform web scraping. BeautifulSoup cannot scrape a webpage that displays data lazily, this is where Selenium comes into the picture as it can wait for specific content to load on the page before doing further processing.


4. **AWS Simple Storage System** or Simple Storage System: provides large storage for us to create a __Data Lake__. We will store all the raw data in this location. Also, the preprocessed data will be stored in S3 before being loaded to Redshift.


5. **Apache Spark**: open-source software that can efficiently process Big Data in a distributed or parallel system. We will use PySpark (Spark with Python) to transform the raw data and prepare them for the Data Warehouse on Redshift.


6. **AWS Elastic MapReduce**, a managed cluster platform that allows the running of big data tools such as Spark and Hadoop. We will employ AWS EMR to run our Spark jobs during the transformation phase of the data.


7. **AWS Redshift**, a fully managed and highly scalable data warehouse solution offered by Amazon. We will build our Data Warehouse on Redshift and we will make the data available for visualization tools from there.


8. **Metabase** is another open-source software that allows easy visualization and analytics of structured data. We will build a dashboard with Metabase to better visualise our data stored in Redshift.


9. **Docker**, a set of platforms as a service that containerizes software, allowing them to act the same way across multiple platforms. In this project, we will run Airflow and Metabase on Docker.


## Scalability

It is always a good practice to consider scalability scenarios when building a data pipeline. A significant increase in the data in the future is much expected. 

For instance, if the volume of the data has increased 500x or even as high as 1000x, that should not break our pipeline. 


We need to scale our EMR cluster nodes either horizontally or both vertically and horizontally 


- Horizontal scale refers to adding more cluster nodes to process the high-volume data.


- Vertical and Horizontal scale means that we increase the performance of existing nodes. Then we also add new nodes to the cluster.


## Running the project

### 1. Requirements

In order to run the project smoothly, a few requirements should be met:

- AWS account with sufficient permissions to access and work on S3, Redshift, and EMR. 
To do so: 
    *   Go to [IAM](https://console.aws.amazon.com/iam/home) in AWS console.

    *   Create a new user

    *   Add permissions to that new user: `AmazonS3FullAccess`, `AmazonRedshiftFullAccess`, `AdministratorAccess`, `AmazonEMRFullAccessPolicy_v2`, `AmazonEMRServicePolicy_v2`, `AmazonEC2FullAccess`.

    *   In the "Security credentials" tab, create an access key and download the `.csv` file.  


- It is also necessary to have the AWS account preconfigured (i.e. having `~/.aws/credentials` and `~/.aws/config` available in your local environment). 


- Docker and Docker Compose, preinstalled in your local environment. Otherwise, they can be installed from [Get Docker](https://docs.docker.com/get-docker/).


- Terraform preinstalled in your local environment.


### 2. Clone the repository

```bash
git clone https://github.com/jrdegbe/zilch__.git
```

### 3. Run Terraform

We are going to use Terraform to build our AWS infrastructure

From the project root folder, move to the `./terraform` directory

```bash
cd terraform
```

Run terraform commands one by one

- Initialization

    ```bash
    terraform init
    ```

- Planning

    ```bash
    terraform plan
    ```

- Applying

    ```bash
    terraform apply
    ```

### 4. Create the Data Warehouse

- Go to the [AWS Redshift](https://console.aws.amazon.com/redshiftv2/home) cluster which was freshly created from Terraform. 


- Connect to your database then go to `Query Data`.

- Manually `Copy` the content of [CyclingERD.sql](/CyclingERD.sql) into the query field and `RUN` the command. This will create the tables and attach constraints to them.


### 5. Run Airflow

     
- From the project root folder, move to the `./airflow` directory

    ```bash
    cd airflow
    ```

- Create environment variables in the `.env` file for our future Docker containers.

    ```bash
    cp .env.example .env
    ```

- Fill in the content of the `.env` file.
    The value for `AIRFLOW_UID` is obtained from the following command:

    ```bash
    echo -e "AIRFLOW_UID=$(id -u)"
    ```

    Then the value for `AIRFLOW_GID` can be left to `0`.

    - Build our extended Airflow Docker image

    ```bash
    docker build -t airflow-img .
    ```

- Run docker-compose to launch Airflow

    Initialise Airflow

    ```bash

    docker-compose up airflow-init 
    ```

    Launch Airflow

    ```bash
    docker-compose up
    ```
    
    This last command launched `Airflow Postgres` internal database, `Airflow Scheduler` and `Airflow Webserver` which could have been launched separately if we did not use Docker.

### 6. Run the Airflow DAGs
    
Once Airflow is up and running, we can now proceed to the most exciting part of the project.

The initialisation DAGs (`init_?_*_dag`) are interdependent. In essence, each DAG wait the success run of its predecessor before starting its tasks. 
For instance, `init_1_spark_emr_dag` will not be started until `init_0_ingestionto_s3_dag` is complete successfully.
In order to trigger these DAGs, please enable the 4 of them _SIMULTANEOUSLY_.

The processor DAGs (`proc_?_*_dag`) on the other hand, needs to be started individually. 
__It is necessary to wait for 4 initialisation DAGs to complete before starting the processor ones__.
To run these last 3 DAGs, please enable the `proc_0_ingestion_to_s3_dag`, wait for it to finish its tasks before enabling the next DAG: `proc_1_spark_emr_dag`.
Likewise, it is necessary to wait unti the end of `proc_1_spark_emr_dag` process before enabling the last DAG: `proc_2_s3_to_redshift_dag` 


The following screenshot shows a success run of the first DAG.

![Ingestion DAG (init_0_ingestion_to_s3_dag)](/images/dags/init_0.png "Ingestion DAG in the Graph view")


After all dags operations, we can now move to Metabase to visualise the data. 

### 7. Visualise data on Metabase
    
Again we will install and run Metabase in a Docker container.
```bash
docker run -d -p 3033:3000 --name metabase metabase/metabase
```

For the very first time of its execution, the above command downloads the latest Docker image available for Metabase before exposing the application on port `3033`.

Once the above command finishes its execution, Metabase should be available at [http://localhost:3033](http://localhost:3033).

We can now connect our Redshift database to this platform and visualise the data in multiple charts.

The following screenshot displays a part of our final dashboard which clearly shows some useful insights about bicycle rides in different dimensions.

![Final Dashboard](/images/final-dashboard.png "The final dashboard on Metabase")
