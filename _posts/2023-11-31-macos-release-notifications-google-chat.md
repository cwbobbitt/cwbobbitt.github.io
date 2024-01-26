---
title: macOS Update Release Notifications in Google Chat
date: 2024-01-22 11:00:00 -0500
categories: [macos, google workspace ]
tags: [macos, software updates, google chat, google workspace]
---

## Introduction

Those reading this post may already be receiving notifications about Apple Releases through the Apple securityannounce mailing list or Apple Developer RSS feed. Maybe you're the only member of your team receiving these notifications and you want to easily keep the rest of your team up to date on these releases. This was exactly my scenario, but I originally struggled to find a solution as I didn't want to require my colleagues to comb through the securityannouce emails of non macOS updates and I couldn't find an already existing Google Chat integration for the Apple Developer RSS feed. With all this in mind, I set off to discover how I could automate notifications for macOS updates being released through a Google Chat alert.

## End Result

To start off, here's what the end result will look like once everything has been configured.

![image-title-here](/assets/images/macos-update-notifications-chat/example.png){:width="75%"} 

## Requirements
- Google Cloud Project - this was all configured within a Google Cloud project, but you can implement the same logic elsewhere
    - The following GCP services are used:
        - **BigQuery**: table to store known macOS release versions
        - **Cloud Run**: executes image containing python script
        - **Artifact Registry**: stores container image
        - **Cloud Scheduler**: runs Cloud Run job on defined schedule
        - **Secrets Manager**: stores service account credentials
