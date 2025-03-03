分布式计算Spark介绍、本地&ADN实践分享

# 摘要：
围绕AI的数据工程，遵循着摩尔定律。不管模型、应用层怎么迭代，数据处理由单机并发的原始时代已处于集群并发的现代时期，本着“存算分离”原则。动不动TB级甚至PB级原始数据的加工处理，对执行效率的诉求也愈加强烈，Apache Spark就这样应运而生。除了本身对数据做分布式完成‘负载均衡’外，还提供不少例如数据集DataFrame、应用层SQL操作(函数计算)、各数据源连接、各文件类型(包括parquet)读写加载、MLLIB等类库支持。
本篇从大数据相关背景出发、介绍了关于Apache Spark的架构及各组件元素、本地组件集群的快速实践 及 最后关于ADN通信大模型关于引入Spark对语料的ETL处理、安全合规扫描进行分布式计算提速的分享。

# 大数据处理相关背景
AI工程之前，软件工程时代，随着系统用户量的逐渐膨胀，系统架构的演进：计算层面即由单机处理单机并发集群并发，存储层面则由分表(table)分库(schema)分区(patition)。计算 & 存储 两者本身是相互解耦的，也即存算分离、弹性可扩展的原则。当然高可用(韧性)角度，进一步考虑到异地容灾之类(本篇暂不讨论细则)。
过度到AI的大数据，训练一个大模型/传统小模型，一个或若干个数据集总量动不动则上TB级。如果软件时代数据的量称为大数据，则AI时代数据的量可成为大大数据；会更上一个台阶的数量级，少则TB级，多则PB级。
万变不离其宗，这里的大/大大数据处理的思路是一致的。在具体工具/框架应用层面，以Apache Spark作为典型代表，如雨后春笋般正被各方用户广泛接纳。其普及程度，可以形象比喻为JAVA届的Spring Framework。

# 架构层分析
## 1. Scala, JAVA, Python等编程语言之间的关系
首先补充介绍下spark相关的各语言之间的关系：Spark是基于Scala语言编写开发的，而Spark的执行，又得基于JVM。其中，Scala是门解释性语言，跟groovy一样。构建的gradle(本人之前有多年API级使用经验)也是基于groovy编写，运行也基于JVM。
这样发展趋势是因为：JAVA编写代码时，每次执行前都需要静态编译；而scala, groovy等解释型语言，则借鉴了前端(如JS)开发的特性；以及编程范式的演进思路(函数是一等公民)。基于此，JAVA 8相较7是一次里程碑式的变更，引入lambda表达式，也是基于该思路 & 闭包特性不得不做的随波逐流般的适配跟进(此时解释型语言该特性已相当成熟)。

## 2. Spark如何支持Python语言脚本开发的
Spark Framework除了对原生JAVA语言的支持外，对其他编程语言(如Python, R等)亦有重大建树。尤其咱看下AI届的母语python如何支持的？首先通过Pyspark(官方解释：PySpark is the Python API for Apache Spark)，然后通过Py4j(官方解释：enables Python to call Java-based Spark components)这一“桥梁”来完成JVM的适配工作。

本地环境中，当你编写spark的python脚本时，需先装好Pyspark & Py4j这两个组件包依赖。

## 3. 关于Spark各模块
最上层API级Layer，各语言通过‘Adapter’对接Spark；接着是Spark的类库有SQL，Streaming, GraphX, MLlib这几个类库LIB

#### 3.1 Spark RDD (Resilient Distributed Dataset)：
RDD是Spark对数据集的抽象用于后续的数据作业，后续3.2会介绍SQL LIB操作。

#### 3.2 Spark SQL & DataFrames: 
Spark SQL is Apache Spark’s module for working with structured data。With PySpark DataFrames you can efficiently read, write, transform, and analyze data using Python and SQL

这里如下校验的一个python脚本，通过spark RDD加载一个本地csv文件，得到一个DataFrame 实例的句柄。如下：
dataframe = sparkApp.read.schema(schema).csv(r'../files/testData.csv')
dataframe.show()

拿到DataFrame的句柄后，进一步就可以通过编写类似DB的SQL方式，提供了一系列快速数据处理API，包括了含groupby，sum，mapReduce等一系列操作。好比数据平台翘楚Databricks的湖仓一体架构的上层delta SQL协议。支持的数据源有本地文件(含各种类型如csv,json,parquet,orc等)，数据库(如通过JDBC相连)。
df.select(df['name'], df['age']).show()

因此，在应用层面，可直接基于Spark的API，快速完成数据的变形、过滤、分析等数据处理操作。这样的好处是：减少自己额外写很多处理脚本二次开发，以及还要反复考虑处理效率问题。

