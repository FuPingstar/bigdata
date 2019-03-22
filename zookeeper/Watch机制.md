Watch机制

__Znode发生变化（Znode本身的增加，删除，修改，以及子Znode的变化）可以通过Watch机制通知到客户端。Zookeeper中所有读操作（getData()，getChildren()，exists()）都可以设置Watch选项。Watch事件具有one-time trigger（一次性触发）的特性，如果Watch监视的Znode有变化，那么就会通知设置该Watch的客户端。__

+ 一次性触发器
    <p style="text-indent:2em">客户端在Znode设置了Watch时，如果Znode内容发生改变，那么客户端就会获得Watch事件。例如：客户端设置getData("/znode1", true)后，如果/znode1发生改变或者删除，那么客户端就会得到一个/znode1的Watch事件，但是/znode1再次发生变化，那客户端是无法收到Watch事件的，除非客户端设置了新的Watch。</p>
+ 发送至客户端
   <p style="text-indent:2em">Watch事件是异步发送到Client。Zookeeper可以保证客户端发送过去的更新顺序是有序的。例如：某个Znode没有设置watcher，那么客户端对这个Znode设置Watcher发送到集群之前，该客户端是感知不到该Znode任何的改变情况的。换个角度来解释：由于Watch有一次性触发的特点，所以在服务器端没有Watcher的情况下，Znode的任何变更就不会通知到客户端。不过，即使某个Znode设置了Watcher，且在Znode有变化的情况下通知到了客户端，但是在客户端接收到这个变化事件，但是还没有再次设置Watcher之前，如果其他客户端对该Znode做了修改，这种情况下，Znode第二次的变化客户端是无法收到通知的。这可能是由于网络延迟或者是其他因素导致，所以我们使用Zookeeper不能期望能够监控到节点每次的变化。Zookeeper只能保证最终的一致性，而无法保证强一致性。</p> 
+ 设置watch的数据内容
    <p style="text-indent:2em">Znode改变有很多种方式，例如：节点创建，节点删除，节点改变，子节点改变等等。Zookeeper维护了两个Watch列表，一个节点数据Watch列表，另一个是子节点Watch列表。getData()和exists()设置数据Watch，getChildren()设置子节点Watch。两者选其一，可以让我们根据不同的返回结果选择不同的Watch方式，getData()和exists()返回节点的内容，getChildren()返回子节点列表。因此，setData()触发内容Watch，create()触发当前节点的内容Watch或者是其父节点的子节点Watch。delete()同时触发父节点的子节点Watch和内容Watch，以及子节点的内容Watch。


#### watch运行机制
1. Watch是轻量级的，其实就是本地JVM的Callback，服务器端只是存了是否有设置了Watcher的布尔类型。（源码见：org.apache.zookeeper.server.FinalRequestProcessor）
2. 在服务端，在FinalRequestProcessor处理对应的Znode操作时，会根据客户端传递的watcher变量，添加到对应的ZKDatabase（org.apache.zookeeper.server.ZKDatabase）中进行持久化存储，同时将自己NIOServerCnxn做为一个Watcher callback，监听服务端事件变化
3. Leader通过投票通过了某次Znode变化的请求后，然后通知对应的Follower，Follower根据自己内存中的zkDataBase信息，发送notification信息给zookeeper客户端。
4. Zookeeper客户端接收到notification信息后，找到对应变化path的watcher列表，挨个进行触发回调。
![watch监听流程图](/zookeeper/images/watch监听流程图.png)

