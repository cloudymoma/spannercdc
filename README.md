# One Workaround for Cloud Spanner Change Log Capture
## Briefing

The workaround uses Cloud Spanner's commit timestamp feature to capture change log.

It works in pull mode, to query the changed data from Cloud Spanner, deliver the changed data to Cloud Pub/sub; then the streaming dataflow job deals with the changed data, and stores in Cloud store in AVRO file format.

## Sample codes 

Samples codes are only for the purpose of demo.

## How to run the demo

1.Create table, and Create a TIMESTAMP column with the column option allow_commit_timestamp set to true in the schema definition;
```$xslt
CREATE TABLE spez_poller_table (
	ID INT64 NOT NULL,
	Color STRING(MAX),
	Name STRING(MAX),
	Timestamp TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
) PRIMARY KEY (ID)
```
2.Create Cloud Pub/Sub topic and subscription, the name is table name;

3.Create Cloud Storage bucket;

4.Create dataflow streaming job to move data in Cloud Pub/Sub to Cloud Storage;

4.1.Get the template of PubsubToAvro from github;

git clone https://github.com/GoogleCloudPlatform/DataflowTemplates

4.2.Please modify PubsubToAvro.java to support read from subscription instead of topic

```$xslt
//change line 181
181 PubsubIO.readMessagesWithAttributes().fromTopic(options.getInputTopic()))
//To
181 PubsubIO.readMessagesWithAttributes().fromSubscription(options.getInputSubscription()))
```

```$xslt
//Add the following part in the descripion

@Description(
        "The Cloud Pub/Sub subscription to consume from. "
            + "The name should be in the format of "
            + "projects/<project-id>/subscriptions/<subscription-name>.")
    ValueProvider<String> getInputSubscription();

    void setInputSubscription(ValueProvider<String> value);
```

4.3.Submit the PubsubToAvro job;

The job runs in time window,reading the change log from Cloud Pub/sub subscription and delivering it to Cloud store.
```$xslt
  ~/Downloads/apache-maven-3.6.3/bin/mvn compile exec:java \
 -Dexec.mainClass=com.google.cloud.teleport.templates.PubsubToAvro \
 -Dexec.cleanupDaemonThreads=false \
 -Dexec.args=" \
 --project=abc-learning-centre \
 --tempLocation=gs://abc-dump/tmp \
 --runner=DataflowRunner \
 --windowDuration=5m \
 --numShards=1 \
 --inputSubscription=projects/abc-learning-centre/subscriptions/spez_poller_table \
 --outputDirectory=gs://abc-dump/spez_poller_table/ \
 --avroTempDirectory=gs://abc-dump/tmp/"
```
5.Execute Java change log capture application;

5.1.Set up the configuration file;
```$xslt
spez {
  avroNamespace="spez"
  instanceName="testing"
  dbName="spez_poller_db"
  tableName="spez_poller_table"
  pollRate=10000
  recordLimit="200"
  startingTimestamp="2020-03-03T08:07:31.170731713Z"
  publishToPubSub=true
}

StartingTimestamp is the lastProcessedTimestamp
```
In the root directory, execute the following command 
```$xslt
mvn compile exec:java -Dexec.mainClass=com.google.spez.Spez

or

mvn package assembly:single
java -jar xxx.jar
```

6.Simulate data DML operation
```$xslt
Insert table(id,color,name,timestamp) values(1,'Red','Tom',current_timestamp());
```
Check the console log and dataflow job log, as well as check the Cloud store bucket.

7.Notes.

This workaround does not deal with delete operation.

This workaround is originally from [spanner-event-export](https://github.com/GoogleCloudPlatform/spanner-event-exporter), and dataflow job is added to deal with change log.






