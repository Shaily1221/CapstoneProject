# NETCOM Censys Capstone Project

Brent Rodhouse, Clifford Rosenberg, Shaily Shah, and Iris Zhang  
Carnegie Mellon University, Heinz College  
Fall 2021

# Introduction

Network Enterprise Technology Command (NETCOM) seeks to automate data scraping processes and integrate available
information from the Censys platform to generate risk scores for IP addresses and domains. Our project makes use of 3
data sources:
Censys.io data which was provided to us by Censys through a BigQuery API, Newly Registered Domains data which came from
whoisds.com, and Blocklist data which was taken from snort.org. These data sources are used to derive cybersecurity
vulnerabilities and provide risk analysis for specific IPs and domains. By providing this information, the client will
have the capability to anticipate attacks from these malicious domains by identifying and blocking their traffic, patch
and remediate identified devices with known vulnerabilities within their operational domain (Army.mil), and rectify any
hosts using expired certificates. This will increase the security posture of the client by enabling them to take a more
proactive, rather than reactive, approach to managing and mitigating cybersecurity risks.

# Data Sources

The following section will outline the various data sources we are using as part of this project.

## [Censys](censys.io)

Data about the hosts and services on the Internet that "scans 2,329 Ports across the entire IPv4 address space, detects
36 protocols across the 2,029 ports, allowing for intelligent identification of services running on non-standard ports,
and had the largest repository of TLS certifcates." (https://censys.io/product/product-data/)

## Newly-Registered Domains from [whoisds.com](https://whoisds.com/newly-registered-domains)

Thousands of new registered domains (NRDs) are displayed each day, many for useful purposes such as introducing new
products, hosting new sites, and creating new brands. However, the majority are suspicious and many are malicious.

"A domain is considered newly registered if it has been registered or had a change in ownership within the last 32
days." (https://www.paloaltonetworks.com/cyberpedia/what-are-malicious-newly-registered-domains)

## IP Blocklist from [snort.org](https://snort.org)

SNORT can be used to track traffic entering and exiting a network. When it detects potentially harmful packets or
threats on Internet Protocol (IP) networks, it will monitor traffic in real time and send out alerts to users.

Snort.org provides a list of domains that are considered to be malicious which we use to compare to the newly registered
domains list to see if there are any IPs that should be blocked.

# Tools

