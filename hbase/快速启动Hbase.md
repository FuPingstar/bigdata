#### 在伪分布式模式下安装安装Hbase

`__因为文章是博主的学习过程，不是系列学习教程，所以基础的部分博主在这里就不详细写了，直接上在伪分布式模式下安装HBase__`

<p style="text-indent:2em">伪分布模式意味着HBase仍然在单个主机上完全运行，但是每个HBase守护进程（HMaster，HRegionServer和ZooKeeper）作为一个单独的进程运行：在独立模式下，所有守护进程都运行在一个jvm进程/实例中。</p>

__Hbase配置__

1. 编辑hbase-site.xml配置。首先，添加以下指示HBase以分布式模式运行的属性，每个守护进程有一个JVM实例。

````
<property>
  <name>hbase.cluster.distributed</name>
  <value>true</value> 
</property>
````
接下来，将 hbase rootdir 从本地文件系统更改为您的 HDFS 实例的地址，使用 HDFS:////或 URI 语法。在这个例子中，HDFS在端口8020的本地主机上运行。
```
<property>
  <name>hbase.rootdir</name>
  <value>hdfs://localhost:8020/hbase</value> 
</property>
```
<font color="green">您不需要在HDFS中创建目录。HBase会为你做这个。如果你创建了这个目录，HBase会试图做一个迁移，这不是你想要的</font>
2. 启动 HBase。使用bin/start-hbase.sh命令启动HBase。如果您的系统配置正确，该jps命令应显示HMaster和HRegionServer进程正在运行。
3. 检查HDFS中的HBase目录。如果一切正常，HBase在HDFS中创建它的目录。在上面的配置中，它存储在HDFS上的/hbase/中。您可以使用 hadoop 的 bin/目录中的 hadoop fs 命令来列出此目录。
```
$ ./bin/hadoop fs -ls /hbase
Found 7 items
drwxr-xr-x   - hbase users          0 2014-06-25 18:58 /hbase/.tmp
drwxr-xr-x   - hbase users          0 2014-06-25 21:49 /hbase/WALs
drwxr-xr-x   - hbase users          0 2014-06-25 18:48 /hbase/corrupt
drwxr-xr-x   - hbase users          0 2014-06-25 18:58 /hbase/data
-rw-r--r--   3 hbase users         42 2014-06-25 18:41 /hbase/hbase.id
-rw-r--r--   3 hbase users          7 2014-06-25 18:41 /hbase/hbase.version
drwxr-xr-x   - hbase users          0 2014-06-25 21:49 /hbase/oldWALs
```
4. 启动和停止备份HBase主（HMaster）服务器。

HMaster服务器控制HBase集群。你可以启动最多9个备份HMaster服务器，这个服务器总共有10个HMaster计算主服务器。

要启动备份HMaster，请使用local-master-backup.sh。对于要启动的每个备份主节点，请添加一个表示该主节点的端口偏移量的参数

每个HMaster使用三个端口（默认情况下为16010,16020和16030）。端口偏移量被添加到这些端口，因此使用偏移量2，备份HMaster将使用端口16012,16022和16032。

以下命令使用端口：16012/16022/16032，16013/16023/16033和16015/16025/16035启动3个备份服务器。
```
$ ./bin/local-master-backup.sh 2 3 5
```
要在不杀死整个群集的情况下杀死备份主机，则需要查找其进程ID（PID）。PID存储在一个名为/tmp/hbase-USER-X-master.pid的文件中。该文件的唯一内容是PID。

您可以使用该kill -9命令来杀死该PID。以下命令将终止具有端口偏移1的主服务器，但保持群集正在运行：
```
$ cat /tmp/hbase-testuser-1-master.pid |xargs kill -9
```
5. 启动和停止其他RegionServers。

HRegionServer按照HMaster的指示管理StoreFiles中的数据。通常，一个HRegionServer在集群中的每个节点上运行。

在同一个系统上运行多个HRegionServers对于伪分布式模式下的测试非常有用。该local-regionservers.sh命令允许您运行多个RegionServer。它以类似的local-master-backup.sh命令的方式工作，因为您提供的每个参数都代表实例的端口偏移量。

每个RegionServer需要两个端口，默认端口是16020和16030。但是，由于HMaster使用默认端口，所以其他RegionServers的基本端口不是默认端口，而HMaster自从HBase版本1.0.0以来也是RegionServer。基本端口是16200和16300。

您可以在服务器上运行另外99个不是HMaster或备份HMaster的RegionServer。以下命令将启动另外四个RegionServers，它们在从16202/16302（基本端口16200/16300加2）开始的顺序端口上运行。

```
$ .bin/local-regionservers.sh start 2 3 4 5
```

手动停止RegionServer，请使用带有stop参数和服务器偏移量的local-regionservers.sh命令停止。
```
$ .bin/local-regionservers.sh stop 3
```

6. 停止HBase

bin/stop-hbase.sh