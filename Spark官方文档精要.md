## Spark官方文档精要 V2.0.2
### Spark Programming Guide
- **overview**  
每一个Spark应用都会包含一个运行着用户main函数的driver程序和并行运行在集群上的executors。Spark提供的最主要的抽象就是RDD，RDD是分布在集群节点上的经过预分区的元素的集合，RDD实现了数据的并行处理。RDD可以从HDFS上的文件上创建，也可以从已有的Scala集合上创建，在driver程序中，可对已创建的RDD进行transform操作，使之成为另一个RDD。Spark能够将已创建的RDD持久化在内存中，这样在迭代计算和并行计算中，就能够重用这些持久化的RDD，使计算更加高效。RDDs还能够从失败的节点上自动恢复。
Spark中的第二个主要的抽样是**共享变量**，它支持并行操作。默认情况下，当Spark在集群节点上以一组tasks的方式运行程序的时候，Spark为每一个task都创建了程序中的每一个变量的拷贝。在有些情况下，某个变量需要在不同task之间或者task和driver程序之间共享。Spark实现了两种共享变量：`broadcast variables`和`accumulators`，其中`broadcast variables`在所有集群节点上都会缓存一个变量；`accumulators`起到一个累加器的作用，它只支持`added`的操作，当你要实现`counters`或者`sums`的时候，累加器会非常有用。
  - **broadcast variables**  
`解析`
  - **accumulators**  
`解析`
- **Linking with Spark**  
Spark的编译运行默认使用Scala-2.11版本。如果使用Scala编写Spark应用，推荐使用Scala2.11.x系列版本编译应用程序。 
	 
  编写maven工程的Spark应用程序，需添加依赖：
	> groupId = org.apache.spark  
	> artifactId = spark-core_2.11  
	> version = 2.0.1  
		
	如果想要通过Spark访问HDFS，需添加依赖：  
	>groupId = org.apache.hadoop  
	>artifactId = hadoop-client  
	>version = <your-hdfs-version>  

- **Initializing Spark**  
	编写Spark应用程序的第一步就是初始化一个SparkContext对象，它是连接Spark和集群计算资源的桥梁，是所有后续计算的入口。SparkContext的初始化需要SparkConf对象，这里面有包含了Spark应用的信息。  
	> val conf = new SparkConf().setAppName(appName).setMaster(master)  
	> sc = new SparkContext(conf)
	
	值得注意的是，每一个JVM里面只允许存在一个SparkContext对象，如果想创建一个新的SparkContext，需要先用stop()将之前的SparkContext关闭。  
	> sc.stop()  

	使用spark-submit提交应用可以动态的传入spark任务参数，如`master`参数、`driver-memory`参数等。如果需要本地调试，则可以把`master`字段设置为`local`
	
- **Resilient Distributed Datasets (RDDs)**  
1、spark每个分区对应一个任务，每颗CPU对应2~4个分区
2、可以从本地文件创建RDD，但是使用本地文件路径必须保证在每个节点都能访问该文件，如每个节点都有一份文件拷贝或者使用共享文件路径
3、textFile(filePath)中的文件路径支持目录和通配符格式；使用textFile可以重新分区，但是分区的数目不能小于HDFS的block数目
4、所有的变换都是lazy的，及变换只有在需要返回给driver运行结果时才会进行计算
5、默认情况下，每次对一个RDD执行action运算，这个RDD都需要被重算。但是可用cache或者persist将RDD缓存在内存里面。中间结果缓存提高后续计算性能。
6、理解RDD里面的闭包。看下面一段代码：
>var counter = 0
>var rdd = sc.parallelize(data)
>// Wrong: Don't do this!!
>rdd.foreach(x => counter += x)
>println("Counter value: " + counter)  

	在执行任务之前，spark首先会对task的闭包函数进行验证，闭包函数中使用的变量和函数对于executor来说必须是可见的，进而应用在RDD的元素上。  

	如果在local模式下运行，这段代码可能会输出预期的结果，因为counter变量和driver进程是在一个jvm中；但是在集群模式下，counter变量已经被分发到执行器节点的jvm中，对counter的操作只能发生在执行器所在jvm的内存中，因此打印的内容也只能显示在executor的stdout里。通过在执行器jvm中改变counter值是无法改变driver端counter值的，所以在上面的代码中，counter的值还是0。如果确实有如上面代码逻辑的需求，spark里面提供了`accumulator`可以使用。打印RDD元素也是同样的道理，你无法在driver端打印executor端内存里面的变量。


- **Shuffle Operation**  
	**会触发shuffle的操作:**
	1. **Repartition**: `repartition`,`coalesce`
	2. **Bykey**:`groupByKey` , `reduceByKey`
	3. **join**: `cogroup`, `join`  

	**Performance Impact:**  
	1. Shuffle是一种资源密集型的操作，涉及到disk I/O,  network I/O和数据的序列化。Spark会产生一组map tasks来组织数据，一组reduce tasks来聚合数据。
	2. Shuffle操作同时会在磁盘上产生大量的临时文件，直到相对应的RDD不再使用并垃圾回收后这些临时文件才会删除。对于长时间运行的Spark任务来说，临时文件会耗费大量的磁盘空间。临时空间所在的目录通过spark.local.dir配置。

- **RDD Persistence**  
1. RDD第一次在action操作中被计算时，才会缓存在内存中：
		```
		The first time it is computed in an action, it will be kept in memory on the nodes.
		```
2. RDD如果缓存在内存中，为了节省空间，是以序列化Java对象的方式存储的。
3. 在Shuffle过程中，为了提高计算性能，Spark也会主动去persist一些中间结果。
4. 对于`resulting RDD`，如果在后面的计算中要重用，建议进行persist提升计算性能。
5.  尽量不要采取spill磁盘的方法缓存数据，除非重算的代价特别大或者有大量的中间结果数据。一般来说，重算一个partition跟从磁盘读数据耗时是同样的。

- **Shared Variables**  
1. broadcast variables:只读变量
使用场景：每个task计算都需要一个公共的而且数据量比较大的数据集时，可以将这个数据集广播到各个节点。
2. accumulators
能够并行实现adds和sums运算，只能增加。
task线程不能读取accumulators的值，只能向acc增加数据；只有driver端进程才能读取acc数据。
acc操作需要在action中触发，在具有lazy属性的map操作中，accu不能实现递增的效果。
	
