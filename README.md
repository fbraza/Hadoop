# Hadoop fundamentals

**Apache Hadoop** is a tightly integrated ecosystem of different software products built to provide scalable and reliable distributed storage and distributing processing. The technical foundations of Hadoop stem from successive innovative approaches proposed and published by Google in the 2000's to produce:

- reliable storage (Google file system with become Hadoop Distributed File System, `HDFS`)
- reliable processing (`MapReduce`)
- low-latency random access queries on hundreds to thousands unreliable servers

The key innovation in this approach was to distribute data and split-up computations on this data in small chunks over different machines / servers. By design distributed systems scale very well. As each machine is independent, scaling the infrastructure only consists of adding some servers (horizontal scaling).

Distributed systems are:
- *Scalable*
- *Open* (easy to modify the features)
- *Heterogeneous*, meaning it can deal with several different type of data:
    - structured data (RDBMS, strongly typed with fixed schema)
    - Semi-structured data (JSON, XML, NoSQL databases)
    - Unstructured data (images, sound or videos)
- Permit to ensure the 3 main **V** on Big Data: *volume*, *velocity* and *variety*
- However distributed systems are *hard to set up* and *complex to maintain and deploy*

##  The Hadoop ecosystem

Hadoop is an open source implementation of these techniques. At its core, it offers a distributed file system (HDFS) and a means of running processes across a cluster of servers (YARN, Yet Another Ressource Manager). The original distributed processing application built on Hadoop was MapReduce, but since a wide range of additionnal software frameworks and librairies have emerged around Hadoop.

## What is a cluster?

A cluster is a bunch of servers grouped together to provide one or more functions such as storage or computation. On Hadoop clusters are deployed following a **master / slaves (or workers)** architecture. 
Slaves machines are where the real work happens - these machines store data, perform computations, offer services like lookups and search and much more. Master machines are responsible for coordination, maintaining metadata about the data and services on the slaves machines and ensuring that the services keep running even if some slaves machines fail. Typically there are two or three master machines for redundancy to avoid any single point of failure. 
A cluster is scaled up by adding more slave servers. Often we want to allow access to the clusters by users by providing some machines to act as *gateway* or *edge* servers. These servers often do not run any services at all but provide access cluster services.

## Core Components

**HDSF**, **YARN** and **MapReduce** represent the core building blocks of the Hadoop infrastructure.

### HDFS

The Hadoop distributed file system is the scalable, fault-tolerant file system for Hadoop. It is optimized to store very large amount of **immutable data* (*Write Once Read Many*) with files being typically accessed in long sequential scan. When storing data, HDFS breaks up a file into *blocks* of configurable size, usually 128MB and store *replicas* of each block on multiple servers ensuring data resilience and parallelism.
Each worker node runs a deamon called a **DataNode** which accepts new block of data and write them to its local disks. The DataNode is only aware of blocks and their IDs: it does not have the knowledge about the file to which a particular block or replicate belongs. 
This information is hold by the **NameNode** which runs on the master servers and is responsible for maintaining a mapping of files to the blocks as well as metadata about the files themselves (names, permissions, attributes). All this data is saved in memory and is by definition non-persistent. 

Having these informations in RAM has some trade-offs:
- This allows to load very fast these details / data
- However we are limited by the size of the RAM
- One consequence of this is that we should avoid putting millions or billions of small files ! This would just overload the memory

![](Images_course_review/Name_and_Data_nodes.png)

It is also possible to run a *secondary NameNode* which despite its name does not act as the primary NameNode. Its main role is to **keep a copy of the File System image of the primary NameNode** which contains all details and metadata. This image is periodically built and updated by the secondary NameNode through interaction with the **JournalNodes** which contains all the logs related to the primary NameNode operations. The secondary NameNode runs on a separate machine as it requires CPU power and as much memory as the NameNode.</p>

![](Images_course_review/Secondary_namenode.png)

### HDFS high availability

