##Spark API##
> 资源来源：http://homepage.cs.latrobe.edu.au/zhe/ZhenHeSparkRDDAPIExamples.html

### aggregate ###
- 它允许应用两个renduce函数，第一个reduce函数做`partition`分区内（**intra partitions**）的计算，  
- 第二个reduce做所有跨`partition`分区（**across partitions**）的计算。
- 它有初始化值得概念：`initial value`，这个值会应用在分区内和分区间计算两个层面。
- 分区间的计算不保证顺序  

- 例子1
>     val z = sc.parallelize(List(1,2,3,4,5,6), 2)
>     
>     // lets first print out the contents of the RDD with partition labels
>     def myfunc(index: Int, iter: Iterator[(Int)]) : Iterator[String] = {
>       iter.toList.map(x =>"[partID:" +  index + ", val: " + x + "]").iterator
>     }
>     
>     z.mapPartitionsWithIndex(myfunc).collect
>     res28: Array[String] = Array([partID:0, val: 1], [partID:0, val: 2],   
>     [partID:0, val: 3], [partID:1, val: 4], [partID:1, val: 5], [partID:1, val: 6])
>     
>     z.aggregate(0)(math.max(_, _), _ + _)
>     res40: Int = 9
>     
>     // This example returns 16 since the initial value is 5
>     // reduce of partition 0 will be max(5, 1, 2, 3) = 5
>     // reduce of partition 1 will be max(5, 4, 5, 6) = 6
>     // final reduce across partitions will be 5 + 5 + 6 = 16
>     // note the final reduce include the initial value
>     z.aggregate(5)(math.max(_, _), _ + _)
>     res29: Int = 16

- 例子2  
>     val z = sc.parallelize(List("a","b","c","d","e","f"),2)
>     
>     //lets first print out the contents of the RDD with partition labels
>     def myfunc(index: Int, iter: Iterator[(String)]) : Iterator[String] = {
>       iter.toList.map(x =>"[partID:" +  index + ", val: " + x + "]").iterator
>     }
>     
>     z.mapPartitionsWithIndex(myfunc).collect
>     res31: Array[String] = Array([partID:0, val: a], [partID:0, val: b],   
>     [partID:0, val: c], [partID:1, val: d], [partID:1, val: e], [partID:1, val: f])
>     
>     z.aggregate("")(_ + _, _+_)
>     res115: String = abcdef
>     
>     // See here how the initial value "x" is applied three times.
>     //  - once for each partition
>     //  - once when combining all the partitions in the second reduce function.
>     z.aggregate("x")(_ + _, _+_)
>     res116: String = xxdefxabc  注：多次计算时，结果也可能出现xxabcxdef，即分区间的计算  
>     不保证计算顺序

- 例子3  
>     // Below are some more advanced examples. Some are quite tricky to work out.
>     
>     val z = sc.parallelize(List("12","23","345","4567"),2)
>     z.aggregate("")((x,y)=>math.max(x.length, y.length).toString, (x,y) =x + y)
>     res141: String = 42
>     
>     z.aggregate("")((x,y)=>math.min(x.length, y.length).toString, (x,y) =x + y)
>     res142: String = 11
>     
>     val z = sc.parallelize(List("12","23","345",""),2)
>     z.aggregate("")((x,y)=>math.min(x.length, y.length).toString, (x,y) =x + y)
>     res143: String = 10 
>     这个例子比较有意思。aggragate有个迭代的概念在里面。分区内计算时会把每次计算的结果作为  
>     第二次计算的因子进行迭代计算。以这个例子为例，它分为两个分区的计算，第一个分区计算时，首先  
>     计算("", "12")=>math.min(x.length, y.length),结果是0，0这个结果会作为第二次计算的因子，  
>     即(0, "23")=>math.min(x.length, y.length)计算，结果是1，所以分区1计算的结果是1；来看分区2  
>     的计算，首先计算("", "345")=>math.min(x.length, y.length),结果是0，然后计算(0, "")  
>     =>math.min(x.length, y.length),结果是0。分区2的计算结果是0.最终的计算结果是10.那么在看另一  
>     个例子：  
>     val z = sc.parallelize(List("12","23","","345"),2)  
>     z.aggregate("")((x,y) => math.min(x.length, y.length).toString, (x,y) => x + y)  
>     res144: String = 11  
>     计算步骤：分区1的计算同上一个例子，结果为1，看分区2的计算：首先计算的是("","")=>math.min  
>     (x.length, y.length)，结果是0，然后计算的是(0, "345")=>math.min(x.length, y.length),结果  
>     是1，最后的结果是11。从这个例子看出，计算结果依赖了分区里面的数据排序，这是一种不好的设计。  

