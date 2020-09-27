# Hadoop fundamentals

<p style='text-align: justify;'><b>Apache Hadoop</b> is a tighly integrated ecosystem of different software products built to provide scalable and reliable dsitributed storage and distributing processing. The technical foundations of Hadoop stem from successive innovative approaches proposed and published by Google in the 2000's to produce:</p>

- reliable storage (Google file system with become Hadoop Distributed File System, `HDFS`)
- reliable processing (`MapReduce`)
- low-latency random access queries on hundreds to thousands unreliable servers

<p style='text-align: justify;'>The key innovation in this approach was to distribute data and split-up computations on this data in small chunks over different machines / servers. By design distributed system scales very well. As each machine is independent scaling the infrastructure only consists of adding some servers (horizontal scaling).</p>

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

<p style='text-align: justify;'>Hadoop is an open source implementation of these techniques. At its core, it offers a distributed file system (HDFS) and a means of running processes across a cluster of servers (YARN, Yet Another Ressource Manager). The original distributed processing application built on Hadoop was MapReduce, but since a wide range of additionnal software frameworks and librairies have emerged around Hadoop.</p>



## What is a cluster?

<p style='text-align: justify;'>A cluster is a bunch of servers grouped together to provide one or more functions such as storage or computation. On Hadoop clusters are deployed following a <b>master / slaves (or workers)</b> architecture. Slaves machines are where the real work happens - these machines store data, perform computations, offer services like lookups and search and much more. Master machines are responsible for coordination, maintaining metadata about the data and services on the slaves machines and ensuring that the services keep running even if some slaves machines fail. Typically there are two or three master machines for redundancy to avoid any single point of failure. A cluster is scaled up by adding more slave servers. Often we want to allow access to the clusters by users by providing some machines to act as <i>gateway</i> or <i>edge</i> servers. These servers often do not run any services at all but provide access cluster services.</p>



## Core Components

<p style='text-align: justify;'><b>HDSF</b>, <b>YARN</b>, <b>Apache ZooKeeper</b> and the <b>Apache Hive Metastore</b> represent the core building blocks of the Hadoop infrastructure.</p>

### HDFS

<p style='text-align: justify;'> The Hadoop distributed file system is the scalable, fault-tolerant file system for Hadoop. It is optimized to store very large amount of <b>immutable</b> data (<i>Write Once Read Many</i>) with files being typically accessed in long sequential scan. When storing data, HDFS breaks up a file into <i>blocks</i> of configurable size, usually 128MB and store <i>replicas</i> of each block on multiple servers ensuring data resilience and parallelism. Each worker node runs a deamon called a <b>DataNode</b> which accepts new block of data and write them to its local disks. The <b>DataNode</b> is only aware of blocks and their IDs: it does not have the knowledge about the file to which a particular block or replicate belongs. This information is hold by the <b>NameNode</b> which runs on the master servers and is responsible for maintaining a mapping of files to the blocks as well as metadata about the files themselves (names, permissions, attributes). All this data is saved in memory and is by definition non-persistent.</p>

![Data_and_Name_nodes](Images_course_review/Name_and_Data_nodes.png)

<p style='text-align: justify;'> It is also possible to run a <i>secondary NameNode</i> which despite its name does not act as the primary NameNode. Its main role is <b>to keep a copy of the File System image of the primary NameNode</b> which contains all details and metadata. This image is periodically built and updated by the <i>secondary NameNode</i> through interaction with the <i>JournalNodes</i> which contains all the logs related to the primary NameNode operations. The secondary NameNode runs on a separate machine as it requires CPU power and as much memory as the NameNode.</p>

![Secondary_Name_Node](Images_course_review/Secondary_namenode.png)

### HDFS high availability

<p style='text-align: justify;'>The combination of replicating NameNode data and using secondary NameNode to create checkpoints do protect against data loss but <b>do not provide high availability of the file system</b>. In this configuration the primary NameNode is still a <b>single point of failure</b>. If the primary NameNode fails all clients would be unable to read or write data. In such an event the whole Hadoop system would be out of service until a new NameNode is brought online.</p>

<p style='text-align: justify;'> Since Hadoop 2 there is now the implementation of a pair of NameNodes in an <b>active-standby</b> configuration. In case of the failure of the active primary NameNode, the standby takes over and ensure continuity of services. To reach this goal a few consideration have to be taken:</p>

- The standby NameNode synchronizes its state with the active NameNode and continues to read new entries.
- DataNodes must send block reports to both NameNodes because the block mapping are stored in memory and not on disk.
- We must ensure that only one NameNode is in active state to avoid any split / brain scenario (thank to *ZooKeeper*).
- The standby also ensure the role of the secondary NameNode by taking periodic checkpoints of the active NameNode state.

![High_availability](Images_course_review/ha_archi.png)

### HDFS: Practice with Labs

