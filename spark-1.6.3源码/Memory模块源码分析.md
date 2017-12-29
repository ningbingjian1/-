# spark内存管理

![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/spark%E5%86%85%E5%AD%98%E6%A8%A1%E5%9D%97%E6%9E%B6%E6%9E%84%E5%9B%BE.png?raw=true)


spark大部分调优都和内存有关，所以spark内部有专门的实现来针对内存进行管理，spark中把内存分为两种，1是用于存储的内存[MemoryStore]，1是用于在任务执行任务，例如shuffle期间所需要的内存[TaskMemoryManager]。spark1.6以前，两种内存是固定大小的，spark1.6之后，存储内存是固定的，但是执行任务期间所需的内存是可以向存储内存"借"的，这样当执行任务期间内存不够的时候，能即时动态增加内存大小而不用重新配置再重新提交spark应用程序

目前spark支持两种内存管理:

<1> ```StaticMemoryManager``` [静态内存管理]

<2> ```UnifiedMemoryManager```[统一内存管理]

两种内存管理都实现了接口```MemoryManager``` ，

```scala
  //用于存储最大内存
  def maxStorageMemory: Long

  //请求存储内存 如果可以，会清除缓存中的某些块来达到足够内存的
    def acquireStorageMemory(
      blockId: BlockId,//块ID
      numBytes: Long,//请求字节数
      evictedBlocks: mutable.Buffer[(BlockId, BlockStatus)]//清除的内存块
      ): Boolean

//请求可用于展开的内存  
def acquireUnrollMemory(
      blockId: BlockId,//块ID
      numBytes: Long,//请求字节数
      evictedBlocks: mutable.Buffer[(BlockId, BlockStatus)]): Boolean

//请求执行内存
      private[memory]
  def acquireExecutionMemory(
      numBytes: Long,//请求内存字节数
      taskAttemptId: Long,//任务ID
      memoryMode: MemoryMode//内存模式 堆内和堆外
      ): Long//返回可分配的字节数
```

从上面的抽象方法容易得知，内存管理主要是针对执行期间和块存储的内存管理。

# TaskMemoryManager执行任务内存管理
1.对内存对象使用64 bit位的方式进行寻址
2.分页使用内存
3.针对堆外，直接使用原始地址索引对象
4.针对堆内。使用高位的前13位来作为页码编号的索引，低位的51[64-13]位来作为页内的索引
5.总共可以容纳8192页，堆内模式，每页最大的内存主要由long[] array数据的最大内存限制，每页可容纳内存达到((1L << 31) - 1) * 8L=16G bytes ，总共可管理内存达到8192 * 2^32 * 8 ,达到35t bytes
6.内存中每夜用MemoryBlock表示
7.MemoryConsumer内存的消费者，主要用来分配内存和内存溢写
## 分配页内存
一般情况下会有
1.调用memoryManager.acquireExecutionMemory请求执行内存，这个时候可能会发生spill
2.寻找第一个未使用的页码，BitSet实现
3.分配页内存 【堆内或者堆外】
4.分配有可能失败，这个时候memoryManager会报内存不足的异常
### MemoryAllocator
内存分配器，分为堆内堆外分配
#### HeapMemoryAllocator
1.先从尚未回收的内存池中寻找可分配MemoryBlock
对于1kB的以下的内存块，spark在释放的时候并没有马上释放，而是先放缓存池。
2.1不行，就直接分配堆内内存

#### UnsafeMemoryAllocator
堆外内存分配比较简单，直接调用平台底层API，记住分配地址即可
##### 分配内存
调用Unsafe.allocateMemory来分配内存，返回分配的内存地址，分配的是直接堆外内存，不受JVM控制
##### 释放内存
```java
 @Override
  public void free(MemoryBlock memory) {
    assert (memory.obj == null) :
      "baseObject not null; are you trying to use the off-heap allocator to free on-heap memory?";
    Platform.freeMemory(memory.offset);
  }
  最终会调用到Unsafe.freeMemory
```

#### 有关内存管理和shuffle的博客

https://github.com/hustnn/TungstenSecret

http://blog.csdn.net/u011564172/article/details/72763978

http://blog.csdn.net/u011564172

http://blog.csdn.net/u011564172/article/details/70183729

# MemoryManager

在spark中，内存管理抽象成MemoryManager,在内存管理中分别包含存储内存管理，执行期间内存管理，执行期间内存管

理还分为堆内和对外,存储内存管理委托给storageMemoryPool，执行期间内存管理委托给onHeapExecutionMemoryPool,

或者offHeapExecutionMemoryPool


MemoryPool负责内存池的内存管理：
1.记录每个任务的内存使用情况
2.返回所有任务的内存消耗情况
3.接收内存增加的请求
4.释放内存

## StaticMemoryManager
1.存储内存大小确定:

systemMaxMemory:spark.testing.memory JVM申请内存 

memoryFraction:内存因子 spark.storage.memoryFraction  0.6

safetyFraction:安全因子 spark.storage.safetyFraction  0.9

存储内存 = systemMaxMemory * memoryFraction * safetyFraction = 0.54 JVM申请内存
2.执行任务内存大小确定:

systemMaxMemory:spark.testing.memory JVM申请内存 

memoryFraction:内存因子 spark.shuffle.memoryFraction  0.2

safetyFraction:安全因子 spark.shuffle.safetyFraction  0.8

执行内存 = systemMaxMemory * memoryFraction * safetyFraction = 0.16 JVM 申请内存

假设申请的executor内存是1G 那么默认给JVM预留 1024M * (1 - (0.54 + 0.16)) = 307.2M

存储内存 = 1024M * 0.54 = 552.96M

执行内存 = 1024M * 0.16 = 163.84M 

## UnifiedMemoryManager

统一内存管理和静态内存管理最大的区别就是在任务执行期间可以向存储内存“借”内存，这样可以避免执行任务时候内存不足，需要重新配置内存，重新提交应

用的风险。

systemMemory：JVM申请内存内存  spark.testing.memory

reservedMemory：保留内存 spark.testing.reservedMemory * 1.5 ，默认是：300M * 1.5 = 450M  否则启动不了Executor

usableMemory：可用内存 systemMemory - reservedMemory = 

memoryFraction ： 0.75 spark.memory.fraction  内存因子是执行任务期间内存和存储内存的共同系数

最大可用内存：usableMemory * memoryFraction   执行任务期间和存储内存共同使用

例如申请了1G 那么最大可用内存就是（1024M - 450M） * 0.75 = 430.5M,由执行任务期间所需内存和存储所需内存共同拥有

# 内存管理配置
spark.memory.useLegacyMode  默认false 使用UnifiedMemoryManager










