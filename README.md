# Distributed Image Processing in Cloud Dataproc
## Overview
The goal of this project is to use Apache Spark on Cloud Dataproc to run a simple facial recognition model on a few images using distributed computing. These images are stored in a Cloud Storage Bucket, and when processed, a new folder with the outputted images will be generated. This project was done as a demo of distributed computing for Noah Gift's Cloud Computing Course at Duke University. 

Reference Qwiklab: [Distributed Image Processing in Cloud Dataproc](https://www.qwiklabs.com/focuses/5834?catalog_rank=%7B%22rank%22%3A7%2C%22num_filters%22%3A0%2C%22has_search%22%3Atrue%7D&parent=catalog&search_id=4914974)


## Procedure
### Part 1: Create Development Machine in Compute Engine
1. Go to **Compute Engine** in the GCP Console, selecting **VM Instances**, then select **Create**.
2. Configure the fields below and leave the rest as default.
    a. **Name**: devhost (or whatever you prefer)
    b. **Series**: N1
    c. **Machine Type**: 2 vCPUs (n1-standard-2 instance)
    d. **Identity and API Access**: Allow full access to all Cloud APIs
    
### Part 2: Install Software

#### Step 1: Set up Scala and sbt

```
sudo apt-get install -y dirmngr unzip

sudo apt-get update

sudo apt-get install -y apt-transport-https

echo "deb https://dl.bintray.com/sbt/debian /" | \

sudo tee -a /etc/apt/sources.list.d/sbt.list

sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 642AC823

sudo apt-get update

sudo apt-get install -y bc scala sbt
```

#### Step 2: Set up the Feature Detector Files

```
sudo apt-get update

gsutil cp gs://spls/gsp124/cloud-dataproc.zip .

unzip cloud-dataproc.zip

cd cloud-dataproc/codelabs/opencv-haarcascade
```

#### Step 3: Launch build
```
sbt assembly
```

### Part 3: Create a Cloud Storage Bucket and Collect Images

#### Step 1: Create Bucket ID
```
# Obtain Project ID
GCP_PROJECT=$(gcloud config get-value core/project)

# Name bucket and create shell variables
MYBUCKET="${USER//google}-image-${RANDOM}"
echo MYBUCKET=${MYBUCKET}
```

#### Step 2: Create Bucket 
```
gsutil mb gs://${MYBUCKET}
```

#### Step 3: Download Sample Image to Bucket
```
curl https://www.publicdomainpictures.net/pictures/20000/velka/family-of-three-871290963799xUk.jpg | gsutil cp - gs://${MYBUCKET}/imgs/family-of-three.jpg

curl https://www.publicdomainpictures.net/pictures/10000/velka/african-woman-331287912508yqXc.jpg | gsutil cp - gs://${MYBUCKET}/imgs/african-woman.jpg

curl https://www.publicdomainpictures.net/pictures/10000/velka/296-1246658839vCW7.jpg | gsutil cp - gs://${MYBUCKET}/imgs/classroom.jpg
```

#### Step 4: See Contents of Bucket
```
gsutil ls -R gs://${MYBUCKET}
```

### Part 3: Create a Cloud Dataproc Cluster
#### Step 1: Create Cluster Name
```
# Create name and shell variable
MYCLUSTER="${USER/_/-}-qwiklab"
echo MYCLUSTER=${MYCLUSTER}
```

#### Step 2: Configure Cluster
```
# Configure region
gcloud config set dataproc/region us-central1

# Configure cluster settings
gcloud dataproc clusters create ${MYCLUSTER} --bucket=${MYBUCKET} --worker-machine-type=n1-standard-2 --master-machine-type=n1-standard-2 --initialization-actions=gs://spls/gsp010/install-libgtk.sh --image-version=2.0  
```

### Part 4: Submit Your Job to Cloud Dataproc
#### Step 1: Load Face Detection Configuration File
```
curl https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_frontalface_default.xml | gsutil cp - gs://${MYBUCKET}/haarcascade_frontalface_default.xml
```
#### Step 2: Submit the Job
```
cd ~/cloud-dataproc/codelabs/opencv-haarcascade

gcloud dataproc jobs submit spark \
--cluster ${MYCLUSTER} \
--jar target/scala-2.12/feature_detector-assembly-1.0.jar -- \
gs://${MYBUCKET}/haarcascade_frontalface_default.xml \
gs://${MYBUCKET}/imgs/ \
gs://${MYBUCKET}/out/
```

#### Step 3: Monitor Job and Inspect Bucket
1. To monitor job, go to **Navigation menu > Dataproc > Jobs**
2. To inspect the bucket, go to **Navigation menu > Storage** and find the bucket. Processed images will be in the **out/** directory