- [Google Chat Webhook URL](https://developers.google.com/chat/how-tos/webhooks#register_the_incoming_webhook)

## What's Happening?
I wanted to learn more on containers and container deployment, so I decided to go the route of using Artifact Registry and Cloud Run. The Cloud Run job is scheduled to run every 6 hours to parse the known updates listed at [https://gdmf.apple.com/v2/pmv](https://gdmf.apple.com/v2/pmv). After the first initial run to populate data to your BigQuery table, if an update is found that doesn't exist in the BigQuery table, the table is updated and a wehbook call is made to send a chat notification. 

It's worth noting that upon first run, you will receive a chat notification for each previous macOS release in a single thread. So prepare yourself!

The three files downloaded as part of the provided script below are the main python script, Docker file, and requirements file. 

### Python Script
The main.py file contains everything necessary to parse the data from gdmf.apple.com/v2/pmv to determine all available macOS updates, store those within BigQuery, and send a chat webhook call if the update is not yet known.

```python
import requests
import json
import urllib3

from datetime import datetime
from google.cloud import bigquery
from google.oauth2 import service_account
from google.cloud import secretmanager

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

project_id = 
project_number = 
dataset_name = "macos_updates"
table_name = "updates"
secret_name = f'projects/{project_number}/secrets/service-account-json/versions/1'
webhook_url = 

# Access Secret to retrieve service account credentials
secret_client = secretmanager.SecretManagerServiceClient()
secret_request = secret_client.access_secret_version(name=secret_name)
service_info = json.loads(secret_request.payload.data.decode('UTF-8'))

# Create credentials from service account information
creds = service_account.Credentials.from_service_account_info(service_info)

# Build BQ client and define dataset/table
client = bigquery.Client(credentials=creds, project=project_id)
dataset_ref = client.dataset(dataset_name)
table_ref = dataset_ref.table(table_name)
table = client.get_table(table_ref) 

# Intialize updates list
updates = []

# Retrieve macOS updates from Apple. Can be switched to iOS if needed.
url="https://gdmf.apple.com/v2/pmv"
get_updates = requests.get(url, verify=False)
macOS=json.loads(get_updates.content)['AssetSets']['macOS']

# Query Big Query table for list of known updates
QUERY = (
    f""" SELECT * FROM `{project_id}.{dataset_name}.{table_name}`"""
    )
query_job = client.query(QUERY) 
rows = query_job.result() 
known_updates = rows.to_dataframe()

# Make webhook call to send chat
def send_chat(update, posting_date): 
    update_name = "macOS " + update
    release_date = datetime.strptime(posting_date, "%Y-%m-%d").strftime('%m/%d/%Y')
    data = {"cards": {
            'sections': [
                {
                'widgets': [
                    {
                    'keyValue': {
                        'topLabel': 'Title',
                        'content': update_name
                    }
                    },
                    {
                    'keyValue': {
                        'topLabel': 'Release Date',
                        'content': release_date
                    }
                    }
                ]
                }
            ],
            'header': {
                'title': 'Mac Update Detector',
                'subtitle': 'A new update has been released',
                'imageUrl': 'https://w7.pngwing.com/pngs/901/839/png-transparent-mac-mini-finder-macos-computer-icons-cool-miscellaneous-furniture-smiley.png',
                'imageStyle': 'IMAGE'
            }
            },
            "thread": {"threadKey": "macUpdates"}
            }
    requests.post(webhook_url, data=json.dumps(data), headers = {'Content-Type': 'application/json; charset=UTF-8'})

# Send new updates to BQ table
def update_bigquery(update):
    row_to_insert = [{'title': update}]
    client.insert_rows(table, row_to_insert)

# Parse all updates and check if in list of known updates. If not known, update BQ table and send chat.
for update in macOS:
    if update['ProductVersion'] not in known_updates['title'].values:
        #Add to bigquery array
        update_bigquery(update['ProductVersion'])
        #Make request to send chat webhook
        send_chat(update['ProductVersion'], update['PostingDate'])
        print("%s added to list of known updates and chat sent." % (update['ProductVersion']))
    else:
        print("%s is already a known update." % (update['ProductVersion']))
  ```

### Dockerfile
The Dockerfile contains the instructions/commands used to assemble the image.

```
FROM python:3.10
WORKDIR /app
COPY requirements.txt /app
RUN pip install --no-cache-dir -r requirements.txt
COPY . /app
CMD [ "python", "main.py" ]
```

### Requirements File
The requirements.txt file contains the list of needed Google libraries (and pandas)

```
db-dtypes
pandas
google-cloud-secret-manager
google-cloud-bigquery
google-api-python-client 
google-auth-httplib2 
google-auth-oauthlib
```

## GCP Steps

To make this easier for everyone, I've included everything necessary to "automate" almost all of the necessary steps. This can simply be pasted within your GCP console or you can save it as a local script and upload to GCP to execute.

```bash
project_id="" #Set a globally unique project id that is between 6 and 30 characters.
region="" #Set a region to deploy the various GCP resources in. See https://cloud.google.com/build/docs/locations#restricted_regions_for_some_projects for possible limitations for Cloud Build. May be easiest to set in region defined from that link.
webhook_url="" #Generate a webhook url within a Google Chat Space https://developers.google.com/chat/how-tos/webhooks#create_a_webhook
billing_account_id="" #Billing Account ID that the project will be associated to

#Create Project
gcloud projects create $project_id --name="macOS Update Detector Bot" 

#Set working project
gcloud config set project $project_id 

#Grab project number to be used later within secret
project_number=$(gcloud projects describe $project_id --format="value(projectNumber)") 

#Link billing account to project
gcloud billing projects link $project_id --billing-account $billing_account_id 

#Enable the required services 
gcloud services enable bigquery.googleapis.com secretmanager.googleapis.com cloudscheduler.googleapis.com run.googleapis.com cloudscheduler.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com 

#Create service account to be used across various services
gcloud iam service-accounts create macos-detector-service --display-name="macOS Detector Service Account" 

#Grant service account Secret Accessor, BigQuery Data Editor, and BigQuery Job User
gcloud projects add-iam-policy-binding $project_id \
--member="serviceAccount:macos-detector-service@${project_id}.iam.gserviceaccount.com" \
--role="roles/secretmanager.secretAccessor"
gcloud projects add-iam-policy-binding $project_id \
--member="serviceAccount:macos-detector-service@${project_id}.iam.gserviceaccount.com" \
--role="roles/bigquery.dataEditor"
gcloud projects add-iam-policy-binding $project_id \
--member="serviceAccount:macos-detector-service@${project_id}.iam.gserviceaccount.com" \
--role="roles/bigquery.jobUser"

#Generate key.json to be used for auth
gcloud alpha iam service-accounts keys create key.json --iam-account="macos-detector-service@${project_id}.iam.gserviceaccount.com"

#Create secret from key.json content
gcloud secrets create service-account-json --replication-policy="user-managed" --data-file="key.json" --location=$region

#Delete key.json
rm key.json

#Create main macos_updates dataset and updates table
gcloud alpha bq datasets create macos_updates --description 'macOS Updates'
gcloud alpha bq tables create updates --dataset macos_updates --description 'Updates' --schema=title=STRING

#Create directory for main.py, Dockerfile, and requirements.txt
mkdir update-detector-files
cd update-detector-files

#Download files
curl -LJO https://github.com/cwbobbitt/macos-update-webhook-alerts/blob/main/Dockerfile
curl -LJO https://github.com/cwbobbitt/macos-update-webhook-alerts/blob/main/requirements.txt
curl -LJO https://github.com/cwbobbitt/macos-update-webhook-alerts/blob/main/main.py

#Replace project specific variables in main.py
sed -i '/project_id =/ s/$/ "'${project_id}'"/' main.py
sed -i '/project_number =/ s/$/ "'${project_number}'"/' main.py
sed -i '/webhook_url =/ s/$/ "'${webhook_url}'"/' main.py

#Create Artifact Registry Repo
gcloud artifacts repositories create updates-repo --repository-format=docker --location=$region

#Submit Cloud Build
gcloud builds submit --region=$region --tag "${region}-docker.pkg.dev/${project_id}/updates-repo/get-macos-updates-image:tag1"

#Create Cloud Run Job
gcloud run jobs create get-macos-updates --image "${region}-docker.pkg.dev/${project_id}/updates-repo/get-macos-updates-image:tag1" --service-account="macos-detector-service@${project_id}.iam.gserviceaccount.com" --region=$region

#Schedule Cloud Run Job to run on specified interval. Every 6 hours used below, see  <> for more information
gcloud scheduler jobs create http get-macos-updates-scheduler \
  --location=$region \
  --schedule="0 */6 * * *" \
  --uri="https://${region}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/${project_id}/jobs/get-macos-updates:run" \
  --http-method POST \
  --oauth-service-account-email "macos-detector-service@${project_id}.iam.gserviceaccount.com"

#Execute Job for first time
gcloud run jobs execute get-macos-updates-scheduler --region=$region

#Clean up. Feel free to comment out to retain files for testing. This in theory should work as is, but nothing is perfect.
rm -rf update-detector-files
```
## Cost
During initial build-out and testing, I did incur a few dollars in cost due to not knowing what I was doing, but after ironing out all the kinks and half knowing what I'm doing now, I dropped it down to roughly $0.20 a month. 

As of writing this, I discovered that this can most likely be knocked down to no cost as I was still storing a few extra images in my repo that bumped me over the free 0.5GB limit, which can be deleted, and a $0.06 monthly charge for Secrets Manager, which I think could be eliminated by switching to a single user-managed replication policy vs automatic. 

The gcloud commands above reflect the user-managed replication policy change, so you should be safe to consider this solution free!

## Closing Thoughts

Is this a lot just to achieve notifications for macOS releases? Probably. Does it get the job done? Yes. Currently, this is just going to tell you general release information, but I hope to expand it in the future to also include what devices support the update.

If you have any questions or suggestions, feel free to shoot me a message over on the <a href="https://macadmins.slack.com/team/U011JUC8SLE" target="_blank">MacAdmins Slack</a>.