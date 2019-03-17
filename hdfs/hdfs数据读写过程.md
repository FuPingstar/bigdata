## hdfs文件的读写过程

### HDFS中的block packet chunk

1. 文件上传前需要分块，这个块就是block，一般为128MB，当然你可以去改，不顾不推荐。因为块太小：寻址时间占比过高。块太大：Map任务数太少，作业执行速度变慢。它是最大的一个单位。
2. packet是第二大的单位，它是client端向DataNode，或DataNode的PipLine之间传数据的基本单位，默认64KB。
3. chunk是最小的单位，它是client向DataNode，或DataNode的PipLine之间进行数据校验的基本单位，默认512Byte，因为用作校验，故每个chunk需要带有4Byte的校验位。所以实际每个chunk写入packet的大小为516Byte。由此可见真实数据与校验值数据的比值约为128 : 1。（即64*1024 / 512）

例如，在client端向DataNode传数据的时候，HDFSOutputStream会有一个chunk buff，写满一个chunk后，会计算校验和并写入当前的chunk。之后再把带有校验和的chunk写入packet，当一个packet写满后，packet会进入dataQueue队列，其他的DataNode就是从这个dataQueue获取client端上传的数据并存储的。同时一个DataNode成功存储一个packet后之后会返回一个ack packet，放入ack Queue中。


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


![读取](/hdfs/images/读取过程1.png)
![读取](/hdfs/images/读取过程2.png)

### 详细过程
1. 打开HDFS文件： HDFS客户端首先调用DistributedFileSystem
.open方法打开HDFS文件，底层会调用ClientProtocal.open方法，返回一个用于读取的HdfsDataInputStream对象
2. 从NameNode获取DataNode地址：在构造DFSInputStream的时候，对调用ClientPortocal.getBlockLocations方法向NameNode获取该文件起始位置数据块信息。NameNode返回的数据块的存储位置是按照与客户端距离远近排序的。所以DFSInputStream可以选择一个最优的DataNode节点,然后与这个节点建立数据连接读取数据块
3. 连接到DataNode读取数据块： HDFS客户端通过调用DFSInputSttream从最优的DataNode读取数据块，数据会以数据包packet形式从DataNode以流式接口传送到客户端，当达到一个数据块末尾的时候,DFSInputStream就会再次调用ClientProtocal.getBlockL
Octions获取下一个数据块的位置信息，并建立和这个新的数据块的最优节点之间的连接，然后HDFS继续读取数据块
4. 客户端关闭输入流
5. 在数据读取过程中，如果客户端在与数据节点通信时出现错误，则会尝试读取包含有此数据块的下一个数据节点，并且失败的数据节点会被记录，以后不会再连接

