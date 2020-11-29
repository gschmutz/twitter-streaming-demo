# Twitter Big Data & Fast Data Sample

This sample shows how to retrieve live twitter data and process it using both a batch processing pipeline, storing the data first in object storage as well as a stream analytics pipeline, where the data is analyzed without storing it at all. 

## Ingesting Twitter Data into Kafka

In the 1st step, we will be using StreamSets Data Collector to consume live data from Twitter and publish the tweets retrieved into a Kafka topic.

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

In the 2nd step, we will be using StreamSets Data Collector to consume the data we have previously written to the Kafka topic and store it in minIO Object Storage (which is S3 compliant).

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

## Processing Tweets in real-time using ksqlDB

In the 3rd step, we will be using a Stream Analytics component called ksqlDB to process the tweets form the `tweets-json` Kafka topic in real-time. ksqlDB offers a familiar SQL-like dialect, which we can use to query from data streams. 

![Alt Image Text](./images/use-case-step3.png "Use Case Step 3")

We first start the ksqlDB CLI and connect to the ksqlDB engine

``` bash
docker exec -it ksqldb-cli ksql http://ksqldb-server-1:8088
```

Now we have to tell ksqlDB the structure of the data in the topic, as the Kafka topic itself is "schema-less", the Kafka cluster doesn't know about the structure of the message. We do that by creating a STREAM object, which we map onto the Kafka topic `tweet-json`. 

```sql
CREATE STREAM tweet_json_s (created_at VARCHAR
                    , id BIGINT
                    , text VARCHAR
                    , source VARCHAR
                    , user STRUCT<screen_name VARCHAR, location VARCHAR>
                    , favourites_count INT
                    , statuses_count INT
                    , favorited BOOLEAN
                    , retweeted BOOLEAN
                    , lang VARCHAR
                    , entities STRUCT<hashtags ARRAY<STRUCT<text VARCHAR>>
                                            , urls ARRAY<STRUCT<display_url VARCHAR, expanded_url VARCHAR>>
                                            , user_mentions ARRAY<STRUCT<screen_name VARCHAR>>>  
            ) WITH (KAFKA_TOPIC='tweet-json', VALUE_FORMAT='JSON');
```

We don't map all the properties of the tweet, only some we are interested in. 

Now with the STREAM in place, we can use the SELECT statement to query from it

```
SELECT * FROM tweet_json_s EMIT CHANGES;
```

We have to specify the emit changes, as we are querying from a stream and by that the data is continuously pushed into the query (so called push query).

We can also selectively only return the data we are interested in, i.e. the `id`, the `text` and the `source` fields.

```
SELECT id, text, source FROM tweet_json_s EMIT CHANGES;
```

Let's work with some nested fields. Inside the `entities` structure there is a `user_mentions` array, which holds all the users being mentioned in a tweet. 

```
SELECT text, entities->user_mentions 
FROM tweet_json_s 
EMIT CHANGES;
```

With that we can easily check the array for all the tweets where `@realDonaldTrump` is mentioned

```
SELECT id
		, text 
FROM tweet_json_s 
WHERE array_contains(entities-> user_mentions, STRUCT(screen_name :='realDonaldTrump')) = true  
EMIT CHANGES;
```

In the `entities` structure, there is also a `hashtags` array, which lists all the hashtags used in a tweet. 


Let's say we want to produce trending hashtags over a given time. In KSQL, we also have a way to do aggregations with the `SELECT COUNT(*) FROM ... GROUP BY` statement, similar to using it with relational databases. 

In order to use that, we first have to flatten the array, so we only have one hashtag per result. We can do that using the `EXPLODE` function

```
SELECT id
	, text
	, EXPLODE(entities->hashtags)->text AS hashtag
FROM tweet_json_s 
EMIT CHANGES; 
```

Unfortunately we can't use the `EXPLODE` function in the `GROUP BY`, so we first have to produce the flattened result into a new Stream (with an other Kafka Topic behind) and then do the aggregation on that. 

```sql
CREATE STREAM tweet_hashtag_s
WITH (kafka_topic='tweet_hashtag')
AS
SELECT id
	, text
	, EXPLODE(entities->hashtags)->text AS hashtag
FROM tweet_json_s 
EMIT CHANGES; 
```

