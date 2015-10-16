
<h1>Phoenix Demo on HDP sandbox 2.3</h1>

<hr>
<h2>Enviroment preparations before the demo: </h2>





--Set all required special parameter in hbase-site.xml for phoenix secondary indexes
--Copy the phoenix-server jar to the Hbase lib directory

--Restart HBASE

--log in to sqlline.py

drop table diagnosis.VOD; </br>
drop view  channelHistoryDevices; </br>
drop table diagnosis.channelHistory; </br>
drop table VOD_noschema; </br>



create table diagnosis.VOD(customerId BIGINT NOT NULL,</br>
eventTime TIME NOT NULL ,</br>
modemId BIGINT,</br>
movieId BIGINT ,</br>
movieName CHAR(250) ,</br>
watchedTillTheEnd BOOLEAN,price BIGINT, CONSTRAINT pk PRIMARY KEY (customerId, eventTime)) SALT_BUCKETS=2; </br>
</br></br>
create table VOD_noschema(customerId BIGINT NOT NULL,</br>
eventTime TIME NOT NULL ,</br>
modemId BIGINT,</br>
movieId BIGINT ,</br>
movieName CHAR(250) ,</br>
watchedTillTheEnd BOOLEAN,price BIGINT, CONSTRAINT pk PRIMARY KEY (customerId, eventTime)); </br>
</br></br>


create table diagnosis.channelHistory(</br>
customerId BIGINT NOT NULL, </br>
eventTime TIME NOT NULL,</br>
modemId BIGINT , </br>
channelId BIGINT, </br>
watchingLength BIGINT</br>  
  , constraint pk primary key(customerId, eventTime)); </br>
  </br></br>
    
  
upsert into  diagnosis.vod values (1234,current_date(),4456,999,'The Lion King',True,21);</br>
upsert into  diagnosis.vod values(1234,TO_DATE('2015-02-13', 'yyyy-MM-dd', 'GMT+1'),4456,556,'Lord Of The Ring',False,43);</br>
upsert into  diagnosis.vod values(1234,TO_DATE('2015-08-22', 'yyyy-MM-dd', 'GMT+1'),4456,111,'The Cable Guy',True,1);</br>
upsert into  diagnosis.vod values(1235,current_date(),4457,999,'The Lion King',True,21);</br>
upsert into  diagnosis.vod values(1235,TO_DATE('2014-08-22', 'yyyy-MM-dd', 'GMT+1'),4457,433,'Friends, Season 1, Episode 4',True,24);</br>
upsert into  diagnosis.vod values(1237,TO_DATE('2015-08-23', 'yyyy-MM-dd', 'GMT+1'),4458,964,'Titanic',True,21);</br>

upsert into  vod_noschema values (1234,current_date(),4456,999,'The Lion King',True,21);</br></br>
<hr>

<h1>The Demo:</h1>

--cd /usr/hdp/2.3.0.0-2130/phoenix/bin </br>
--python sqlline.py  localhost:2181 </br>
--(by default, in hdp, the hbase node in the zookeeper is called hbase-unsecured
and then you need to loging with: python sqlline.py  localhost:2181:/hbase-unsecured ) </br>
</br>
<h2>show table creation sample </h2>
--Not for run, already exists:  
</br>
create table diagnosis.VOD </br>
(customerId BIGINT NOT NULL,</br>
eventTime TIME NOT NULL ,</br>
modemId BIGINT,</br>
movieId INTEGER,</br>
movieName CHAR(250) ,</br>
watchedTillTheEnd BOOLEAN,</br>
price INTEGER, </br>
CONSTRAINT pk PRIMARY KEY (customerId, eventTime)) </br>
SALT_BUCKETS=2;</br>
</br>
<h2>Using the key:</h2>
explain select customerid from diagnosis.vod where customerid=1234;
  </br>
select count(*) from diagnosis.vod;  
</br>
upsert into  diagnosis.vod values(1234,TO_DATE('2015-02-13', 'yyyy-MM-dd', 'GMT+1'),4456,556,'Lord Of The Ring',False,43);</br>

