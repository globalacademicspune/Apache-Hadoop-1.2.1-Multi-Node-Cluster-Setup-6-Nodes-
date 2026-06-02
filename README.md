# Apache-Hadoop-1.2.1-Multi-Node-Cluster-Setup-6-Nodes-
This repository contains the installation and configuration steps for setting up an Apache Hadoop 1.2.1 multi-node cluster on Ubuntu Linux using six nodes.

Cluster Architecture
Node	Role
nn	NameNode
snn	Secondary NameNode
jt	JobTracker
dn1	DataNode
dn2	DataNode
dn3	DataNode

Features

Multi-node Hadoop cluster deployment
Passwordless SSH authentication
HDFS configuration
MapReduce configuration
Distributed administration using DSH
Cluster verification using JPS
Hadoop Version
Apache Hadoop 1.2.1

Repository Structure
.
├── README.md
├── installation-guide.md

Verification

Run the following command on all nodes:

dsh -a jps

Expected Hadoop daemons:

NameNode (nn)
NameNode
Secondary NameNode (snn)
SecondaryNameNode
JobTracker (jt)
JobTracker
DataNodes (dn1, dn2, dn3)
DataNode
TaskTracker