#### 3.3 Pandas API
提供Python Pandas类库的API支持

#### 3.4 Structured Streaming
流式计算Streaming模块是在Spark SQL模块上演进而来的。流式计算其实可以认为是一个‘无边界’的批处理batch processing. Spark 提供了不同时间段的动态控制，定制获取。
Structured Streaming is a scalable and fault-tolerant stream processing engine built on the Spark SQL engine. You can express your streaming computation the same way you would express a batch computation on static data.

#### 3.5 MLlib & GraphX
Spark也提供关于machine learning & 图处理的类库支持。当前使用暂且不多，因此点到即止。

# 关于Spark内核各组件
这里最重要的是把几个核心的组件搞明白。
**Application**: User program built on Spark. Consists of a driver program and executors on the cluster. 我们平台开发的每个spark程序，其实就是以一个application应用为单元。场景化可理解为映射到DataOps作业流中具体的一个处理插件，每个作业处理脚本，封装为一个插件，每个插件就是一个application单元。
**Driver program**: The process running the main() function of the application and creating the SparkContext。Driver Node为每个spark application创建相应的SparkContext上下文. 上下文可以加载任何configuration配置(后续如何分配资源，如何控制并发，performance turning均跟这些conf相关)
**Cluster manager**: An external service for acquiring resources on the cluster (e.g. standalone manager, Mesos, YARN, Kubernetes) 。Cluster manager用于分配集群的资源(根据上面的Configuration配置) 到具体的Executor Node。集群的类型有Spark自身的standalone(一会本地实践会介绍)，也有云计算方式的Mesos、Yarn、K8S集群(Dataops所采用)。
**Deploy mode**: Distinguishes where the driver process runs. In "cluster" mode, the framework launches the driver inside of the cluster. In "client" mode, the submitter launches the driver outside of the cluster. Spark有两种部署模式，Cluster & Client方式；区别在于Spark Driver是位于Cluster集群内(可选择任一节点)？还是集群外。
**Worker node**：Any node that can run application code in the cluster。集群内的工作节点，也即Spark Driver初始化好SparkContext，通过ClusterManager分配资源后，具体干活Executor所在集群内的工作节点
**Executor**：A process launched for an application on a worker node, that runs tasks and keeps data in memory or disk storage across them. Each application has its own executors. 具体执行作业任务的程序，真正消耗内存、磁盘、CPU的地方。
**Task**：A unit of work that will be sent to one executor。将一个executor划分为多个逻辑的task单元。
**Job**：A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action (e.g. save, collect); you'll see this term used in the driver's logs.
**Stage**：Each job gets divided into smaller sets of tasks called stages that depend on each other (similar to the map and reduce stages in MapReduce); you'll see this term used in the driver's logs.

# 实现代码剖析
其实就是从用户触发脚本执行开始，将上述的各个组件对象的交互工作，具体回味下。
【参考文献https://dongma.github.io/2021/02/18/spark-standalone-mode/】
以Spark的standalone模式为例：

## 1. Client端
1)	客户端通过spark-submit.sh运行脚本
2)	进而调用spark-class.sh
3)	执行Spark launcher的主main函数：org.apache.spark.launcher.Main#main(String[] argsArray)，提取submit的各项运行参数
4)	执行Spark deploy的主函数：org.apache.spark.deploy#main(), 解析上述提交的参数，拆解具体提交的Action
5)	执行SparkSubmit#submit(SparkSubmitArguments, Boolean)，将提交的参数依赖包加载到的classpath，完成执行前的依赖解析
6)	执行SparkApplication#start(childArgs.toArray, sparkConf), 过滤系统中的环境变量，只保留以 SPARK_ or MESOS_开头的环境变量
7)	执行RestSubmissionClientApp#run()，将sparkConf转换为sparkProperties并进行过滤
8)	执行RestSubmissionClientApp#createSubmission()，验证所有masters地址，开始构建submitUrl然后逐个向master发送请求