The combination of replicating NameNode data and using secondary NameNode to create checkpoints do protect against data loss but do not provide high availability of the file system. In this configuration the primary NameNode is still a **single point of failure**. If the primary NameNode fails all clients would be unable to read or write data. In such an event the whole Hadoop system would be out of service until a new NameNode is brought online.
Since Hadoop 2 there is now the implementation of a pair of NameNodes in an **active-standby** configuration. In case of the failure of the active primary NameNode, the standby takes over and ensures continuity of services. To reach this goal a few consideration have to be taken:

- The standby NameNode has to synchronize its state with the active NameNode and keeps reading new entries.
- DataNodes must send block reports to both NameNodes because the block mapping are stored in memory and not on disk.
- We must ensure that only one NameNode is in active state to avoid any **split / brain** scenario (thank to *ZooKeeper*).
- The standby NameNode also ensures the role of the secondary NameNode by taking periodic checkpoints of the active NameNode state.

![](Images_course_review/ha_archi.png)

### HDFS: Practice with Labs

In the *practice with labs section you will find a link toward some practice. The lab consists of a set of instructions that you can do by yourself or with help from the walk-through which is my solution for the lab. 
All the labs have been performed in the frame of the Data Engineering applied master proposed by the Data SicenceTech institute. These were possible with the help of a consulting company Adaltas which provided us access to a Hadoop cluster. 
If you do not have access to such service you can still follow the walk-through and these notes to get an idea about the way the Hadoop infrastructure and services work or deploy an Hadoop cluster locally. 
The general setup for the lab looks as depicted in the figure below. We basically connect by SSH and through VPN to an **edge node** from which we can perfom local operations and send HDFS-related operations.

![](Images_course_review/lab_setup.png)

### YARN

Apache YARN (**Y**et **A**nother **R**esource **N**egotiator). is the cluster resource manager of Hadoop that was introduced in Hadoop 2. It provides an API to resquest and work with cluster resources. 
Nevertheless these API are abstracted from the user code. Instead a typical user will leverage some high-level APIs providing by some distributed frameworks that are built on top of YARN. 
Among others MapReduce, Spark, Tez or Hive are examples of computing frameworks running as YARN applications on the compute layer (provided by YARN) and the storage layer (provided by HDFS). 
There is also a extra layer of applications that build on top of these frameworks. For example *Pig* and *Hive* run on MapReduce, Spark or Tez and don't directly interact with YARN. 

![](Images_course_review/yarn_applications.png)

### Overview of YARN running
In its core YARN provides two types of daemon: a **resource manager** (two by cluster in a stand-by configuration to ensure high availability) and several node managers. As indicated by its name the resource manager manages the use of resource across the cluster and the node managers launch and monitor **containers** where computations will happen. The figure below depicts possible scenarios when using YARN applications.

![](Images_course_review/overview_of_yarn.png)

1. To run an application the client will contact the ressource manager **(1)** and ask it to run an *application master* process **(2)**.
2. The resource manager finds a node manager that can launch the application master in a container **(3)**.
3. What the application master does once it is running depends on the YARN application:
	- *It could simply run a computation in the current container and return the result*
	- *Or it could request more resources for more containers to perform distributed computations*
	- *For the latter we have a step of resource negotiation that starts with the resource manager* **(4)**
4. YARN has a flexible model for resource allocation but basically a request will:
	- *Express the amount of computer resource required for each containers*
	- *Express locality constraints meaning that containers should be run as close as possible from the data to be processed*
	- *Because RAM and CPU are the limiting factors in distributed YARN can **queue** the request and manage the ressource accordingly*
	- *If one of the request suddenly needs 100% of the ressources it will kill all other processes finish this task and return to the other ones*
5. Once negotiation is done. The resource manager look for additional nodes to launch containers **(5)**

### MapReduce

MapReduce is a programming model for data processing. Hadoop can run programs written in various languages (Python, Ruby Java ...). MapReduce programs are inherently parallel. It works by breaking down the data processing into two main phases: **map** and **reduce**. Each phase has **key-value pairs** as input and output. Their types are chosen by the programmer.

