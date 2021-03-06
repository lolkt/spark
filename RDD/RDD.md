1.RDD是一个基本的抽象，操作RDD就像操作一个本地集合一样，降低了编程的复杂度

RDD的算子分为两类，一类是Transformation（lazy），一类是Action（触发任务执行）

**RDD不存真正要计算的数据，而是记录了RDD的转换关系**（调用了什么方法，传入什么函数）

RDD与普通集合的区别：
* RDD里边记录的是描述信息（从哪里读数据，以后对数据如何进行计算），RDD的方法分为两类，Transformation（lazy）和Action（生成task并发送到Executor中执行）
* scala的集合里面存储的是真正的数据，执行方法后立即返回结果
#### RDD的属性
 
1）一系列分区（Partition），即数据集的基本组成单位。对于RDD来说，每个分片都会被一个计算任务处理，并决定并行计算的粒度。用户可以在创建RDD时指定RDD的分区个数，如果没有指定，那么就会采用默认值。默认值就是程序所分配到的CPU Core的数目。

2）一个计算每个分区的函数。每一个输入切片都会有一个函数作用在上面

Spark中RDD的计算是以分片为单位的，每个RDD都会实现compute函数以达到这个目的。compute函数会对迭代器进行复合，不需要保存每次计算的结果。
一个分区对应一个Task（同一个stage中），一个分区对应的Task只能在一台机器中（Executor），一台机器上可以有多个分区对应的Task

3）RDD与RDD之间存在依赖关系。

RDD的每次转换都会生成一个新的RDD，所以RDD之间就会形成类似于流水线一样的前后依赖关系。在部分分区数据丢失时，Spark可以通过这个依赖关系重新计算丢失的分区数据，而不是对RDD的所有分区进行重新计算。

4）（可选）如果存储的是KV类型，shuffle的时候回产生一个分区器Partitioner，默认是hash partitioner

一个Partitioner，即RDD的分片函数。当前Spark中实现了两种类型的分片函数，一个是基于哈希的HashPartitioner，另外一个是基于范围的RangePartitioner。只有对于于key-value的RDD，才会有Partitioner，非key-value的RDD的Parititioner的值是None。Partitioner函数不但决定了RDD本身的分片数量，也决定了parent RDD Shuffle输出时的分片数量。

5）如果读取的是HDFS的数据，那么会有一个最优位置。（在调度task之前会与namenode进行通信）

一个列表，存储存取每个Partition的优先位置（preferred location）。对于一个HDFS文件来说，这个列表保存的就是每个Partition所在的块的位置。按照“移动数据不如移动计算”的理念，Spark在进行任务调度的时候，会尽可能地将计算任务分配到其所要处理数据块的存储位置。


在Driver端创建一个RDD的实例，在其中记录了计算逻辑，可以操作分布在多台机器上的数据（通过hdfs api等接口）

#### 创建RDD
1）由一个已经存在的Scala集合创建。
```
val rdd1 = sc.parallelize(Array(1,2,3,4,5,6,7,8))
```

2）由外部存储系统的数据集创建，包括本地的文件系统，还有所有Hadoop支持的数据集，比如HDFS、Cassandra、HBase等
```
val rdd2 = sc.textFile("hdfs://node1:9000/words.txt")
```
3）调用一个已经存在的RDD的Transformation，会生成一个新的RDD
#### RDD分区个数
1.如果是将Driver端的Scala集合并行化创建RDD，并且没有指定RDD的分区，RDD的分区就是为该app分配的中的和核数

2.如果是重hdfs中读取数据创建RDD，并且设置了最小分区数量是1，那么RDD的分区数据即使输入切片的数据，如果不设置最小分区的数量，即spark调用textFile时会默认传入2，那么RDD的分区数量会大于等于输入切片的数量
	
先计算所有文件大小之和，会设置目标大小为该值除以最小分区数量的大小（默认值是2，也就是取所有文件大小之和的一半作为目标大小），如果某个文件大小超过目标值的1.1倍，就会将该文件按照目标大小切分
 
#### mapPartitionsWithIndex
RDD的map方法，是Executor中执行时，是一条一条的将数据拿出来处理

mapPartitionsWithIndex 一次拿出一个分区（分区中并没有数据，而是记录要读取哪些数据，真正生成的Task会读取多条数据），并且可以将分区的编号取出来

功能：取分区中对应的数据时，还可以将分区的编号取出来，这样就可以知道数据是属于哪个分区的（哪个区分对应的Task的数据）

	//该函数的功能是将对应分区中的数据取出来，并且带上分区编号
	
    val func = (index: Int, it: Iterator[Int]) => {
      it.map(e => s"part: $index, ele: $e")
    }
	rdd.mapPartitionsWithIndex(fun)

#### cache
RDD的cache方法是lazy的，不会产生一个新的RDD，只是标记原来的RDD要被cache（在第一次触发action的时候缓存数据）。如果任务需要触发多次action，可以考虑将数据cache到内存。   
- 缓存数据是在executor中，**占用executor所在的机器的内存**（一个task存放一块内存），如果hdfs数据不在当前节点上，需要通过网络拉取
- 什么时候进行cache（要求计算很快，内存资源足够，**cache的数据会多次触发action**）
- 建议根据精准条件filter之后，缩小范围之后再进行cache
- 可以使用cache.unpersist(true/false)将缓存的数据清空，true表示阻塞
- 如果内存空间不足，会自动采用内存+磁盘方式，只在内存中缓存一部分数据

注：在缓存数据的时候，放到内存的数据会比原来在文件系统的数据所用的空间大，因为直接存储了java对象，没有进行序列化，节省时间

#### checkpoint
使用场景
- 迭代计算，要求保证数据安全
- 对速度要求不高（与cache到内存相比）
- 将中间结果保存到hdfs

步骤：
- 设置checkpoint目录（hdfs目录）
- 经过复杂计算，得到中间结果
- 将中间结果checkpoint到指定的hdfs目录
- 后续计算就可以使用前面ck的数据了

做了checkpoint的RDD就会将该RDD之前的依赖关系释放掉，下次使用该RDD的数据的时候直接从hdfs中读取。

注：checkpoint和cache一样，不会生成新的RDD，只是标记当前的RDD需要checkpoint，然后在第一次触发action的时候执行chrckpoint，此时会有两个stage，一个进行计算，另一个进行checkpoint（将数据写入hdfs）


