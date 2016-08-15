## Usage Analysis System 
### Objective
Usage Tracking gives developers an insight into how an application is being used. It could lead to optimal use of a developer's time. Performance analysis tracks client side performance metrics - like GC,memory and server side metrics like response time of the server, time taken by server to serialize data,etc. The requirements of both systems are widely different - performance analysis data is likely to be of a larger scale than usage data, and real-time response would be more important here. That is, it may be less tolerant to "eventual consistency" than usage data. It might be better if the same datastore were used for both usage and performance analysis - a tool correlating usage and performance data could be built on top of it. The most important requirement of a usage analysis system is that we should be able to group and query by any dimension/feature and it should be easy to add any custom dimension to the database. Flexibility is more important than real-time updates, so eventual consistency is tolerable here. Maybe an AP(Availability+Partition Tolerance) system is best suited for this case.

### Drawbacks of SQL 
* The structure of the data must be known in advance. Changing the schema later can be costly. This does not scale well with modern web applications where the structure of the data being logged could change rapidly - new dimensions need to be added and some older dimensions may not make sense anymore. NoSQL stores allow pushing new dimensions on the fly, it is purely the decision of the developer what to track.(However, there is an _implicit_ schema in the sense that you need to be aware of the structure of the logged data to extract meaningful information from a noSQL datastore). 
* In SQL, there is an impedence mismatch between the relational model of an rdbms and the object-oriented programming style. Object Relational Mappers(ORMs) help bridge the gap, but most noSQL databases like MongoDB offer drivers which adopt an object oriented approach to writing and reading data from the database.

### Block Diagram
![](http://s3.postimg.org/5suef4nnn/block_diagram_1.png)

Web Applications are the sources of the usage data and these are stored in MongoDB through a writer API. This data can be queried through a python API or javaScript API. For queries that are run frequently over large datasets, pre-computation could avoid processing the same records multiple times and improve response time.


## Use Cases : 
#### Usage Analysis:
* To find out which components/widgets of a web application are most used. By knowing this, developer time can be optimally used. They can focus on  those components that are heavily used and upgrade them.
* This data can also be used for defaults analysis. The developer can then figure out if the default options are indeed the most frequently used and change the defaults if they are not so frequently used.
* Each user’s data can be analysed to create a “user story” which outlines what he typically does on a workday. This data can be used to customize user experience.
* Application specific dimensions can be added and removed easily.
* Providing support for multi-valued dimensions.
* Load time breakdown is the depiction of the events which takes place from when the page The analysis system can be used to provide a flexilble API for load time breakdown.
* The usage analysis system can be extensible to allow developers to build on top of the existing API to gain more information.

#### Performance Analysis :

* To monitor the memory usage of applications,garbage collection, time taken to make an API call,etc.
* Provide an alerting mechanism that monitors some metric(like memory used), in pseudo real-time, and sends alerts if the metrics indicate any cause for concern.
