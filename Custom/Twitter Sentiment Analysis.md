**PREREQUISITES**
=================

	* Download JSON Serde at:
	* http://files.cloudera.com/samples/hive-serdes-1.0-SNAPSHOT.jar
	* and to renominate it as hive-serdes-1.0.jar

- Add Jar to **HIVE_AUX_JARS_PATH** of HiveServer2:

	1. Copy the JAR files to the host on which HiveServer2 is running. Save the JARs to any directory you choose, and make a note of the path (create directory in /usr/share/).
		1. In the Cloudera Manager Admin Console, go to the Hive service.
		2. Click the Configuration tab.
		3. Expand the Service-Wide > Advanced categories
		4. Configure the Hive Auxiliary Jars Directory property with the path from the first step.
		5. Click Save Changes. The JARs are added to HIVE_AUX_JARS_PATH environment variable.
		6. Redeploy the Hive client configuration.
			1. In the Cloudera Manager Admin Console, go to the Hive service.
			2. From the Actions menu at the top right of the service page, select Deploy Client Configuration.
			3. Click Deploy Client Configuration.
		7. Restart the Hive service. If the Hive Auxiliary Jars Directory property is configured but the directory does not exist, HiveServer2 will not start.

- Execute the following commands on terminal:
	+ hive> add jar hive-serdes-1.0.jar;

**PREPARE DATA**
================

- Step 1: Download and Extract the Sentiment Tutorial Files.

	* You can download a set of sample Twitter data contained in a compressed (.zip) folder here:
		+ http://s3.amazonaws.com/hw-sandbox/tutorial13/SentimentFiles.zip
	* The Twitter data was obtained using Flume. Flume can be used as a log aggregator, collecting log data from many diverse sources and moving it to a centralized data store. In this case, Flume was used to capture the Twitter stream data, which we can now load into the Hadoop Distributed File System (HFDS).
	* Save the SentimentFiles.zip file to your computer, and then extract the files. You should see a SentimentFiles folder.

- Step 2: Load Twitter Data into the HDFS with HUE.

	* Click on the File Browser tab at the top and then select Upload -> Zip File.
	* You will see a file selection box and navigate into the SentimentFiles folder. You will see a file called upload.zip. Select that and start the upload.

**SENTIMENT ANALYSIS**
======================

Create the tweets_raw table containing the records as received from Twitter
---------------------------------------------------------------------------
```
CREATE EXTERNAL TABLE tweets_raw (
   id BIGINT,
   created_at STRING,
   source STRING,
   favorited BOOLEAN,
   retweet_count INT,
   retweeted_status STRUCT<
      text:STRING,
      user:STRUCT<screen_name:STRING,name:STRING>>,
   entities STRUCT<
      urls:ARRAY<STRUCT<expanded_url:STRING>>,
      user_mentions:ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
      hashtags:ARRAY<STRUCT<text:STRING>>>,
   text STRING,
   user STRUCT<
      screen_name:STRING,
      name:STRING,
      friends_count:INT,
      followers_count:INT,
      statuses_count:INT,
      verified:BOOLEAN,
      utc_offset:STRING, -- was INT but nulls are strings
      time_zone:STRING>,
   in_reply_to_screen_name STRING,
   year int,
   month int,
   day int,
   hour int
)
ROW FORMAT SERDE 'com.cloudera.hive.serde.JSONSerDe '
LOCATION '/user/YOURUSER/upload/upload/data/tweets_raw'
;
```
Create sentiment dictionary
---------------------------
```
CREATE EXTERNAL TABLE dictionary (
    type string,
    length int,
    word string,
    pos string,
    stemmed string,
    polarity string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/user/YOURUSER/upload/upload/data/dictionary';

CREATE EXTERNAL TABLE time_zone_map (
    time_zone string,
    country string,
    notes string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/user/YOURUSER/upload/upload/data/time_zone_map';
```
Clean up tweets
---------------
```
CREATE VIEW tweets_simple AS
SELECT
  id,
  cast ( from_unixtime( unix_timestamp(concat( '2013 ', substring(created_at,5,15)), 'yyyy MMM dd hh:mm:ss')) as timestamp) ts,
  text,
  user.time_zone
FROM tweets_raw
;

CREATE VIEW tweets_clean AS
SELECT
  id,
  ts,
  text,
  m.country
 FROM tweets_simple t LEFT OUTER JOIN time_zone_map m ON t.time_zone = m.time_zone;
```
Compute sentiment
-----------------

