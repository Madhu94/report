### Comparison 
#### Raw performance

MongoDB's aggregation pipeline performs better for smaller datasets. Larger datasets, where the data size each stage works with exceeds 100MB, have to use disk,
so the pipeline's performance degrades.  InfluxDB performs well for queries that involve aggregations over a single field, but performance degrades and
memory usage spikes when we have to perform queries like finding hitcount by user,department as these involve merging all the series.

## Flexibility
Flexibility in adding new dimensions is an important use case of the application usage analysis system.
### SQL
With SQL's rigid schema, to add new dimensions we can
* normalise the data so that pulling meaningful data from the table requires several joins, or
* keep the data denormalised. All writes go to a single unindexed table. When the user wants to query/do analysis on a parameter, all records with that
 parameter are read from the main table,written to another table and that column is indexed in the main table. A metadata table containing the parameters
 being queried, along with the time of the last update is also kept. All further queries for that parameter will be from the new indexed table. A separate
 'read table' for each application can be maintained.

### MongoDB
Since this is a document store that works with bson data, new fields can be introduced anytime. There is also a "$exists" operator which can be used to
filter out documents that contain a particular field and optionally dump it to a new collection.

### InfluxDB
The InfluxDB data model involves tags,fields,series and measurements. 
Briefly,
**Measurements** => Like a SQL table

**Tags** => What you would use in a where clause or a group by clause typically.
        Like dimensions in the current app-usage system

**Fields** => What would be on the y-axis typically.
          Like measures in the current app-usage system.

**Series** => A unique set of tags make up a series

Tags are indexed,fields are not. We can create a new tag anytime. Although the concept of tags sounds promising, there is a huge catch. The memory usage
and query performance of InfluxDB are directly affected by our series cardinality(number of unique series in our measurement/database).
We might need to stop users from adding tags with high cardinality. Adding it as a field and then using some preprocessing may get the desired result
and prevent OOM. Another caveat is that points belong to the same series (same set of tags) should not have the same timestamp, else the old value is
updated.

