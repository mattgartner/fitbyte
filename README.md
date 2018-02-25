# FitByte Demo

Demo for an IoT Data Pipeline on Google Cloud Platform (GCP) using: 
  1. Google Compute Engine
  2. Google Cloud Pub/Sub
  3. Google Cloud IoT Core
  4. Google Cloud Dataflow
  5. Google BigQuery
  6. Google Cloud Storage
  
## Demo Setup via Google Cloud SDK 
### Create Project
```
gcloud projects create iot-data-pipeline-1
```
### Set project
```
gcloud config set project iot-data-pipeline-1
```
Enable billing on the Cloud Console Dashboard

### Enable APIs
```
gcloud services enable cloudiot.googleapis.com --async
gcloud services enable pubsub.googleapis.com --async 
gcloud services enable dataflow.googleapis.com --async
```
### Confirm APIs enabled
```
gcloud services list
```
May take a few minutes for each API to enable and show up

### Create a Pub/Sub Topic
```
gcloud pubsub topics create fitbyte
```

### Create a BigQuery dataset (switch to GUI)
1. Create new dataset
2. Name it fitbyte
3. Location: US
4. Create empty table
5. Table name: metrics
6. "Edit as Text" under Schema, paste: 
```
user_id:STRING,calories_burned:FLOAT,calories_consumed:FLOAT,sleep_hours:FLOAT,water_consumed:FLOAT,steps:FLOAT,distance:FLOAT,bmi:FLOAT,heart_rate:FLOAT,weight:FLOAT,timestamp:TIMESTAMP,device:STRING
```

### Create Cloud Storage Bucket
```
gsutil mb -p iot-data-pipeline-1 -c multi_regional -l US gs://fitbyte/
```

### Create device registry in IoT Core
```
gcloud iot registries create iot-devices \
    --project=iot-data-pipeline-1 \
    --region=us-central1 \
    --event-notification-topic=projects/iot-data-pipeline-1/topics/fitbyte
```

### Create a Dataflow Pipeline (switch to GUI)
1. Create job from template
2. Name: fitbyte-data
3. Region: us-central1
4. Cloud Dataflow template: PubSub to BigQuery
5. Cloud Pub/Sub input topic: projects/iot-data-pipeline-1/topics/fitbyte
6. BigQuery output table: iot-data-pipeline-1:fitbyte.metrics
7. Temporary Location: gs://fitbyte/tmp/
8. Max-workers: 2
9. Machine-type: n1-standard-1
10. Run Job

### Create a compute instance for simulated IoT devices
```
gcloud compute instances create iot-simulator 
```

### SSH into VM and perform manual commands:
```
sudo apt-get update
sudo apt-get install python-pip openssl git google-cloud-sdk -y
sudo pip install pyjwt paho-mqtt cryptography
git clone https://github.com/mattgartner/fitbyte.git
export PROJECT_ID=iot-data-pipeline-1
export MY_REGION=us-central1
export REGISTRY=iot-devices
cd $HOME/fitbyte/
openssl req -x509 -newkey rsa:2048 -keyout rsa_private.pem \
    -nodes -out rsa_cert.pem -subj "/CN=unused"
gcloud init
gcloud iot devices create fitbit-1 \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=$REGISTRY \
  --public-key path=rsa_cert.pem,type=rs256
gcloud iot devices create fitbit-2 \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=$REGISTRY \
  --public-key path=rsa_cert.pem,type=rs256
gcloud iot devices create fitbit-3 \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=$REGISTRY \
  --public-key path=rsa_cert.pem,type=rs256
gcloud iot devices create apple-1 \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=$REGISTRY \
  --public-key path=rsa_cert.pem,type=rs256
gcloud iot devices create apple-2 \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=$REGISTRY \
  --public-key path=rsa_cert.pem,type=rs256
gcloud iot devices create apple-3 \
  --project=$PROJECT_ID \
  --region=$MY_REGION \
  --registry=$REGISTRY \
  --public-key path=rsa_cert.pem,type=rs256
```



## Run Simulated Devices
```
cd $HOME/fitbyte/
wget https://pki.google.com/roots.pem
python cloudiot_mqtt_example_json.py \
    --project_id=$PROJECT_ID \
    --registry_id=$REGISTRY \
    --device_id=fitbit-1 \
    --private_key_file=rsa_private.pem \
    --message_type=event \
    --algorithm=RS256 > fitbit-1-log.txt 2>&1 &
    
python cloudiot_mqtt_example_json.py \
    --project_id=$PROJECT_ID \
    --registry_id=$REGISTRY \
    --device_id=fitbit-4 \
    --private_key_file=rsa_private.pem \
    --message_type=event \
    --algorithm=RS256
```

## Demo Walkthrough

### Upload Cloud Storage csv
1. Dataflow template name: storage-test
2. Cloud Dataflow Template: GCS Text to Cloud PubSub
3. Input Cloud Storage Files: gs://fitbyte/upload/*.csv
4. Output Pub/Sub Topic: projects/iot-data-pipeline-1/topics/fitbyte
5. Temporary Location: gs://fitbyte/tmp/