select count(*) from diagnosis.vod ; </br>
--no Change!
</br>
<h2>Secondary Indexes (mutable / not mutable):</h2>

CREATE INDEX movie_idx  ON diagnosis.vod(movieid DESC) INCLUDE (moviename) SALT_BUCKETS=2;

explain select * from diagnosis.vod where movieid=999;</br>
explain select moviename from diagnosis.vod where movieid=999;</br>
explain select moviename  from diagnosis.vod;</br>
explain select count(*)  from diagnosis.vod where movieid=999;</br>
</br>
--Many tuning options</br>
--performances: http://phoenix-bin.github.io/client/performance/latest.htm</br>

<h2>Dynamic columns:</h2>
--upsert static record:

UPSERT INTO diagnosis.channelhistory</br>
(</br>
customerId , </br>
eventTime ,</br>
modemId , </br>
channelId , </br>
watchingLength) values</br>
(</br>
1233,current_time(),444,3,50);</br>
</br>
--upsert dynamic record:

UPSERT INTO diagnosis.channelhistory</br>
(</br>
customerId , </br>
eventTime ,</br>
modemId ,</br> 
channelId ,</br> 
watchingLength,deviceType CHAR(250),deviceId BIGINT ) values</br>
(</br>
1233,current_time(),444,4,45,'MobilePhone',0478899842)</br>
</br>
--select dynamic column:</br>
</br>
SELECT customerId,eventTime,channelId,deviceType</br>
FROM diagnosis.channelHistory(deviceType CHAR(250))</br>
WHERE deviceId>400;</br>
</br>
--Didn’t work. why?</br>
--Fixed query:</br>
</br>
SELECT customerId as id,eventTime,channelId,deviceType</br>
FROM diagnosis.channelHistory(deviceType CHAR(250),deviceId BIGINT)</br>
WHERE deviceId>400;</br>
</br>
--Rows without dynamic values would display null:</br>
SELECT customerId,eventTime,channelId,deviceType</br>
FROM diagnosis.channelHistory(deviceType CHAR(250));</br>
</br>
<h2>VIEWS:</h2></br>
--	You must create the view with “select *”</br>
</br>
--View over table that has dynamic columns:</br>
</br>
CREATE VIEW channelHistoryDevices  </br>
(deviceType char(250),deviceId bigint)</br>
 AS</br>
SELECT *</br>
FROM diagnosis.channelHistory;</br>
Now we can select * from the view and see all columns without specifiy dynamic columns again</br>
</br>
select customerid, devicetype  FROM channelHistoryDevices ;</br>
</br>
</br>
</br
<h2>Joins : </h2>
SELECT  v.customerId,v.movieid,c.channelid</br>
FROM diagnosis.VOD AS v</br>
INNER JOIN diagnosis.channelHistory AS c</br>
ON v.CustomerID = c.CustomerID</br>
where v.movieid=999;</br>
</br>
--The join used the secondary index. Phoenix also supports grouped join.</br>
--Phoenix now has both hash join and sort-merge join  (SE_SORT_MERGE_JOIN hint)</br>
--There are more configurations, and nice optimizations that were implemented.</br>

<h2>Spark loading and Processing </h2></br>

--Basic support for column and predicate pushdown using the Data Source API</br>
--The Data Source API does not support passing custom Phoenix settings in configuration, you must create the</br> DataFrame or RDD directly if you need fine-grained configuration.</br>
--No support for aggregate or distinct queries as explained in our Map Reduce Integration documentation.</br>
</br>
--Load the columns MovieId and MovieName from VOD_NOSCHEMA as an RDD</br>
      val sc = new SparkContext("local", "phoenix-test")</br>
    val rdd: RDD[Map[String, AnyRef]] = sc.phoenixTableAsRDD(</br>
      "VOD_NOSCHEMA", Seq("MOVIEID", "MOVIENAME"), zkUrl = Some("localhost:2181")</br>
    )</br>
    println("First Line: "+rdd.first());</br>

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







