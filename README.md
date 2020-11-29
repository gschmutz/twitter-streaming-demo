# Twitter Big Data & Fast Data Sample

## Ingesting Twitter Data into Kafka

In the first step, we will be using StreamSets Data Collector to consume live data from Twitter and publish the tweets retrieved into a Kafka topic.

![Alt Image Text](./images/use-case-step1.png "Use Case Step 1")

Navigate to <http://dataplatform:18630> to open StreamSets and create a new pipeline. 

Add a `HTTP Client` orgin and a `Kafka Producer` destination.

For the `HTTP Client` configure these properties:

* Tab **HTTP**
  * **Resource URL**: `https://stream.twitter.com/1.1/statuses/filter.json?track=trump,biden`
  * **Authentication Type**: `OAuth`
* Tab **Credentials**
  * Set **Consumer Key**, **Consumer Secret**, **Token** and **Token Secret** to the values gotten from the Twitter application
* Tab **Data Format**
  * **Data Format**: `JSON`
  * **Max object Length (chars)**: `409600`     

For the `Kafka Producer` configure these properties:

* Tab **Kafka**
  * **Broker URI**: `kafka-1:19092`
  * **Topic**: `tweet-json`
* Tab **Data Format**
  * **Data Format**: `JSON`  

Create the Kafka Topic using the command line

```
docker exec -ti kafka-1 kafka-topics --create --topic tweet-json --partitions 8 --replication-factor 3 --zookeeper zookeeper-1:2181
```

To check for message, let's start a console consumer using the `kafkacat` utility. We can either have `kafkacat` installed locally or use the containerized version which is part of the platform.

```
docker exec -ti kafkacat kafkacat -b kafka-1:19092 -t tweet-json
```

Start the pipeline in StreamSets.

## Moving Data from Kafka to S3 compliant Object Storage

In the second step, we will be using StreamSets Data Collector to consume the data we have previously written to the Kafka topic and store it in minIO Object Storage (which is S3 compliant).

![Alt Image Text](./images/use-case-step2.png "Use Case Step 2")

Navigate to <http://dataplatform:18630> to open StreamSets and create a new pipeline. 

Add a `Kafka Consumer` orgin and a `Amazon S3` destination.

For the `Kafka Consumer` configure these properties:

* Tab **Kafka**
  * **Broker URI**: `kafka-1:19092`
  * **Consumer Group**: `TweetConsumerV1`
  * **Topic**: `tweet-json`
  * **Zookeeper URI**: `zookeeper-1:2181`
  * **Max Batch Size (records)**: `10000`
  * **Batch Wait Time (ms): `200000`
* Tab **Data Format**
  * **Data Format**: `JSON`
  * **Max object Length (chars)**: `409600`     

For the `Amazon S3` configure these properties (as the MinIO is part of the self-contained platform, we can put the Access key and Secret Access key here in the text):

* Tab **Amazon S3**
  * **Authentication Method**: `AWS Keys`
  * **Access Key ID**: `V42FCGRVMK24JJ8DHUYG`
  * **Secret Access Key**: `bKhWxVF3kQoLY9kFmt91l+tDrEoZjqnWXzY9Eza`
  * **Use Specific Region**: `X`
  * **Region**: `Other - specify`
  * **Endpoint**: `http://minio:9000`
  * **Bucket**: `tweet-bucket`
  * **Common Prefix**: `raw`
  * **Object Name Suffix**: `json`
  * **Delimiter**: `/`
  * **Use Path Style Address Model**: `X`
* Tab **Data Format**
  * **Data Format**: `JSON`

Before we can now start the pipeline, we have to create the bucket in MinIO object storage. We can either do that in the MinIO Browser or through the s3cmd available in the `awscli` service. 

```
docker exec -ti awscli s3cmd mb s3://tweet-bucket
```

Start the pipeline in StreamSets. As soon as there are `10000` tweets available in the Kafka topic, the first object should be written out to MinIO object storage. 

Navigate to the MinIO Browser on <http://dataplatform:9000> and check that the bucket is being filled.

## Processing Tweets using Apache Spark

In the third step, we will be using Apache Spark to process the tweets we have stored in object storage. We will be using Apache Zeppelin for executing the Apache Spark statements in an "ad-hoc" fashion.

![Alt Image Text](./images/use-case-step3.png "Use Case Step 3")

Navigate to <http://dataplatform:28080> to open Apache Zeppelin and login as user `admin` with password `abc123!`. Create a new notebook using the **Create new note** link. 