## 2. Master端
1)	Master接收到客户端post请求：start-master.sh中调用守护进程spark-daemon.sh
2)	调用Master#receive(), 接收到netty提交的请求，先注册Spark Application
3)	执行StandaloneRestServer#handleSubmit(String, SubmitRestProtocolMessage, HttpServletResponse)，Master接收到submit请求，通过DeployMessages.RequestSubmitDriver(driverDescription)申请启动Spark Driver
4)	执行Master#receiveAndReply(context: RpcCallContext)，用createDriver(description)对DriverDescription再进行一次封装，同时通过schedule()进行资源调度到Worker上
5)	执行Worker#receive()，调用LaunchDriver(driverId, driverDesc)，
6)	执行driver.start()，创建Driver所需要的工作目录，同时download用户自定义的jar包 然后开始运行Driver
7)	执行worker#prepareAndRunDriver()，调用CommandUtils.buildProcessBuilder()，结合command所要运行的环境，重新构建一个命令
8)	执行DriverManager#main(args: Array[String])，通过自定义的classLoader加载jar包，根据mainClass通过反射执行其main()方法，触发用户程序的执行【Spark Driver已启动】
9)	执行SparkContext#createTaskScheduler(SparkContext, String, String)，初始化StandaloneSchedulerBackend类
10)	执行StandaloneSchedulerBackend#start()，用CoarseGrainedExecutorBackend构建command命令，然后构建ApplicationDescription对象，将其传入appClient并向Master发起应用注册的请求
11)	执行StandaloneAppClient#tryRegisterAllMasters()，方法中发送RegisterApplication(appDescription, self)，Master端收请求后会重新运行schedule()
12)	执行Worker#receive()方法，根据case匹配到LaunchExecutor的请求，构建ExecutorRunner对象(参数中包含masterUrl, appId, execId, appDesc, cores_, memory_等重要信息)并调用其start()方法
13)	执行ExecutorRunner#start()，首先创建了一个worker线程用于执行任务，要执行的方法为fetchAndRunExecutor()。在方法中通过CommandUtils.buildProcessBuilder()创建进程，然后设置执行路径、环境变量以及spark UI相关内容，然后启动进程CoarseGrainedExecutorBackend【Spark Executor已启动】
14)	执行CoarseGrainedExecutorBackend#receive()，接收case LaunchTask(data)的请求，当executor初始化好之后执行executor.launchTask(this, taskDesc)方法 
15)	TaskRunner#run()，设置TaskMemoryManager、序列化jar文件、初始化各种Metrics统计信息，然后通过task.run()的任务就正常执行了【Spark Task已启动执行】

# 快速实践
## 1. 安装
a. 参考https://spark.apache.org/downloads.html, 选择相应的Spark版本及基对应的Hadoop包类型，下载该Spark压缩包。
b. 解压后，设置好HADOOP_HOME及PATH环境变量
c. 设置好PATH环境变量后，{HADOOP_HOME}/bin下的.sh(Linux)及.cmd(windows)文件，便可执行了。

**Tips**: 这里特别注意，由于原生下载的spark-3.4.5-bin-hadoop3.tgz，hadoop默认只支持Linux环境，若需要在windows环境下执行，还得额外下载对应hadoop版本的
Hadoop.dll & winutils.exe文件，将其至于{HADOOP_HOME}/bin目录下；否则会报执行不起来。至于具体原因呢，参考官方解释https://cwiki.apache.org/confluence/display/HADOOP2/WindowsProblems，引入Hadoop.dll & winutils.exe该两文件后，hadoop即可使用windows API来实现posix-like file access permission。

## 2. 快速DEMO 
【环境变量PATH设置好{HADOOP_HOME}/bin的前提下】

### 2.1 执行pyspark
直接执行pyspark，即可快速启动一个Python的spark application, 类似启动一个SpringBoot应用小程序；可通过启动后的4040端口查看相应的Spark Context信息

### 2.2 执行spark-submit
通过spark-submit处理脚本，执行Spark example中的SparkPi类，并指定部署的模式及执行的Executor并发数，最后释放相关资源

### 2.3 启动Spark standalone模式的集群
【本地快速验证直接使用裸金属BMC方式，K8S等容器执行，见后续的生产应用】
a. 先启动Spark Master：spark-class org.apache.spark.deploy.master.Master
可通过8080端口查看MasterUI
b. 启动worker，注册到上述Spark Master: 
spark-class org.apache.spark.deploy.worker.Worker spark://10.37.122.26:7077
c. 按上述步骤b，再启动一个worker
d. 执行一起python的处理程序，启动一个Spark Application, 指派到上述Spark Cluster中，并执行完任务
如上图，在Intellij IDE中，编写python脚本，并执行。
注意，上述配置特意指定了dynamicAllocation，用于动态分配Worker Node的最大、最小Executor。默认会是10个；为了减少初始资源浪费 & 方便查找executor执行后的具体log, 将最小线程数放小

### 2.4 监控观测
**Spark Driver**: 通过4040端口，可以看到执行的Executor、JOB情况
**Spark Master**：默认通过8080端口；可以查看到集群中的Workers, 正在运行及已完成的applications
**Spark Worker**: 默认通过8081端口；可以查看正在执行&已完成的executors
**Executor running event logs**：通过上图Worker中执行的Executor Logs链接，查看详情如下：
