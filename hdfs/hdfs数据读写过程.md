## hdfs文件的读写过程

![写入](/hdfs/images/写入过程1.png)
![写入](/hdfs/images/写入过程2.png)

#### 详细过程
1. HDFS客户端调用DistributedFileSystem.create()函数创建文件

2. 底层再调用ClientProtocal.create()方法在NameNode的文件系统目录树中创建一个文件，并且创建新文件的操作记录到editlog中，返回一个HdfsDataOutputStream对象，底层是对DFSOutputStream进行了一个包装。

3. HDFS 客户端根据返回的输出流对象调用write方法来写数据

4. 由于之间创建的新文件是一个空文件，并没有申请任何数据块，所以DFSOutputStream首先会调用ClientProtocal.addBlock向NameNode申请数据块，数据块的大小可以由用户自己配置，默认128M，NameNode返回一个LocatedBlock对象，这个对象保存了这个数据块所有DataNode位置信息，然后就可以建立数据流管道写数据块了
5. 建立通向DataNode的数据流管道，写入数据：建立数据流管道之后，HDFS客户端就可以向数据流管道写数据。它会将数据切分成一个个的数据包packet，然后通过数据流管道发送到DataNode
__解释__
DFSDataOutputStream将数据分成块，写入Data Queue。Data Queue由Data Streamer读取，并通知元数据节点分配数据节点用来存储数据块(每块默认复制3份)。分配的数据节点放在一个Pipeline中。Data Streamer将数据块写入Pipeline中的第一个数据节点；第一个数据节点再将数据块发送给第二个数据节点；第二个数据节点再将数据发送给第三个数据节点。
__解释__
DFSDataoutputStream为发出去的数据块保存了ACK Queue,等待Pipeline中的数据节点告知数据已成功写入。如果数据节点在写入过程中失败了，则关闭Pipeline，将Ack Queue中的数据块放入到Data Queue的开始
           
6. 每一个packet都有一个确认包，逆序的通过数据流管道回到输出流。输出流在确认了所有的DataNode已经写入了这个数据包，就会从对应的缓存队列删除这个数据包
7. 当DataNode成功接收一个数据块时，DataNode会通过DataNodeProtocal.blockReceivedAndDelete方法向NameNode汇报，NameNode会更新内存中数据块和数据节点的对应关系
8. 完成操作后，客户端关闭输出流,此操作将所有的数据块写入pipeline中的数据节点，并等待ACK Queue成功返回。最后通知元数据节点写入完毕。