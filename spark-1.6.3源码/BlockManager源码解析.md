
blockManager负责spark中的数据存储的管理，不管是调用了cache,persist,还是broadcast等与存储相关的API，其数据都是由blockManager进行管理的，driver和executor都有一个blockManager实例来负责数据块的读写，而数据块的元数据管理是由driver端来管理。

BlockManager是主从结构,先看看BlockManager的架构图
![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/BlockManager%E6%9E%B6%E6%9E%84%E5%9B%BE.png?raw=true)
可以发现,和BlockManager相关的内容很多，也足以见得BlockManager是spark很重要的一个模块。

# BlockManager实例化
   在spark中，入口是SparkContext类，在SparkContext中会创建一个SparkEnv来初始化很多spark需要的实例，BlockManger就是在SparkEnv创建的时候实例化的。BlockManager在Driver和Executor启动的时候进行实例化，大致步骤如下
   ![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/BlockManager%E5%AE%9E%E4%BE%8B%E5%8C%96.jpg?raw=true)
   
   ```scala
     val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
      serializer, conf, memoryManager, mapOutputTracker, shuffleManager,
      blockTransferService, securityManager, numUsableCores)
   ```

# BlockManager初始化
&emsp;BlockManager实例化之后还不能用，需要初始化操作才能使用.初始化需要调用```BlockManager.initialize```，为什么呢？因为在BlockManager实例化的时候可能还不知道应用的appId,而启动executor的时候才知道appId,所以在Executor初始化的时候把appId传给```BlockManager.initialize```方法，这个时候开始初始化。
&emsp;那初始化都做了什么工作?   有下面几点
* ```blockTransferService.init``` 这是一个block相关的传输服务
* ```shuffleClient.init``` shuffle请求客户端初始化 负责shuffle过程block的获取
* ```BlockManagerId```实例化 和executorId,host,port 构成BlockManagerId
* ```master.registerBlockManager```向driver端注册

# 存储Block
当调用RDD的persis()方法或者cache等方法，都会把RDD的计算结果进行持久化，持久化写入就是由BlockManager来完成的，主要调用了BlockManager，具体可以查看```CacheManager.getOrCompute```,而存储数据块的方式有好几种，包括
```
DISK_ONLY
DISK_ONLY_2
MEMORY_ONLY
MEMORY_ONLY_2
MEMORY_ONLY_SER
MEMORY_ONLY_SER_2
MEMORY_AND_DISK
MEMORY_AND_DISK_2
MEMORY_AND_DISK_SER
MEMORY_AND_DISK_SER_2
OFF_HEAP
```
在BlockManager存储block的方法主要有```BlockManager.putIterator```,```BlockManager.putBlockData```,```BlockManager.putBytes```,而这几个方法最终都会调用BlockManager.doPut方法
```scala
  private def doPut(
      blockId: BlockId, //block的唯一标识
      data: BlockValues,//block内容
      level: StorageLevel,//存储级别
      tellMaster: Boolean = true, //是否向master汇报
      effectiveStorageLevel: Option[StorageLevel] = None) //
    : Seq[(BlockId, BlockStatus)]
```

doPut实际调用过程大概如下
* 根据存储级别选取存储方式:```memoryStore[内存],externalBlockStore[堆外],diskStore[磁盘]```
* `根据对应的Store类型，调用```blockStore.putIterator```或者```blockStore.putArray```或者```blockStore.putBytes```,这里是真正的存储动作
* 特别针对MemoryStore，可能出现内存不足的情况，如果可以，就存入磁盘。否则抛出OOM异常
* 更新block状态BlockStatus。会在memoryStore章节的时候详细说明
* 写入block完成,标记block,释放当前block
* 判断是否有副本，继续保存副本的block

# 读取Block
在spark中，一般读取Block的流程是这样的,当Executor从上游的Stage获取数据的时候会发起读取Block的操作，Block的读取可能有两种情况，一个是读取存储在本地的Block，另一个是读取在远程节点上的Block,远程读取是通过ShuffleClient和NettyBlockTransferService配合实现的。但是最终都会调用BlockManager.getBlockData,这个方法代码并不多可以直接贴出来

```scala
  override def getBlockData(blockId: BlockId): ManagedBuffer = {

    if (blockId.isShuffle) {
    //shuffle 下一个stage请求上一个stage的结果
      shuffleManager.shuffleBlockResolver.getBlockData(blockId.asInstanceOf[ShuffleBlockId])
    } else {
      //从本地获取Block
      val blockBytesOpt = doGetLocal(blockId, asBlockResult = false)
        .asInstanceOf[Option[ByteBuffer]]
      if (blockBytesOpt.isDefined) {
        val buffer = blockBytesOpt.get
        new NioManagedBuffer(buffer)
      } else {
        throw new BlockNotFoundException(blockId.toString)
      }
    }
  }

