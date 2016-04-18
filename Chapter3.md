### MongoDB
MongoDB is a document-oriented database. In SQL terminology, a document is equivalent to a record, a MongoDB collection is equivalent to a SQL table. A short summary of MongoDB's features :
* Developer friendly API and Community Support
* Pluggable Storage Engine(data structure, compare to Mmap!)
* Replication and Automatic Failover
* Horizontal Scaling

How MongoDB stores data : 
MongoDB stores data as BSON(Binary JSON), which is a superset of JSON(JavaScript Object Notation). JSON is light-weight, human-readable data interchange format. It is becoming the default format of data interchange on the web, and is widely used in the "noSQL movement" because a JSON result can be used directly in the application layer and most proramming languages support JSON. JSON supports number(64-bit float),string,boolean,array and object. 
So why BSON? 
* BSON extends the JSON datatypes to include datatypes like date,int32,int64,double and binary. 
* It's traversable. For the JSON document {"hello":"world"}, this is what a BSON  document looks like   
  \x16\x00\x00\x00               // total document size   
  \x02                           // 0x02 = type String  
  hello\x00                      // field name  
  \x06\x00\x00\x00world\x00      // field value (size of value, value, null terminator)   
  \x00                           // 0x00 = type EOO ('end of object')  

Storing the length of the document and the fields allows scanning through the document for a specific field very quickly. So, this metadata that BSON stores with each document can be a little ineffecient in terms of storage, but that's a tradeoff we make for fast traversal.
* It uses C data types, so drivers can efficiently convert BSON to an equivalent data structure in the programming language.

#### Pluggable Storage Engine
A storage engine or a database engine is the part of the database that determines how the database stores data on disk and in memory It also decides what indexes, what kind of locks/concurrency control is supported, and also handles the CRUD operations. So storage engines can have a big impact on performance - one storage engine could be optimised for high write throughput, but may not work well with an update-heavy workload.MongoDB versions 3.0 and up, ship with a pluggable storage engine. That is, you can choose between MMaPv1, the original MongoDB storage engine that uses memory mapped files and WiredTiger - built by WiredTiger.Inc(now a part of MongoDB. 
WiredTiger), now the default in version 3.2. 
#### WiredTiger
WiredTiger brings 
* Document Level Concurrency - this is WiredTiger's flagship feature that improves performance. Earlier versions of MongoDB offered only collection-level locks.
* Better performance with NFS - Since WiredTiger does not use memory mapping, its performance is decent on NFS.
* Compression - uses snappy by default, can be configured to use zlib. Prefix compression is used for storing indexes.
* Wired Tiger uses a BTree, the data structure used for database indexes, for accessing its datafiles. Btrees allow for insertions, deletions and searches in O(log n) time.

#### MMaPv1 
The storage engine got its name because it uses UNIX's mmap() system call to persist data. This was a choice made by MongoDB in its formative years that allowed them to focus on developing MongoDB's features and leaving the task of persisting data to the operating system. MMaPv1 did not work well with NFS as a result of this. Also it did not offer any compression - BSON data stored on disk was the same as BSON data in memory.

#### Replication and Fault Tolerance 
MongoDB uses replica sets to provide fault tolerance, failover and high availability. A replica set is a group of mongod processes that maintain the same data.  
**Replica set members**  
* Primary - the primary is "elected" by the other members of the replica set. All writes and by default all reads go to the primary. The primary maintains a list of all the operations it performs on its data in a collection called the "oplog". 
* Secondary - the secondary syncs the oplog from the primary and performs the same operations on its data. This sync-ing is asyncronous. In case the primary fails, one of the secondaries become the primary. A replica set can have upto 50 members but only 7 of them can vote. 
* Arbiter - this is a lightweight mongod process that does not maintain a copy of the data, but is used to have odd number of votes in case of a failover.
![](https://docs.mongodb.org/manual/_images/replica-set-primary-with-secondary-and-arbiter.png)

Replica sets allow secondaries to be configured with various options 
* Priority 0 - Changing to priority 0 from the default 1 means that this secondary can never be primary in case of a failover.
* Hidden - these are priority 0 members that cannot be accessed by client applications. Can be used for a different kind of workload from the primary.
* Delayed - these are priority 0 members that maintain a delayed version of the data. This can be used for rolling backups. These members should be hidden, and should not be queried directly.
* Read preferences - by default, the read preference is "primary". But other preferences can be set for different kind of workloads - secondary, primary preferred, seondary preferred and nearest(useful for geographically distributed recplica sets). Reads on secondaries could be used in cases where "eventually consistent" results are acceptable.


### Sharding 
MongoDB's horizontal scaling method is sharding - this allows data of a database to be distributed across many shards. A shard could be a replica set or just a single mongod process.
  
![](https://docs.mongodb.org/manual/_images/sharded-cluster-production-architecture.png)
Sharding is done on a per-collection basis, based on a shard key. The shard key should be an indexed field that appears in every field in the collection. The choice of a shard key depends on the access pattens. Sharding is irreversible, and performance may suffer if access patterns change after sharding. 
**Components of a Sharded Cluster**
* Shards - these store a portion of the data in the collection. Each shard stores data as "chunks". A chunk is a contiguous range of shard key values. The balancer is a background process that manages the even distribution of chunks across shards. Tag-aware sharding is a technique that can be used to override the balancer's rules.
* Mongos - this is a lightweight application that abstracts away the internals of the sharded cluster from the client applications. It takes care of routing the queries to the right shards. When it starts up, it caches the metadata about the shards( which chunk is where) from the config servers.
* Config servers - these maintain metadata about the sharded cluster. These are mongod processes that must be deployed as a replica set, and failure of all the config servers could render the cluster unusable.
The shard key should be chosen is such a way that it distributes writes and isolates queries. Distributed writes allow writes to be spread across shards so that write performance is not restricted to the capacity of one shard. Queries to shards can be targetted - sent to one or two shards or broadcast - sent to all shards.

















