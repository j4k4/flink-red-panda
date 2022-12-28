# Apache Flink meets Red Panda
 
I've created this repository in the context of testing the interoperability of Apache Flink and Red Panda. 
In some sense, it is also summarizes the capabilities of Apache Flink's Apache Kafka Connector. 

## Environment Setup
This starts a local environment running consisting of Red Panda, a Flink Cluster and a SQL Client to submit Flink SQL queries to the Flink Cluster.
```
docker-compose up -d
docker-compose run sql-client
```
## Writing Exactly-Once to Red Panda with Transactions ~~(fails)~~

```sql
/* enable checkpointing on the Flink side */
SET 'execution.checkpointing.interval' = '100ms';

/* Test Data Generator */ 
CREATE TABLE animal_sightings (
  `timestamp` TIMESTAMP(3),
   `name` STRING,
   `country` STRING,
   `number` INT
) WITH (
  'connector' = 'faker', 
  'fields.name.expression' = '#{animal.name}',
  'fields.country.expression' = '#{country.name}',
  'fields.number.expression' = '#{number.numberBetween ''0'',''1000''}',
  'fields.timestamp.expression' = '#{date.past ''15'',''SECONDS''}'
);

CREATE TABLE animal_sightings_panda_txn (
   `timestamp` TIMESTAMP(3) METADATA,
   `name` STRING,
   `country` STRING,
   `number` INT
) WITH (
   'connector' = 'kafka',
   'topic' = 'animal_sightings_txn',
   'properties.bootstrap.servers' = 'redpanda:29092',
   'format' = 'json', 
   'scan.startup.mode' = 'latest-offset',
   'sink.delivery-guarantee' = 'exactly-once',
   'sink.transactional-id-prefix' =  'flink',
   'properties.max.in.flight.requests.per.connection'='1'
);

INSERT INTO animal_sightings_panda_txn SELECT * FROM animal_sightings;

/* See if it works by reading from the Red Panda topic */
SELECT * FROM animal_sightings_panda_txn;
```

### check that the job is running 
- Checking the Flink UI under https://localhost:8081 will now show that the Job is working correctly:
  - there should be no exceptions
  - you should see data being written
  - you should see successful checkpoints

### enforce a recovery from checkpoint
- Run `docker-compose restart taskmanager` to restart the taskmanger. This will also restart and recover the job.
- The job should be switching its state between running to restarting all the time. Unable to recover and throws exceptions like. You should see this in the exception overview in the Flink UI of the job.
```
java.lang.IllegalStateException: Failed to commit KafkaCommittable{producerId=282, epoch=0, transactionalId=flink-0-274}
	at org.apache.flink.streaming.runtime.operators.sink.committables.CommitRequestImpl.signalFailedWithUnknownReason(CommitRequestImpl.java:77)
	at org.apache.flink.connector.kafka.sink.KafkaCommitter.commit(KafkaCommitter.java:119)
	at org.apache.flink.streaming.runtime.operators.sink.committables.CheckpointCommittableManagerImpl.commit(CheckpointCommittableManagerImpl.java:126)
	at org.apache.flink.streaming.runtime.operators.sink.CommitterOperator.commitAndEmit(CommitterOperator.java:176)
	at org.apache.flink.streaming.runtime.operators.sink.CommitterOperator.commitAndEmitCheckpoints(CommitterOperator.java:160)
	at org.apache.flink.streaming.runtime.operators.sink.CommitterOperator.initializeState(CommitterOperator.java:121)
	at org.apache.flink.streaming.api.operators.StreamOperatorStateHandler.initializeOperatorState(StreamOperatorStateHandler.java:122)
	at org.apache.flink.streaming.api.operators.AbstractStreamOperator.initializeState(AbstractStreamOperator.java:283)
	at org.apache.flink.streaming.runtime.tasks.RegularOperatorChain.initializeStateAndOpenOperators(RegularOperatorChain.java:106)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.restoreGates(StreamTask.java:726)
	at org.apache.flink.streaming.runtime.tasks.StreamTaskActionExecutor$1.call(StreamTaskActionExecutor.java:55)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.restoreInternal(StreamTask.java:702)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.restore(StreamTask.java:669)
	at org.apache.flink.runtime.taskmanager.Task.runWithSystemExitMonitoring(Task.java:935)
	at org.apache.flink.runtime.taskmanager.Task.restoreAndInvoke(Task.java:904)
	at org.apache.flink.runtime.taskmanager.Task.doRun(Task.java:728)
	at org.apache.flink.runtime.taskmanager.Task.run(Task.java:550)
	at java.base/java.lang.Thread.run(Unknown Source)
Caused by: org.apache.flink.kafka.shaded.org.apache.kafka.common.KafkaException: Unhandled error in EndTxnResponse: The server experienced an unexpected error when processing the request.
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.producer.internals.TransactionManager$EndTxnHandler.handleResponse(TransactionManager.java:1646)
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.producer.internals.TransactionManager$TxnRequestHandler.onComplete(TransactionManager.java:1322)
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.ClientResponse.onComplete(ClientResponse.java:109)
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.NetworkClient.completeResponses(NetworkClient.java:583)
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.NetworkClient.poll(NetworkClient.java:575)
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.producer.internals.Sender.maybeSendAndPollTransactionalRequest(Sender.java:418)
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.producer.internals.Sender.runOnce(Sender.java:316)
	at org.apache.flink.kafka.shaded.org.apache.kafka.clients.producer.internals.Sender.run(Sender.java:243)
	... 1 more
```