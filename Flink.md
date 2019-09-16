# `Flink`

[参考的地址](<https://confucianzuoyuan.github.io/flink-tutorial/>)

批处理的特点：

* 有界，持久，大量，适合访问全套记录，用于离线

流处理：

* 无界，实时，无序针对整个数据集执行操作，用于实时

`Flink`的几大模块：

* `Flink `Table & SQL(还没开发完)
* `Flink Gelly`(图计算)
* `Flink `CEP(复杂事件处理)

**`Flink`是一个框架和一个分布式处理引擎，用于对无界或者有界数据流进行状态计算。**

原本是德国的项目，约在14年的时候代码捐赠给了apache基金会，`flink`在德语中有快速灵活的意思。

流式处理就是来一条数据，处理一条数据，会遇到比批处理更多的问题，并且实现：

* 低延迟
* 高吞吐
* 结果的准确性和容错性

传统的数据一本分为两大类，一类是要做事务处理，一类是要做分析处理。

**OLAP**：联机的是分析处理

![1564499273098](img/flk/1.png)

**TP**：联机的事务处理

![1564498866690](img/flk/2.png)

传统的两种处理数据的方式，延迟都很高，都不太满足实时性的要求，所以需要做流式处理。

**有状态的流式处理**

![1564530460899](img/flk/3.png)

**起初的流式处理框架，为了实时性牺牲了正确性**，上图的架构表明：数据进入了`Flink`之后进行的处理逻辑，在本地的内存存储一份，还要在远程备份一份以达到数据可靠性的需求。但是在分布式系统在处理流式的数据的时候还要面对一个问题：**数据达到的先后顺序问题，举一个例子来说A事件可能先于B事件发生，但是B事件可已经处理完了，A事件还没有到达**。但是批处理的框架就不会遇到这样的问题，因为批处理过程会将连续的时间暂存起来，能够确定事件发生的先后顺序。

最早的实时处理框架是storm，这个框架不能承受高吞吐，能够保证低延迟，但是无法保证准确性。

由于流处理没有计算结束时间的概念，我们使用延迟和吞吐量来表征流处理的性能要求。

## 流处理框架的演变

为了保证流处理框架的低延迟和准确性，出现了新的架构***Lambda*** 架构：

![1564530920732](img/flk/4.png)

Lambda架构尽管解决了低延迟和准确性的问题，但是达到相同的功能却需要写两套API，提高了学习的成本和维护的成本，所以此时`flink`出现了。

## `Flink的优势`

* 低延迟，
* 高吞吐
* 在压力下保持正确性
* 时间正确和语义化窗口（也即乱序数据有能很好的处理）

好的，

## `Flink`的特点

### 事件驱动（event-driven）

这种以流为世界观的架构，获得的最大好处就是具有极低的延迟。

![1564614315541](img/flk/5.png)

 这里可以发现，`flink`的流式处理和传统的事务处理方式做了一个对比。

### 基于流

在`Flink`的世界观中，一切都是流，离线数据是有界的流，实时数据是无界的流。

![1564614576916](img/flk/6.png)

### 分层API

`Flink`的API，顶层抽象，使用方便，底层具体使用灵活。

### 其他特点：

* 支持事件时间，和处理时间
* 精确一次的状态一致性的保证
* 低延迟，美妙处理数百万事件，毫秒级延迟
* 与众多常用的存储系统连接
* 高可用，动态扩展，实现7 * 24小时全天候运行

## Spark和`Flink`的对比

### 流和批

Micro-Batching计算模式认为"流是批的特例"，流计算就是将连续不断的批进行持续计算，如果批足够小那么就有足够小的延时，在一定程度上满足了99%的实时计算场景。那么那1%为啥做不到呢? 这就是架构的魅力，***在Micro-Batching模式的架构实现上就有一个自然流数据流入系统进行攒批的过程，这在一定程度上就增加了延时。***

![1564615104348](img/flk/7.png)

Native Streaming计算模式认为批是流的特例"，这个认知更贴切流的概念，比如一些监控类的消息流，数据库操作的binlog，实时的支付交易信息等等自然流数据都是一条，一条的流入。Native Streaming计算模式每条数据的到来都进行计算，这种计算模式显得更自然，并且延时性能达到更低。



### 数据模型

* spark使用的是RDD模型，Spark streaming的DStream实际是RDD的组合
* `Flink`的数据模型就是数据流

### 运行架构

* Spark是批处理，将DAG划分为不同的stage，一个完成之后才可以计算下一个
* `Flink`是标准的流执行模式，一个事件在一个节点处理完成之后可以直接发往下一个节点进行。

### `Flink`的处理流程： 

~~~shell
enviroment -> source -> transformation -> sink
# 在程序中更多的操作就是add一个source，add一个sink。
~~~

## `Flink`-Hello-world

**对`Flink`的单词统计的实现**

### 添加依赖

~~~xml
<dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-scala_2.11</artifactId>
            <version>1.7.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-streaming-scala -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.11</artifactId>
            <version>1.7.2</version>
        </dependency>
    </dependencies>

<build>
    <plugins>
    <!-- 该插件用于将Scala代码编译成class文件 -->
    <plugin>
        <groupId>net.alchim31.maven</groupId>
        <artifactId>scala-maven-plugin</artifactId>
        <version>3.4.6</version>
        <executions>
            <execution>
                <!-- 声明绑定到maven的compile阶段 -->
                <goals>
                    <goal>testCompile</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>3.0.0</version>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>

~~~

### 批处理

添加`scala`包，然后在该包下编程

~~~Scala
package com.isea.helloworld

import org.apache.flink.api.scala.{DataSet, ExecutionEnvironment}

object WordCount {
  def main(args: Array[String]): Unit = {
    // 创建执行环境
    val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

    // 从文件中读取数据
    val inPath = "G:\\scala_learn\\Flink\\Flink-Hello-World\\src\\main\\resources\\hello.txt"

    // 将文本文件转为DataSet
    val inputDS: DataSet[String] = env.readTextFile(inPath)

    // 做word count的操作，这里的方法，groupBy(0)指的是按照二元组的第一个元素进行分组，按照第二个元素进行求和

    import org.apache.flink.api.scala._
    val wordCountDS: AggregateDataSet[(String, Int)] = inputDS.flatMap(_.split(" ")).map((_, 1)).groupBy(0).sum(1)

    // 输出
    wordCountDS.print()
  }
}
~~~

如果是发现

~~~Scala
Could  not find implicit value for evidence parameter of type org.apache.flink.common.typeinfo.TypeInfomation[......]`

这表明的是需要进行隐式转换，需要做的事情就是引入即可
import org.apache.flink.api.scala._
~~~

### 流处理

~~~Scala
package com.isea.helloworld

import org.apache.flink.api.java.utils.ParameterTool
import org.apache.flink.streaming.api.scala.{DataStream, StreamExecutionEnvironment}

object StreamWordCount {
  def main(args: Array[String]): Unit = {
    // 获取配置的相关端口
    val params: ParameterTool = ParameterTool.fromArgs(args)
    val host: String = params.get("host")
    val port: Int = params.getInt("port")
    
    // 创建执行环境
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

    // source ,这里使用socket的方式进行
    val socketDS: DataStream[String] = env.socketTextStream(host,port)

    import org.apache.flink.api.scala._
    // 做word count的操作，这里的方法，groupBy(0)指的是按照二元组的第一个元素进行分组，按照第二个元素进行求和
    val streamDS: DataStream[(String, Int)] = socketDS.flatMap(_.split(" "))
      .filter(_.nonEmpty).map((_,1))
      .keyBy(0)
      .sum(1)

    // 输出
    streamDS.print()

    //streamDS.print().setParallelism(1) // 这里可以指定并行度，有意思的是每一个算子都可以设置并行度
    
    // 启动executor
    env.execute("socket stream...")
  }
}

~~~

此时需要在特定的主机的某个端口开启服务，由该程序来负责监听以完成测试。

## `Flink的部署`

### Standalone模式

注意这里使用的版本是带`hadoop`的，这说明我们的这里可能需要使用到`hdfs`等，如果没有用到（比如只有`redis,kafka`等）如果运行在yarn模式下，那么你 需要下载带Hadoop的版本了。下载的地址：

~~~html
https://flink.apache.org/dow nloads.html
~~~

这里说一下Hadoop Free版本的`Flink`，表示的是该`Flink`并不依赖于Yarn和HDFS

这里我使用的版本是：

~~~shell
/opt/module/flink-1.7.2
~~~

在`/opt/module/flink-1.7.2/conf/flink-conf.yaml`配置文件中可以对`flink`配置，主要配置的信息包括：

`jobmanager`等信息，还有一个配置文件：`/opt/module/flink-1.7.2/conf/slaves`中来配置具体的干活的节点。在配置信息配置好了之后，我们就可以完成启动的操作：

~~~shell
[isea@hadoop101 flink-1.7.2]$ ./bin/start-cluster.sh  #启动命令
Starting cluster.
Starting standalonesession daemon on host hadoop101. # fink运行在哪台机器上
Starting taskexecutor daemon on host hadoop101.
[isea@hadoop101 flink-1.7.2]$   jps
3969 Jps
3865 TaskManagerRunner    # 
3401 StandaloneSessionClusterEntrypoint  # 
~~~

在`flink`启动了之后，我们可以使用web页面的方式来进行管理：`hadoop101:8081`

![1564802990565](img/flk/8.png)

将上述程序打包之后，选择`Entry Class 和 host port参数`上传然后在监听的服host进行如下的操作：

~~~shell
nc -lk 7777  # Linux下开启特定的端口的服务
~~~

或者使用下面的方式运行`flink`程序：

~~~shell
[isea@hadoop101 flink-1.7.2]$ ./bin/flink run -c com.isea.helloworld.StreamWordCount
--host hadoop101 --port 7777

~~~

## `Flink`的原理

### `Flink架构`

![1564817443989](img/flk/9.png)



上图的架构结合**Spark**原理，很清楚，不需要再做解释。

### `Flink任务调度原理`

![1564817467516](img/flk/10.png)



client端只是任务的预处理，并没有在`Flink`整体的运行架构之中。每一个`TaskManager`可以认为是一个JVM进程，每一个`taskManager`下面可以有多个`TaskSolt`去共同处理一个task。

客户端不是运行时和程序执行的一部分，但它用于准备并发送Dataflow `（JobGraph）`给 `Master（JobManager）`，然后，客户端断开连接或者维持连 接以等待接收计算结果。 当 `Flink `集群启动后，首先会启动一个` JobManger `和一个或多个的 `TaskManager`。由 Client 提交任务给 `JobManager，JobManager `再调度任 务到各个 `TaskManager `去执行，然后` TaskManager` 将心跳和统计信息汇报给 `JobManager。TaskManager `之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。 Client 为提交 Job 的客户端，可以运行在任何机器上 (与 `JobManager `环境连通即可)。提交 Job 后，Client 可以结束进程 (Streaming 的任务)，也 可以不结束并等待结果返回。 `JobManager `主要负责调度 Job 并协调 Task 做 `Checkpoint`。从 Client 处接收到 Job 和 JAR 包等资源后，会生成优化后的执行计划，并以 Task 为单元调度到各个 `TaskManager `去执行。`TaskManager `在启动的时候就设置好了槽位数 (Slot)，每个 Slot 能启动一个 Task，Task 为线程。从 `JobManager `处接收需要部署的 Task，部署启动后，与自己的上游建立 `Netty `footnote:[Java 异步 IO 库] 连接，接收数 据并处理。 

:apple:`akka =>netty=>epoll`

### 执行图

`Flink`中的执行图有四层，`StreamGraph（由client完成）->JobGraph（JobManager完成）->ExecutionGraph（TaskManager完成）->物理执行图（并不是一种数据结构，taskManager上部署task之后形成的图）` 

![](img/spk/24.png)

* **`StreamGraph`**：是根据用户通过 Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构
* **`JobGraph`**：`StreamGraph`经过优化后生成了 `JobGraph`，提交给 `JobManager` 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。
* **`ExecutionGraph`**：`JobManager` 根据 `JobGraph`生成`ExecutionGraph`。`ExecutionGraph`是`JobGraph`的并行化版本，是调度层最核心的数据结构。
* 物理执行图：`JobManager `根据 `ExecutionGraph `对 Job 进行调度后， 在各个 `TaskManager `上部署 Task 后形成的” 图”，并不是一个具体的数据结构。

![](img/spk/25.png)

### Worker和Slots

每一个worker(`TaskManager`)都是一个**JVM**进程**，它可能会在**独立的线程上执行一个或多个subtask。为了控制一个worker能接收多少个task，worker通过task slot来进行控制（一个worker至少有一个task slot）。每个task slot表示`TaskManager`拥有资源的**一个固定大小的子集**。假如一个`TaskManager`有三个slot，那么它会将其管理的内存分成三份给各个slot。资源slot化意味着一个subtask将不需要跟来自其他job的subtask竞争被管理的内存，取而代之的是它将拥有一定数量的内存储备。需要注意的是，这里不会涉及到CPU的隔离，slot目前仅仅用来隔离task的受管理的内存。这样就做到了避免了竞争带来的内耗。

通过调整task slot的数量，允许用户定义subtask之间如何互相隔离。如果一个`TaskManager`一个slot，那将意味着每个task group运行在独立的JVM中（该JVM可能是通过一个特定的容器启动的），而一个`TaskManager`多个slot意味着更多的subtask可以共享同一个JVM。而在同一个JVM进程中的task将共享TCP连接（基于多路复用）和心跳消息。它们也可能共享数据集和数据结构，因此这减少了每个task的负载。

![](img/spk/26.png)

**Task Slot**是静态的概念，是指`TaskManager`具有的并发执行能力，可以通过参数`taskmanager.numberOfTaskSlots`进行配置；而并行度`parallelism`是动态概念，即**`TaskManager`**运行程序时实际使用的并发能力，可以通过参数parallelism.default进行配置。

:exclamation:也就是说，假设一共有 3 个 `TaskManager`，每一个 `TaskManager `中的 分配 3 个 Task Slot，也就是每个 `TaskManager `可以接收 3 个 Task，一共 9 个 Task Slot，如果我们设置 parallelism.default=1，即运行程序默认的并行度为 1，9 个 `TaskSlot `只用了 1 个，有 8 个空闲，因此，设置合适的并 行度才能提高效率。这里有三个worker，每个worker有三个槽点，那么表示可以同时开三个进程，9个线程来进行任务的处理，例子

![1564818288980](img/flk/11.png)

![1564818438510](img/flk/12.png)

### 程序和数据流

![](img/spk/27.png)

所有的`Flink `程序都是由三部分组成的：***Source、Transformation 和 Sink。*** Source 负责读取数据源，Transformation 利用各种算子进行处理加工， Sink 负责输出。 在运行时，`Flink `上运行的程序会被映射成 Streaming `Dataflows`，它包含了这三部分。每一个 Dataflow 以一个或多个 sources 开始以一个或多个 sinks 结束。dataflow 类似于任意的有向无环图 (DAG)。在大部分情况下， 程序中的 transformations 跟 dataflow 中的 operator 是一一对应的关系，但 有时候，一个 transformation 可能对应多个 operator。

### 每个算子的并行

**一个operator的的subtask的个数被称作是算子的并行度**，所以每一个算子理论上有其自己并行度。Stream在operator之间的操作有两种形式一个是不会发生`shuffle`，另外一一个是可以发生`shuflle` ，在`Flink`中称前者为**forward（one by one）**，后者称之为redistribution（再分发），所以对于能够合成链条（**chains**）的算子，必须要满足的条件是：**one by one操作 +  并行度相同**。例如：常见的one by one的算子有：`flapmap，map`。

### 并行数据流

`Flink `程序的执行具有并行、分布式的特性。在执行过程中，一个 stream 包含一个或多个 stream partition，而每一个 operator 包含一个或多个 operator subtask，这些 operator subtasks 在不同的线程、不同的物理机或不同的容器中彼此互不依赖的执行。 一个特定 operator 的 subtask 的个数被称之为其 parallelism(并行度)。 一个 stream 的并行度总是等同于其 producing operator的并行度。一个程序中，不同的operator可能具有不同的并行度。

![](img/spk/28.png)

Stream在operator之间传输数据的形式可以是one-to-one(forwarding) 的模式也可以是redistributing 的模式，具体是哪一种形式，取决于operator的种类。 

:one:One-to-One：stream(比如在 source 和 map operator 之间) 维护着分 区以及元素的顺序。那意味着 map operator 的subtask看到的元素的个数以及顺序跟`source operator `的 subtask 生产的元素的个数、顺序相同，`map、 fliter、flatMap `等算子都是 one-to-one 的对应关系。类似于 spark 中的窄依赖。

:two: Redistributing：stream（map() 跟 `keyBy/window `之间或者 `keyBy/window `跟 sink 之间）的分区会发生改变。每一个operator subtask 依据所选择的 transformation 发送数据到不同的目标 subtask。例如，`keyBy()`基于 `hashCode `重分区、broadcast 和 rebalance 会随机重新分区，这些算子 都会引起 redistributing 过程，而 redistributing 过程就类似于 Spark 中的 shuffle 过程。类似于 spark 中的宽依赖。

### task和操作链

相同并行度的 one to one 操作，`Flink `这样相连的 operator 链接在一 起形成一个 task，原来的 operator 成为里面的 subtask。将 operators 链接成 task 是非常有效的优化：它能减少线程之间的切换和基于缓存区的数据交换，在减少时延的同时提升吞吐量。链接的行为可以在编程 API中进行指定。

![](img/spk/29.png)

### 系统架构

`Flink` 是一个用于有状态的并行数据流处理的分布式系统。它由多个进程构成，这些进程一般会分布运行在不同的机器上。对于分布式系统来说， 面对的常见问题有：集群中资源的分配和管理、进程协调调度、持久化和高可用的数据存储，以及故障恢复。 对于这些分布式系统的经典问题，业内已有比较成熟的解决方案和服务。所以 `Flink`并不会自己去处理所有的问题，而是利用了现有的集群架构和服务，这样它就可以把精力集中在核心工作——分布式数据流处理上了。`Flink` 与一些集群资源管理工具有很好的集成，比如 `Apache Mesos、YARN `和 `Kubernetes`；同时，也可以配置为独立（stand-alone）集群运行。`Flink` 自己并不提供持久化的分布式存储，而是直接利用了已有的分布式文件系 统（比如 HDFS）或者对象存储（比如 S3）。对于高可用的配置，`Flink` 需要依靠 `Apache ZooKeeper `来完成。 在本节中，我们将介绍 `Flink `的不同组件，以及在运行程序时它们如何 相互作用。我们会讨论部署 `Flink `应用程序的两种模式，并且了解每种模式 下分发和执行任务的方式。最后，我们还会解释一下 `Flink` 的高可用性模式 是如何工作的。

#### `flink运行时组件`

`Flink `运行时架构主要包括四个不同的组件，它们会在运行流处理应用 程序时协同工作：作业管理器（`JobManager`）、资源管理器（`ResourceManager`）、任务管理器（`TaskManager`），以及分发器（`Dispatcher`）。因为 `Flink `是用 Java 和 Scala 实现的，所以所有组件都会运行在 Java 虚拟机（JVMs） 上。每个组件的职责如下：

:one: 作业管理器（`JobManager`）是控制一个应用程序执行的主进程，也就是说，每个应用程序都会被一个不同的 `JobManager `所控制执行。 `JobManager `会先接收到要执行的应用程序。这个应用程序会包括：作业图（`JobGraph`）、逻辑数据流图（`logical dataflow graph`）和打包了所有的类、库和其它资源的 JAR包。`JobManager `会把 `JobGraph `转换成一个物理层面的数据流图，这个图被叫做“执行图”（`ExecutionGraph`），包含了所有可以并发执行的任务。`JobManager `会向资源管 理器（`ResourceManager`）请求执行任务必要的资源，也就是任务管 理器（`TaskManager`）上的插槽（slot）。一旦它获取到了足够的资源， 就会将执行图分发到真正运行它们的` TaskManager `上。而在运行过程中，`JobManager `会负责所有需要中央协调的操作，比如说检查点 （checkpoints）的协调。 • 

:two: `ResourceManager `主要负责管理任务管理器（`TaskManager`）的插槽 （slot），`TaskManger `插槽是` Flink` 中定义的处理资源单元。`Flink` 为不同的环境和资源管理工具提供了不同资源管理器（`ResourceManager`），比如 YARN、`Mesos`、K8s，以及 standalone 部署。当 `JobManager `申请插槽资源时，`ResourceManager` 会将有空闲插槽的`TaskManager `分配给` JobManager`。如果 `ResourceManager `没有足够的插槽来满足 `JobManager `的请求，它还可以向资源提供平台发起会话，以提供启动 `TaskManager `进程的容器。另外，`ResourceManager `还负责终止空闲的`TaskManager`，释放计算资源。 

:three: 任务管理器（`TaskManager`）是`Flink `中的工作进程。通常在 `Flink `中会有多个`TaskManager `运行，每一个 `TaskManager `都包含了一定数量的插槽（slots）。插槽的数量限制了` TaskManager `能够执行的任务数量。启动之后，`TaskManager `会向资源管理器注册它的插槽；收到资源管理器的指令后，`TaskManager `就会将一个或者多个插槽提供给` JobManager `调用。`JobManager `就可以向插槽分配任务（tasks）来执 行了。在执行过程中，一个 `TaskManager `可以跟其它运行同一应用程 序的 `TaskManager `交换数据。任务的执行和插槽的概念会在“任务执行”一节做具体讨论。 

:four: 分发器（Dispatcher）可以跨作业运行，它为应用提交提供了 REST 接 口。当一个应用被提交执行时，分发器就会启动并将应用移交给一个 `JobManager。`由于是 REST 接口，所以 Dispatcher 可以作为集群的 一个 HTTP 接入点，这样就能够不受防火墙阻挡。Dispatcher 也会启动一个Web UI，用来方便地展示和监控作业执行的信息。Dispatcher 在架构中可能并不是必需的，这取决于应用提交运行的方式。

![](img/spk/30.png)

## `Flink`的流处理API

### 环境的获取

~~~scala
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
// getExecutionEnvironment 会自动获取执行的环境，不管是standalone的方式还是集群的方式去运行flink。也就是说flink能够自己判断执行的环境，然后返回当前的环境

// 如果是local的方式，并行度默认是电脑的cpu的核数，如果是集群的方式的话，读取yaml配置文件中的并行度。
~~~

### source

数据的来源可以有多种，常见的我们可以从集合中读取数据，可以从文件中读取数据，从`kafka`等消息队列中读取数据，还可以自定义消息来源。其中从集合中，从文件中读取数据相当于使用`flink`做批处理操作，更常用的应用场景是：`Flink`读取`Kafka`中的数据。这里演示一个从`Kafka`中获取数据：

**添加依赖**

~~~xml
<!-- https://mvnrepository.com/artifact/org.apache.flink/flink-connector-kafka-0.11 -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.11_2.11</artifactId>
    <version>1.7.0</version>
</dependency>
~~~

**代码**

~~~Scala
package com.isea.helloworld

import java.util.Properties

import org.apache.flink.api.common.serialization.SimpleStringSchema
import org.apache.flink.streaming.api.scala.{DataStream, StreamExecutionEnvironment}
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer011
import org.apache.kafka.clients.consumer.ConsumerConfig

object KafkaFlink {
  def main(args: Array[String]): Unit = {
    val environment: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

    // kafka做为配置源：
    val properties = new Properties()
    properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,"hadoop101:9092")
    import org.apache.kafka.clients.consumer.ConsumerConfig
    properties.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    properties.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer")
    properties.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,"latest")

    var topic : String = "sensor"

    import org.apache.flink.api.scala._
    val streamDS: DataStream[String] = environment.addSource(new FlinkKafkaConsumer011[String](topic,new SimpleStringSchema(),properties))

    streamDS.print("flink-kafka:").setParallelism(1);
    environment.execute("API-Flink-Kafka")
  }
}
~~~

上述的代码完成的逻辑就是在`kafka`开启了之后，然后创建生产者，上述的代码能够消费生产者生产的数据做到**实时打印**。

* UDP：at most once ，最多一次，广播出去就算了
* at least once ，最少一次，可能存在多次
* exactly-once 恰好一次

流：迭代器就是流，一个指针，一个一个的去遍历流，列表是将所有的数都放置到内存中。

逻辑执行图->物理执行图