This produces a Stream Analytics job which now continuously run in the background and read from the `tweet_json_s`, flattens the hashtags and produces the result into the newly created `tweet_hashtag_s` stream.

On that new stream we can now do aggregation. Because a stream is unbounded, we have to specify a window over which the aggregation should take place. We are using a 1 minute window.

```sql
SELECT hashtag
	, count(*) 
FROM tweet_hashtag_s 
WINDOW TUMBLING (SIZE 60 seconds) 
GROUP BY hashtag
EMIT FINAL;
```

By using the `EMIT FINAL`, we specify that we only want to get a result at the end of the window.

## Processing Tweets using Apache Spark

In the 4th step, we will be using Apache Spark to process the tweets we have stored in object storage. We will be using Apache Zeppelin for executing the Apache Spark statements in an "ad-hoc" fashion.

![Alt Image Text](./images/use-case-step4.png "Use Case Step 4")

Navigate to <http://dataplatform:28080> to open Apache Zeppelin and login as user `admin` with password `abc123!`. Create a new notebook using the **Create new note** link. 

Now let's read all the data we have stored to Minio object storage so far, using the `spark.read.json` command

```
val tweets = spark.read.json("s3a://tweets/raw/")
```

Spark returns the result as a Data Frame, which is backed by a schema, derived from the JSON structure. We can use the `printSchema` method on the data frame to view the schema.

```
tweets.printSchema
```

the schema is quite lengthy, here only the start and the end is shown, leaving out the details in the middle

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
 ...
 |-- source: string (nullable = true)
 |-- text: string (nullable = true)
 |-- timestamp_ms: string (nullable = true)
 |-- truncated: boolean (nullable = true)
 |-- user: struct (nullable = true)
 |    |-- contributors_enabled: boolean (nullable = true)
 |    |-- created_at: string (nullable = true)
 |    |-- default_profile: boolean (nullable = true)
 |    |-- default_profile_image: boolean (nullable = true)
 |    |-- description: string (nullable = true)
 |    |-- favourites_count: long (nullable = true)
 |    |-- follow_request_sent: string (nullable = true)
 |    |-- followers_count: long (nullable = true)
 |    |-- following: string (nullable = true)
 |    |-- friends_count: long (nullable = true)
 |    |-- geo_enabled: boolean (nullable = true)
 |    |-- id: long (nullable = true)
 |    |-- id_str: string (nullable = true)
 |    |-- is_translator: boolean (nullable = true)
 |    |-- lang: string (nullable = true)
 |    |-- listed_count: long (nullable = true)
 |    |-- location: string (nullable = true)
 |    |-- name: string (nullable = true)
 |    |-- notifications: string (nullable = true)
 |    |-- profile_background_color: string (nullable = true)
 |    |-- profile_background_image_url: string (nullable = true)
 |    |-- profile_background_image_url_https: string (nullable = true)
 |    |-- profile_background_tile: boolean (nullable = true)
 |    |-- profile_banner_url: string (nullable = true)
 |    |-- profile_image_url: string (nullable = true)
 |    |-- profile_image_url_https: string (nullable = true)
 |    |-- profile_link_color: string (nullable = true)
 |    |-- profile_sidebar_border_color: string (nullable = true)
 |    |-- profile_sidebar_fill_color: string (nullable = true)
 |    |-- profile_text_color: string (nullable = true)
 |    |-- profile_use_background_image: boolean (nullable = true)
 |    |-- protected: boolean (nullable = true)
 |    |-- screen_name: string (nullable = true)
 |    |-- statuses_count: long (nullable = true)
 |    |-- time_zone: string (nullable = true)
 |    |-- translator_type: string (nullable = true)
 |    |-- url: string (nullable = true)
 |    |-- utc_offset: string (nullable = true)
 |    |-- verified: boolean (nullable = true)
 |-- withheld_in_countries: array (nullable = true)
 |    |-- element: string (containsNull = true)
