# README 

Demos of streaming data into BigQuery using pyspark jobs. 
- The first demo is running locally on a textSocketStream
- The second demo runs on dataproc


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

In the second demo, we are going to read dummy sensory data from pubsub.
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

 # TODO find the pub, of the empty pubsub message 

### Run the the script to generate fake message 

``` bash
source env/bin/activate
python publish.py 
```

### DEMO notes
Build a batch pipeline (SAP CSV on GCS)
    Explore [ ] hamburger menu: (SDK) 
                Portable (SDK, management)
                pipelines, studio, wrangler & metadata 
                Explore local data export
    
            SAP export
                Deploy
                Clean (missing values, filtering)
                JavaScript 
                Join
                JDBC connector 
            BigQuery
                Count the join

Deploying a pipeline
    Show the summary 
    Show the pipelines

Real-time
    Time series features
    Error cleaning
    Aggregate windows  [30sec]
    Metadata 

# Clean up 

```bash
bq rm streaming_dataset
```