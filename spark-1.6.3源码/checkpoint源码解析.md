# 如何checkpoint?
```
RDD.checkpoint()
```
# checkpoint是什么意思?

所谓checkpoint就是把RDD的计算结果进行持久化，在下次获取该RDD结果数据的时候，直接从chekcpoint中提取，免去了重新计算的时间。这对于需要重复使用的RDD特别有用
# 何时checkpoin?

RDD被重复多次使用的时候需要checkpoint

# checkpoint源码解析
## 第一步 检查RDD是否需要checkpoint
首先，checkpoint的执行时机是在当前job计算完成后才查询当前RDD有没有checkpoint标记，如果需要checkpoint，此时就启用一个新的job进行 checkpoint操作
证据如下```SparkContext.runJob```:
```scala
   
  def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
    //省略了无关代码 
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    progressBar.foreach(_.finishAll())
    //查看是否需要checkpoint
    rdd.doCheckpoint()
  }
```
doCheckpoint方法先检查当前RDD是否需要checkpoint,如果不需要，就继续检查其依赖RDD是否需要checkpoint,如果RDD带有checkpoint标记，就进行checkpoint操作

## 第二步 spark怎么实现的Checkpoint
从```RDD.checkpoint()```的源码
```
  def checkpoint(): Unit = RDDCheckpointData.synchronized {
      //在SparkContext设置checkpoint目录
    if (context.checkpointDir.isEmpty) {
      throw new SparkException("Checkpoint directory has not been set in the SparkContext")
    } else if (checkpointData.isEmpty) {
        //返回ReliableRDDCheckpointData
      checkpointData = Some(new ReliableRDDCheckpointData(this))
    }
  }
```

ReliableRDDCheckpointData实现了RDDCheckpointData的抽象方法```doCheckpoint```，该方法实现了真正的checkpoint功能

* 通过依赖```ReliableCheckpointRDD.writeRDDToCheckpointDirectory```把RDD计算结果写入磁盘
* 每个分区写一个文件
* 根据写出文件创建一个CheckPointRDD返回ReliableCheckpointRDD

## 第三步  如何利用RDD的checkpoint结果?
在RDD的compute方法里面，会调用iterator方法，在iterator方法，调用computeOrReadCheckpoint方法，最终会调用到ReliableCheckpointRDD的compute方法
```scala
  override def compute(split: Partition, context: TaskContext): Iterator[T] = {
    val file = new Path(checkpointPath, ReliableCheckpointRDD.checkpointFileName(split.index))
    ReliableCheckpointRDD.readCheckpointFile(file, broadcastedConf, context)
  }
```