```
针对shuffle的请求，目前存在两种读取，一个是```FileShuffleBlockResolver.getBlockData```
一种是```IndexShuffleBlockResolver.getBlockData```,分别对应于hash和sort Shuffle的情况。可以参考shuffle模块的写出和读取来详细了解是如何操作的。

针对非Shuffle的读取，会调用doGetLocal方法，根据Block的存储类型,调用```DiskStore```,```MemoryStore```,```ExternalBlockStore```三者之一获取块数据,由存储级别选择.


# BlockManagerMaster,BlockManagerSlave
之前提过，BlockManager是主从Master-Slave模式的，其中Master是存在Driver端，Slave存在于Executor端. 而在Exectuor端持有BlockManagerMasterEndpoint的RpcEndpointRef.

BlockManagerMaster负责管理Block元数据，向BlockManagerSlave发送操作Block的消息，而Executor端的BlockManagerSlave负责向Master端发送Block的状态，并负责Block的操作的执行。

在Executor端，启动的时候会主动向Driver端注册自己的,可在BlockManager.scala找到注册的代码
```master.registerBlockManager(blockManagerId, maxMemory, slaveEndpoint)```,可以通过spark-shell启动来验证这个注册过程.
```
bin/spark-shell --master yarn  --num-executor 1
```

在这里我们申请了1个executor,观察截图，发现注册了两次BlockManager,1次是driver端的BlockManager注册，一次是executor端的blockManager注册。

![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/blockManager%E5%90%AF%E5%8A%A8%E6%B3%A8%E5%86%8C%E6%88%AA%E5%9B%BE.png?raw=true)




# CacheManager
CacheManager功能非常单一，在spark中负责对RDD的计算结果缓存的管理，RDD真正执行的时候会调用```RDD.compute``` --> ```RDD.iterator``` -->```cacheManager.getOrCompute```

```CacheManager.getOrCompute```方法会先从缓存中查找是否已经有缓存结果，如果有直接返回，如果没有就计算，然后放入缓存。


```scala

  def getOrCompute[T](
      rdd: RDD[T],
      partition: Partition,
      context: TaskContext,
      storageLevel: StorageLevel): Iterator[T] = {

    val key = RDDBlockId(rdd.id, partition.index)
    logDebug(s"Looking for partition $key")
    //从BlockManager获取块
    blockManager.get(key) match {
      case Some(blockResult) =>
        
        //....删除了一些无关紧要的代码.....

      //获取到block，直接返回即可
        val iter = blockResult.data.asInstanceOf[Iterator[T]]
        new InterruptibleIterator[T](context, iter) {
          override def next(): T = {
            existingMetrics.incRecordsRead(1)
            delegate.next()
          }
        }
      case None =>
      //缓存中没有对应的Block,需要重新计算
      //获取锁  避免有其他线程也在计算同样的block --[推测执行]
        val storedValues = acquireLockForPartition[T](key)
        if (storedValues.isDefined) {
          return new InterruptibleIterator[T](context, storedValues.get)
        }

        // Otherwise, we have to load the partition ourselves
        try {
          //从checkpoint中读取
          val computedValues = rdd.computeOrReadCheckpoint(partition, context)


          // Otherwise, cache the values and keep track of any updates in block statuses
          val updatedBlocks = new ArrayBuffer[(BlockId, BlockStatus)]
          //将计算结果存入缓存[disk,memory,external,具体哪个由storageLevel决定]
          val cachedValues = putInBlockManager(key, computedValues, storageLevel, updatedBlocks)
          //这里删除了部分无关紧要代码
          //返回计算结果
          new InterruptibleIterator(context, cachedValues)

        } finally {
          loading.synchronized {
            loading.remove(key)
            loading.notifyAll()
          }
        }
    }
```

# DiskStore,MemoryStore,ExternalBlockStore

前面提到，BlockManager负责计算过程的Block的写入和读取，而根据存储级别的设置，Block的存储可以分为```DiskStore,MemoryStore,ExternalBlockStore```,BlockManager根据存储级别的设定
# BlockTransferService
















