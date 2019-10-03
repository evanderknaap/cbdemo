# README 

Demos of streaming data into BigQuery using pyspark jobs and Data Fusion. 
- The first demo is running locally on a textSocketStream
- The second demo runs on dataproc
- The third demo runs on DataFusion

## Demo 1

### Setup 
Create a dataset, to hold your data
```bash
bq --location=eu mk streaming_dataset
```

Next go, to the console and create a table, called "tableA" which
has a column "id" of type integer, and "text" forof type String.

### Running the demo
Start a stream on TCP socket 9999, using netcat server.
```bash
$ nc -lk 9999
```

Next, we are going to stream to BigQuery, using the *client.insertall* method.
This uses the [tabledata.insertall](https://cloud.google.com/bigquery/docs/
reference/rest/v2/tabledata/insertAll) API. This is limited to 10K rows, per 
client. We can play with the number of partitions, to manage the amount
of rows written by each executor.

Run the pyspark locally 
```bash
$ python streaming_app.py
```

Alternatively, we can do a spark submit.
```bash
spark-submit streaming_app.py
```

Each microbatch, a client is created (or reused) for each partition of the RDD
and the results written to BigQuery. You can play with
- The *microbatch* in the stream context, to see that every microbatch data is sent
- The *repartition* method, to change the amount of threads writing to BigQuery

## Demo 2 

