# 使用

```scala
val arr1 = (0 until num).toArray
val barr1 = sc.broadcast(arr1)
val observedSizes = sc.parallelize(1 to 10, slices).map(_ =>barr1.value.size)
```
![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/%E6%9E%B6%E6%9E%84%E5%9B%BE.png?raw=true)

# broadcast的作用
1.针对数据量较少的对象进行缓存，避免在每个task初始化该对象
# broadcast的实现类结构

![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/broadcast-%E7%B1%BB%E7%BB%93%E6%9E%84.png?raw=true)

# broadcast要点
  1. &ensp;&ensp;broadcast使用很简单，只需要调用```SparkContext..broadcast()``` 就可以。
  2. 广播变量在Spark中是由BroadcastManager统一进行管理
  3. 在spark中广播变量的广播方式包含两种:http和Torrent的方式,默认是Torrent方式
下面详细讲述BroadcastManager,http和Torrent的广播变量存取的详细实现过程
# BroadcastManager
```BroadcastManager```在SparkEnv构造的时候通过调用```new BroadcastManager```进行初始化,在构造```BroadcastManager``` ，默认会调用其```initialize```方法.通过配置```spark.broadcast.factory```查找```BroadcastFactory```,默认情况是```org.apache.spark.broadcast.TorrentBroadcastFactory```

# Broadcast
是一个抽象类包含有两个子类```TorrentBroadcast```,,```HttpBroadcastFactory```,主要包含几个抽象方法
```scala
protected def getValue(): T //如何获取广播变量
protected def doUnpersist(blocking: Boolean) // 移除持久化的广播变量
protected def doDestroy(blocking: Boolean) //销毁广播变量时回调
```
## TorrentBroadcast
主要包含两个重要的方法，存储和读取广播变量
### 存储广播变量
存储广播变量的方法是```TorrentBroadcast.TorrentBroadcast```

1.默认情况下会在driver端存放一份广播变量

2.将广播变量拆分成多个block存储到BlockManager进行存储,默认每个block大小是4M,通过```spark.broadcast.blockSize```控制.  

3.block的存储方式MEMORY_AND_DISK_SER


总结:
通过源码可以得出结论，默认在driver端存储了两次，一次是未经过分块和未序列化未压缩的存储，一份是分成多块的序列化压缩存储


### 读取广播变量
TorrentBroadcast.readBroadcastBlock

1.广播变量获取，默认情况是从local中的blockManager直接通过BrocastId获取，如果获取成功就直接返回```BlockResult```,否则从远端。一般情况如果是driver端调用的话是能获取成功，如果是executor调用，则不能。
2.如果上一步获取不成功，就根据block数量，挨个获取，首先也是从本机获取，然后从远端获取 ，一般是executor端会走这个流程
3.如果是第一次获取到block，需要在本机备份一份。这样下次就可以直接从本机获取

![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/%E5%B9%BF%E6%92%AD%E5%8F%98%E9%87%8F%E5%86%99%E5%85%A5%E8%AF%BB%E5%8F%96--Torrent.png?raw=true)
## HttpBroadcastFactory

使用http管理广播变量和使用BlockManager管理广播变量有所不同，主要表现在几个方面:
1.和blockManager一样，都会在driver端预存一份，然后也会在driver端以文件的形式写入磁盘
2.在executor通过http调用的方式获取block
3.不会在executor备份，每次都会调用http来获取block

![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/%E5%B9%BF%E6%92%AD%E5%8F%98%E9%87%8F%E5%86%99%E5%85%A5%E8%AF%BB%E5%8F%96--http.png?raw=true)





  

   






