# 使用

```scala
val arr1 = (0 until num).toArray
val barr1 = sc.broadcast(arr1)
val observedSizes = sc.parallelize(1 to 10, slices).map(_ =>barr1.value.size)
```




