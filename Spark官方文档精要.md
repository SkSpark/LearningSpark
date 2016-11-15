## Spark官方文档精要
### Spark Programming Guide
- **overview**  
每一个Spark应用都会包含一个运行着用户main函数的driver程序和并行运行在集群上的executors。Spark提供的最主要的抽象就是RDD，RDD是分布在集群节点上的经过预分区的元素的集合，RDD实现了数据的并行处理。RDD可以从HDFS上的文件上创建，也可以从已有的Scala集合上创建，在driver程序中，可对已创建的RDD进行transform操作，使之成为另一个RDD。Spark能够将已创建的RDD持久化在内存中，这样在迭代计算和并行计算中，就能够重用这些持久化的RDD，使计算更加高效。RDDs还能够从失败的节点上自动恢复。
Spark中的第二个主要的抽样是**共享变量**，它支持并行操作。默认情况下，当Spark在集群节点上以一组tasks的方式运行程序的时候，Spark为每一个task都创建了程序中的每一个变量的拷贝。在有些情况下，某个变量需要在不同task之间或者task和driver程序之间共享。Spark实现了两种共享变量：`broadcast variables`和`accumulators`，其中`broadcast variables`在所有集群节点上都会缓存一个变量；`accumulators`起到一个累加器的作用，它只支持`added`的操作，当你要实现`counters`或者`sums`的时候，累加器会非常有用。
  - **broadcast variables**  
	> 解析
  - **accumulators**    
	> 解析 
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
	