- Explode each tweet text in array of words.
	* Example: 330166074362433536 ["waiting","for","iron","man","3","to","start"]
```
create view l1 as select id, words from tweets_raw lateral view explode(sentences(lower(text))) dummy as words;
```
- Explode each tweet -> word on multiple raws

	* 330166074362433536 waiting
	* 330166074362433536 for
	* 330166074362433536 iron
	* 330166074362433536 man
	* 330166074362433536 3
	* 330166074362433536 to
	* 330166074362433536 start
```
create view l2 as select id, word from l1 lateral view explode( words ) dummy as word ;
```
- Used the dictionary file to score the sentiment of each Tweet by the number of positive words compared to the number of negative
words, and then assigned a positive, negative, or neutral sentiment value to each Tweet.
```
create view l3 as select
    id,
    l2.word,
    case d.polarity
      when  'negative' then -1
      when 'positive' then 1
      else 0 end as polarity
 from l2 left outer join dictionary d on l2.word = d.word;

 create table tweets_sentiment stored as orc as select
  id,
  case
    when sum( polarity ) > 0 then 'positive'
    when sum( polarity ) < 0 then 'negative'
    else 'neutral' end as sentiment
 from l3 group by id;
```
- Put everything back together and re-number sentiment
```
CREATE TABLE tweetsbi
STORED AS ORC
AS
SELECT
  t.*,
  case s.sentiment
    when 'positive' then 2
    when 'neutral' then 1
    when 'negative' then 0
  end as sentiment
FROM tweets_clean t LEFT OUTER JOIN tweets_sentiment s on t.id = s.id;
```
**N-GRAM**
==========

The commands above will return the top-10 (1-gram) from all tweet.
--------------------------------------------------------------------
```
create table twitter_trending_words_struct (NEW_ITEM ARRAY<STRUCT<ngram:array<string>, estfrequency:double>>);

INSERT OVERWRITE TABLE twitter_trending_words_struct SELECT context_ngrams(sentences(lower(text)), array(null), 10) AS snippet from tweets_raw;

create table twitter_trending_words (ngram string, estfrequency double);

INSERT OVERWRITE TABLE twitter_trending_words select X.ngram[0], X.estfrequency  from (select explode(new_item) as X from twitter_trending_words_struct ) Z;
```
The commands above will return the top-100 bigrams (2-grams)
--------------------------------------------------------------------
```
create table twitter_trending_two_words_struct ( NEW_ITEM ARRAY<STRUCT<ngram:array<string>, estfrequency:double>>);

INSERT OVERWRITE TABLE twitter_trending_two_words_struct SELECT context_ngrams(sentences(lower(text)), array(null,null), 100) AS snippet from tweets_raw;

create table twitter_trending_two_words (term1 string, term2 string,estfrequency double);

INSERT OVERWRITE TABLE  twitter_trending_two_words select X.ngram[0],X.ngram[1],X.estfrequency  from (select explode(new_item) as X from twitter_trending_two_words_struct) Z;
```
**REFERENCES**
==============
- http://hortonworks.com/hadoop-tutorial/how-to-refine-and-visualize-sentiment-data/
- http://bigdatabloggin.blogspot.it/2012/08/trending-topics-in-hive-ngrams.html
- https://cwiki.apache.org/confluence/display/Hive/StatisticsAndDataMining