# I heart logs
## Table of contents:
1. [Chapter 1: Introduction](#Chapter1)
2. [Chapter 2: Data Integration](#Chapter2)
3. [Chapter 3: Logs and Real-Time Stream Processing](#Chapter3)
4. [Chapter 4: Building Data Systems with Logs](#Chapter4)


## Chapter 1: Introduction<a name="Chapter1"></a>
### What is a log
Most programmers think of log as a series of loosely structured requests, errors, or other messages in a sequence of rotating text files. Application log should allow for programmatic access.
The log discuss here is more general and closer to what in the database or systems world might be called a commit log or journal. It is an append-only sequence of records ordered by time. Records 
in the log are stored in the order they were appended. Reads proceed from left to right. Each entry appended to the log is assigned a unique, sequential log entry number that acts as its unique key.
The log entry number can be thought of as the timestamp of the entry. Describing this ordering as a notion of time seems a bit odd at first, but it has the convenient property of being decoupled 
from any particular physical clock.

### Logs in Databases
The usage in databases has to do with keeping in sync a variety of data structures and indexes in the presence of crashes. To make this atomic and durable, a database uses a log to write out 
information about the records it will be modifying before applying the changes to all the various data structures that it maintains. Since the log is immediately persisted, it is used as the 
authoritative source in restoring all other persistent structures in the event of a crash. This log can also be used to replicate data across databases, therefore:

    * The log is used as a publish/subscribe mechanism to transmit data to other replicas
    * The log is used as a consistency mechanism to order the updates that are applied to multiple replicas
    
### Logs in Distributed Systems
The log-centric approach to distributed systems arises from a simple observation called the state machine replication principle: _If two identical, deterministic processes begin in the same state 
and get the same inputs in the same order, they will produce the same output and end in the same state._ The application to distributed computing is that you can reduce the problem of making 
multiple machines all do the same thing to the problem of implementing a consistent log to feed input to these processes. The purpose of the log here is to squeeze all the nondeterminism out of the
input stream to ensure that each replica that is processing this input stays in sync. The discrete log entry numbers now act as a clock for the state of the replicas, you can describe the state of 
each replica by a single number: the timestamp for the maximum log entry that it has processed. Two replicas at the same time will be in the same state. This gives a discrete, event-driven notion 
of time that, unlike the machine’s local clocks, is easily comparable between different machines.

### Variety of Log-Centric Designs
The distributed systems literature commonly distinguishes two broad approaches to processing and replication. The _state machine model_ usually refers to an active-active model, where we keep a log
of the incoming requests and each replica processes each request in log order. The_primary-backup model_, is to elect one replica as the leader, which processes requests in the order they arrive 
and logs the changes to its state that occur as a result of processing the requests. The other replicas apply the state changes that the leader makes so that they will be in sync and ready to take
over as leader, should the leader fail.

#### An Example
Say we want to implement a replicated arithmetic service that maintains a set of variables (initialized to zero) and applies additions, multiplications, subtractions, divisions, and queries on 
these values. Our remote web service based on HTTP will respond to the following commands:

```text
x? // get the current value of x
x+=5 // add 5 to x
x-=2 // subtract 2 from x
y*=2 // double y
```

In case of multiple servers processing this commands, the problem of having sync servers can be resolved in multiple ways:

    * The state-machine replication approach would involve first writing to the log the operation that is to be performed, then having each replica apply the operations in the log order
    * In a primary-backup approach would choose one of the replicas to act as the primary (or leader or master) which would locally execute whatever command it receives in the order requests 
    arrive, and it would log out the series of variable values that result from executing the commands. The remaining replicas would act as backups, they subscribe to this log and passively apply 
    the new variable values to their local stores so they are ready to act as a master in case the current master fails
    
### Logs and Consensus
The distributed log can be seen as the data structure that models the problem ofconsensus.A log represents a series of decisions on the next value to append.

### Changelog 101: Tables and Events Are Dual 
There is a duality between a log of changes and a table: The log is similar to the list of all credits and debits a bank processes, while a table would be all the current account balances. If you have
a log of changes, you can apply these changes in order to create the table and capture the current state. This table will record the latest state for each key. This log can also transform it to 
 create all kinds of derived tables. You can see tables and events as dual: tables support data at rest and logs capture change.

 
## Chapter 2: Data Integration<a name="Chapter2"></a>
Data integration means making available all the data that an organization has to all the services and systems that need it.

### Data Integration: Two Complications
#### Data is More Diverse
Event data records things that happen rather than things that are. In web systems, this means user activity logging, as well as the machine-level events and statistics required to reliably operate 
and monitor a data center's machines. This type of event data shakes up traditional data integration approaches because it tends to be several orders of magnitude larger than transactional data.

#### The Explosion of Specialized Data Systems
Specialized systems exist for OLAP, search, simple online storage, batch processing, graph analysis, and so on. The combination of more data of more varieties and a desire to get this data into more 
systems leads to a huge data integration problem.

### Log-Structured Data Flow
The log is the natural data structure for handling data flow between systems. The recipe is very simple: _Take all of the organization’s data and put it into a central log for real-time subscription_.
Each logical data source can be modeled as its own log. Each subscribing system reads from this log as quickly as it can, applies each new record to its own store, and advances its position in the 
log, the log concept gives a logical clock for each change against which all subscribers can be measured. The log also acts as a buffer that makes data production asynchronous from data consumption.
You can think of the log as acting as a kind of messaging system with durability guarantees and strong ordering semantics. In distributed systems, this model of communication is sometimes named as 
_atomic broadcast_.

### My Experience at LinkedIn
Key factors to take in consideration:

    * Making data available in a new processing system (Hadoop) unlocked many possibilities. Many new products and analysis came from simply putting together multiple pieces of data that had 
    previously been locked up in specialized systems
    * Reliable data loads would require deep support from the data pipeline (capture structure, schema changes...)
    * New data sources requires a considerable effort to be integrated. The custom ingestion processes were not appropriate
    
From this factors, it can be derived that it is important to isolate the consumers from the source of data. Consumers ideally would need to integrate with a single data repository given them access 
to all the data.
    
### Relationship to ETL and the Data Warehouse
The data warehouse is meant to be a repository for the clean, integrated data structured to support analysis. The data warehousing methodology involves periodically extracting data from source 
databases, munging it into some kind of understandable form, and loading it into a central data warehouse, although the mechanics of getting this are a bit out of date.
A data warehouse is a piece of batch query infrastructure that is well suited to many kinds of reporting and ad hoc analysis, particularly when the queries involve simple counting, aggregation, and
filtering. But having a batch system be the only repository of clean, complete data means the data is unavailable for systems requiring a real-time feed: real-time processing, search indexing, 
 monitoring systems, and so on.

### ETL and Organizational Scalability
The classic problem of the data warehouse team is that they are responsible for collecting and cleaning all the data generated by every other team. Data producers are often not very aware of the use
of the data in the data warehouse and end up creating data  that is hard to extract or requires heavy, hard-to-scale transformation to get it into usable form. A better approach is to have a 
central pipeline, the log, with a well-defined API for adding data. The addition of new storage systems is of no consequence to the data warehouse team, as they have a central point of integration.

### Where Should We Put the Data Transformations?
This architecture also raises a set of different options for where a particularcleanup or transformation can reside:

   * It can be done by the data producer prior to adding the data to the companywide log
    * It can be done as a real-time transformation on the log (which in turn produces a new, transformed log)
    * It can be done as part of the load process into some destination data system
    
The best model is to have the data publisher docleanup prior to publishing the data to the log, ensuring that the data is in a canonical form and doesn’t retain any holdovers from the particular 
code that produced it or the storage system in which it might have been maintained. Any kind of value-added transformation that can be done in real-time should be done as post-processing on the raw
log feed that was produced. Also, only aggregation that is specific to the destination system should be performed as part of the loading process.

### Decoupling Systems
The typical approach to activity data in the web industry is to log it out to text files where it can be scrapped into a data warehouse or into Hadoop for aggregation and querying, but this 
approach couples the data flow to the data warehouse’s capabilities and processing schedule. However, with a log centric event data handling, decoupled systems would have to decide how to consume 
and transform the data.

### Scaling a Log
If you want to keep a commit log that acts as amultisubscriber real-time journal of everything happening on a consumer-scale website, scalability will be a primary challenge. Some techniques to 
achieve this are:

    * Partitioning the log
    * Optimizing throughput by batching reads and writes
    * Avoiding needless data copies
    
Each partition is a totally ordered log, but there is no global ordering between partitions The writer controls the assignment of the messages to a particular partition, with most users choosing to
partition by some kind of key. Partitioning allows log appends to occur without coordination between shards, and allows the throughput of the system to scale linearly with the Kafka cluster size 
while still maintaining ordering within the sharding key. Each partition is replicated across a configurable number of replicas, each of which has an identical copy of the partition’s log. At any 
time, a single partition will act as the leader; if the leader fails, one of the replicas will take over as leader.
The guarantees that we provide are that each partition is order preserving, and Kafka guarantees that appends to a particular partition from a single sender will be delivered in the order they are 
sent. A log, like a filesystem, is easy to optimize for linear read and write patterns. The log can group small reads and writes together into larger, high-throughput operations. Finally, Kafka 
uses a simple binary format that is maintained between in-memory log, on-disk log, and in-network data transfers.


## Chapter 3: Logs and Real-Time Stream Processing<a name="Chapter3"></a>
Stream processing is basically infrastructure for **continuous** data processing (as opposed to batch processing). A stream processing system produces output at a user-controlled frequency instead 
of waiting for the “end” of the data set to be reached.

### Data Flow Graphs