### aggregateByKey [Pair] ###
- 计算原理跟aggregate是类似的，不同之处在于：**聚集操作作用在同一个key上面，初始化值参加分区间的计算**  

- Listing Variants  
> 
>     def aggregateByKey[U](zeroValue: U)(seqOp: (U, V) ⇒ U, combOp: (U, U) ⇒ U)(implicit arg0: ClassTag[U]): RDD[(K, U)]
>     def aggregateByKey[U](zeroValue: U, numPartitions: Int)(seqOp: (U, V) ⇒ U, combOp: (U, U) ⇒ U)(implicit arg0: ClassTag[U]): RDD[(K, U)]
>     def aggregateByKey[U](zeroValue: U, partitioner: Partitioner)(seqOp: (U, V) ⇒ U, combOp: (U, U) ⇒ U)(implicit arg0: ClassTag[U]): RDD[(K, U)]

- 例子  
> 
>     val pairRDD = sc.parallelize(List( ("cat",2), ("cat", 5), ("mouse", 4),("cat", 12), ("dog", 12), ("mouse", 2)), 2)
>     
>     // lets have a look at what is in the partitions
>     def myfunc(index: Int, iter: Iterator[(String, Int)]) : Iterator[String] = {
>       iter.toList.map(x ="[partID:" +  index + ", val: " + x + "]").iterator
>     }
>     pairRDD.mapPartitionsWithIndex(myfunc).collect
>     
>     res2: Array[String] = Array([partID:0, val: (cat,2)], [partID:0, val: (cat,5)], [partID:0, val: (mouse,4)], [partID:1, val: (cat,12)], [partID:1, val: (dog,12)], [partID:1, val: (mouse,2)])
>     
>     pairRDD.aggregateByKey(0)(math.max(_, _), _ + _).collect
>     res3: Array[(String, Int)] = Array((dog,12), (cat,17), (mouse,6))  
>     分析：分两个分区进行计算，首先计算出分区1中cat，mouse和dog的最大值，初始值参加计算，结果是(cat,5),  
>     (mouse,4);分区2中cat，mouse和dog的最大值为(cat,12),(mouse,2),(dog,12)。最后结果是把key值相同的结  
>     果相加(cat,17),(mouse,6),(dog,12)。
>     
>     pairRDD.aggregateByKey(100)(math.max(_, _), _ + _).collect
>     res4: Array[(String, Int)] = Array((dog,100), (cat,200), (mouse,200))  
>     分析：分两个分区进行计算，首先计算出分区1中cat，mouse和dog的最大值，初始值参加计算，结果是(cat,100),  
>     (mouse,100);分区2中cat，mouse和dog的最大值为(cat,100),(mouse,100),(dog,100)。最后结果是把key值相同的结  
>     果相加(cat,200),(mouse,200),(dog,100)，此时初始值是不参加计算的。  

### cartesian ###
- 笛卡尔乘积:将两个RDD进行计算，第一个RDD的每一个元素都与第二个RDD的每一个元素进行join操作，将结果返回成为新  的RDD。笛卡尔积对内存消耗很大。  
  
- Listing Variants  
>     def cartesian[U: ClassTag](other: RDD[U]): RDD[(T, U)]  

- 例子  
>     val x = sc.parallelize(List(1,2,3,4,5))
>     val y = sc.parallelize(List(6,7,8,9,10))
>     x.cartesian(y).collect
>     res0: Array[(Int, Int)] = Array((1,6), (1,7), (1,8), (1,9), (1,10), (2,6), (2,7), (2,8), (2,9), (2,10), (3,6), (3,7), (3,8), (3,9), (3,10), (4,6), (5,6), (4,7), (5,7), (4,8), (5,8), (4,9), (4,10), (5,9), (5,10))  
>     分析：在构造rdd时，可以进行分片处理，处理后计算的任务数就是两个rdd分片的乘积。

