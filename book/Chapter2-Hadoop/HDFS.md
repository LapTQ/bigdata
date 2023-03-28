# HDFS

HDFS is a file system where 

* each file is divided into blocks of a pre-determined size.
* these blocks are stored across a cluster comprising of a single NameNode (Master node) and several DataNodes (Slave nodes).
* these DataNode can be run on a single machine,but in the practical world, these DataNodes are spread across various machines.



## NameNode

Functions of NameNode:

* It is the master daemon that maintains and manages the DataNode.
* It records the metadata of all the files stored in the cluster, e.g. the location of block stored,the size of the files, permissions, hierarchy, etc. 
* There are two files associated with the metadata:
  - **FsImage:** It contains the complete state of the file system namespace since the start of the NameNode.
  - **EditLogs:** It contains all the recent modifications made to the file system with respect to the most recent FsImage.
* It records each change that takes place to the file system metadata. For example, if a file is deleted in HDFS, the NameNode will immediately record this in the EditLog.
* It regularly receives a Heartbeat and a block report from all the DataNodes in the cluster to ensure that the DataNodes are live.
* It keeps a record of all the blocks in HDFS and in which nodes these blocks are located.
* The NameNode is also responsible to take care of the **replication** **factor** of all the blocks which we will discuss in detail later in this HDFS tutorial blog.
* In **case of the DataNode failure**, the NameNode chooses new DataNodes for new replicas, balance disk usage and manages the communication traffic to the DataNodes.

## DataNode

DataNodes are the slave nodes in HDFS. Unlike NameNode, DataNode is a commodity hardware, that is, a non-expensive system which is not of high quality or high-availability. The DataNode is a block server that stores the data in the local file ext3 or ext4.

### *Functions of DataNode:*

- These are slave daemons or process which runs on each slave machine.
- The actual data is stored on DataNodes.
- The DataNodes perform the low-level read and write requests from the file system’s clients.
- They send heartbeats to the NameNode periodically to report the overall health of HDFS, by default, this frequency is set to 3 seconds.







## Further Readings

* [Edureka - Apache Hadoop HDFS Architecture](https://www.edureka.co/blog/apache-hadoop-hdfs-architecture/):))