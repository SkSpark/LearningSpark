### MAVEN编译打包发布版本	

#### 编译
- 运行以下命令，防止编译时内存溢出   

	```
export MAVEN_OPTS="-Xms512m -Xmx3g -XX:MaxPermSize=256m -XX:ReservedCodeCacheSize=64m"
	```  

- 编译Spark可用./build/mvn来编译，如果没连外网，需要将scala版本和zinc版本拷贝至build目录下。zinc主要用来编译提速和增量编译：  
**scala：** [scala-2.11.8](http://www.scala-lang.org/download/2.11.8.html)  
**zinc：** [zinc-0.3.9](https://github.com/alixGuo/Resources/blob/master/zinc-0.3.9.rar)  

	**编译命令：**
	```
./build/mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.4.0.8 -Phive -Phive-thriftserver -DskipTests clean package
	```  
	尽量不要clean，编译时间会延长。  

	spark包含多个子项目，可通过`-pl`参数来指定子模块，从而单独编译某个子项目。如：
	`mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.4.0.8 -Phive -Phive-thriftserver -pl sql/hive -DskipTests clean package`  

	如使用公司内私服，需要在spark根目录pom文件中设置：  
![Alt text](https://github.com/alixGuo/Resources/blob/master/2016121201.png)  

	顺便写一下，Spark应用工程引用公司私服，设置：  
![Alt text](https://github.com/alixGuo/Resources/blob/master/2016121202.png)

#### 打包
- spark/dev目录下有make-distribution.sh，这个脚本是编译打包一体的。也可以改写该脚本，将编译命令去掉，手动编译，查看编译过程。

#### 发布
- 将本地定制化的spark上传至公司maven库。  
**需要配置几个地方：**  
编辑机maven的setting文件，设置server，上传文件时会进行用户校验。
![Alt text](https://github.com/alixGuo/Resources/blob/master/2016121203.png)  

	配置spark根目录pom文件，配置distribution管理信息：
![Alt text](https://github.com/alixGuo/Resources/blob/master/2016121204.png)


	整体发布命令：  
	```
mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.4.0.8 -Phive -Phive-thriftserver  -DskipTests  deploy
	```

	也可以使用`-pl`命令单独发布一个子项目：
	```
mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.4.0.8 -Phive -Phive-thriftserver -pl sql/catalyst -DskipTests clean deploy
	```

	或者单独上传一个jar包（脚本批量上传多个jar包）：  
	```
for i in $(ls *.jar);do mvn deploy:deploy-file -DgroupId=org.apache.spark -DartifactId=spark-assembly_2.11 -Dversion=2.0.2.1-SNAPSHOT -Dpackaging=jar -Dfile=$i -Durl=http://maven.cnsuning.com/content/repositories/snapshots/ -DrepositoryId=snapshots; done
	```



