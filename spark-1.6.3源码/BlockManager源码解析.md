[TOC]


blockManager负责spark中的数据存储的管理，不管是调用了cache,persist,还是broadcast等与存储相关的API，其数据都是由blockManager进行管理的，driver和executor都有一个blockManager实例来负责数据块的读写，而数据块的元数据管理是由driver端来管理。

BlockManager是主从结构,先看看BlockManager的架构图
![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/BlockManager%E6%9E%B6%E6%9E%84%E5%9B%BE.png?raw=true)
可以发现,和BlockManager相关的内容很多，也足以见得BlockManager是spark很重要的一个模块。

# BlockManager实例化
   在spark中，入口是SparkContext类，在SparkContext中会创建一个SparkEnv来初始化很多spark需要的实例，BlockManger就是在SparkEnv创建的时候实例化的。BlockManager在Driver和Executor启动的时候进行实例化，大致步骤如下
   ![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/BlockManager%E5%AE%9E%E4%BE%8B%E5%8C%96.png?raw=true)
   
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

doPut方法包含的涉及的逻辑不少，实际调用过程大概如下
* 根据存储级别选取存储方式:```memoryStore[内存],externalBlockStore[堆外],diskStore[磁盘]``

* `根据对应的Store类型，调用```blockStore.putIterator```或者```blockStore.putArray```或者```blockStore.putBytes```,这里是真正的存储动作
* 更新block状态BlockStatus
* 写入block完成,标记block,释放当前block
* 判断是否有副本，继续保存副本的block

# 读取Block


# BlockManagerMaster BlockManagerSlave












