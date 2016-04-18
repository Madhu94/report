### InfluxDB
This is an open-source time-series database. A few assumptions made about time-series data:  
* Time-series data are usually write-once and read-many type of workloads. Updates and deletes to individual points are rare. Deletes are usually done in bulk and for cold data. InfluxDB emphasises "CR-ud" over CRUD, updates and deletes are not performance, but writes and reads are optimised.
* Individual points are not important. InfluxDB's operations are optimised for large aggregations.
* Each record(or "point") contains a timestamp. The timestamps are usually in increasing order.
* Scale is very important. High throughput for writes, and good performance for aggregations over millions of points. 
* If the same data is sent multiple times, it treated as duplicate data. This eliminates the need for conflict resolution. Each database handles conflict resolution differently - Riak uses vector clocks, Cassandra just takes the last update as the right one. Eliminating this conflict resolution simplifies InfluxDB's design but it may cause data loss in certain circumstances.

#### Data Model  
InfluxDB stores data as _points_ in _measurements_(like a sql table. But measurement is more of a logical partitioning of the data). Each point has optional _tags_ and atleast one or more _fields_. A unique combination of measurement name and tagset is called a _series_.  
Tags are indexed, fields are not. The number of series in a measurement is called _series cardinality_. The series cardinality affects query performance and may cause out of memory exceptions if it is too high.  

#### Query Model 
InfluxDB has drivers in python, java, node.js and other languages. It has an admin UI and a command line shell which provides a "sql-ish" syntax. Internally, the drivers and the shell send the query as a rest call to the influxd process. Groupby can only be done on tags, and aggregations like counts can only be dones on fields.

#### Storage engine 
The original InfluxDB storage engine was LevelDB, which used the log-structured merge tree, for high write throughput. Other databases like Cassandra and Riak also used the LSM for their storage engine.The LSM was not performant on large scale deletes, which was a typical need in time series data. A delete was as expensive as an insert. So, a memory mapped B+ tree was tried in the next storage engine, BoltDB, but it could not handle high IOPS. Finally a variant of the LSM, called the time-structured-merge tree was used to get high write throughput and good speed for aggregation queries.   
TSM basically has three components :  
* Write Ahead Log(WAL) - all writes go the write-ahead-log. Before that the incoming points, and all metadata associated with that point (tags,series,etc) are cached. Then the data is serialized, compressed using snappy and written to WAL. Only then is the write acknowledged. So as soon as the write is acknowledged, we can query the data, since its been cached.
* Index Files - Periodically the WAL flushes to the indexfiles. Indexfiles undergo _compactions_, basically merging two index files into one(or merging two time windows into a larger time interval).

The WAL is flushed on restart, so having a lot of tags and series could slow down time needed to start up. 


#### Tick Stack and Grafana  
InfluxDB is part of the TICK stack, a suite of tools built around InfluxDB for time-series data. The whole stack is written in Go. 
* T - Telegraf, an open source metrics collector. There are many others, but Telegraf was built to easily integrate with InfluxDB's data model. Telegraf can
collect system metrics like cpu,memory from any host, it has plugins for mysql,mongodb and many other datasources and even messaging queues like apache kafka.
* I - InfluxDB, an open source time-series database. 
* C -Chronograf, a closed-source(for now) visualization tool for influxDB. Does not seem to be a mature tool yet.
* K - Kapacitor, a data processing and alerting tool for influxdb. The alerting can be done as part of a stream or a batch task, using TICKScript, a Domain
 Specific Language(DSL) for Kapacitor. If the alert criteria is met, an event handler is called(post to a log file, a http post, email and many other
 event handlers are supported). Information about the event is also passed to the handler and can be used inside it. Custom anomaly detection algorithms
 can be created using python scripts(numpy,scipy algorithms) and referencing the python script in our TICKScript program. Any valid input for influxdb
 is also valid for kapacitor so that alerts can be triggered at collection time itself.

#### Grafana
Grafana is a fork of Kibana 3, with a focus on time-series data visualization and graphs (built for graphite originally).
It can create dashboards comprising graphs, tables, singlestats(queries that reduce to single number). A dashboard can pull data from multiple datasources
(multiple influxdb instances) and constuct panels with them. A panel is a graph,table or singlestat. However each panel can only be tied to a single datasource.
Grafana works well for aggregation queries - based on the size of the panel and the number of points being queried, it uses influxdb's group by time() clause
to reduce the number of points returned. Non-aggregation queries that do not use the group by time() feature should atleast use time filters in the where
clause to avoid slowing down Grafana. Scripted dashboards is a powerful feature to generate dynamic dashboards using javascript.