### Columnar datastores 
### SRQ
SRQ supports two formats - archived/historical and realtime. The archived format cannot be queried, while we're writing to it(i.e. till we close the
writer) and once closed, we can't write to the table again. This is clearly not suitable for us- we have to maintain several tables and keep track of them.
The realtime format allows querying and writing to the table at once. However, there can't be more than one writer at a time. So,we may need to set up
a service and all writes can be sent to that. Currently, only a Java writer API is available and several aggregation operations outlined [here](https://gitlab.nyc.deshaw.com/nmadhum/database_evaluation/blob/bb93d0b2c15bdaac1601ac06d8aa6797072e1016/GroupBy_Specs_Final_v3.pdf) are yet
to be implemented.

### Cassandra
Cassandra's schema,along with its [partition key](http://stackoverflow.com/questions/24949676/difference-between-partition-key-composite-key-and-clustering-key-in-cassandra) must be specified at creation time. The partition key resticts what queries can be performed on the
table, see [here](http://www.datastax.com/dev/blog/a-deep-look-to-the-cql-where-clause). Cassandra does not offer group by options, although [user-defined functions and aggregates](http://www.planetcassandra.org/blog/user-defined-functions-in-cassandra-3-0/) can be used to simulate a group by.

## Maintainability
MongoDB is written in C++. The admin shell in mongodb is a full javascript interpreter that can run scripts. There are several third-party open-source and
 commercial tools to view the documents and visualise them like mongo-express,adminMongo,Fluentd and Edda.
InfluxDB is written in Go. It offers an admin user interface that runs on the browser, default port 8083 and an admin shell too. Data is written to
influxdb using a rest api - whether we use the admin UI, shell or one of the [client libraries](https://docs.influxdata.com/influxdb/v0.11/clients/api/)

## Preprocessing
How could we pre-computing the results of frequently used and known queries can improve performance.
### SQL
A stored procedure could run at regular intervals and output the results to another table.
### MongoDB 
There are three ways to do aggregations in MongoDB -
* Aggregation pipeline(natively supported by MongoDB)
* Map-Reduce (javascript) 
* single purpose aggregations

Of these, aggregation pipelines give the best performance. Aggregation pipeline could be triggered at regular intervals and pre-computed results could
be stored in another collection. Or incremental map-reduce could be done.

### InfluxDB 
[Continuous queries](https://docs.influxdata.com/influxdb/v0.11/query_language/continuous_queries/) in InfluxDB could run in the background and the results could be written to another measurement. We can configure how frequently they
 run and the time interval over which they aggregate data. Visualization tools like Grafana could query the downsampled measurement directly for plotting.


## Scalability
### SQL
Vertical Scaling is the only option.
### MongoDB
Horizontal scaling using sharding is supported. Shards can be added, removed easily.The sharding details are abstracted and the application driver
can communicate with a cluster the same way it does with a standalone mongod instance. Sharding is done based on a shard key, and it is done on
a per-collection basis. The [shard key]() has the following restrictions :
* The shard key should be an indexed field appearing in every document in the collection.
* It should provide writes distribution and query isolation. If writes are not distributed,this could cause a write bottleneck(the write speed for the
cluster would be the write speed of a single shard). Since writes could be batched and done offline, we need not be concerned about this. However, query
isolation implies that only those queries that use the shard key will be efficient. Else, all the shards need to be queried and the results need to be
merged and then presented to the user. These scatter/gather queries take up time.
Sharding is a one-time process and its difficult to "un-shard" a collection. This may cause problems if our access patterns change after some time.
 Sharding is to be done only when we have to, when the standalone mongod/replica set runs out of memory.

### InfluxDB
As of the current release(0.11) influxdb supports clustering. Future versions will [not offer open source support for clustering](https://influxdata.com/blog/update-on-influxdb-clustering-high-availability-and-monetization/). An InfluxDB cluster consists of
* 1 or more data nodes - these run the data service, which persists data.
* An odd number of consensus nodes(at least 3) - these run the consensus service, which participates in the [Raft algorithm](http://thesecretlivesofdata.com/raft/).
* Hybrid nodes, which run both data and consensus services.
Clustering is now still in beta version.

## Deployment 
MongoDB can be deployed as
* A single standalone mongod server - this is not recommended in production environments as it does not offer any fault tolerance.
* A replica set consisting of one primary and 2 secondaries or one primary,one secondary and one arbiter. An arbiter has voting privileges but does
not carry any data, so its hardware requirements are minimal.This is the simplest setup which offers fault tolerance. All writes go to the primary,
the secondaries copy the primary's oplog and replicate it. Reads can be from secondaries too.  If the primary fails, a secondary can be elected as primary.
 A replica set can consist of upto 50 members, but only 7 voting members.
* A cluster consisting of many shards. Each shard can be a replica set or a single standalone server. Sharding environments also require config
servers which maintain metadata about the cluster and a mongos(mongo shard utility) instance. Mongos abstracts away the internal sharding details
and allows our application's driver to talk to it as if it were talking to a single mongod server.

InfluxDB can be deployed as
1)Standalone instance - no fault tolerance.
2)Cluster - replication factor can be set from 1 to all nodes in the cluster. There are data nodes which persist our data,consensus nodes which
take care of consistency across nodes. The cluster must have an odd number of consensus nodes running the Raft consensus algorithm.


## Memory 
Tags are indexed. This can cause InfluxDB to quickly consume large amounts of memory(3 to 5 times as much as MongoDB). Queries that involve points
from multiple series can cause InfluxDB to consume a lot of memory and then crash.
So,filtering by user/action/time/deprtment or some combination of these is necessary to make the query work without consuming a lot of memory.
InfluxDB's memory consumption spikes from 3GB to about 70-80GB when running queries.

## Tuning
What can we do to get better performance from the database?
### SQL 
SQL offers a variety of index, each meant for a specific kind of query. Other than the usual secondary indexes, 
* Filtered indexes - when you have an additional filter based on a column other than the one you're indexing on. Say, if you frequently query the
users who use dportal, putting an index on users might speed up performance. But if you create a filtered index on the users where application = 'dportal',
 the performance gains are much much higher, and the space consumed by the index is also minimised. This index will not be helpful if you query for users
 of some other application though.
* Columnstore indexes - store data in a columnstore format and not the usual rowstore format. This is based on the assumption that queries
rarely ask for more than a few column at a time and aggregations are done over a column. Also, same-type data is stored together, this means
 higer compression ratios. This can't be used if your query uses a lot of where clauses to search deep into the data.
 * Full text indexes  - For columns that involve a lot of text, this index can give better performance than the SQL LIKE claues.
* Indexed Views - A view/subset of the table that is persisted. If a filtered index can be used, this approach is better avoided. Indexed views must be
created with schema binding(meaning that the underlying table cannot be altered without dropping the view). Also, the query optimizer always chooses the
filtered index over the equivalent indexed view.
* Included Columns - For example,specifying "create index on table(col1) include columns(col2,col3)" at the time of creation of an index will store the columns col2,col3 in the col1 index. But these will not be used to construct the index and they will take up more space, but they give very good performance for queries which use the col1 index and ask for col2,col3.

### MongoDB 
* Secondary indexes - these can be a single column or a compound index of multiple columns. If an index does not cover a query, MongoDB does not use the index.
In those cases, forcing mongoDB to use the indexing explicitly improves performance. Forcing it to use the wrong index can degrade performance badly.
* Multikey index - an index over an array and all its fields. Multikey indexes are subject to a lot of restrictions.
* Text indexes - similar to sql server's full text index.
* Hashed index - Used mostly for distributing writes evenly in a sharded environment. It should work well for equality match queries, but not range queries. 
* Aggregation pipeline - Do not create pipeline stages that handle more than 100MB data, this might need to use disk, and can be slow.
Maybe,using "$project" to get only the fields we need from the document can improve performance.

### InfluxDB
Tags are indexed in influxdb, there is no indexing on fields. So,filtering by using a where clause on a field value will be slow. Query performance is
not affected by the number of points in the measurement but by the series cardinality. If the series cardinality is too high,influxdb consumes a lot of RAM
and crashes for queries that require points from many series. That is, influxdb was built to handle a few large series and not many small ones. Rethinking
the schema to include tags with fewer distinct values can improve query performance. As a rule of thumb, not more than 100000 series.


## Complementary tools with Influxdb
### The Tick Stack
InfluxDB is part of the TICK stack which is an end-to-end solution for collecting,storing,processing and visualising data. All components of the stack are
written in Go.
* T - Telegraf, an open source metrics collector. There are many others, but Telegraf was built to easily integrate with InfluxDB's data model. Telegraf can
collect system metrics like cpu,memory from any host, it has plugins for mysql,mongodb and many other datasources and even messaging queues like apache kafka.
* I - InfluxDB, an open source time-series database. 
* C -Chronograf, a closed-source(for now) visualization tool for influxDB. Does not seem to be a mature tool yet.
* K - Kapacitor, a data processing and alerting tool for influxdb. The alerting can be done as part of a stream or a batch task, using TICKScript, a Domain
 Specific Language(DSL) for Kapacitor. If the alert criteria is met, an event handler is called(post to a log file, a http post, email and many other
 event handlers are supported). Information about the event is also passed to the handler and can be used inside it. Custom anomaly detection algorithms
 can be created using python scripts(numpy,scipy algorithms) and referencing the python script in our TICKScript program. Any valid input for influxdb
 is also valid for kapacitor so that alerts can be triggered at collection time itself.

### Grafana
Grafana is a fork of Kibana 3, with a focus on time-series data visualization and graphs (built for graphite originally).
It can create dashboards comprising graphs, tables, singlestats(queries that reduce to single number). A dashboard can pull data from multiple datasources
(multiple influxdb instances) and constuct panels with them. A panel is a graph,table or singlestat. However each panel can only be tied to a single datasource.
Grafana works well for aggregation queries - based on the size of the panel and the number of points being queried, it uses influxdb's group by time() clause
to reduce the number of points returned. Non-aggregation queries that do not use the group by time() feature should atleast use time filters in the where
clause to avoid slowing down Grafana. Scripted dashboards is a powerful feature to generate dynamic dashboards using javascript.

### Comparison Summary

|**Parameter\Database** |**SQL** | **MongoDB**  |**InfluxDB**   |
|-----------------------|---------------- |-----------------|----------------------|
|**Flexibility**       |Alter table to add new dimension| No schema needs to specified.A new dimension/measure can be added anytime. If some schema needs to be imposed,there are Object Document Mappers(ODMs) that provide schema validation on top of the default MongoDB drivers.| A new [tag/field](https://docs.influxdata.com/influxdb/v0.11/concepts/key_concepts/) can be added anytime. Tags should not have too many unique values, this could cause the influxd process to quickly use up a lot of memory.|
|**Maintainability**    |    | Written in C++. Mongo's default admin interface is the mongo shell, which is a full JavaScript interpreter.Written in GO. Admin web interface.|Influx shell. Uses a SQL-like query language, InfluxQL|
|**Preprocessing**      |Stored Procedures can be triggered periodically to pre-compute and store results of frequently used queries |Aggregation pipeline/ Map-reduce. These operations could be triggered at intervals and the results stored in separate collection. Server side javascript.But this can have [performance issues](http://stackoverflow.com/questions/17550244/does-server-side-javascript-function-have-performance-issues-in-mongodb). |Continuous queries - aggregate data periodically and store as new measurement.The intervals at which these should be run and the time interval over which they should aggregate data can be configured|
|**Scalability**        |Vertical Scaling | Sharding split data across multiple mongod/replica sets based on a shard keya one-time process,its not easy to unshard a collection It does not improve query performance,but the queries which do not invlove the shard key will be inefficient | Clustering(in beta). Future versions of InfluxDB will [not have open source support](https://influxdata.com/blog/update-on-influxdb-clustering-high-availability-and-monetization/) for clustering|
|**Deployment**         |  | A single mongod instance - no fault tolerance, not recommended for production. A replica set - 1 primary, 1 secondary, 1 arbiter/another secondary. Secondaries contain copies of primary's data. This is the simplest production model. Sharded cluster- split the data across multiple shards. Each shard can be standalone mongod or a replica set|  Standalone/Clustered. |
|**Memory**             |  | Uses up 0.738GB on disk | Uses up to 0.405GB on disk.RAM memory spikes when queries are run, memory consumed by influxd increases with series cardinality and if the query involves points across multiple series. |
|**Expire data**        |  | Use TTL(Time To Live) indexes to expire data after a certain time| Retention policies - specifies how many nodes to write to, and the duration of the data. This can be specified at insert time. |
|**Tuning**             | Use filtered indexes, included columns and indexed views to improve query performance | Secondary indexes. Aggregation pipeline can use indexes if $match is used to filter needed documents at the beginning of the pipeline.Sparse indexes- By default, secondary indexes are built on every document in the collection. A sparse index will be built only on those documents that have the specified field. This could save a lot of space.| Design schema to reduce series cardinality.|
|**Complementary Tools**|  |  |[Kapacitor](https://influxdata.com/time-series-platform/kapacitor/) - Alerting tool and data processing engine. [Grafana](http://grafana.org/) It can create dashboards comprising panels of graphs, tables, singlestats(queries that reduce to single number). A dashboard can pull data from multiple datasources (multiple influxdb instances) and constuct panels with them. Scripted dashboards is a powerful feature to generate dynamic dashboards using javascript. |
|**Community Support**  |  |  [Google groups](https://groups.google.com/forum/#!forum/mongodb-user)[IRC chat](http://irc.lc/perl/mongodb/irctc@@@).Github stars : 9033. Forks:2523.Stackoverflow tags for "mongodb" - 60462 | [Github issues page](https://groups.google.com/forum/#!forum/influxdb)[Google groups](https://groups.google.com/forum/#!forum/influxdb)Github stars :7707 Forks:1008 .Stackoverflow tags for "influxdb"-222|


#### When should you use InfluxDB? 
InfluxDB is a great tool when 
* you have a write-heavy load, it can scale to above 100k writes/s.
* the dimensions by which you query are few and are not likely to change much, but you need to process tens of millions of records
* you need real-time alerting. InfluxDB can easily integrate with Kapacitor and send alerts through email, write to log files, etc.
* visualise aggregated data in real-time, through Grafana. 

InfluxDB should not be used if
* the data has many dimensions/features by which we may want to analyse/group/filter
* the dimensions have many distinct values and these may change frequently,
* our queries may involve any/all of these dimensions
 
#### When should you use MongoDB?
MongoDB should be used for  
* datasets that are very unstructured.
* the data stored is more complex than the primitive int,string,float,date. MongoDB can support storing arrays and nested documents.


MongoDB should not be used if 
* our working set size is too large. Working set size means the data that we actively use/query for. In our case, the upper bound of the working set size should be a year's data. 
* We need to aggregate over millions of records. The 100MB limit on the aggregation pipeline is a nice way to sandbox MongoDB and prevent it from consuming too much memory. But if it is exceeded, the pipeline has to use disk and this severely degrades performance.

 
## Our Use Cases 

#### For the Usage Analysis System 

These are the use cases where we need aggregations(basically,group by any dimension and count, with filters on any dimension)
  * Find out which features of an application are used the most, so that developers can spend more time optimizing those features
  * Find out the frequently used values so that they can be made default
  * It should be easy to add new application-specific dimensions
  * Provide support for multivalued dimensions    

We should go with MongoDB here because 
1. The current scale of the app-usage data seems to be well within the aggregation pipeline's capacity.  
2. With InfluxDB, the series cardinality makes it hard to tell how its going to scale. That is, the number of dimensions and the number of unique values of those dimensions and the number 
of unique combinations of the various dimension values(this is the series cardinality) can't be predicted.
3. MongoDB's JSON data model allows for richer and complex datatypes - multivalued dimensions could be stored as arrays and even indexed. This would be hard to do in InfluxDB or SQL.  
 

#### For the Performance Analysis System
 
Here we need real-time updates, monitoring and alerts :
  * To monitor memory,performance of an API call.
  * Provide an alerting mechanism that monitors these metrics in realtime and sends alerts if the metrics cross some pre-defined threshold
  
InfluxDB will do great here because
1. The dataset is likely to have a few large series instead of many medium-sized ones(unlike the usage analysis data) and the dimensions are less dynamic and more predictable. 
2. It can integrate with Grafana,Kapacitor,Telegraf.
Grafana can auto-refresh data at set time intervals. It makes use of InfluxDB's group by time() clause to intelligently pull data from the database. Scripted dashboards is a powerful feature to generate dashboards on the fly.
Kapacitor can send alerts if a performance measure crosses some pre-defined threshold. Telegraf is a metrics collector that can get cpu,memory,io,etc from any host and write to InfluxDB. It would be difficult to build the same functionality that these tools provide, around MongoDB.

 
And these are the use cases that must be implemented by the higher level API:
   * Provide a "user story" for each user, describing his typical workday.
   * Provide a flexible API for load-time breakdown
  
  
  