[Google BigQuery](https://cloud.google.com/bigquery): a fully-managed, serverless data warehouse that enables scalable
analysis over Censys data

[Jupyter Notebook](https://jupyter.org/): an open-source web application that allows our team to integrate live code and
visualizations using python programming language

[Docker](https://www.docker.com/): an open-source containerization platform that enables our team to package our python
applications into containers

[Apache Airflow](https://airflow.apache.org/): an open-source tool to automate, schedule, and monitor our process
workflows.

[AWS](https://aws.amazon.com/): a premier cloud computing platform offering a variety of cloud services which we use to
store our entire process within EC2 and final reports within S3.

# Airflow Task Structure

All tasks associated with this project are defined in [`netcom_censys_dag.py`](#dag). Below is a visual breakdown of the
Airflow tasks, as well as a description of each task:  
![Airflow Task Graph](images/airflow_task_graph.png)

- `get_army_mil`: This task is responsible for fetching the army.mil Censys data from Google BigQuery.
- `army_mil_analysis`: This task analyzes the data fetched by the `get_army_mil` task and finds potential
  vulnerabilities. It then pushes its findings to the S3 bucket.
- `snort`: This task fetches SNORT's IP blocklist.
- `get_purge_domains`: This task fetches any newly-registered domains lists that haven't yet been downloaded, and
  deletes the oldest lists to prevent buildup over time.
- `get_analyze_narrow`: This task fetches the Censys data from Google BigQuery that will be matched with
  newly-registered and blocklisted domains, affectionately referred to as the "narrow table." It then combines this with
  newly-registered and blocklisted domains, then pushes the result to the S3 bucket.
- `get_certificates`: This task is responsible for fetching the Censys data on certificates associated with army.mil. It
  then pushes its findings to the S3 bucket.
- `cleanup`: After all of the upstream tasks have finished, this task removes any files that were generated during
  previous tasks and need to be removed, then pushes the logs that were generated during previous tasks to the S3
  bucket. This task will run even if upstream tasks fail or are skipped. This ensures that the logs are pushed to S3
  even if something goes wrong upstream.

**Interval Triggers**

- `weekly_trigger`: This task evaluates whether today (UTC) is a Wednesday. If it is, it allows downstream tasks
  (`get_army_mil`, `snort`) to execute. If it is not, these downstream tasks are skipped. The reason for this is that we
  only want the downstream tasks to execute once per week, as some of the data we're collecting only is updated weekly.
- `quarterly_trigger`: This task evaluates if its downstream task (`get_certificates`) has run recently. If it has run
  in the past 90 days, then it will cause the downstream task to skip. The 90-day interval is configurable via the
  `certificate_fetch_period` variable in [`constants.py`](#constants).

# File Structure

The project's root directory is `airflow-docker`, which was based on the result of the `docker-compose.yaml` file found
[here](https://airflow.apache.org/docs/apache-airflow/2.1.3/docker-compose.yaml). For more information on Airflow's
basic Docker implementation, see
[this](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html).

Below is a breakdown of the file structure:

```angular2html
airflow-docker/
├── dags/
│   ├── src/
│   │   ├── __init__.py
│   │   ├── activity_logger.py
│   │   ├── army_mil_vulnerabilities.py
│   │   ├── censys_bigquery.py
│   │   ├── client_secrets.json
│   │   ├── constants.py
│   │   ├── credentials.pkl
│   │   ├── domain_file_manager.py
│   │   ├── fetch_new_domains.py
│   │   ├── ip_resolver.py
│   │   ├── newly_registered_join.py
│   │   ├── s3_utility.py
│   │   └── snort_blocklist.py
│   └── netcom_censys_dag.py
├── intermediate_data/
│   ├── censys_bigquery/
│   ├── ip_blocklist/
│   │   └── snort_blocklist.txt
│   ├── netcom_logs/
│   │   ├── log_2021-11-22.txt
│   │   └── log_2021-11-28.txt
│   └── newly_registered_domains/
├── plugins/
├── Dockerfile
├── airflow.sh*
├── docker-compose.yaml
└── requirements.txt
```

## airflow-docker/

- [`Dockerfile`](airflow-docker/Dockerfile): This file configures the setup so that we have all Python dependencies
  needed within the docker containers.
- [`airflow.sh`](airflow-docker/airflow.sh): A convenience script to quickly log into the container (
  found [here](https://airflow.apache.org/docs/apache-airflow/2.1.3/airflow.sh)). You can call `./airflow.sh bash` to
  get a `bash` shell within the container, `./airflow.sh python` to access the Python CLI in the container,
  or `./airflow.sh info` to get general info on the containers.
- <a id='dcyaml'>[`docker-compose.yaml`](airflow-docker/docker-compose.yaml)</a>: This file is responsible for building
  all the Docker containers necessary to run Apache Airflow within Docker. Running `docker-compose up --build` within
  the `airflow-docker/`
  directory will call on this file to build images and start Airflow.
- [`requirements.txt`](airflow-docker/requirements.txt): This file contains all the necessary Python packages to run our
  project. This is passed to `pip install -r`
  within `Dockerfile`.

### airflow-docker/dags

The [`dags`](airflow-docker/dags) folder holds the Airflow DAG definition for the project, as well as all of the
supporting source files.

- <a id='dag'>[`netcom_censys_dag.py`](airflow-docker/dags/netcom_censys_dag.py)</a>: This file contains the DAG
  definition for our project. It imports the source files (see below) that are needed to run the entire project.

#### airflow-docker/dags/src

This directory contains the main source files for the project.

- [`activity_logger.py`](airflow-docker/dags/src/activity_logger.py): This file contains logic to create a custom log to
  show the progress of a dag run.
- [`army_mil_vulnerabilities.py`](airflow-docker/dags/src/army_mil_vulnerabilities.py): This file takes the raw Censys
  data for hosts associated with the army.mil domain and identifies their potential vulnerabilities.
- [`censys_bigquery.py`](airflow-docker/dags/src/censys_bigquery.py): This file provides an interface to easily fetch
  different data queries from BigQuery.
- `client_secrets.json`: This is a JSON key file that you will use to create credentials to access Google BigQuery. This
  is something that must be done with your own Google account that has been provided access to the Censys data. For more
  information, see
  [this](https://cloud.google.com/bigquery/docs/authentication/end-user-installed).
- `credentials.pkl`: This is the credentials file you will use to authenticate requests to Google BigQuery. It is
  created from
  `client_secrets.json` (See `censys_bigquery.py`'s
  `CensysBigQuery._get_credentials()`). If this file has not yet been created, you'll need to run the following code to
  generate it. Note that this code will require interaction with a browser, but it will only have to be done once to
  create the credentials file.

```angular2html
from google_auth_oauthlib import flow
# Note: client_secrets.json is the OAuth key you must download from Google Cloud
appflow = flow.InstalledAppFlow.from_client_secrets_file(
"client_secrets.json", scopes=["https://www.googleapis.com/auth/bigquery"]
)
appflow.run_local_server()
credentials = appflow.credentials
# For saving the credentials to file
import pickle
with open('credentials.pkl', 'wb') as f:
pickle.dump(credentials, f)
```

- <a id='constants'>[`constants.py`](airflow-docker/dags/src/constants.py)</a>: This file holds a lot of constant values
  that are used by several of the different tasks. It exists to provide consistent changes if a file or directory name
  wants to be changed. It prevents the need to change the name in several different places across several scripts.
- [`domain_file_manager.py`](airflow-docker/dags/src/domain_file_manager.py): This script contains code to maintain the
  locally-stored newly-registered domain repository, which is a store of the last 40 fetched newly-registered domains
  lists from
  [https://whoisds.com/newly-registered-domains](https://whoisds.com/newly-registered-domains). It deletes old ones and
  calls functions to fetch new ones from the website as needed and resolve the domain names to IP addresses. It does
  these with the help of `fetch_new_domains.py` and `ip_resolver.py` (see below).
- [`fetch_new_domains.py`](airflow-docker/dags/src/fetch_new_domains.py): This file contains code to fetch the
  newly-registered domains. See above.
- [`ip_resolver.py`](airflow-docker/dags/src/ip_resolver.py): This file's responsibility is to resolve domain names
  (like "google.com") to their IPv4 address (like 128.54.62.4).
- [`newly_registered_join.py`](airflow-docker/dags/src/newly_registered_join.py): This script combines the
  newly-registered domains with the IP addresses from the SNORT blocklist and returns them for further processing.
- [`s3_utility.py`](airflow-docker/dags/src/s3_utility.py): This script combines the logic for interacting with the
  project's S3 bucket.
- [`snort_blocklist.py`](airflow-docker/dags/src/snort_blocklist.py): This file is responsible for fetching the SNORT
  blocklist.

### airflow-docker/intermediate_data/

This directory is responsible for storing data that is used in the intermediate steps of the process. It contains four
subdirectories:

- [`censys_bigquery/`](airflow-docker/intermediate_data/censys_bigquery): This temporarily holds data fetched from
  BigQuery. It is emptied when a DAG run is completed.
- [`ip_blocklist`](airflow-docker/intermediate_data/ip_blocklist): This contains the newest IP blocklist obtained from
  SNORT, located at [https://snort.org/downloads/ip-block-list](https://snort.org/downloads/ip-block-list)
  and stored in `snort_blocklist.py`. This file is overwritten when a new list is fetched, but is kept in case that a
  new list is not able to be fetched on the next run.
- [`netcom_logs/`](airflow-docker/intermediate_data/netcom_logs): This directory contains log files, one for each DAG
  run date. Note that these logs are independent of Airflow's default logging. They are much more simplified and custom
  to the specific tasks that are implemented for this project. At the end of a DAG run, they are pushed to the S3
  bucket. In the file structure shown above, there are a few example log files from November 22 and 28.
- [`newly_registered_domains/`](airflow-docker/intermediate_data/newly_registered_domains): This directory contains the
  repository of the last 40 fetched newly-registered domains lists from [whoisds.com](https://whoisds.com).

# AWS Infrastructure

The following section provides an outline of the infrastructure on AWS that we used as the foundation of the project.

## Hosting

We chose an [EC2](https://aws.amazon.com/ec2/) instance backed by
[EBS](https://aws.amazon.com/ebs) to host our project. Early on, we anticipated high memory usage, so we opted for a
m5a.4xlarge instance, but after further development, we were able to scale down to a m5a.2xlarge. This could most likely
be reduced even further to reduce cost. For more information on AWS's m5 instances, see
[their page](https://aws.amazon.com/ec2/instance-types/m5/) on it.  
The gp3 EBS volume for this instance was chosen as 100 GB to be safe, but this could be scaled down as well.  
In terms of software, we went with the standard Amazon Linux 2 Kernel 5.10
(ID ami-0d718c3d715cec4a7). On this, we installed Docker and docker-compose, then configured the Docker daemon to run
upon startup.

### Accessing the EC2 Instance

We created two points of access for the EC2 instance. We did this by configuration of a
[security group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html).

1. The first was the typical SSH access on port 22 for accessing the CLI.
2. The second and more used access point was the
   [Airflow Web UI](https://airflow.apache.org/docs/apache-airflow/stable/ui.html)
   accessible over HTTP on port 8989 (this port is configurable in
   [`docker-compose.yaml`](#dcyaml) in the `airflow-webserver` section).

We made each of these access points only accessible via a single client IP address (Brent's laptop) to prevent undesired
access.

_One should note that on every startup of the EC2 instance, the IP changes, and the URL changes, so you'd need to ping a
new URL each time. This can be prevented by attaching an
[Elastic IP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
to the instance. While we did not do this in the end, it will save a lot of headache to implement this if the instance
is needing to be accessed often via either SSH or the Airflow Web UI._

### EC2 Start/Stop Automation

The project runs daily, with some parts running weekly and some running every 90 days, so it doesn't make sense to keep
the EC2 instance running constantly. Instead, we created an automation process to start and stop the instance each day.

We did this with AWS's [Lambda](https://aws.amazon.com/lambda/) service. Lambda allows one to run a block of code (say,
Python) whenever it receives a specified trigger. We did this by having a Lambda function fire a few minutes before 00:
00 UTC. This function would send a command to start the EC2 instance. Right after 00:00 UTC, the Airflow DAG will run,
generate reports, then push the generated logs to the S3 bucket as a final step
([see the section below](#final-report-storage-on-s3)).

To stop the instance, we created another Lambda function that would stop the instance, and set a trigger for it to
listen for the log file coming into the S3 bucket. Once it saw the log file come in, it would send the command to stop.
The instance would then shut down, ensuring that the instance would not need to run longer than was necessary to
complete the job.

_Note that if something did happen, and the log file was not able to make it to the S3 bucket, the EC2 instance would
continue running. While we did not experience this issue, it's definitely possible via a network partition between the
EC2 instance and the bucket. To mitigate this, we added a second trigger that would shut down the instance 2 hours after
it had started
(a few minutes before 01:55 UTC)._

## Final Report Storage on S3

The outputs of our process needed to be easily accessible regardless of the EC2 instance's state, so we decided to send
all the reports to an S3 bucket. The current setup is designed to send the reports to a bucket called
`netcom-censys-bucket`, but this can be easily changed by changing the value of the `s3_bucket_name` variable in
`airflow-docker/dags/src/constants.py`. The prefixes of the bucket objects correspond to the DAG run date that they are
associated with. For example, the army vulnerabilities report that was generated from DAG run 2021-11-22 will be located
at `2021-11-22/army_vulnerabilities.csv`. Over the course of DAG runs, you'll see the following objects appear in the
bucket, where "[YYYY-MM-DD]" is the date of the DAG run:

- `[YYYY-MM-DD]/netcom_log.txt`: The custom generated logs we made so the processes being done during the DAG run can be
  tracked. This should be generated every day, as there are parts of the process that run every day.
- `[YYYY-MM-DD]/army_vulnerabilities.csv`: The report concerning the vulnerabilities of hosts associated with the
  army.mil domain. This report will only be generated once per week, as the Censys data is only updated weekly on Google
  BigQuery.
- `[YYYY-MM-DD]/newly_registered_and_blocked.csv`: The report containing the domains which are newly-registered, but
  whose IPs were on the SNORT blocklist. This report will only be generated once per week, as the Censys data is only
  updated weekly on Google BigQuery.
- `[YYYY-MM-DD]/expired_certificates_plus_6m.csv`: The report containing certificates associated with army.mil that are
  either expired or will expire within 6 months. You won't see this very often, as the certificates are only fetched
  every 90 days.

## AWS Permissions

To do all of these processes, a series of permissions on AWS must be established:

- **EC2 S3 Access Permissions:** Permissions must be given to the EC2 instance to access the S3 bucket. We gave our EC2
  the "AmazonS3FullAccess" policy. This allowed our EC2 instance to write to and read from the S3 bucket without having
  to pass it an Access key ID or a Secret access key.
- **Lambda EC2 Start/Stop Permissions:** The Lambda functions we created needed permission to start and stop the EC2
  instance, so we needed to implement this. For information on how to do this, see
  [AWS's guide to starting/stopping EC2 instances with Lambda](https://aws.amazon.com/premiumsupport/knowledge-center/start-stop-lambda-cloudwatch/)
  .
