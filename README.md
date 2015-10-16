
<b>Phoenix Demo on HDP sandbox 2.3</b>

<hr>
Enviroment preparations before the demo: 





Set all required special parameter in hbase-site.xml for phoenix secondary indexes
Copy the phoenix-server jar to the Hbase lib directory

Restart HBASE

log in to sqlline.py

drop table diagnosis.VOD;
drop view  channelHistoryDevices;
drop table diagnosis.channelHistory;
drop table VOD_noschema;



create table diagnosis.VOD(customerId BIGINT NOT NULL,
eventTime TIME NOT NULL ,
modemId BIGINT,
movieId BIGINT ,
movieName CHAR(250) ,
watchedTillTheEnd BOOLEAN,price BIGINT, CONSTRAINT pk PRIMARY KEY (customerId, eventTime)) SALT_BUCKETS=2;

create table VOD_noschema(customerId BIGINT NOT NULL,
eventTime TIME NOT NULL ,
modemId BIGINT,
movieId BIGINT ,
movieName CHAR(250) ,
watchedTillTheEnd BOOLEAN,price BIGINT, CONSTRAINT pk PRIMARY KEY (customerId, eventTime));



create table diagnosis.channelHistory(
customerId BIGINT NOT NULL, 
eventTime TIME NOT NULL,
modemId BIGINT , 
channelId BIGINT, 
watchingLength BIGINT  
  , constraint pk primary key(customerId, eventTime));
  
    
  
upsert into  diagnosis.vod values (1234,current_date(),4456,999,'The Lion King',True,21);
upsert into  diagnosis.vod values(1234,TO_DATE('2015-02-13', 'yyyy-MM-dd', 'GMT+1'),4456,556,'Lord Of The Ring',False,43);
upsert into  diagnosis.vod values(1234,TO_DATE('2015-08-22', 'yyyy-MM-dd', 'GMT+1'),4456,111,'The Cable Guy',True,1);
upsert into  diagnosis.vod values(1235,current_date(),4457,999,'The Lion King',True,21);
upsert into  diagnosis.vod values(1235,TO_DATE('2014-08-22', 'yyyy-MM-dd', 'GMT+1'),4457,433,'Friends, Season 1, Episode 4',True,24);
upsert into  diagnosis.vod values(1237,TO_DATE('2015-08-23', 'yyyy-MM-dd', 'GMT+1'),4458,964,'Titanic',True,21);

upsert into  vod_noschema values (1234,current_date(),4456,999,'The Lion King',True,21);
<hr>

<h1>The Demo:</h1>

cd /usr/hdp/2.3.0.0-2130/phoenix/bin
python sqlline.py  localhost:2181
(by default, in hdp, the hbase node in the zookeeper is called hbase-unsecured
and then you need to loging with: python sqlline.py  localhost:2181:/hbase-unsecured )

show table creation sample 
Not for run, already exists:  

create table diagnosis.VOD
(customerId BIGINT NOT NULL,
eventTime TIME NOT NULL ,
modemId BIGINT,
movieId INTEGER,
movieName CHAR(250) ,
watchedTillTheEnd BOOLEAN,
price INTEGER, 
CONSTRAINT pk PRIMARY KEY (customerId, eventTime)) 
SALT_BUCKETS=2;

--Using the key:
explain select customerid from diagnosis.vod where customerid=1234;
  
select count(*) from diagnosis.vod;  

upsert into  diagnosis.vod values(1234,TO_DATE('2015-02-13', 'yyyy-MM-dd', 'GMT+1'),4456,556,'Lord Of The Ring',False,43);

select count(*) from diagnosis.vod ; 
--no Change!

--Secondary Indexes (mutable / not mutable):

CREATE INDEX movie_idx  ON diagnosis.vod(movieid DESC) INCLUDE (moviename) SALT_BUCKETS=2;

explain select * from diagnosis.vod where movieid=999;
explain select moviename from diagnosis.vod where movieid=999;
explain select moviename  from diagnosis.vod;
explain select count(*)  from diagnosis.vod where movieid=999;

--Many tuning options
--performances: http://phoenix-bin.github.io/client/performance/latest.htm

--Dynamic columns:
--upsert static record:

UPSERT INTO diagnosis.channelhistory
(
customerId , 
eventTime ,
modemId , 
channelId , 
watchingLength) values
(
1233,current_time(),444,3,50);

--upsert dynamic record:

UPSERT INTO diagnosis.channelhistory
(
customerId , 
eventTime ,
modemId , 
channelId , 
watchingLength,deviceType CHAR(250),deviceId BIGINT ) values
(
1233,current_time(),444,4,45,'MobilePhone',0478899842)

--select dynamic column:

SELECT customerId,eventTime,channelId,deviceType
FROM diagnosis.channelHistory(deviceType CHAR(250))
WHERE deviceId>400;

--Didn’t work. why?
--Fixed query:

SELECT customerId as id,eventTime,channelId,deviceType
FROM diagnosis.channelHistory(deviceType CHAR(250),deviceId BIGINT)
WHERE deviceId>400;

--Rows without dynamic values would display null:
SELECT customerId,eventTime,channelId,deviceType
FROM diagnosis.channelHistory(deviceType CHAR(250));

--VIEWS:
--	You must create the view with “select *”

--View over table that has dynamic columns:

CREATE VIEW channelHistoryDevices  
(deviceType char(250),deviceId bigint)
 AS
SELECT *
FROM diagnosis.channelHistory;
Now we can select * from the view and see all columns without specifiy dynamic columns again

select customerid, devicetype  FROM channelHistoryDevices ;



--Joins : 
SELECT  v.customerId,v.movieid,c.channelid
FROM diagnosis.VOD AS v
INNER JOIN diagnosis.channelHistory AS c
ON v.CustomerID = c.CustomerID
where v.movieid=999;

--The join used the secondary index. Phoenix also supports grouped join.
--Phoenix now has both hash join and sort-merge join  (SE_SORT_MERGE_JOIN hint)
--There are more configurations, and nice optimizations that were implemented.

--Spark loading and Processing

--Basic support for column and predicate pushdown using the Data Source API
--The Data Source API does not support passing custom Phoenix settings in configuration, you must create the DataFrame or RDD directly if you need fine-grained configuration.
--No support for aggregate or distinct queries as explained in our Map Reduce Integration documentation.

--Load the columns MovieId and MovieName from VOD_NOSCHEMA as an RDD
      val sc = new SparkContext("local", "phoenix-test")
    val rdd: RDD[Map[String, AnyRef]] = sc.phoenixTableAsRDD(
      "VOD_NOSCHEMA", Seq("MOVIEID", "MOVIENAME"), zkUrl = Some("localhost:2181")
    )
    println("First Line: "+rdd.first());

--Upsirting new record:

import java.util.Calendar
import org.apache.spark.SparkContext
import org.apache.phoenix.spark._

object PhoenixLoader {

def main(args: Array[String]) {

val sc = new SparkContext("local", "phoenix-test")

val dataSet = List((445, Calendar.getInstance().getTime(), 556, 777, "The Hobbit", true, 32))
    sc.parallelize(dataSet).saveToPhoenix("DIAGNOSIS.VOD",Seq("CUSTOMERID", "EVENTTIME", "MODEMID", "MOVIEID", "MOVIENAME", "WATCHEDTILLTHEEND", "PRICE"),zkUrl = Some("localhost:2181"))
  }
}







