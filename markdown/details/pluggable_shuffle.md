# spark shuffle 插件式机制

本文主要介绍spark的shuffle机制和插件式shuffle架构分析

## shuffle manager

在spark里面shuffle manager是一个接口类，用户可以实现各种manager，具体的shuffle manager实例是在初始化sparkenv时创建的，根据用户配置manager类创建对应的实例。

``` 
// Let the user specify short names for shuffle managers
val shortShuffleMgrNames = Map(
  "hash" -> "org.apache.spark.shuffle.hash.HashShuffleManager",
  "sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager",
  "tungsten-sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager")
val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
val shuffleMgrClass = shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase, shuffleMgrName)
val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)
```

shuffle manger如何使用的呢

* 在shuffle map task里面通过env拿到manager，获取writer写数据

``` 
var writer: ShuffleWriter[Any, Any] = null
try {
  val manager = SparkEnv.get.shuffleManager
  writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
  writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
  writer.stop(success = true).get
}
```

* 在shuffledRdd里面通过manager获取reader读数据

``` 
val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
SparkEnv.get.shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
  .read()
  .asInstanceOf[Iterator[(K, C)]]
```

架构何其清爽！



## shuffle handle

从上一节中，我们可以看到无论是获取shuffle writer还是reader，都需要一个shuffle handle，那么它是用来做什么的呢？

``` 
/**
 * Subclass of [[BaseShuffleHandle]], used to identify when we've chosen to use the
 * serialized shuffle.
 */
private[spark] class SerializedShuffleHandle[K, V](
  shuffleId: Int,
  numMaps: Int,
  dependency: ShuffleDependency[K, V, V])
  extends BaseShuffleHandle(shuffleId, numMaps, dependency) {
}

/**
 * Subclass of [[BaseShuffleHandle]], used to identify when we've chosen to use the
 * bypass merge sort shuffle path.
 */
private[spark] class BypassMergeSortShuffleHandle[K, V](
  shuffleId: Int,
  numMaps: Int,
  dependency: ShuffleDependency[K, V, V])
  extends BaseShuffleHandle(shuffleId, numMaps, dependency) {
}
```



可以看到它就只有两个实现，其实就是其一个标识作用。用于标识sort based shuffle writer时使用哪个具体的writer

``` 
override def getWriter[K, V](
    handle: ShuffleHandle,
    mapId: Int,
    context: TaskContext): ShuffleWriter[K, V] = {
  numMapsForShuffle.putIfAbsent(
    handle.shuffleId, handle.asInstanceOf[BaseShuffleHandle[_, _, _]].numMaps)
  val env = SparkEnv.get
  handle match {
    case unsafeShuffleHandle: SerializedShuffleHandle[K @unchecked, V @unchecked] =>
      new UnsafeShuffleWriter(
        env.blockManager,
        shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
        context.taskMemoryManager(),
        unsafeShuffleHandle,
        mapId,
        context,
        env.conf)
    case bypassMergeSortHandle: BypassMergeSortShuffleHandle[K @unchecked, V @unchecked] =>
      new BypassMergeSortShuffleWriter(
        env.blockManager,
        shuffleBlockResolver.asInstanceOf[IndexShuffleBlockResolver],
        bypassMergeSortHandle,
        mapId,
        context,
        env.conf)
    case other: BaseShuffleHandle[K @unchecked, V @unchecked, _] =>
      new SortShuffleWriter(shuffleBlockResolver, other, mapId, context)
  }
}
```





## shuffle writer



## shuffle reader

