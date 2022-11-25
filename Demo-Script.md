### Platys

```
platys init -f -n tdwi-demo -s trivadis/platys-modern-data-platform -w 1.13.0-preview
```

```
      KAFKA_enable: true
      KAFKA_KSQLDB_enable: true
      KAFKA_KSQLDB_suppress_enabled: true
      KAFKA_AKHQ_enable: true
      SPARK_enable: true
      HIVE_METASTORE_enable: true
      STREAMSETS_enable: true
      STREAMSETS_stage_libs: 'streamsets-datacollector-apache-kafka_2_7-lib,streamsets-datacollector-aws-lib'
      STREAMSETS_http_authentication: 'form'      
      ZEPPELIN_enable: true
      TRINO_enable: true
      MINIO_enable: true
```
                        
### Streamsets

#### from Twitter

```
https://stream.twitter.com/1.1/statuses/filter.json?track=fifa2022,WorldCup2022,fifaworldcup,QatarWorldCup2022,Qatar2022
```

* `DcQ8rThsnifDurvouQz0GpZRa`
* `daVHHwTjiCSQbWt0Sd0bePCdkHNys1vsKWgqDLr5KkH9YvXiCO`
* `18898576-h6719llGmPkTRaMwJ3ope8WLUkfS0UGbJnrYID99F`
* `Yxy9298sBZZYthTdp7Lok5gLJvWO3M6zhjh1NHErNdKs6`

```bash
docker exec -ti kafka-1 kafka-topics --create --topic tweet-json --partitions 8 --replication-factor 3 --zookeeper zookeeper-1:2181
```

#### to S3

<http://dataplatform:9000>

  * **Access Key ID**: ``
  * **Secret Access Key**: ``

```bash
docker exec -ti awscli s3cmd mb s3://tweet-bucket
```  

### ksqlDB

``` bash
docker exec -it ksqldb-cli ksql http://ksqldb-server-1:8088
```

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

```sql
SELECT * FROM tweet_json_s EMIT CHANGES;
```

```sql
SELECT * FROM tweet_json_s WHERE user->screen_name = 'gschmutz' EMIT CHANGES;
```

```
Demonstrating a data platform created with #platys using #streamsets, #kafka, #minio, #spark and #trino
```

```
SELECT id
	, text
	, EXPLODE(entities->hashtags)->text AS hashtag
FROM tweet_json_s 
EMIT CHANGES; 
```

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

```sql
SELECT hashtag
	, count(*) AS count
FROM tweet_hashtag_s 
WINDOW TUMBLING (SIZE 30 seconds) 
GROUP BY hashtag
EMIT FINAL;
```


### Spark / Zeppelin

Import Zeppelin Notebook

### Trino

```bash
docker exec -ti hive-metastore hive
```

```sql
CREATE DATABASE twitter_data;
```

```sql
USE twitter_data;
```

```sql
DROP TABLE IF EXISTS hashtag_count_t;
CREATE EXTERNAL TABLE hashtag_count_t (hashtag string, nof integer)
STORED AS PARQUET LOCATION 's3a://tweet-bucket/result/hashtag-counts';  
```

```bash
docker exec -ti trino-cli trino --server trino-1:8080 --catalog minio
```

```sql
use minio.twitter_data;
```

```sql
SELECT * FROM hashtag_count_t;
```