<p style="text-align: justify">In the <i>practice with labs</i> section you will find a link toward some practice. The lab consists of a set of instructions that you can do by yourself or with help from the walk-through which is my solution for the lab. All the labs have been performed in the frame of the Data Engineering applied master proposed by the Data SicenceTech institute. These were possible with the help of a consulting company Adaltas which provided us access to a Hadoop cluster. If you do not have access to such service you can still follow the walk-through and these notes to get an idea about the way the Hadoop infrastructure and services work. The general setup for the lab looks as depicted in the figure below. We basically connect by SSH and through VPN to an <b>edge node</b> from which we can perfom local operations and send HDFS-related operations.</p>


![Lab_setup](Images_course_review/lab_setup.png)

You can access the LAB1 [here]().

>PUT THE LINK


### YARN

<p style="text-align: justify"> Apache YARN (<b>Yet</b> <b>A</b>nother <b>R</b>esource <b>N</b>egotiator). is the cluster resource manager of Hadoop that was introduce in Hadoop 2. It provides an API to resquest and work with cluster resources. Nevertheless these API are abstracted from the user code. Instead a typical user will leverage some high-level APIs providing by some distributed frameworks that are built on top of YARN. Among others MapReduce, Spark, Tez or Hive are examples of computing frameworks running as YARN applications on the compute layer (provided by YARN) and the storage layer (provided by HDFS). <p style="text-align: justify">There is also a extra layer of applications that build on top of these frameworks. For example <i>Pig</i> and <i>Hive</i> run on MapReduce, Spark or Tez and don't directly interact with YARN.</p> 

![YARN_applications](Images_course_review/yarn_applications.png)

### Overview of YARN running
<p style="text-align: justify">In its core YARN provides two types of daemon: a <b>resource manager</b> (two by cluster in a stand-by configuration to ensure high availability) and several node managers. As indicated by its name the resource manager manages the use of resource across the cluster and the node managers launch and monitor <b>containers</b> where computations will happen. The figure below depicts possible scenarios when using YARN applications.</p>

![Anatomy_YARN_run](Images_course_review/overview_of_yarn.png)

1. To run an application the client will contact the ressource manager **(1)** an ask it to run an *application master* process **(2)**.
2. The resource manager finds a node manager that can launch the application master in a container **(3)**.
3. What the application master does once it is running depends on the YARN application:
	- *It could simply run a computation in the current container and return the result*
	- *Or it could request more resources for more containers to perform distributed computations.*
	- *For the latter we have a step of resource negotiation that starts with the resource manager.* **(4)**
4. YARN has a flexible model for resource allocation but basically a request will:
	- *Express the amount of computer resource required for each containers*
	- *Express locality constraints meaning that containers should be run as close as possible from the data to be processed*
	- *Because RAM and CPU are the limiting factors in distributed  YARN can **queue** the request and manage the ressource accordingly*
	- *If one of the request suddenly needs 100% of the ressources it will kill all other processes finish this task and return to the other ones.*

5. Once negotiation is done. The resource manager look for additional nodes to launch containers **(5)**

### MapReduce

<p style="text-align: justify"><b>MapReduce</b> is a programming model for data processing. Hadoop can run programs written in various languages (Python, Ruby Java ...). MapReduce programs are inherently parallel. It works by breaking down the data processing into two main phases: **map** and **reduce**. Each phase has **key-value pairs** as input and output. Their types is chosen by the programmer. <code>map()</code> and <code>reduce()</code></p>.

### Overview of a MapReduce job
<p style="text-align: justify">A <b>MapReduce job</b> is a unit of work that the client wants to perform and is composed of (i) the input data, (ii) MapReduce and (iii) some configuration. Hadoop runs the job by splitting it into two main tasks: the <b>map</b> and the <b>reduce</b>. These tasks are scheduled using YARN and run in the cluster nodes (remember the containers !).</p>

<p style="text-align: justify"> Hadoop splits the input into several chunks. To each chunk is associated a <b>map</b> task. For each job a good split size should be around the size of a block (around 128MB). if splits are too small you may overload the process and managing all the map tasks might become tedious. In respect to <b>data locality</b> Hadoop will do its best to run the tasks where the data reside. Map tasks are written on the local disk not on HDFS. This is because map outputs are intermediary data that will be processed by the reduced tasks. When all the job is done, the data produced by the map tasks be removed. So storing it in HDFS (which implies the replication of this data) sounds like an overkill.</p>

<p style="text-align: justify">Reduce tasks do not take advantage of data locality. Indeed the input for a reduce task is normally the <b>merged</b>, <b>shuffled</b> and <b>sorted</b> output of all the mappers. The output fo the reduce task is saved on HDFS for reliability. A typical workflow of this process is depicted in the figure below.</p>

![MapReduce](Images_course_review/MapReduce.png)
