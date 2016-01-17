# spark TaskInfo中的taskId，index，attemptNumber的关系

看代码时，看到TaskInfo中有这几个变量，正好分析spark长稳问题时也遇到对它的理解，特别是为什么要弄一个index在这里，所以写一篇文章来理解

``` 
class TaskInfo(
    val taskId: Long,
    val index: Int,
    val attemptNumber: Int,
    val launchTime: Long,
    val executorId: String,
    val host: String,
    val taskLocality: TaskLocality.TaskLocality,
    val speculative: Boolean) {
```

## what is TaskInfo

主要用来记录task 运行信息，包括启动时间，运行的节点，executorId， locality level。TaskInfo是在调度任务时创建的，具体而言是在TasksetManager的resourceOffer中创建的，这个方法其实是task调度的核心原子方法，主要负责给指定的资源调度一个满足条件（比如本地性）的task，从其签名也不难理解：

``` 
def resourceOffer(
    execId: String,
    host: String,
    maxLocality: TaskLocality.TaskLocality)
  : Option[TaskDescription] 
```

每调度一个task 都会创建一个对应的 TaskInfo，这个对应关系维护在TasksetManager内部。

## relation between taskId，index and attemptNumber

taskId 是在同样也是在TasksetManager的resourceOffer中创建的，其主要作用是给每个task一个唯一的id，类型为Long；注意这个唯一性是强唯一性，一个task失败后第二次再跑又会生成新的taskId。

index 是指分配的task在TasksetManager中tasks数组中的索引位置，所以同一个task失败后再运行其index是不会变的，其在逻辑上唯一标识一个task。

attemptNumber 是指同一个task尝试的次数，其实是通过TasksetManager中taskAttempts每个task对应的attempts的长度来计算的。

另外我们在跑任务识，driver打印的任务启动，完成日志其实是使用index 和 attemptNumber

``` java
16/01/17 14:51:52 INFO TaskSetManager: Starting task 0.0 in stage 3.0 (TID 32, 192.168.1.100, partition 0,PROCESS_LOCAL, 2078 bytes)

16/01/17 14:51:52 INFO TaskSetManager: Starting task 1.0 in stage 3.0 (TID 33, 192.168.1.100, partition 1,PROCESS_LOCAL, 2078 bytes)

16/01/17 14:51:52 INFO TaskSetManager: Starting task 2.0 in stage 3.0 (TID 34, 192.168.1.100, partition 2,PROCESS_LOCAL, 2078 bytes)

16/01/17 14:51:52 INFO TaskSetManager: Starting task 3.0 in stage 3.0 (TID 35, 192.168.1.100, partition 3,PROCESS_LOCAL, 2135 bytes)
  
  ...
  
16/01/17 14:51:52 INFO TaskSetManager: Finished task 0.0 in stage 3.0 (TID 32) in 31 ms on 192.168.1.100 (1/4)
16/01/17 14:51:52 INFO TaskSetManager: Finished task 3.0 in stage 3.0 (TID 35) in 29 ms on 192.168.1.100 (2/4)
16/01/17 14:51:52 INFO TaskSetManager: Finished task 2.0 in stage 3.0 (TID 34) in 31 ms on 192.168.1.100 (3/4)
16/01/17 14:51:52 INFO TaskSetManager: Finished task 1.0 in stage 3.0 (TID 33) in 32 ms on 192.168.1.100 (4/4)

```

注意这里的`task 0.0` 前面这个数字0指的就是index，从逻辑上唯一标识了一个task；后一个数字0表示attemptNumber，表示第几次跑

可能细心的同学发现了，在一个stage里面 taskId 其实和 index.attemptNumber是一一对应的；跨多个stage则不成立。 如果一个job有多个stage，则在日志里面我们可能观察到多个 `Starting task 1.0 `,所以严格来说，taskId是和 stageId.stageAttemptId.index.index.attemptNumber 一一对应。