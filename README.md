# README 

Demos of streaming data into BigQuery using pyspark jobs and Data Fusion. 
- The first demo is running locally on a textSocketStream
- The second demo runs on dataproc
- The third demo runs on DataFusion

## Setup
- Create a pubsub topuc and subscription, note down the names
- Create a new Data Fusion instance
- Create a bucket bucket in the US, with a checkpointing folder

## Deploy the pipeline
- Navigate to the pipeline studio and hit the green, *+* button
- Select *import pipeline*
- Upload *stream_pipeline-cdap-data-streams.json* from the project folder
- Select *fix all*
- Check that the right topic and subscriptions are set in the pubsub plugin 
- In the pipeline config settings, set the checkpoint folder
- In the BigQuery plugins, make sure to set the temporary golder to the checkpoints folder
- Deploy the pipeline 

### Run the the script to generate fake message 

``` bash
source env/bin/activate
python publish.py 
```

## Clean up (TODO)