### Overview of a MapReduce job
**A MapReduce job** is a unit of work that the client wants to perform and is composed of (i) the input data, (ii) MapReduce and (iii) some configurations. Hadoop runs the job by splitting it into two main tasks: the **map** and the **reduce**. These tasks are scheduled using YARN and run in the cluster nodes (remember the containers !).
Hadoop splits the input into several chunks. To each chunk is associated a **map** task. For each job a good split size should be around the size of a block (around 128MB). If splits are too small you may overload the process and managing all the map tasks might become tedious. 
In respect to **data locality** Hadoop will do its best to run the tasks where the data reside. Map tasks are written on the local disk not on HDFS. This is because map outputs are intermediary data that will be processed by the reduced tasks. When all the job is done, the data produced by the map tasks are removed. So storing it in HDFS (which implies the replication of this data) sounds like an overkill.
Reduce tasks do not take advantage of data locality. Indeed the input for a reduce task is normally the **merged, shuffled and sorted output of all the mappers**. The output fo the reduce task is saved on HDFS for reliability. A typical workflow of this process is depicted in the figure below.

![MapReduce](Images_course_review/MapReduce.png)

### MapReduce: Practice with Labs

In this lab we will process the file we downloaded during Lab1 and apply some MapReduce algorithms to (i) count all words in the file and (ii) find the most representative word of this file. The algorithm will be implemented in both Python and Ruby.

## Hadoop applications

In the next sections we will describe and work with some of the most useful applications that sit on top of the Hadoop core components. 

### Hive

In normal use, Hive converts SQL queries (HiveQL) into a series of MapReduce jobs. Hive organizes data into tables, which allows to attach structures or schemas to data stored on HDFS. Hive also saves and generates metadata (including the schema) that are stored in a database called the **Metastore**. Hive can handle a large spectrum of files as semi-structured data (e.g., CSV, JSON ...), sequence files (designed for large-scale processing), columnar file or HBase tables.
Hive is optimized for analytical processing (**OLAP**) and not for transactional processing (**OLTP**). OLTP is typically the type of processing that happens between a website and its database. The website needs to connect regularly to the database to get data, update it, insert new fields and so on. 
In contrast to this and given the fact that Hive harnesses the MapReduce algorithm to process HQL queries, its proper use should be related to data analytics. It makes total sense since data analysis related queries directly deals with columns (i.e., reduce an entire column to an average, sort a column, get maximum ...). These types of queries also fit perfectly with the MapReduce design. To go even further and enforce this aspect Hive is preferably used with some specific columnar file like the ORCFile (*Optimized Record Columnar File*).

### Anatomy of Hive

![](Images_course_review/hive_anatomy.png)

### Tables in Hive
A Hive table is made up of the data being stored and the associated metadata describing the layout of the data in the table. The data typically resides in HDFS. Hive stores the metadata in a relational database (metastore) and not in HDFS.
By default, newly created tables will be managed by Hive meaning that Hive will move the data into its warehouse directory: this is a called a **managed table** (Query: `CREATE TABLE ...`). 
Alternatively, we can create an **external table** which tells Hive to refer to data that exists at a defined location on HDFS so, outside the warehouse directory (Query:`CREATE EXTERNAL TABLE ...`).

 >When creating tables we need to specify the location of the data. Keep in mind that you need to specify the location of the folder where lies your data not the file itself.
 
The workflow we saw in class was the following. If no restriction use external tables! As such they can be available for other Hadoop applications. 
To create the table you first need to read the data from your file (for example csv) and load it into a intermediary table. Once done you need to transfer the data into the final table that will be stored on the right format for future queries (ORCFile for Hive). More on this subject in the first Hive's Lab.
 
 ### Partition and buckets
 
 Hive can organize tables into partitions. It is a way of dividing a table based on the value of a partition column, such as date, country name or a category. Using partition can speed up queries.
 
 ```sql
 ￼CREATE TABLE ...
 PARTITIONED BY <your condition>
 ```
 
Tables or partitions may be subdivided further into buckets to give more structure to the data that may be used for more efficient and specific queries. Additionally buckets are particularly useful if we want to create smaller samples of a big data set to perform sampling of our data instead of querying the whole.
