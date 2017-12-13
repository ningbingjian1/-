blockManager负责spark中的数据存储的管理，不管是调用了cache,persist,还是broadcast等与存储相关的API，其数据都是由blockManager进行管理的，driver和executor都有一个blockManager实例来负责数据块的读写，而数据块的元数据管理是由driver端来管理。

BlockManager是主从结构,先看看BlockManager的架构图
![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/BlockManager%E6%9E%B6%E6%9E%84%E5%9B%BE.png?raw=true)
可以发现,和BlockManager相关的内容很多，也足以见得BlockManager是spark很重要的一个模块。

# BlockManager实例化
   在spark中，入口是SparkContext类，在SparkContext中会创建一个SparkEnv来初始化很多spark需要的实例，BlockManger就是在SparkEnv创建的时候初始化的
   
# BlockManager初始化