In the second demo, we are going to read dummy sensor data from pubsub.
We will create this dummy data with a [faker](https://github.com/joke2k/faker)
python scripts, which we will run locally. 

With a second python (pyspark) scripts, we are going to parse these message
and stream to bigquery. For this we will create a second table, *tableB*, 
with the following schema.

### Setup 

#### Creating a pubsub topic and subscription 
```bash
gcloud pubsub topics create sensors
```
Create a subscription
```bash
gcloud pubsub subscriptions create sub --topic=sensors
```
Next, navigate to the pubsub menu. Select *topics*, select *sensors* and then
click *+publish message*. Type "hello world" and click publish. We'll use this
for testing later. 

#### Setting up the connector
To stream messages from pubsub in pyspark, we'll use a [connector](https://github.com/SignifAi/Spark-PubSub) 
We need to create a jar file, and python egg file to pass as dependencies. Clone the git repo, and cd in the directory. 

```bash
git clone https://github.com/SignifAi/Spark-PubSub.git
cd spark-pubsub
```
 Build the Java file

 ```bash
 cd java
 mvn clean install
 ```

 build the Python file
 ```bash
 cd..
 cd python
 python setup.py bdist_egg
 ```

 Export environment variables
 ```bash
 export SPARK_PUBSUB_JAR="/Users/evanderknaap/Desktop/spark-pubsub/java/target/spark_pubsub-1.1-SNAPSHOT.jar"
 export SPARK_PUBSUB_PYTHON_EGG="/Users/evanderknaap/Desktop/spark-pubsub/python/dist/spark_pubsub-1.0.0-py2.7.egg"
 ```

Run the example
 ```bash
spark-submit --jars ${SPARK_PUBSUB_JAR} --driver-class-path ${SPARK_PUBSUB_JAR} --py-files \
${SPARK_PUBSUB_PYTHON_EGG} sensor_streaming.py
 ```
TODO: fix pubsub error

## Demo 3 -  Data Fusion

- Create a new Data Fusion instance
- Create a bucket bucket in the US, with a checkpointing folder

#### Deploy the pipeline
- Navigate to the pipeline studio and hit the green, *+* button
- Select *import pipeline*
- Upload *stream_pipeline-cdap-data-streams.json* from the project folder
- Select *fix all*
- Check that the right topic and subscriptions are set in the pubsub plugin 
- In the pipeline config settings, set the checkpoint folder
- In the BigQuery plugins, make sure to set the temporary golder to the checkpoints folder
- Deploy the pipeline 

#### Run the the script to generate fake message 

``` bash
source env/bin/activate
python publish.py 
```
## Demo 4 - Kafka to BigQuery 

In this demo, we create a Kafka cluster on DataProc and generate fake sensor data. 
On a second dataproc cluster we deploy a pyspark application that parses the data and streams it into BigQuery.

#### Setup 
Set environment variables
```
PROJECT=<your-project>
BUCKET=<your-bucket-name>
CLUSTER=<your-cluster-name>
```

Create a DataProc Cluster with 3 master nodes, running Kafka. Note the initialization script that is passed
to install Kafka on the nodes.
```bash
gcloud beta dataproc clusters create $CLUSTER \
    --enable-component-gateway --region europe-west1 --subnet default --zone "" --num-masters 3 \
    --master-machine-type n1-standard-2 --master-boot-disk-size 500 --num-workers 2 \
    --worker-machine-type n1-standard-2 --worker-boot-disk-size 500 --image-version 1.4-debian9 \
    --optional-components ANACONDA,JUPYTER --scopes 'https://www.googleapis.com/auth/cloud-platform' \
    --project $PROJECT --initialization-actions 'gs://dataproc-initialization-actions/kafka/kafka.sh'
```

From your local terminal, list the names of the worker nodes and note the name. We need it later.
``` bash
gcloud dataproc clusters describe $CLUSTER --region europe-west1 | grep w-
```

Navigate **dataproc > cluster > vm-instance** in the google cloud console. Click the SSH button next to any master node to create a topic. 
```bash
kafka-topics.sh --zookeeper localhost:2181 --create --replication-factor 1 --partitions 1 --topic test 
```
From within the same window, create an environment variable for one of the worker node names.
```bash
WORKER=<worker-name>
```
Generate random data and add it to the topuc. We create a comma seperate string of a device_id, event_time timestamp, and two random numbers that mimic a sensor reading. 

```bash
  for i in {1..10000}; 
          do for i in {1..3}; 
                  do timestamp=$(date +%s);
                  echo "device_${i},${timestamp},$(($RANDOM%40+1)),$(($RANDOM%10+1))";
          done;
          sleep 1s; 
   done | /usr/lib/kafka/bin/kafka-console-producer.sh --broker-list $WORKER:9092 --topic test
```
Navigate **dataproc > cluster > vm-instance** in the console. Create a second SSH tunnel on any of the nodes.
Store the name of a worker.
```bash
WORKER=<your-worker-name>
```
Check if the messages exist. If all goes well, you should see messages flowing in.
```bash
/usr/lib/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server $WORKER:9092 \
    --topic test --from-beginning
```
#### Kafka stream into Spark on DataProc
Next, lets's see if we can get messages incoming into our pyspark application. We are going to deploy a pyspark application, that prints 10 records, every 5 seconds. 
``` Bash
gcloud dataproc jobs submit pyspark --cluster=$CLUSTER\
    --region europe-west1\
    --properties spark.jars.packages=org.apache.spark:spark-streaming-kafka-0-8_2.11:2.0.1\
    sensor_streaming.py     
```

Grab the Job_id of the running spark job, and copy it. 
```bash
gcloud dataproc jobs list --region=europe-west1
```
Paste the Job_ID in the follow command to kill it
```bash
gcloud dataproc jobs kill <JOB_ID> --region=europe-west1
```
#### Streaming to BigQuery
Next, let's deploy a second cluster to run our pyspark job, called **spark**. Since we are using the Python SDK wrapper around the BigQuery **table.Insertall** API, we will add the bigquery library and intall it using Pip.
One can also use conda to install packages. However, the channels used in the startup script do not include the google cloud SDK, so we'll use Pip instead.

```bash
gcloud dataproc clusters create spark\
    --optional-components ANACONDA\
    --metadata 'PIP_PACKAGES=google-cloud-bigquery'\
    --region europe-west1\
    --initialization-actions gs://dataproc-initialization-actions/python/pip-install.sh
```
Next, submit our job to our fresh cluster.
``` Bash
gcloud dataproc jobs submit pyspark --cluster=spark\
    --region europe-west1\
    --properties spark.jars.packages=org.apache.spark:spark-streaming-kafka-0-8_2.11:2.0.1\
    sensor_stream_to_bigquery.py     
```
If all goes well we should have data streaming in.

## TODO Clean up 

```bash
bq rm streaming_dataset
dataproc clusters delete $CLUSTER --region europe-west1
```