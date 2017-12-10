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

## HttpBroadcastFactory




  

   