### checkpoint ###
- checkpoint是将一个rdd以二进制文件的形式存放在指定的路径下，该路径可以使用SparkContext来指定。checkpoint操作需  要一个action操作来触发。
- checkpoint路径如果是本地的，需要在每一个节点上都存在，也可以用hdfs路径替代。
- Listing Variants
>     def checkpoint()  

- 例子  
>     sc.setCheckpointDir("my_directory_name")
>     val a = sc.parallelize(1 to 4)
>     a.checkpoint
>     a.count
>     14/02/25 18:13:53 INFO SparkContext: Starting job: count at <console>:15
>     ...
>     14/02/25 18:13:53 INFO MemoryStore: Block broadcast_5 stored as values to memory (estimated size 115.7 KB, free 296.3 MB)
>     14/02/25 18:13:53 INFO RDDCheckpointData: Done checkpointing RDD 11 to file:/home/cloudera/Documents/spark-0.9.0-incubating-bin-cdh4/bin/my_directory_name/65407913-fdc6-4ec1-82c9-48a1656b95d6/rdd-11, new parent is RDD 12
>     res23: Long = 4  

- 解读  
>     转自：http://blog.csdn.net/xiao_jun_0820/article/details/50475351
>     /**
>     * Mark this RDD for checkpointing. It will be saved to a file inside the checkpoint
>     * directory set with `SparkContext#setCheckpointDir` and all references to its parent
>     * RDDs will be removed. This function must be called before any job has been
>     * executed on this RDD. It is strongly recommended that this RDD is persisted in
>     * memory, otherwise saving it on a file will require recomputation.
>     */
>     
>     这是源码中RDD里的checkpoint()方法的注释，里面建议在执行checkpoint()方法之前先对rdd进行persisted操作。
>     
>     为啥要这样呢？因为checkpoint会触发一个Job,如果执行checkpoint的rdd是由其他rdd经过许多计算转换过来的，如果你  
>     没有persisted这个rdd，那么又要重头开始计算该rdd，也就是做了重复的计算工作了，所以建议先persist rdd然后再  
>     checkpoint，checkpoint会丢弃该rdd的以前的依赖关系，使该rdd成为顶层父rdd，这样在失败的时候恢复只需要恢复该  
>     rdd,而不需要重新计算该rdd了，这在迭代计算中是很有用的，假设你在迭代1000次的计算中在第999次失败了，然后你没有  
>     checkpoint，你只能重新开始恢复了，如果恰好你在第998次迭代的时候你做了一个checkpoint，那么你只需要恢复第998  
>     次产生的rdd,然后再执行2次迭代完成总共1000的迭代，这样效率就很高，比较适用于迭代计算非常复杂的情况，也就是恢复  
>     计算代价非常高的情况，适当进行checkpoint会有很大的好处。  

- 引申阅读  
>     http://www.tuicool.com/articles/qQ3eYv 《Spark的Cache和Checkpoint》

###coalesce, repartition
- 将RDD重新组合成确定数目的分区，对于coalesce来说，如果第二个参数shuffle设置为false，则不能讲分区数目增大，只能减小。
- Listing Variants
> def coalesce ( numPartitions : Int , shuffle : Boolean = false ): RDD
> [T] def repartition ( numPartitions : Int ): RDD [T]

- 例子
> val y = sc.parallelize(1 to 10, 10) val z = y.coalesce(2, false)
> z.partitions.length res9: Int = 2

### cogroup [Pair], groupWith [Pair]
- 对两个RDD中的KV元素，每个RDD中相同key中的元素分别聚合成一个集合。与reduceByKey不同的是针对两个RDD中相同的key的元素进行合并.

