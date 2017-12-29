# 简要介绍
shuffle模块是spark两个stage之间传递数据的实现模块，涉及到数据的读写操作，对性能的影响非常重要。
# shuffleManager

在spark中 shuffle模块是以插件形式提供的，要自定义shuffle,只要实现了ShuffleManager即可。目前spark提供两种shuffle模式：

sort --> SortShuffleManager

hash --> HashShuffleManager

ShuffleManager需要实现下面的方法:

# SortShuffleManager

# HashShuffleManager