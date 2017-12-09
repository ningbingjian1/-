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

![](https://github.com/ningbingjian1/reading/blob/master/spark-1.6.3%E6%BA%90%E7%A0%81/resources/%E6%9E%B6%E6%9E%84%E5%9B%BE.png?raw=true)







