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
当调用RDD的persis()方法或者cache等方法，都会把RDD的计算结果进行持久化，持久化写入就是由BlockManager来完成的，主要调用了BlockManager，具体可以查看```CacheManager.getOrCompute```
# 读取Block
# BlockManagerMaster BlockManagerSlave