```

We can see that the a tweet has a hierarchical structure and that the tweet text it self, the `text` field is only one part of much more information.

Let`s see 10 records of the data frame

```
tweets.show(10)
```

We can also ask the data frame for the number of records, using the `count` method

```
tweets.count()
```

Now let's use the `cache` method to cache the data in memory, so that further queries are more efficient

```
tweets.cache()
```

Spark SQL allows to use the SQL language to work on the data in a data frame. We can register a table on a data frame.
 
```
tweets.createOrReplaceTempView("tweets")
```

With the `tweets` table registered, we can use it in a SELECT statement. Inside spark, you can use the `spark.sql()` to execute the SELECT statement.

But with Zeppelin, there is also the possibility to use the `%sql` directive to directly work on the tables registered

```
 %sql
SELECT * FROM tweets
```

We can also do the count using SQL

```
%sql
SELECT COUNT(*) FROM tweets
```

and there are many SQL functions available and we can also work on the hierarchical data, such as `entities.hashtags.text`. First let's see only tweets, which have at least one hashtag
 
```
%sql
SELECT text, entities.hashtags.text 
FROM tweets 
WHERE SIZE(entities.hashtags) > 0
```

We can use that statement in a so called inline view and count how often a hashtag has been mentioned over all and sort it descending.

```
%sql
SELECT hashtag, COUNT(*) nof 
FROM (
	SELECT LOWER(hashtag.text) AS hashtag 
	FROM tweets LATERAL VIEW EXPLODE(entities.hashtags) hashtags AS hashtag ) 
	GROUP BY hashtag
ORDER BY nof desc
```

Let's say this is the result we want to make available. We can now turn it into a spark statement by placing it inside the `spark.sql` method call. 

```
val resultDf = spark.sql("""
select hashtag, count(*) nof from (
select lower(hashtag.text) as hashtag from tweets lateral view explode(entities.hashtags) hashtags as hashtag ) group by hashtag
order by nof desc
""")
```

The result of the `spark.sql` is another, new data frame, which we can either do further processing on, or store it using the `write.parquet` method

```
resultDf.write.parquet("s3a://tweet-bucket/result/hashtag-counts")
```

We store it in Minio object storage in the same bucket as the raw data, but use another path `result/hashtag-counts` so that we can distinguish the raw twitter data from the `hashtag-counts` data. We also no longer use json as the data format but the more efficient parquet data format.

## Using Presto to query the result in object storage

In this last step, we are using Presto SQL engine to make the result data we have created in the previous step available for querying over SQL. 

![Alt Image Text](./images/use-case-step5.png "Use Case Step 5")

To make the data in object storage usable in Presto, we first have to create the necessary meta data in the Hive Metastore. The Hive Metastore is similar to a data dictionary in the relational world, it stores the meta information about the data stored somewhere else, such as in object storage. 

We first start the Hive Metastore CLI

```
docker exec -ti hive-metastore hive
```

and create a database and then switch to that database

```
CREATE DATABASE twitter_data;
USE twitter_data;
```

Databases allow to group similar data. 

Inside the database, we can now create a table, which wraps the data in object storage. We create a table for the data in the `result/hashtag-counts` folder

```
DROP TABLE IF EXISTS hashtag_count_t;
CREATE EXTERNAL TABLE hashtag_count_t (hashtag string
									 , nof integer)
STORED AS PARQUET LOCATION 's3a://tweet-bucket/result/hashtag-counts';  
```

With the table in place, we can quit the CLI using the `exit;` command.

Now let's start the Presto CLI and connect to the `presto-1` service

```
docker exec -ti presto-cli presto --server presto-1:8080 --catalog minio
```

We then switch to the `minio` catalog and the `twitter_data` database, which matches the name of the database we have created in the Hive Metastore before

```
use minio.twitter_data;
```

A show tables should show the one table `hashtag_count_t` we have created before

```
show tables;
```

Now we can use `SELECT` to query from the table

```
SELECT * FROM hashtag_count_t;
```

We can also count how many hashtags we have seen so far

```
SELECT count(*) FROM hashtag_count_t;
```

Presto can be integrated into many standard data analytics tools, such as PowerBI or Tableau any many others.