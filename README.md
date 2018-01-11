# data-misc-tools

[![License](https://img.shields.io/github/license/cfcodefans/data-misc-tools.svg)](LICENSE)

This project hosts several tools to help with development using Hive, Spark

To study and apply how to leverage new features of Hive, Spark, Hadoop
for data process, I created and am developing some functions for Hive and spark.


### Requirement
1. Spark 2.2
2. Hive 2.3
3. Hadoop 3.0.0

## 1. Spark Runner

Spark Runner is inspired by SparkShell, It uses the same Scala Interpreter with SparkSession bound in
Interpreter context, so it can:

1. interpret scala code from a file on hdfs(will extend to load scala/java/python from more source/storage)
2. compile the code into some scala function that takes parameters of current time, sparksession, arbitaray parameters such as some Rdd
3. run and process the data according the timing.

It will run at one minute interval periodically and detect the change of scala source,
change of code will trigger scala interpreter to compile and update the ongoing scala function.

Basically a timer to run spark jobs without compiling, stop, restart spark process.


## 2. Hive functions

Regarding Hive UDF(User Defined Function), I hope to develop more functions to enable hive to connect/interact with many other data sources(as long as it provides data through some URI). Therefore this can become an option for ETL and different approach to apply machine learning algorithms to data in hive. 

### Some notes (might be wrong/insufficient)

Hive has 3 different types of UDF
1. [GenericUDF](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-CreateFunction) This kind of UDF executes on each row given by sql, applies arbitrary business logic and returns the transformed data. It has initialize method invoked before the first row is passed into evaluate() method, however it does not get notified when the last row is passed. Therefore it is unsuitable for some cases such as using some clients to access data sources like Kafka, since it might not know when to close the client. Otherwise it is expensive to create a kafka client then close it for each row's execution.
  
2. [GenericUDTF](https://cwiki.apache.org/confluence/display/Hive/DeveloperGuide+UDTF) The output of UDTF is treated as a complete table, therefore in select query, the UDTF expression must be the only one term exclusively, it can be nested inside other functions. However its close() method is called. Therefore I develop t_zk_write(udf to write zookeeper) as UDTF, so it can create single zookeeper client and use it to write all the rows to zookeeper, kafka UDTFs are similar cases.


3. [GenericUDAFResolver2](https://cwiki.apache.org/confluence/display/Hive/GenericUDAFCaseStudy) (I am still studying this) 


### HTTP UDF (get/post)
1. url is always string
2. timeout is always int, default by 3000
3. headers is a map<string, string>
4. content is string
 
|type|name|parameters|return|
| --- | --- | --- | --- |
|udf|http_get|_FUNC_(url, timeout, headers) - send get request to url with headers in timeout|http code, headers, content|
|udf|http_post|_FUNC_(url, timeout, headers, content) - send post to url with headers in timeout|http code, headers, content|
|udf|url_encode|_FUNC_(string) - send get request to url with headers in timeout|encoded string|
|udtf|t_http_get|_FUNC_(url, timeout, headers) - send get request to url with headers in timeout|http code, headers, content|
|udtf|t_http_post|_FUNC_(url, timeout, headers, content) - send post to url with headers in timeout|http code, headers, content|

This call of UDF http_get will need to open and close http client instance of each of 10 rows.
```sql
select city_name, http_get(printf("http://solr:8984/solr/user/select?fl=gender,age,nickname&q=city:'%s'&wt=json", city_name)) from p_city limit 10;
```
This call of UDTF t_http_get will only open one http client instance and share with all rows, then close it
```sql
select t_http_get(city_name, printf("http://solr:8984/solr/user/select?fl=gender,age,nickname&q=city:%s&wt=json", url_encode(city_name))) from p_city limit 30;
```
city_name is a context column to mark result of each row.

### ZOOKEEPER UDF (read/write/delete, can't watch, sorry)

1. zkAddress is string like host1:port,host2:port...
2. timeout is int
3. pathToRead is string

|type|name|parameters|return|
| --- | --- | --- | --- |
|udf|zk_read|_FUNC_(zkAddress, timeout, pathToReads...) recursively read data from zookeeper paths|struct of "p", "v", path and value|
|udf|zk_write|_FUNC_(zkAddress, timeout, pathAndValues...) recursively write values paired with zookeeper paths|struct of "p", "v", path and old value|
|udf|zk_delete|_FUNC_(zkAddress, timeout, pathToDeletes) recursively delete zookeeper paths|struct of "p", "v", path and old value|
|udtf|t_zk_write|_FUNC_(zkAddress, timeout, pathAndValues...) recursively write values paired with zookeeper paths|struct of "p", "v", path and old value|
|udtf|t_zk_delete|_FUNC_(zkAddress, timeout, pathToDeletes) recursively delete zookeeper paths|struct of "p", "v", path and old value|

### Kafka (/default producer configurations/default consumer configuration/list topics/pull/push)
|type|name|parameters|return|
| --- | --- | --- | --- |
|udf|m_add|_FUNC_(map1, map2....) - Returns a combined map of N maps|a map|

I am working on functions to pull from/push to kafka.