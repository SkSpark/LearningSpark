### Structured Streaming ###

----------
#### Overview
Structured Streaming是建立于Spark-SQL引擎之上的流处理引擎，使用Structured Streaming可以使用户向处理静态数据一样处理流数据。Structured Streaming通过checkpoint和WAL来保证端到端的`exactly-once`容错。

#### 模式

```
val lines = spark.readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()
  
// Split the lines into words
val words = lines.as[String].flatMap(_.split(" "))

// Generate running word count
val wordCounts = words.groupBy("value").count()
```
lines DataFrame代表了从socket接受的流式文本数据，可以把lines看成是一个无边界表，流式文本数据的每一行流进这个表成为一个row，Structured Streaming就是对每一行进行计算，然后将结果输出的。

**Result Table**  
每一批计算出来的结果又会更新到另一个无限表Result Table中。

**Output三种模式**：  
- complete Mode：将整个更新的Result Table表都输出到外部存储
- Append Mode：只将上次触发以后Result Table新增的结果行输出到外部存储。这种模式适用于Result Table已存在的结果行不发生改变的查询。
- Update Mode：只将上次触发以后Result Table更新的结果行输出到外部存储。跟Complete Mode不同之处是，Update Mode对于没有发生改变的结果行是不输出到外部存储的。现在的Spark2.0还未支持Update Mode。

这三种输出模式只适用于特定的查询，并不能支持所有的查询。

**significantly different**  
Structed Streaming跟其他流计算引擎很大的不同是，它能够代替用户做更新结果表的工作，其他的引擎需要用户自己去更新Result Table
