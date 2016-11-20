# Persistent HDFS 
REF : https://github.com/kiwenlau/hadoop-cluster-docker

## First time Startup
This branch modifies the  Dockerfile for kiwenlau's docker-based hadoop cluster to enable using the host's storage via a -v mapping.  This makes it possible for the hadoop volume to survive restarts of the host and the docker containers since the hadoop file system is persisted to storage.  However this change introduces a number of new steps which must be performed to ensure the cluster is configured correctly. 

## Changes to Dockerfile
First let's discuss the changes that were made to the Dockerfile and why they are needed.

### commented out  ...
```

# RUN mkdir -p ~/hdfs/namenode && \ 
#    mkdir -p ~/hdfs/datanode && \
```
This was done because we are no longer using the local storage and we need to create local directories for each of the docker instances which will be running on the same host. Otherwise, if we left this directory and just created a common volume to a fixed  host location (like /hdfs), it would result in each of the docker instances trying to use the exact same host folder and this will not work.
### commented out  ...

```
RUN /usr/local/hadoop/bin/hdfs namenode -format
```
We will need each image to format it's storage and if we could use the VOLUME keyword in the Dockerfile to map our images to their host folders, then we could leave this command. Since each instance must have a unique host location, this must be deferred until after the instance has been started.  Note, this infers that we will need to start each of the instances with the -v command in the start-container.sh script. 
## Changes to the start-container.sh 
```
# add  to docker run (master)
-v /hdfs/master:/root/hdfs \

# add to docker run (slaves where $i is the slave number)
-v /hdfs/slave$i:/root/hdfs \
```


### Before First Start

As these changes imply, we also need to add matching paths to the host file system at so these changes will work.  The host file system should have these folders created. (where N is the number of slave nodes)
/hdfs/master
/hdfs/slave1
/hdfs/slave2
...
/hdfs/slaveN
```
# on the host
mkdir /hdfs/master & mkdir /hdfs/slave1 & mkdir /hdfs/slave2 
...

```
### First Startup
```
./start-container.sh 
```
### After first Start
Now we must login to each master/slave image and format their respective hdfs.
```
docker exec -it hadoop-slave1 bash
root@hadoop-slave1:~# /usr/local/hadoop/bin/hdfs namenode -format

# repeat for each slave and the master

```   
Finally, we can start the cluster from the master terminal. 
```
root@hadoop-master:~# ./start-hadoop.sh 
```
At this point, running jps on the slave nodes will show : 
```
root@hadoop-slave1:~# jps
1250 Jps
1124 NodeManager
1012 DataNode

```
 If DataNode is shown then we have correctly setup dfs.  We can also run the dfsadmin -report on the master node to see the DFS status...

```
root@haddop-master:~# hdfs dfsadmin -report
Configured Capacity: 201625673728 (187.78 GB)
Present Capacity: 108036644864 (100.62 GB)
DFS Remaining: 108036595712 (100.62 GB)
DFS Used: 49152 (48 KB)
DFS Used%: 0.00%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0

-------------------------------------------------
Live datanodes (2):
...


```

### Test to verify dfs persistence
1. Copy files/folders to hdfs on the from the master terminal
```
root@hadoop-master:~# hdfs dfs -mkdir /test
root@hadoop-master:~# touch test.txt
root@hadoop-master:~# hdfs dfs -put test.ext /test/
root@hadoop-master:~# hdfs dfs -ls /test
Found 1 items
-rw-r--r--   2 root supergroup         20 2016-11-20 18:39 /test/test.txt

```
2. Shutdown hadoop cluster 
```
root@hadoop-master:~# stop-all.sh
...
root@hadoop-master:~# exit
exit
<host>$ docker stop hadoop-slave1 hadoop-slave2 hadoop-master
...
```
3. Restart hadoop cluster and verify test file exists
```
<host>$ ./start-container.sh
root@hadoop-master:~# ./start-hadoop.sh
Starting namenodes on [hadoop-master]
...
root@hadoop-master:~# hdfs dfs -ls /test
Found 1 items
-rw-r--r--   2 root supergroup         20 2016-11-20 18:39 /test/test.txt

```