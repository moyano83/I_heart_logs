# I heart logs
## Table of contents:
1. [Chapter 1: Introduction](#Chapter1)
2. [Chapter 2: Data Integration](#Chapter2)
3. [Chapter 3: Logs and Real-Time Stream Processing](#Chapter3)
4. [Chapter 4: Building Data Systems with Logs](#Chapter4)


## Chapter 1: Introduction<a name="Chapter1"></a>
### What is a log
Most programmers think of log as a series of loosely structured requests, errors, or other messages in a sequence of rotating text files. Application log should allow for 
programmatic access. The log discuss here is more general and closer to what in the database or systems world might be called a commit log or journal. It is an append-only 
sequence of records ordered by time. Records in the log are stored in the order they were appended. Reads proceed from left to right. Each entry appended to the log is assigned
 a unique, sequential log entry number that acts as its unique key. The log entry number can be thought of as the timestamp of the entry. Describing this ordering as a notion of
  time seems a bit odd at first, but it has the convenient property of being decoupled from any particular physical clock.

### Logs in Databases
The usage in databases has to do with keeping in sync a variety of data structures and indexes in the presence of crashes. To make this atomic and durable, a database uses a 
log to write out information about the records it will be modifying before applying the changes to all the various data structures that it maintains. Since the log is 
immediately persisted, it is used as the authoritative source in restoring all other persistent structures in the event of a crash. This log can also be used to replicate data 
across databases, therefore:

    * The log is used as a publish/subscribe mechanism to transmit data to other replicas
    * The log is used as a consistency mechanism to order the updates that are applied to multiple replicas
    
### Logs in Distributed Systems
The log-centric approach to distributed systems arises from a simple observation called the state machine replication principle: _If two identical, deterministic processes 
begin in the same state and get the same inputs in the same order, they will produce the same output and end in the same state._ The application to distributed computing is 
that you can reduce the problem of making multiple machines all do the same thing to the problem of implementing a consistent log to feed input to these processes. The purpose 
of the log here is to squeeze all the nondeterminism out of theinput stream to ensure that each replica that is processing this input stays in sync. The discrete log entry 
numbers now act as a clock for the state of the replicas, you can describe the state of each replica by a single number: the timestamp for the maximum log entry that it has 
processed. Two replicas at the same time will be in the same state. This gives a discrete, event-driven notion of time that, unlike the machine’s local clocks, is easily 
comparable between different machines.

### Variety of Log-Centric Designs
The distributed systems literature commonly distinguishes two broad approaches to processing and replication. The _state machine model_ usually refers to an active-active 
model, where we keep a log of the incoming requests and each replica processes each request in log order. The_primary-backup model_, is to elect one replica as the leader, which
 processes requests in the order they arrive and logs the changes to its state that occur as a result of processing the requests. The other replicas apply the state changes 
 that the leader makes so that they will be in sync and ready to take over as leader, should the leader fail.

#### An Example
Say we want to implement a replicated arithmetic service that maintains a set of variables (initialized to zero) and applies additions, multiplications, subtractions, 
divisions, and queries on these values. Our remote web service based on HTTP will respond to the following commands:

```text
x? // get the current value of x
x+=5 // add 5 to x
x-=2 // subtract 2 from x
y*=2 // double y
```

In case of multiple servers processing this commands, the problem of having sync servers can be resolved in multiple ways:

    * The state-machine replication approach would involve first writing to the log the operation that is to be performed, then having each replica apply the operations in the 
    log order
    * In a primary-backup approach would choose one of the replicas to act as the primary (or leader or master) which would locally execute whatever command it receives in the 
    order requests 
    arrive, and it would log out the series of variable values that result from executing the commands. The remaining replicas would act as backups, they subscribe to this log 
    and passively apply 
    the new variable values to their local stores so they are ready to act as a master in case the current master fails
    
### Logs and Consensus
The distributed log can be seen as the data structure that models the problem ofconsensus.A log represents a series of decisions on the next value to append.

### Changelog 101: Tables and Events Are Dual 
There is a duality between a log of changes and a table: The log is similar to the list of all credits and debits a bank processes, while a table would be all the current 
account balances. If you have a log of changes, you can apply these changes in order to create the table and capture the current state. This table will record the latest state 
for each key. This log can also transform it to create all kinds of derived tables. You can see tables and events as dual: tables support data at rest and logs capture change.

 
## Chapter 2: Data Integration<a name="Chapter2"></a>
Data integration means making available all the data that an organization has to all the services and systems that need it.

### Data Integration: Two Complications
#### Data is More Diverse
Event data records things that happen rather than things that are. In web systems, this means user activity logging, as well as the machine-level events and statistics required
 to reliably operate and monitor a data center's machines. This type of event data shakes up traditional data integration approaches because it tends to be several orders of 
 magnitude larger than transactional data.

#### The Explosion of Specialized Data Systems
Specialized systems exist for OLAP, search, simple online storage, batch processing, graph analysis, and so on. The combination of more data of more varieties and a desire to 
get this data into more systems leads to a huge data integration problem.

### Log-Structured Data Flow
The log is the natural data structure for handling data flow between systems. The recipe is very simple: _Take all of the organization’s data and put it into a central log for 
real-time subscription_. Each logical data source can be modeled as its own log. Each subscribing system reads from this log as quickly as it can, applies each new record to its
 own store, and advances its position in the log, the log concept gives a logical clock for each change against which all subscribers can be measured. The log also acts as a 
 buffer that makes data production asynchronous from data consumption. You can think of the log as acting as a kind of messaging system with durability guarantees and strong 
 ordering semantics. In distributed systems, this model of communication is sometimes named as _atomic broadcast_.

### My Experience at LinkedIn
Key factors to take in consideration:

    * Making data available in a new processing system (Hadoop) unlocked many possibilities. Many new products and analysis came from simply putting together multiple pieces of
     data that had 
    previously been locked up in specialized systems
    * Reliable data loads would require deep support from the data pipeline (capture structure, schema changes...)
    * New data sources requires a considerable effort to be integrated. The custom ingestion processes were not appropriate
    
From this factors, it can be derived that it is important to isolate the consumers from the source of data. Consumers ideally would need to integrate with a single data 
repository given them access to all the data.
    
### Relationship to ETL and the Data Warehouse
The data warehouse is meant to be a repository for the clean, integrated data structured to support analysis. The data warehousing methodology involves periodically extracting 
data from source databases, munging it into some kind of understandable form, and loading it into a central data warehouse, although the mechanics of getting this are a bit out
 of date. A data warehouse is a piece of batch query infrastructure that is well suited to many kinds of reporting and ad hoc analysis, particularly when the queries involve 
 simple counting, aggregation, and filtering. But having a batch system be the only repository of clean, complete data means the data is unavailable for systems requiring a 
 real-time feed: real-time processing, search indexing, monitoring systems, and so on.

### ETL and Organizational Scalability
The classic problem of the data warehouse team is that they are responsible for collecting and cleaning all the data generated by every other team. Data producers are often not
 very aware of the use of the data in the data warehouse and end up creating data  that is hard to extract or requires heavy, hard-to-scale transformation to get it into usable 
 form. A better approach is to have a central pipeline, the log, with a well-defined API for adding data. The addition of new storage systems is of no consequence to the data 
 warehouse team, as they have a central point of integration.

### Where Should We Put the Data Transformations?
This architecture also raises a set of different options for where a particularcleanup or transformation can reside:

   * It can be done by the data producer prior to adding the data to the companywide log
    * It can be done as a real-time transformation on the log (which in turn produces a new, transformed log)
    * It can be done as part of the load process into some destination data system
    
The best model is to have the data publisher docleanup prior to publishing the data to the log, ensuring that the data is in a canonical form and doesn’t retain any holdovers 
from the particular code that produced it or the storage system in which it might have been maintained. Any kind of value-added transformation that can be done in real-time 
should be done as post-processing on the raw log feed that was produced. Also, only aggregation that is specific to the destination system should be performed as part of the 
loading process.

### Decoupling Systems
The typical approach to activity data in the web industry is to log it out to text files where it can be scrapped into a data warehouse or into Hadoop for aggregation and 
querying, but this approach couples the data flow to the data warehouse’s capabilities and processing schedule. However, with a log centric event data handling, decoupled 
systems would have to decide how to consume and transform the data.

### Scaling a Log
If you want to keep a commit log that acts as a multisubscriber real-time journal of everything happening on a consumer-scale website, scalability will be a primary challenge. 
Some techniques to achieve this are:

    * Partitioning the log
    * Optimizing throughput by batching reads and writes
    * Avoiding needless data copies
    
Each partition is a totally ordered log, but there is no global ordering between partitions The writer controls the assignment of the messages to a particular partition, with 
most users choosing to partition by some kind of key. Partitioning allows log appends to occur without coordination between shards, and allows the throughput of the system to 
scale linearly with the Kafka cluster size while still maintaining ordering within the sharding key. Each partition is replicated across a configurable number of replicas, each
 of which has an identical copy of the partition’s log. At any time, a single partition will act as the leader; if the leader fails, one of the replicas will take over as leader.
The guarantees that we provide are that each partition is order preserving, and Kafka guarantees that appends to a particular partition from a single sender will be delivered 
in the order they are sent. A log, like a filesystem, is easy to optimize for linear read and write patterns. The log can group small reads and writes together into larger, 
high-throughput operations. Finally, Kafka uses a simple binary format that is maintained between in-memory log, on-disk log, and in-network data transfers.


## Chapter 3: Logs and Real-Time Stream Processing<a name="Chapter3"></a>
Stream processing is basically infrastructure for **continuous** data processing (as opposed to batch processing). A stream processing system produces output at a 
user-controlled frequency instead of waiting for the “end” of the data set to be reached.

### Data Flow Graphs
Stream processing allows us to also include feeds computed off of other feeds and not only feeds or logs of primary data—that is, the events and rows of data directly produced 
in the execution of various applications. These derived feeds can encapsulate arbitrary complexity and intelligence in the processing, so they can be quite valuable. 
A stream processing job, for our purposes, will be anything that reads from logs and writes output to logs or other systems. The logs used for input and output connect these 
processes into a graph of processing stages. Using a centralized log in this fashion, you can view all the organization’s data capture, transformation, and flow as just a 
series of logs and the processes that read from them and write to them.

### Logs and Stream Processing
Logs in streaming are needed (instead of direct messaging) for several reasons:

    * It makes the data set to be multisubscriber
    * To ensure that order is maintained in the processing done by each consumer of data
    * To provide buffer‐ ing and isolation to the individual processes: the log acts as a very large buffer that allows the process to be restarted or fail without slowing down 
    other parts of the processing graph
    
### Reprocessing Data: The Lambda Architecture and an Alternative
An interesting application of this kind of log-oriented data modeling is the Lambda Architecture.

#### What is a Lambda Architecture and How Do I Become One?
The way this works is that an immutable sequence of records is captured and fed into a batch-and-stream processing system in parallel. You implement your transformation logic 
twice, once in the batch system and once in the stream processing system. Then you stitch together the results from both at query time to produce a complete answer.

#### What’s Good About This? 
The Lambda Architecture emphasizes retaining the original input data unchanged. This architecture highlights the problem of reprocessing data.

#### And the Bad...
The problem with the Lambda Architecture is that maintaining code that needs to produce the same result in two complex distributed systems is very complex. One proposed 
approach to fixing this is to have a language or framework that abstracts over both the real-time and batch framework, but still, the operational burden of running and 
debugging two systems is going to be very high.

#### An Alternative
An alternative approach to the lambda architecture can be actually very simple:

    * Use Kafka or some other system that will let you retain the full log of the data you want to be able to reprocess and that allows for multiple subscribers
    * To reprocess, start a second stream processing job that starts processing from the beginning of the retained data, but direct this output data to a new output table
    * When the second job has caught up, switch the application to read from the new table
    * Stop the old version of the job and delete the old output table
    
Unlike the Lambda Architecture, in this approach you only do reprocessing when your processing code changes and you actually need to recompute your results. The Lambda 
Architecture requires running both reprocessing and live processing all the time, whereas the second proposal requires running the second copy of the job when you  need 
reprocessing. This proposal requires temporarily having twice the storage space in the output database and requires a database that supports high-volume writes for the reload.

### Stateful Real-Time Processing
Some real-time stream processing is just stateless record-at-a-time transformation, but many of the uses are more sophisticated counts, aggregations, or joins over windows in 
the stream. The simplest alternative would be to keep state in memory. However, if the process crashed, it would lose its intermediate state. An alternative is to simply store 
all state in a remote storage system and join over the network to that store.
An alternative is that a stream processor can keep its state in a local table or index. The contents of this store are fed from its input streams and it can journal out a 
changelog for this local index that it keeps to allow it to restore its state in the event of a crash and restart. When the process fails, it restores its index from the changelog.
In this approach, the state of the processors is also maintained as a log. A changelog can be extracted from a database and indexed in different forms by various stream 
processors to join against event streams.

### Log Compaction
In Kafka, cleanup has two options depending on whether the data contains pure event data or keyed updates (events that specifically record state changes in entities identified 
by some key). For event data, Kafka supports retaining a window of data. The window can be defined in terms of either time (days) or space (GBs). An important property for keyed 
data is that you can replay it from the log to recreate the state of the source system.  


## Chapter 4: Building Data Systems with Logs<a name="Chapter4"></a>
There is an analogy here between the role a log serves for data flow inside a distributed database and the role it serves for data integration in a larger organization. In both 
cases, it is responsible for data flow, consistency, and recovery. There is one possible simplification in the handling of data in the move to distributed systems: coalescing 
many little instances of each system into a few big clusters.

### Unbundling?
The explosion of different systems is caused by the difficulty of build‐ ing distributed data systems. There are three different possibilities that would lead to:

    * The separation of systems remains more or less as it is for a good deal longer
    * There could be a reconsolidation in which a single system with enough generality starts to merge all the different functions back into a single uber-system
    * Open source allows another possibility: data infrastructure could be unbundled into a collection of services and application-facing system API. You can piece these 
    ingredients together to create a vast array of possible systems
    
### The Place of the Log in System Architecture
Here are some things a log can do:

    * Handle data consistency by sequencing concurrent updates to nodes
    * Provide data replication between nodes
    * Provide “commit” semantics to the writer (like ACKs)
    * Provide the external data subscription feed from the system
    * Provide the capability to restore failed replicas that lost their data or bootstrap new replicas
    * Handle rebalancing of data between nodes  
    
The majority of what is left in order to have a distributed system is related to the final client-facing query API and indexing strategy. A pattern used to build out many real-time
 query systems it to take as input either the log directly generated by database updates or else a log derived from other real-time processing, and provide a particular 
 partitioning, indexing, and query capability on top of that data stream.
 
### Conclusion
Logs give us a principled way to model changing data as it cascades through a distributed system. Having this kind of basic abstraction in place gives us a way of gluing 
together disparate data systems, process‐ ing real-time changes, as well as a being an interesting system and application architecture in its own right.