- 例子
> val a = sc.parallelize(List(1, 2, 1, 3), 1) 
> val b = a.map((_, "b"))
> val c = a.map((_, "c")) b.cogroup(c).collect 
> res7: Array[(Int,(Iterable[String], Iterable[String]))] = Array(
> (2,(ArrayBuffer(b),ArrayBuffer(c))),(3,(ArrayBuffer(b),ArrayBuffer(c))), (1,(ArrayBuffer(b,b),ArrayBuffer(c, c)))) val d = a.map((_, "d")) b.cogroup(c,
> d).collect res9: Array[(Int, (Iterable[String], Iterable[String],
> Iterable[String]))] = Array( (2，(ArrayBuffer(b),ArrayBuffer(c),ArrayBuffer(d))),
> (3,(ArrayBuffer(b),ArrayBuffer(c),ArrayBuffer(d))), (1,(ArrayBuffer(b,b),ArrayBuffer(c, c),ArrayBuffer(d, d)))) 
> val x =sc.parallelize(List((1, "apple"), (2, "banana"), (3, "orange"), (4,"kiwi")), 2) 
> val y = sc.parallelize(List((5, "computer"), (1,
> "laptop"), (1, "desktop"), (4, "iPad")), 2) 
> x.cogroup(y).collect
> res23: Array[(Int, (Iterable[String], Iterable[String]))] = Array(
> (4,(ArrayBuffer(kiwi),ArrayBuffer(iPad))), 
> (2,(ArrayBuffer(banana),ArrayBuffer())), 
> (3,(ArrayBuffer(orange),ArrayBuffer())),
> (1,(ArrayBuffer(apple),ArrayBuffer(laptop, desktop))),
> (5,(ArrayBuffer(),ArrayBuffer(computer))))

### collectAsMap [Pair]

- 根collect算子相似，适用于key-value型的RDD，collect时保持其key-value的结构不变，最终输出scala map的类型。
- 若存在相同key的键值对，后面的value会覆盖前面的value，key值保持唯一
- Listing Variants
> def collectAsMap(): Map[K, V]

- 例子
> val data = sc.parallelize(List((1, "a"), (1, "A"), (2, "b"), (2, "B"), (3, "c"), (3, "C")))
> data.collectAsMap 
> scala.collection.Map[Int,String] = Map(2 -> B, 1 -> A, 3 -> C)
> **解读**：collect结果是Array[(Int, String)] = Array((1,a), (1,A), (2,b), (2,B), (3,c), (3,C))，collectAsMap结果是scala.collection.Map[Int,String] = Map(2 -> B, 1 -> A, 3 -> C)，可以看出对相同的key值来说，后面一个value将前一个value替换。

### combineByKey[Pair]
- 根据key进行combine操作，第二个算子先combine value，第三个算子combine结果。
- 例子
> val a =
>sc.parallelize(List("dog","cat","gnu","salmon","rabbit","turkey","wolf","bear","bee"),3) 
>val b = sc.parallelize(List(1,1,2,2,2,1,2,2,2), 3) 
>val c = b.zip(a)
>val d = c.combineByKey(List(_), (x:List[String], y:String) => y :: x, (x:List[String], y:List[String]) => x ::: y) 
> d.collect 
> res16:Array[(Int, List[String])] = Array((1,List(cat, dog, turkey)),
> (2,List(gnu, rabbit, salmon, bee, bear, wolf)))

- 参考
> http://www.tuicool.com/articles/miueaqv 这篇文章分析的很好，推荐

### countByKey [Pair]
- 值得注意，该算子返回的是拥有同一个key的value的个数与key的组合

- 例子
> val c = sc.parallelize(List((3, "Gnu"), (3, "Yak"), (5, "Mouse"),
> (3,"Dog")), 2)  c.countByKey  res3: scala.collection.Map[Int,Long] =
> Map(3-> 3, 5 -> 1)

### countByValue
- 返回一个map，key是rdd里面的元素，value是这个元素的存在个数，相同元素个数累加。

- 例子
> val b = sc.parallelize(List(1,2,3,4,5,6,7,8,2,4,2,1,1,1,1,1))
> b.countByValue 
> res27: scala.collection.Map[Int,Long] = Map(5 -> 1, 8-> 1, 3 -> 1, 6 -> 1, 1 -> 6, 2 -> 3, 4 -> 2, 7 -> 1)