```
val tweets = spark.read.json("s3a://tweets/raw/")
```

```
tweets.printSchema
```

```
root
 |-- contributors: string (nullable = true)
 |-- coordinates: struct (nullable = true)
 |    |-- coordinates: array (nullable = true)
 |    |    |-- element: double (containsNull = true)
 |    |-- type: string (nullable = true)
 |-- created_at: string (nullable = true)
 |-- display_text_range: array (nullable = true)
 |    |-- element: long (containsNull = true)
 |-- entities: struct (nullable = true)
 |    |-- hashtags: array (nullable = true)
 |    |    |-- element: struct (containsNull = true)
 |    |    |    |-- indices: array (nullable = true)
 |    |    |    |    |-- element: long (containsNull = true)
 |    |    |    |-- text: string (nullable = true)
 |    |-- media: array (nullable = true)
 |    |    |-- element: struct (containsNull = true)
 |    |    |    |-- additional_media_info: struct (nullable = true)
 |    |    |    |    |-- description: string (nullable = true)
 |    |    |    |    |-- embeddable: boolean (nullable = true)
 |    |    |    |    |-- monetizable: boolean (nullable = true)
 |    |    |    |    |-- title: string (nullable = true)
 
 ...
```
 
```
tweets.createOrReplaceTempView("tweets")
```

```
 %sql
SELECT * FROM tweets
```

```
%sql
select count(*) from tweets
```
 
```
%sql
select text, entities.hashtags.text from tweets where size(entities.hashtags) > 0
```

```
%sql
select hashtag, count(*) nof from (
select lower(hashtag.text) as hashtag from tweets lateral view explode(entities.hashtags) hashtags as hashtag ) group by hashtag
order by nof desc
```



```
val resultDf = spark.sql("""
select hashtag, count(*) nof from (
select lower(hashtag.text) as hashtag from tweets lateral view explode(entities.hashtags) hashtags as hashtag ) group by hashtag
order by nof desc
""")
```

```
resultDf.write.parquet("s3a://tweet-bucket/result/hashtags")
```

  
## Twitter Source

```bash
TWITTER_CONSUMERKEY=
TWITTER_CONSUMERSECRET=
TWITTER_ACCESSTOKEN=
TWITTER_ACCESSTOKENSECRET=
```


```sql
CREATE SOURCE CONNECTOR SOURCE_TWITTER_01 WITH (
    'connector.class' = 'com.github.jcustenborder.kafka.connect.twitter.TwitterSourceConnector',
    'twitter.oauth.accessToken' = '${file:/data/credentials.properties:TWITTER_ACCESSTOKEN}',
    'twitter.oauth.consumerSecret' = '${file:/data/credentials.properties:TWITTER_CONSUMERSECRET}',
    'twitter.oauth.consumerKey' = '${file:/data/credentials.properties:TWITTER_CONSUMERKEY}',
    'twitter.oauth.accessTokenSecret' = '${file:/data/credentials.properties:TWITTER_ACCESSTOKENSECRET}',
    'kafka.status.topic' = 'twitter_01',
    'process.deletes' = false,
    'filter.keywords' = 'devrel,apachekafka,confluentinc,ksqldb,kafkasummit,kafka connect,rmoff,tlberglund,gamussa,riferrei,nehanarkhede,jaykreps,junrao,gwenshap'
);
```

```
docker exec -ti hive-metastore hive
```

Create the database and switch to the database

```
CREATE DATABASE twitter_data;
USE twitter_data;
```


Create the Hive Table

```
DROP TABLE IF EXISTS hashtag_count_t;
CREATE EXTERNAL TABLE hashtag_count_t (hashtag string
									 , nof integer)
STORED AS PARQUET LOCATION 's3a://tweets/hashtags';  
```




```
docker exec -ti presto-cli presto --server presto-1:8080 --catalog minio
```

```
use minio.twitter_data;

show tables;
```

```
SELECT * FROM hashtag_count_t;
```

```
SELECT count(*) FROM hashtag_count_t;
```


## Query from ksqlDB

```
docker exec -it ksqldb-cli ksql http://ksqldb-server-1:8088
```


```
CREATE STREAM twitter_raw (CreatedAt BIGINT, Id BIGINT, Text VARCHAR) 
WITH (KAFKA_TOPIC='tweet-json', VALUE_FORMAT='JSON');
``