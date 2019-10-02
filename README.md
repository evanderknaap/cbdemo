# README 

Demos of streaming data into BigQuery using pyspark jobs. 
- The first demo is running locally on a textSocketStream
- The second demo runs on dataproc

## Setup 
Create a dataset, to hold your data
```bash
bq --location=eu mk streaming_dataset
```

Next go, to the console and create a table, called "tableA" which
has a column "id" of type integer, and "text" forof type String.

## Demo 1
Start a stream on TCP socket 9999, using netcat server.
```bash
$ nc -lk 9999
```

Next, we are going to stream to BigQuery, using the *client.insertall* method.
This uses the [tabledata.insertall](https://cloud.google.com/bigquery/docs/reference/rest/v2/tabledata/insertAll) API. This is limited to 10K rows, per client.

Run the pyspark locally 
```bash
$ python streaming_app.py
```

Each microbatch, a client is created (or reused) for each partition of the RDD
and the results written to BigQuery. You can play with
- The *microbatch* in the stream context, to see that every microbatch data is sent
- The *repartition* method, to change the amount of threads writing to BigQuery

## SETUP 
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