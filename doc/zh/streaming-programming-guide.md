---
layout: global
displayTitle: Spark Streaming 编程指南
title: Spark Streaming
description: 针对 Spark SPARK_VERSION_SHORT 的 Spark Streaming 编程指南和教程
---

* This will become a table of contents (this text will be scraped).
{:toc}

# 概述
Spark Streaming 是 Spark Core API 的扩展, 它支持弹性的, 高吞吐的, 容错的实时数据流的处理.
数据可以通过多种数据源获取, 例如 Kafka, Flume, Kinesis 以及 TCP sockets, 也可以通过例如 `map`, `reduce`, `join`, `window` 等的高级函数组成的复杂算法处理.
最终, 处理后的数据可以输出到文件系统, 数据库以及实时仪表盘中.
事实上, 你还可以在 data streams（数据流）上使用 [机器学习](ml-guide.html) 以及 [图形处理](graphx-programming-guide.html) 算法.

<p style="text-align: center;">
  <img
    src="img/streaming-arch.png"
    title="Spark Streaming architecture"
    alt="Spark Streaming"
    width="70%"
  />
</p>

在内部, 它工作原理如下, Spark Streaming 接收实时输入数据流并将数据切分成多个 batch（批）数据, 然后由 Spark 引擎处理它们以生成最终的 stream of results in batches（分批流结果）.

<p style="text-align: center;">
  <img src="img/streaming-flow.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="70%" />
</p>

Spark Streaming 提供了一个名为 *discretized stream* 或 *DStream* 的高级抽象, 它代表一个连续的数据流.
DStream 可以从数据源的输入数据流创建, 例如 Kafka, Flume 以及 Kinesis, 或者在其他 DStream 上进行高层次的操作以创建.
在内部, 一个 DStream 是通过一系列的 [RDDs](api/scala/index.html#org.apache.spark.rdd.RDD) 来表示.

本指南告诉你如何使用 DStream 来编写一个 Spark Streaming 程序.
你可以使用 Scala , Java 或者 Python（Spark 1.2 版本后引进）来编写 Spark Streaming 程序.
所有这些都在本指南中介绍.
您可以在本指南中找到标签, 让您可以选择不同语言的代码段.

**Note（注意）:** 在 Python 有些 API 可能会有不同或不可用. 在本指南, 您将找到 <span class="badge" style="background-color: grey">Python API</span> 的标签来高亮显示不同的地方.

***************************************************************************************************

# 一个入门示例
在我们详细介绍如何编写你自己的 Spark Streaming 程序的细节之前, 让我们先来看一看一个简单的 Spark Streaming 程序的样子.
比方说, 我们想要计算从一个监听 TCP socket 的数据服务器接收到的文本数据（text data）中的字数.
你需要做的就是照着下面的步骤做.

<div class="codetabs">
<div data-lang="scala"  markdown="1" >
首先, 我们导入了 Spark Streaming 类和部分从 StreamingContext 隐式转换到我们的环境的名称, 目的是添加有用的方法到我们需要的其他类（如 DStream）.
[StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) 是所有流功能的主要入口点.
我们创建了一个带有 2 个执行线程和间歇时间为 1 秒的本地 StreamingContext.

{% highlight scala %}
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // 自从 Spark 1.3 开始, 不再是必要的了   

// 创建一个具有两个工作线程（working thread）并且批次间隔为 1 秒的本地 StreamingContext .
// master 需要 2 个核, 以防止饥饿情况（starvation scenario）.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
{% endhighlight %}

Using this context, we can create a DStream that represents streaming data from a TCP
source, specified as hostname (e.g. `localhost`) and port (e.g. `9999`).
使用该 context, 我们可以创建一个代表从 TCP 源流数据的离散流（DStream）, 指定主机名（hostname）（例如 localhost）和端口（例如 9999）.

{% highlight scala %}
// 创建一个将要连接到 hostname:port 的 DStream，如 localhost:9999 
val lines = ssc.socketTextStream("localhost", 9999)
{% endhighlight %}

上一步的这个 `lines` DStream 表示将要从数据服务器接收到的数据流.
在这个离散流（DStream）中的每一条记录都是一行文本（text）.
接下来，我们想要通过空格字符（space characters）拆分这些数据行（lines）成单词（words）.

{% highlight scala %}
// 将每一行拆分成 words（单词）
val words = lines.flatMap(_.split(" "))
{% endhighlight %}

`flatMap` 是一种 one-to-many（一对多）的离散流（DStream）操作，它会通过在源离散流（source DStream）中根据每个记录（record）生成多个新纪录的形式创建一个新的离散流（DStream）.
在这种情况下，在这种情况下，每一行（each line）都将被拆分成多个单词（`words`）和代表单词离散流（words DStream）的单词流.
接下来，我们想要计算这些单词.

{% highlight scala %}
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
// 计算每一个 batch（批次）中的每一个 word（单词）
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// 在控制台打印出在这个离散流（DStream）中生成的每个 RDD 的前十个元素
// 注意: 必需要触发 action（很多初学者会忘记触发 action 操作，导致报错：No output operations registered, so nothing to execute） 
wordCounts.print()
{% endhighlight %}

上一步的 `words` DStream 进行了进一步的映射（一对一的转换）为一个 (word, 1) paris 的离散流（DStream），这个 DStream 然后被规约（reduce）来获得数据中每个批次（batch）的单词频率.
最后，`wordCounts.print()` 将会打印一些每秒生成的计数.

Note that when these lines are executed, Spark Streaming only sets up the computation it
will perform when it is started, and no real processing has started yet. To start the processing
after all the transformations have been setup, we finally call

请注意，当这些行（lines）被执行的时候， Spark Streaming 仅仅设置了计算, 只有在启动时才会执行，并没有开始真正地处理.
为了在所有的转换都已经设置好之后开始处理，我们在最后调用:

{% highlight scala %}
ssc.start()             // 开始计算
ssc.awaitTermination()  // 等待计算被中断
{% endhighlight %}

该部分完整的代码可以在 Spark Streaming 示例
[NetworkWordCount]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/NetworkWordCount.scala) 中找到.
<br>

</div>
<div data-lang="java" markdown="1">

First, we create a
[JavaStreamingContext](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html) object,
which is the main entry point for all streaming
functionality. We create a local StreamingContext with two execution threads, and a batch interval of 1 second.

{% highlight java %}
import org.apache.spark.*;
import org.apache.spark.api.java.function.*;
import org.apache.spark.streaming.*;
import org.apache.spark.streaming.api.java.*;
import scala.Tuple2;

// Create a local StreamingContext with two working thread and batch interval of 1 second
SparkConf conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount");
JavaStreamingContext jssc = new JavaStreamingContext(conf, Durations.seconds(1));
{% endhighlight %}

Using this context, we can create a DStream that represents streaming data from a TCP
source, specified as hostname (e.g. `localhost`) and port (e.g. `9999`).

{% highlight java %}
// Create a DStream that will connect to hostname:port, like localhost:9999
JavaReceiverInputDStream<String> lines = jssc.socketTextStream("localhost", 9999);
{% endhighlight %}

This `lines` DStream represents the stream of data that will be received from the data
server. Each record in this stream is a line of text. Then, we want to split the lines by
space into words.

{% highlight java %}
// Split each line into words
JavaDStream<String> words = lines.flatMap(x -> Arrays.asList(x.split(" ")).iterator());
{% endhighlight %}

`flatMap` is a DStream operation that creates a new DStream by
generating multiple new records from each record in the source DStream. In this case,
each line will be split into multiple words and the stream of words is represented as the
`words` DStream. Note that we defined the transformation using a
[FlatMapFunction](api/scala/index.html#org.apache.spark.api.java.function.FlatMapFunction) object.
As we will discover along the way, there are a number of such convenience classes in the Java API
that help define DStream transformations.

Next, we want to count these words.

{% highlight java %}
// Count each word in each batch
JavaPairDStream<String, Integer> pairs = words.mapToPair(s -> new Tuple2<>(s, 1));
JavaPairDStream<String, Integer> wordCounts = pairs.reduceByKey((i1, i2) -> i1 + i2);

// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print();
{% endhighlight %}

The `words` DStream is further mapped (one-to-one transformation) to a DStream of `(word,
1)` pairs, using a [PairFunction](api/scala/index.html#org.apache.spark.api.java.function.PairFunction)
object. Then, it is reduced to get the frequency of words in each batch of data,
using a [Function2](api/scala/index.html#org.apache.spark.api.java.function.Function2) object.
Finally, `wordCounts.print()` will print a few of the counts generated every second.

Note that when these lines are executed, Spark Streaming only sets up the computation it
will perform after it is started, and no real processing has started yet. To start the processing
after all the transformations have been setup, we finally call `start` method.

{% highlight java %}
jssc.start();              // Start the computation
jssc.awaitTermination();   // Wait for the computation to terminate
{% endhighlight %}

The complete code can be found in the Spark Streaming example
[JavaNetworkWordCount]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming/JavaNetworkWordCount.java).
<br>

</div>
<div data-lang="python"  markdown="1" >
First, we import [StreamingContext](api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext), which is the main entry point for all streaming functionality. We create a local StreamingContext with two execution threads, and batch interval of 1 second.

{% highlight python %}
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

# Create a local StreamingContext with two working thread and batch interval of 1 second
sc = SparkContext("local[2]", "NetworkWordCount")
ssc = StreamingContext(sc, 1)
{% endhighlight %}

Using this context, we can create a DStream that represents streaming data from a TCP
source, specified as hostname (e.g. `localhost`) and port (e.g. `9999`).

{% highlight python %}
# Create a DStream that will connect to hostname:port, like localhost:9999
lines = ssc.socketTextStream("localhost", 9999)
{% endhighlight %}

This `lines` DStream represents the stream of data that will be received from the data
server. Each record in this DStream is a line of text. Next, we want to split the lines by
space into words.

{% highlight python %}
# Split each line into words
words = lines.flatMap(lambda line: line.split(" "))
{% endhighlight %}

`flatMap` is a one-to-many DStream operation that creates a new DStream by
generating multiple new records from each record in the source DStream. In this case,
each line will be split into multiple words and the stream of words is represented as the
`words` DStream.  Next, we want to count these words.

{% highlight python %}
# Count each word in each batch
pairs = words.map(lambda word: (word, 1))
wordCounts = pairs.reduceByKey(lambda x, y: x + y)

# Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.pprint()
{% endhighlight %}

The `words` DStream is further mapped (one-to-one transformation) to a DStream of `(word,
1)` pairs, which is then reduced to get the frequency of words in each batch of data.
Finally, `wordCounts.pprint()` will print a few of the counts generated every second.

Note that when these lines are executed, Spark Streaming only sets up the computation it
will perform when it is started, and no real processing has started yet. To start the processing
after all the transformations have been setup, we finally call

{% highlight python %}
ssc.start()             # Start the computation
ssc.awaitTermination()  # Wait for the computation to terminate
{% endhighlight %}

The complete code can be found in the Spark Streaming example
[NetworkWordCount]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/network_wordcount.py).
<br>

</div>
</div>

如果你已经 [下载](index.html#downloading) 并且 [构建](index.html#building) Spark, 您可以使用如下方式来运行该示例.
你首先需要运行 Netcat（一个在大多数类 Unix 系统中的小工具）作为我们使用的数据服务器.

{% highlight bash %}
$ nc -lk 9999
{% endhighlight %}

然后，在另一个不同的终端，你可以通过执行如下命令来运行该示例:

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight bash %}
$ ./bin/run-example streaming.NetworkWordCount localhost 9999
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight bash %}
$ ./bin/run-example streaming.JavaNetworkWordCount localhost 9999
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight bash %}
$ ./bin/spark-submit examples/src/main/python/streaming/network_wordcount.py localhost 9999
{% endhighlight %}
</div>
</div>


然后，在运行在 netcat 服务器上的终端输入的任何行（lines），都将被计算，并且每一秒都显示在屏幕上，它看起来就像下面这样:

<table width="100%">
    <td>
{% highlight bash %}
# TERMINAL 1:
# Running Netcat

$ nc -lk 9999

hello world



...
{% endhighlight %}
    </td>
    <td width="2%"></td>
    <td>
<div class="codetabs">

<div data-lang="scala" markdown="1">
{% highlight bash %}
# TERMINAL 2: RUNNING NetworkWordCount

$ ./bin/run-example streaming.NetworkWordCount localhost 9999
...
-------------------------------------------
Time: 1357008430000 ms
-------------------------------------------
(hello,1)
(world,1)
...
{% endhighlight %}
</div>

<div data-lang="java" markdown="1">
{% highlight bash %}
# TERMINAL 2: RUNNING JavaNetworkWordCount

$ ./bin/run-example streaming.JavaNetworkWordCount localhost 9999
...
-------------------------------------------
Time: 1357008430000 ms
-------------------------------------------
(hello,1)
(world,1)
...
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight bash %}
# TERMINAL 2: RUNNING network_wordcount.py

$ ./bin/spark-submit examples/src/main/python/streaming/network_wordcount.py localhost 9999
...
-------------------------------------------
Time: 2014-10-14 15:25:21
-------------------------------------------
(hello,1)
(world,1)
...
{% endhighlight %}
</div>
</div>
    </td>
</table>


***************************************************************************************************
***************************************************************************************************

# 基础概念

接下来，我们了解完了简单的例子，开始阐述 Spark Streaming 的基本知识。

## 依赖

与 Spark 类似，Spark Streaming 可以通过 Maven 来管理依赖.
为了编写你自己的 Spark Streaming 程序，你必须添加以下的依赖到你的 SBT 或者 Maven 项目中.

<div class="codetabs">
<div data-lang="Maven" markdown="1">

	<dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_{{site.SCALA_BINARY_VERSION}}</artifactId>
        <version>{{site.SPARK_VERSION}}</version>
    </dependency>
</div>
<div data-lang="SBT" markdown="1">

	libraryDependencies += "org.apache.spark" % "spark-streaming_{{site.SCALA_BINARY_VERSION}}" % "{{site.SPARK_VERSION}}"
</div>
</div>

针对从 Spark Streaming Core API 中不存在的数据源中获取数据，如 Kafka， Flume，Kinesis ，你必须添加相应的坐标  `spark-streaming-xyz_{{site.SCALA_BINARY_VERSION}}` 到依赖中.
例如，有一些常见的依赖如下.


<table class="table">
<tr><th>Source（数据源）</th><th>Artifact（坐标）</th></tr>
<tr><td> Kafka </td><td> spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}} </td></tr>
<tr><td> Flume </td><td> spark-streaming-flume_{{site.SCALA_BINARY_VERSION}} </td></tr>
<tr><td> Kinesis<br/></td><td>spark-streaming-kinesis-asl_{{site.SCALA_BINARY_VERSION}} [Amazon Software License] </td></tr>
<tr><td></td><td></td></tr>
</table>

想要查看一个实时更新的列表，请参阅 [Maven repository](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.apache.spark%22%20AND%20v%3A%22{{site.SPARK_VERSION_SHORT}}%22) 来了解支持的 sources（数据源）和 artifacts（坐标）的完整列表。

***

## 初始化 StreamingContext

为了初始化一个 Spark Streaming 程序, 一个 **StreamingContext** 对象必须要被创建出来，它是所有的 Spark Streaming 功能的主入口点。

<div class="codetabs">
<div data-lang="scala" markdown="1">

一个 [StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) 对象可以从一个 [SparkConf](api/scala/index.html#org.apache.spark.SparkConf) 对象中来创建.

{% highlight scala %}
import org.apache.spark._
import org.apache.spark.streaming._

val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))
{% endhighlight %}

这个 `appName` 参数是展示在集群 UI 界面上的应用程序的名称.
`master` 是一个 [Spark, Mesos or YARN cluster URL](submitting-applications.html#master-urls),
或者一个特殊的 __"local[\*]"__ 字符串以使用 local mode（本地模式）来运行.
在实践中，当在集群上运行时，你不会想在应用程序中硬编码 `master`，而是 [使用 `spark-submit` 来启动应用程序](submitting-applications.html) , 并且接受该参数.
然而，对于本地测试和单元测试，你可以传递 "local[*]" 来运行 Spark Streaming 进程（检测本地系统中内核的个数）.
请注意，做个内部创建了一个 [SparkContext](api/scala/index.html#org.apache.spark.SparkContext)（所有 Spark 功能的出发点），它可以像 ssc.sparkContext 这样被访问.

这个 batch interval（批间隔）必须根据您的应用程序和可用的集群资源的等待时间要求进行设置.
更多详情请参阅 [优化指南](#setting-the-right-batch-interval) 部分.

一个 `StreamingContext` 对象也可以从一个现有的 `SparkContext` 对象来创建.

{% highlight scala %}
import org.apache.spark.streaming._

val sc = ...                // 已存在的 SparkContext
val ssc = new StreamingContext(sc, Seconds(1))
{% endhighlight %}


</div>
<div data-lang="java" markdown="1">

A [JavaStreamingContext](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html) object can be created from a [SparkConf](api/java/index.html?org/apache/spark/SparkConf.html) object.

{% highlight java %}
import org.apache.spark.*;
import org.apache.spark.streaming.api.java.*;

SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
JavaStreamingContext ssc = new JavaStreamingContext(conf, new Duration(1000));
{% endhighlight %}

The `appName` parameter is a name for your application to show on the cluster UI.
`master` is a [Spark, Mesos or YARN cluster URL](submitting-applications.html#master-urls),
or a special __"local[\*]"__ string to run in local mode. In practice, when running on a cluster,
you will not want to hardcode `master` in the program,
but rather [launch the application with `spark-submit`](submitting-applications.html) and
receive it there. However, for local testing and unit tests, you can pass "local[*]" to run Spark Streaming
in-process. Note that this internally creates a [JavaSparkContext](api/java/index.html?org/apache/spark/api/java/JavaSparkContext.html) (starting point of all Spark functionality) which can be accessed as `ssc.sparkContext`.

The batch interval must be set based on the latency requirements of your application
and available cluster resources. See the [Performance Tuning](#setting-the-right-batch-interval)
section for more details.

A `JavaStreamingContext` object can also be created from an existing `JavaSparkContext`.

{% highlight java %}
import org.apache.spark.streaming.api.java.*;

JavaSparkContext sc = ...   //existing JavaSparkContext
JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(1));
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">

A [StreamingContext](api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext) object can be created from a [SparkContext](api/python/pyspark.html#pyspark.SparkContext) object.

{% highlight python %}
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

sc = SparkContext(master, appName)
ssc = StreamingContext(sc, 1)
{% endhighlight %}

The `appName` parameter is a name for your application to show on the cluster UI.
`master` is a [Spark, Mesos or YARN cluster URL](submitting-applications.html#master-urls),
or a special __"local[\*]"__ string to run in local mode. In practice, when running on a cluster,
you will not want to hardcode `master` in the program,
but rather [launch the application with `spark-submit`](submitting-applications.html) and
receive it there. However, for local testing and unit tests, you can pass "local[\*]" to run Spark Streaming
in-process (detects the number of cores in the local system).

The batch interval must be set based on the latency requirements of your application
and available cluster resources. See the [Performance Tuning](#setting-the-right-batch-interval)
section for more details.
</div>
</div>

在定义一个 context 之后,您必须执行以下操作.

1. 通过创建输入 DStreams 来定义输入源.
1. 通过应用转换和输出操作 DStreams 定义流计算（streaming computations）.
1. 开始接收输入并且使用 `streamingContext.start()` 来处理数据.
1. 使用 `streamingContext.awaitTermination()` 等待处理被终止（手动或者由于任何错误）.
1. 使用 `streamingContext.stop()` 来手动的停止处理.

##### 需要记住的几点:
{:.no_toc}
- 一旦一个 context 已经启动，将不会有新的数据流的计算可以被创建或者添加到它。.
- 一旦一个 context 已经停止，它不会被重新启动.
- 同一时间内在 JVM 中只有一个 StreamingContext 可以被激活.
- 在 StreamingContext 上的 stop() 同样也停止了 SparkContext 。为了只停止 StreamingContext ，设置 `stop()` 的可选参数，名叫 `stopSparkContext` 为 false.
- 一个 SparkContext 就可以被重用以创建多个 StreamingContexts，只要前一个 StreamingContext 在下一个StreamingContext 被创建之前停止（不停止 SparkContext）.

***

## Discretized Streams (DStreams)（离散化流）
**Discretized Stream** or **DStream** 是 Spark Streaming 提供的基本抽象.
它代表了一个连续的数据流, 无论是从 source（数据源）接收到的输入数据流,
还是通过转换输入流所产生的处理过的数据流.
在内部, 一个 DStream 被表示为一系列连续的 RDDs, 它是 Spark 中一个不可改变的抽象,
distributed dataset  (的更多细节请看 [Spark 编程指南](programming-guide.html#resilient-distributed-datasets-rdds).
在一个 DStream 中的每个 RDD 包含来自一定的时间间隔的数据，如下图所示.

<p style="text-align: center;">
  <img src="img/streaming-dstream.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="70%" />
</p>

应用于 DStream 的任何操作转化为对于底层的 RDDs 的操作.
例如，在 [先前的示例](#一个入门示例)，转换一个行（lines）流成为单词（words）中，flatMap 操作被应用于在行离散流（lines DStream）中的每个 RDD 来生成单词离散流（words DStream）的 RDDs .
如下所示.

<p style="text-align: center;">
  <img src="img/streaming-dstream-ops.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="70%" />
</p>

这些底层的 RDD 变换由 Spark 引擎（engine）计算。 DStream 操作隐藏了大多数这些细节并为了方便起见，提供给了开发者一个更高级别的 API 。这些操作细节会在后边的章节中讨论。

***

## Input DStreams 和 Receivers（接收器）
输入 DStreams 是代表输入数据是从流的源数据（streaming sources）接收到的流的 DStream.
在 [一个入门示例](#一个入门示例) 中, `lines` 是一个 input DStream, 因为它代表着从 netcat 服务器接收到的数据的流.
每一个 input DStream（除了 file stream 之外, 会在本章的后面来讨论）与一个 **Receiver** ([Scala doc](api/scala/index.html#org.apache.spark.streaming.receiver.Receiver),
[Java doc](api/java/org/apache/spark/streaming/receiver/Receiver.html)) 对象关联, 它从 source（数据源）中获取数据，并且存储它到 Sparl 的内存中用于处理.

Spark Streaming 提供了两种内置的 streaming source（流的数据源）.

- *Basic sources（基础的数据源）*: 在 StreamingContext API 中直接可以使用的数据源.
  例如: file systems 和 socket connections.
- *Advanced sources（高级的数据源）*: 像 Kafka, Flume, Kinesis, 等等这样的数据源. 可以通过额外的 utility classes 来使用. 像在 [依赖](#依赖) 中讨论的一样, 这些都需要额外的外部依赖.

在本节的后边，我们将讨论每种类别中的现有的一些数据源.

请注意, 如果你想要在你的流处理程序中并行的接收多个数据流, 你可以创建多个 input DStreams（在 [性能优化](#level-of-parallelism-in-data-receiving) 部分进一步讨论）.
这将创建同时接收多个数据流的多个 receivers（接收器）.
但需要注意，一个 Spark 的 worker/executor 是一个长期运行的任务（task），因此它将占用分配给 Spark Streaming 的应用程序的所有核中的一个核（core）.
因此，要记住，一个 Spark Streaming 应用需要分配足够的核（core）（或线程（threads），如果本地运行的话）来处理所接收的数据，以及来运行接收器（receiver(s)）.

##### 要记住的几点
{:.no_toc}

- 当在本地运行一个 Spark Streaming 程序的时候，不要使用 "local" 或者 "local[1]" 作为 master 的 URL.
  这两种方法中的任何一个都意味着只有一个线程将用于运行本地任务.
  如果你正在使用一个基于接收器（receiver）的输入离散流（input DStream）（例如， sockets ，Kafka ，Flume 等），则该单独的线程将用于运行接收器（receiver），而没有留下任何的线程用于处理接收到的数据.
  因此，在本地运行时，总是用 "local[n]" 作为 master URL ，其中的 n > 运行接收器的数量（查看 [Spark 属性](configuration.html#spark-properties) 来了解怎样去设置 master 的信息）.

- 将逻辑扩展到集群上去运行，分配给 Spark Streaming 应用程序的内核（core）的内核数必须大于接收器（receiver）的数量。否则系统将接收数据，但是无法处理它.

### 基础的 Sources（数据源）
{:.no_toc}

我们已经简单地了解过了在 [入门示例](#一个入门示例) 中 `ssc.socketTextStream(...)` 的例子，例子中是通过从一个 TCP socket 连接接收到的文本数据来创建了一个离散流（DStream）.
除了 sockets 之外，StreamingContext API 也提供了根据文件作为输入来源创建离散流（DStreams）的方法。

- **File Streams:** 用于从文件中读取数据，在任何与 HDFS API 兼容的文件系统中（即，HDFS，S3，NFS 等），一个 DStream 可以像下面这样创建:

    <div class="codetabs">
    <div data-lang="scala" markdown="1">
        streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
    </div>
    <div data-lang="java" markdown="1">
		streamingContext.fileStream<KeyClass, ValueClass, InputFormatClass>(dataDirectory);
    </div>
    <div data-lang="python" markdown="1">
		streamingContext.textFileStream(dataDirectory)
    </div>
    </div>

	Spark Streaming 将监控`dataDirectory` 目录并且该目录中任何新建的文件 (写在嵌套目录中的文件是不支持的). 注意

     + 文件必须具有相同的数据格式.
     + 文件必须被创建在 `dataDirectory` 目录中, 通过 atomically（院子的） *moving（移动）* 或 *renaming（重命名）* 它们到数据目录.
     + 一旦移动，这些文件必须不能再更改，因此如果文件被连续地追加，新的数据将不会被读取.

  对于简单的文本文件，还有一个更加简单的方法 `streamingContext.textFileStream(dataDirectory)`.
  并且文件流（file streams）不需要运行一个接收器（receiver），因此，不需要分配内核（core）。

	<span class="badge" style="background-color: grey">Python API</span> 在 Python API 中 `fileStream` 是不可用的, 只有	`textFileStream` 是可用的.

- **Streams based on Custom Receivers（基于自定义的接收器的流）:** DStreams 可以使用通过自定义的 receiver（接收器）接收到的数据来创建. 更多细节请参阅 [自定义 Receiver
  指南](streaming-custom-receivers.html).

- **Queue of RDDs as a Stream（RDDs 队列作为一个流）:** 为了使用测试数据测试 Spark Streaming 应用程序，还可以使用 `streamingContext.queueStream(queueOfRDDs)` 创建一个基于 RDDs 队列的 DStream，每个进入队列的 RDD 都将被视为 DStream 中的一个批次数据，并且就像一个流进行处理.

想要了解更多的关于从 sockets 和文件（files）创建的流的细节, 请参阅相关函数的 API文档, 它们在
[StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) for
Scala, [JavaStreamingContext](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html)
for Java 以及 [StreamingContext](api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext) for Python 中.

### 高级 Sources（数据源）
{:.no_toc}

<span class="badge" style="background-color: grey">Python API</span> 从 Spark {{site.SPARK_VERSION_SHORT}} 开始,
在 Python API 中的 Kafka, Kinesis 和 Flume 这样的外部数据源都是可用的.

这一类别的 sources（数据源）需要使用非 Spark 库中的外部接口，它们中的其中一些还需要比较复杂的依赖关系（例如， Kafka 和 Flume）.
因此，为了最小化有关的依赖关系的版本冲突的问题，这些资源本身不能创建 DStream 的功能，它是通过 [依赖](#依赖) 单独的类库实现创建 DStream 的功能.

请注意, 这些高级 sources（数据源）不能再 Spark shell 中使用, 因此，基于这些高级 sources（数据源）的应用程序不能在 shell 中被测试.
如果你真的想要在 Spark shell 中使用它们，你必须下载带有它的依赖的相应的 Maven 组件的 JAR ，并且将其添加到 classpath.

一些高级的 sources（数据源）如下.

- **Kafka:** Spark Streaming {{site.SPARK_VERSION_SHORT}} 与 Kafka broker 版本 0.8.2.1 或更高是兼容的. 更多细节请参阅 [Kafka 集成指南](streaming-kafka-integration.html).

- **Flume:** Spark Streaming {{site.SPARK_VERSION_SHORT}} 与 Flume 1.6.0 相兼容. 更多细节请参阅 [Flume 集成指南](streaming-flume-integration.html) .

- **Kinesis:** Spark Streaming {{site.SPARK_VERSION_SHORT}} 与 Kinesis Client Library 1.2.1 相兼容. 更多细节请参阅 [Kinesis 集成指南](streaming-kinesis-integration.html).

### 自定义 Sources（数据源）
{:.no_toc}

<span class="badge" style="background-color: grey">Python API</span> 在 Python 中还不支持这一功能.

Input DStreams 也可以从自定义数据源中创建.
如果您想这样做, 需要实现一个用户自定义的 **receiver** （看下一节以了解它是什么）, 它可以从自定义的 sources（数据源）中接收数据并且推送它到 Spark.
更多细节请参阅 [自定义 Receiver 指南](streaming-custom-receivers.html).

### Receiver Reliability（接收器的可靠性）
{:.no_toc}

可以有两种基于他们的 *reliability可靠性* 的数据源.
数据源（如 Kafka 和 Flume）允许传输的数据被确认.
如果系统从这些可靠的数据来源接收数据，并且被确认（acknowledges）正确地接收数据，它可以确保数据不会因为任何类型的失败而导致数据丢失.
这样就出现了 2 种接收器（receivers）:

1. *Reliable Receiver（可靠的接收器）* - 当数据被接收并存储在 Spark 中并带有备份副本时，一个可靠的接收器（reliable receiver）正确地发送确认（acknowledgment）给一个可靠的数据源（reliable source）.
1. *Unreliable Receiver（不可靠的接收器）* - 一个不可靠的接收器（ unreliable receiver ）不发送确认（acknowledgment）到数据源。这可以用于不支持确认的数据源，或者甚至是可靠的数据源当你不想或者不需要进行复杂的确认的时候.

在 [自定义 Receiver 指南](streaming-custom-receivers.html) 中描述了关于如何去编写一个 reliable receiver（可靠的接收器）的细节.

***

##  DStreams 上的 Transformations（转换）
与 RDD 类似，transformation 允许从 input DStream 输入的数据做修改.
DStreams 支持很多在 RDD 中可用的 transformation 算子。一些常用的如下所示 : 

与RDD类似，类似，transformation 允许修改来自 input DStream 的数据.
DStreams 支持标准的 Spark RDD 上可用的许多转换.
一些常见的如下.

<table class="table">
<tr><th style="width:25%">Transformation（转换）</th><th>Meaning（含义）</th></tr>
<tr>
  <td> <b>map</b>(<i>func</i>) </td>
  <td> 利用函数 <i>func</i> 处理原 DStream 的每个元素，返回一个新的 DStream. </td>
</tr>
<tr>
  <td> <b>flatMap</b>(<i>func</i>) </td>
  <td> 与 map 相似，但是每个输入项可用被映射为 0 个或者多个输出项。. </td>
</tr>
<tr>
  <td> <b>filter</b>(<i>func</i>) </td>
  <td> 返回一个新的 DStream，它仅仅包含原 DStream 中函数 <i>func</i> 返回值为 true 的项.</td>
</tr>
<tr>
  <td> <b>repartition</b>(<i>numPartitions</i>) </td>
  <td> 通过创建更多或者更少的 partition 以改变这个 DStream 的并行级别（level of parallelism）. </td>
</tr>
<tr>
  <td> <b>union</b>(<i>otherStream</i>) </td>
  <td> 返回一个新的 DStream，它包含源 DStream 和 <i>otherDStream</i> 的所有元素.</td>
</tr>
<tr>
  <td> <b>count</b>() </td>
  <td> 通过 count 源 DStream 中每个 RDD 的元素数量，返回一个包含单元素（single-element）RDDs 的新 DStream. </td>
</tr>
<tr>
  <td> <b>reduce</b>(<i>func</i>) </td>
  <td> 利用函数 <i>func</i> 聚集源 DStream 中每个 RDD 的元素，返回一个包含单元素（single-element）RDDs 的新 DStream。函数应该是相关联的，以使计算可以并行化.</td>
</tr>
<tr>
  <td> <b>countByValue</b>() </td>
  <td> 在元素类型为 K 的 DStream上，返回一个（K,long）pair 的新的 DStream，每个 key 的值是在原 DStream 的每个 RDD 中的次数.</td>
</tr>
<tr>
  <td> <b>reduceByKey</b>(<i>func</i>, [<i>numTasks</i>]) </td>
  <td> 当在一个由 (K,V) pairs 组成的 DStream 上调用这个算子时，返回一个新的, 由 (K,V) pairs 组成的 DStream，每一个 key 的值均由给定的 reduce 函数聚合起来.
  <b>注意</b>：在默认情况下，这个算子利用了 Spark 默认的并发任务数去分组。你可以用 numTasks 参数设置不同的任务数。</td>
</tr>
<tr>
  <td> <b>join</b>(<i>otherStream</i>, [<i>numTasks</i>]) </td>
  <td> 当应用于两个 DStream（一个包含（K,V）对，一个包含 (K,W) 对），返回一个包含 (K, (V, W)) 对的新 DStream. </td>
</tr>
<tr>
  <td> <b>cogroup</b>(<i>otherStream</i>, [<i>numTasks</i>]) </td>
  <td>当应用于两个 DStream（一个包含（K,V）对，一个包含 (K,W) 对），返回一个包含 (K, Seq[V], Seq[W]) 的 tuples（元组）.</td>
</tr>
<tr>
  <td> <b>transform</b>(<i>func</i>) </td>
  <td> 通过对源 DStream 的每个 RDD 应用 RDD-to-RDD 函数，创建一个新的 DStream. 这个可以在 DStream 中的任何 RDD 操作中使用. </td>
</tr>
<tr>
  <td> <b>updateStateByKey</b>(<i>func</i>) </td>
  <td> 返回一个新的 "状态" 的 DStream，其中每个 key 的状态通过在 key 的先前状态应用给定的函数和 key 的新 valyes 来更新. 这可以用于维护每个 key 的任意状态数据.</td>
</tr>
<tr><td></td><td></td></tr>
</table>

其中一些转换值得深入讨论.

#### UpdateStateByKey 操作
{:.no_toc}
该 `updateStateByKey` 操作允许您维护任意状态，同时不断更新新信息. 你需要通过两步来使用它.

1. 定义 state - state 可以是任何的数据类型.
1. 定义 state update function（状态更新函数） - 使用函数指定如何使用先前状态来更新状态，并从输入流中指定新值.

在每个 batch 中，Spark 会使用状态更新函数为所有已有的 key 更新状态，不管在 batch 中是否含有新的数据。如果这个更新函数返回一个 none，这个 key-value pair 也会被消除.

让我们举个例子来说明.
在例子中，假设你想保持在文本数据流中看到的每个单词的运行计数，运行次数用一个 state 表示，它的类型是整数, 我们可以使用如下方式来定义 update 函数:

<div class="codetabs">
<div data-lang="scala" markdown="1">

{% highlight scala %}
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}
{% endhighlight %}

这里是一个应用于包含 words（单词）的 DStream 上（也就是说，在 [先前的示例](#一个入门示例)中，该 `pairs` DStream 包含了 (word, 1) pair）.

{% highlight scala %}
val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
{% endhighlight %}

update 函数将会被每个单词调用，`newValues` 拥有一系列的 1（来自 (word, 1) pairs），runningCount 拥有之前的次数.

</div>
<div data-lang="java" markdown="1">

{% highlight java %}
Function2<List<Integer>, Optional<Integer>, Optional<Integer>> updateFunction =
  (values, state) -> {
    Integer newSum = ...  // add the new values with the previous running count to get the new count
    return Optional.of(newSum);
  };
{% endhighlight %}

This is applied on a DStream containing words (say, the `pairs` DStream containing `(word,
1)` pairs in the [quick example](#a-quick-example)).

{% highlight java %}
JavaPairDStream<String, Integer> runningCounts = pairs.updateStateByKey(updateFunction);
{% endhighlight %}

The update function will be called for each word, with `newValues` having a sequence of 1's (from
the `(word, 1)` pairs) and the `runningCount` having the previous count. For the complete
Java code, take a look at the example
[JavaStatefulNetworkWordCount.java]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming
/JavaStatefulNetworkWordCount.java).

</div>
<div data-lang="python" markdown="1">

{% highlight python %}
def updateFunction(newValues, runningCount):
    if runningCount is None:
        runningCount = 0
    return sum(newValues, runningCount)  # add the new values with the previous running count to get the new count
{% endhighlight %}

This is applied on a DStream containing words (say, the `pairs` DStream containing `(word,
1)` pairs in the [earlier example](#a-quick-example)).

{% highlight python %}
runningCounts = pairs.updateStateByKey(updateFunction)
{% endhighlight %}

The update function will be called for each word, with `newValues` having a sequence of 1's (from
the `(word, 1)` pairs) and the `runningCount` having the previous count. For the complete
Python code, take a look at the example
[stateful_network_wordcount.py]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/stateful_network_wordcount.py).

</div>
</div>

请注意, 使用 `updateStateByKey` 需要配置的 `checkpoint` （检查点）的目录，这里是更详细关于讨论 [checkpointing](#checkpointing) 的部分.

#### Transform Operation*（转换操作）
{:.no_toc}
transform 操作（以及它的变化形式如 `transformWith`）允许在 DStream 运行任何 RDD-to-RDD 函数.
它能够被用来应用任何没在 DStream API 中提供的 RDD 操作.
例如，连接数据流中的每个批（batch）和另外一个数据集的功能并没有在 DStream API 中提供，然而你可以简单的利用 `transform` 方法做到.
这使得有非常强大的可能性.
例如，可以通过将输入数据流与预先计算的垃圾邮件信息（也可以使用 Spark 一起生成）进行实时数据清理，然后根据它进行过滤.

<div class="codetabs">
<div data-lang="scala" markdown="1">

{% highlight scala %}
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information

val cleanedDStream = wordCounts.transform { rdd =>
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
  ...
}
{% endhighlight %}

</div>
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.spark.streaming.api.java.*;
// RDD containing spam information
JavaPairRDD<String, Double> spamInfoRDD = jssc.sparkContext().newAPIHadoopRDD(...);

JavaPairDStream<String, Integer> cleanedDStream = wordCounts.transform(rdd -> {
  rdd.join(spamInfoRDD).filter(...); // join data stream with spam information to do data cleaning
  ...
});
{% endhighlight %}

</div>
<div data-lang="python" markdown="1">

{% highlight python %}
spamInfoRDD = sc.pickleFile(...)  # RDD containing spam information

# join data stream with spam information to do data cleaning
cleanedDStream = wordCounts.transform(lambda rdd: rdd.join(spamInfoRDD).filter(...))
{% endhighlight %}
</div>
</div>

请注意，每个 batch interval（批间隔）提供的函数被调用.
这允许你做随时间变动的 RDD 操作, 即 RDD 操作, 分区的数量，广播变量，等等.
batch 之间等可以改变。

#### Window Operations（窗口操作）
{:.no_toc}
Spark Streaming 也支持 *windowed computations（窗口计算）*，它允许你在数据的一个滑动窗口上应用 transformation（转换）.
下图说明了这个滑动窗口.

<p style="text-align: center;">
  <img src="img/streaming-dstream-window.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="60%" />
</p>

如上图显示，窗口在源 DStream 上 *slides（滑动）*，合并和操作落入窗内的源 RDDs，产生窗口化的 DStream 的 RDDs。在这个具体的例子中，程序在三个时间单元的数据上进行窗口操作，并且每两个时间单元滑动一次。 这说明，任何一个窗口操作都需要指定两个参数.

 * <i>window length（窗口长度）</i> - 窗口的持续时间（图 3）.
 * <i>sliding interval（滑动间隔）</i> - 执行窗口操作的间隔（图 2）.

这两个参数必须是 source DStream 的 batch interval（批间隔）的倍数（图 1）.  

让我们举例以说明窗口操作.
例如，你想扩展前面的例子用来计算过去 30 秒的词频，间隔时间是 10 秒.
为了达到这个目的，我们必须在过去 30 秒的 `(wrod, 1)` pairs 的 `pairs` DStream 上应用 `reduceByKey` 操作.
用方法 `reduceByKeyAndWindow` 实现.

<div class="codetabs">
<div data-lang="scala" markdown="1">

{% highlight scala %}
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
{% endhighlight %}

</div>
<div data-lang="java" markdown="1">

{% highlight java %}
// Reduce last 30 seconds of data, every 10 seconds
JavaPairDStream<String, Integer> windowedWordCounts = pairs.reduceByKeyAndWindow((i1, i2) -> i1 + i2, Durations.seconds(30), Durations.seconds(10));
{% endhighlight %}

</div>
<div data-lang="python" markdown="1">

{% highlight python %}
# Reduce last 30 seconds of data, every 10 seconds
windowedWordCounts = pairs.reduceByKeyAndWindow(lambda x, y: x + y, lambda x, y: x - y, 30, 10)
{% endhighlight %}

</div>
</div>

一些常用的窗口操作如下所示，这些操作都需要用到上文提到的两个参数 - <i>windowLength（窗口长度）</i> 和 <i>slideInterval（滑动的时间间隔）</i>.

<table class="table">
<tr><th style="width:25%">Transformation（转换）</th><th>Meaning（含义）</th></tr>
<tr>
  <td> <b>window</b>(<i>windowLength</i>, <i>slideInterval</i>) </td>
  <td> 返回一个新的 DStream, 它是基于 source DStream 的窗口 batch 进行计算的.</td>
</tr>
<tr>
  <td> <b>countByWindow</b>(<i>windowLength</i>, <i>slideInterval</i>) </td>
  <td> 返回 stream（流）中滑动窗口元素的数</td>
</tr>
<tr>
  <td> <b>reduceByWindow</b>(<i>func</i>, <i>windowLength</i>, <i>slideInterval</i>) </td>
  <td> 返回一个新的单元素 stream（流），它通过在一个滑动间隔的 stream 中使用 <i>func</i> 来聚合以创建. 该函数应该是 associative（关联的）且 commutative（可交换的），以便它可以并行计算</td>
</tr>
<tr>
  <td> <b>reduceByKeyAndWindow</b>(<i>func</i>, <i>windowLength</i>, <i>slideInterval</i>,
  [<i>numTasks</i>]) </td>
  <td> 在一个 (K, V) pairs 的 DStream 上调用时, 返回一个新的 (K, V) pairs 的 Stream, 其中的每个 key 的 values 是在滑动窗口上的 batch 使用给定的函数 <i>func</i> 来聚合产生的.
  <b>Note（注意）:</b> 默认情况下, 该操作使用 Spark 的默认并行任务数量（local model 是 2, 在 cluster mode 中的数量通过 <code>spark.default.parallelism</code> 来确定）来做 grouping. 您可以通过一个可选的 <code>numTasks</code> 参数来设置一个不同的 tasks（任务）数量.
  </td>
</tr>
<tr>
  <td> <b>reduceByKeyAndWindow</b>(<i>func</i>, <i>invFunc</i>, <i>windowLength</i>,
  <i>slideInterval</i>, [<i>numTasks</i>]) </td>
  <td markdown="1"> 上述 <code>reduceByKeyAndWindow()</code> 的更有效的一个版本，其中使用前一窗口的 reduce 值逐渐计算每个窗口的 reduce值. 这是通过减少进入滑动窗口的新数据，以及 "inverse reducing（逆减）" 离开窗口的旧数据来完成的. 一个例子是当窗口滑动时"添加" 和 "减" keys 的数量. 然而，它仅适用于 “invertible reduce functions（可逆减少函数）”，即具有相应 "inverse reduce（反向减少）" 函数的 reduce 函数（作为参数<i> invFunc </ i>）. 像在 <code>reduceByKeyAndWindow</code> 中的那样, reduce 任务的数量可以通过可选参数进行配置. 请注意, 针对该操作的使用必须启用 [checkpointing](#checkpointing).
</td>
</tr>
<tr>
  <td> <b>countByValueAndWindow</b>(<i>windowLength</i>,
  <i>slideInterval</i>, [<i>numTasks</i>]) </td>
  <td> 在一个 (K, V) pairs 的 DStream 上调用时, 返回一个新的 (K, Long) pairs 的 DStream, 其中每个 key 的 value 是它在一个滑动窗口之内的频次. 像 code>reduceByKeyAndWindow</code> 中的那样, reduce 任务的数量可以通过可选参数进行配置.
</td>
</tr>
<tr><td></td><td></td></tr>
</table>

#### Join 操作
{:.no_toc}
最后，它值得强调的是，您可以轻松地在 Spark Streaming 中执行不同类型的 join.

##### Stream-stream joins
{:.no_toc}
Streams（流）可以非常容易地与其他流进行 join.

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
JavaPairDStream<String, String> stream1 = ...
JavaPairDStream<String, String> stream2 = ...
JavaPairDStream<String, Tuple2<String, String>> joinedStream = stream1.join(stream2);
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
stream1 = ...
stream2 = ...
joinedStream = stream1.join(stream2)
{% endhighlight %}
</div>
</div>

这里，在每个 batch interval（批间隔）中，由 `stream1` 生成的 RDD 将与 `stream2` 生成的 RDD 进行 jion.
你也可以做 `leftOuterJoin`，`rightOuterJoin`，`fullOuterJoin`.
此外，在 stream（流）的窗口上进行 join 通常是非常有用的.
这也很容易做到.

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream2.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2)
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
JavaPairDStream<String, String> windowedStream1 = stream1.window(Durations.seconds(20));
JavaPairDStream<String, String> windowedStream2 = stream2.window(Durations.minutes(1));
JavaPairDStream<String, Tuple2<String, String>> joinedStream = windowedStream1.join(windowedStream2);
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
windowedStream1 = stream1.window(20)
windowedStream2 = stream2.window(60)
joinedStream = windowedStream1.join(windowedStream2)
{% endhighlight %}
</div>
</div>

##### Stream-dataset joins
{:.no_toc}
这在解释 `DStream.transform` 操作时已经在前面演示过了.
这是另一个 join window stream（窗口流）与 dataset 的例子.

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
JavaPairRDD<String, String> dataset = ...
JavaPairDStream<String, String> windowedStream = stream.window(Durations.seconds(20));
JavaPairDStream<String, String> joinedStream = windowedStream.transform(rdd -> rdd.join(dataset));
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
dataset = ... # some RDD
windowedStream = stream.window(20)
joinedStream = windowedStream.transform(lambda rdd: rdd.join(dataset))
{% endhighlight %}
</div>
</div>

实际上，您也可以动态更改要加入的 dataset.
提供给 `transform` 的函数是每个 batch interval（批次间隔）进行评估，因此将使用 `dataset` 引用指向当前的 dataset.

DStream 转换的完整列表可在 API 文档中找到.
针对 Scala API，请看 [DStream](api/scala/index.html#org.apache.spark.streaming.dstream.DStream) 和 [PairDStreamFunctions](api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions).
针对 Java API，请看 [JavaDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html) 和 [JavaPairDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html).
针对 Python API，请看 [DStream](api/python/pyspark.streaming.html#pyspark.streaming.DStream).

***

## DStreams 上的输出操作

输出操作允许将 DStream 的数据推送到外部系统, 如数据库或文件系统.
由于输出操作实际上允许外部系统使用变换后的数据, 所以它们触发所有 DStream 变换的实际执行（类似于RDD的动作）.
目前, 定义了以下输出操作：

<table class="table">
<tr><th style="width:30%">Output Operation</th><th>Meaning</th></tr>
<tr>
  <td> <b>print</b>()</td>
  <td> 在运行流应用程序的 driver 节点上的DStream中打印每批数据的前十个元素. 这对于开发和调试很有用.
  <br/>
  <span class="badge" style="background-color: grey">Python API</span> 这在 Python API 中称为 <b>pprint()</b>.
  </td>
</tr>
<tr>
  <td> <b>saveAsTextFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> 将此 DStream 的内容另存为文本文件. 
  每个批处理间隔的文件名是根据 <i>前缀</i> 和 <i>后缀</i> : <i>"prefix-TIME_IN_MS[.suffix]"</i> 生成的.</td>
</tr>
<tr>
  <td> <b>saveAsObjectFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> 将此 DStream 的内容另存为序列化 Java 对象的 <code>SequenceFiles</code>. 
  每个批处理间隔的文件名是根据 <i>前缀</i> 和 <i>后缀</i> : <i>"prefix-TIME_IN_MS[.suffix]"</i> 生成的.
  <br/>
  <span class="badge" style="background-color: grey">Python API</span> 这在Python API中是不可用的.
  </td>
</tr>
<tr>
  <td> <b>saveAsHadoopFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> 将此 DStream 的内容另存为 Hadoop 文件.
  每个批处理间隔的文件名是根据 <i>前缀</i> 和 <i>后缀</i> : <i>"prefix-TIME_IN_MS[.suffix]"</i> 生成的.
  <br>
  <span class="badge" style="background-color: grey">Python API</span> 这在Python API中是不可用的.
  </td>
</tr>
<tr>
  <td> <b>foreachRDD</b>(<i>func</i>) </td>
  <td> 对从流中生成的每个 RDD 应用函数 <i>func</i> 的最通用的输出运算符. 此功能应将每个 RDD 中的数据推送到外部系统, 例如将 RDD 保存到文件, 或将其通过网络写入数据库. 请注意, 函数 <i>func</i> 在运行流应用程序的 driver 进程中执行, 通常会在其中具有 RDD 动作, 这将强制流式传输 RDD 的计算.</td>
</tr>
<tr><td></td><td></td></tr>
</table>

### foreachRDD 设计模式的使用
{:.no_toc}
`dstream.foreachRDD` 是一个强大的原语, 允许将数据发送到外部系统.但是, 了解如何正确有效地使用这个原语很重要. 避免一些常见的错误如下.

通常向外部系统写入数据需要创建连接对象（例如与远程服务器的 TCP 连接）, 并使用它将数据发送到远程系统.为此, 开发人员可能会无意中尝试在Spark driver 中创建连接对象, 然后尝试在Spark工作人员中使用它来在RDD中保存记录.例如（在 Scala 中）:

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
dstream.foreachRDD(rdd -> {
  Connection connection = createNewConnection(); // executed at the driver
  rdd.foreach(record -> {
    connection.send(record); // executed at the worker
  });
});
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
def sendRecord(rdd):
    connection = createNewConnection()  # executed at the driver
    rdd.foreach(lambda record: connection.send(record))
    connection.close()

dstream.foreachRDD(sendRecord)
{% endhighlight %}
</div>
</div>

这是不正确的, 因为这需要将连接对象序列化并从 driver 发送到 worker. 这种连接对象很少能跨机器转移.
此错误可能会显示为序列化错误（连接对象不可序列化）, 初始化错误（连接对象需要在 worker 初始化）等.
正确的解决方案是在 worker 创建连接对象.

但是, 这可能会导致另一个常见的错误 - 为每个记录创建一个新的连接. 例如: 

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
dstream.foreachRDD(rdd -> {
  rdd.foreach(record -> {
    Connection connection = createNewConnection();
    connection.send(record);
    connection.close();
  });
});
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
def sendRecord(record):
    connection = createNewConnection()
    connection.send(record)
    connection.close()

dstream.foreachRDD(lambda rdd: rdd.foreach(sendRecord))
{% endhighlight %}
</div>
</div>

通常, 创建连接对象具有时间和资源开销. 
因此, 创建和销毁每个记录的连接对象可能会引起不必要的高开销, 并可显着降低系统的总体吞吐量. 
一个更好的解决方案是使用 `rdd.foreachPartition` - 创建一个连接对象, 并使用该连接在 RDD 分区中发送所有记录.

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
dstream.foreachRDD(rdd -> {
  rdd.foreachPartition(partitionOfRecords -> {
    Connection connection = createNewConnection();
    while (partitionOfRecords.hasNext()) {
      connection.send(partitionOfRecords.next());
    }
    connection.close();
  });
});
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
def sendPartition(iter):
    connection = createNewConnection()
    for record in iter:
        connection.send(record)
    connection.close()

dstream.foreachRDD(lambda rdd: rdd.foreachPartition(sendPartition))
{% endhighlight %}
</div>
</div>

  这样可以在多个记录上分摊连接创建开销.

最后, 可以通过跨多个RDD /批次重用连接对象来进一步优化.
可以维护连接对象的静态池, 而不是将多个批次的 RDD 推送到外部系统时重新使用, 从而进一步减少开销.

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
dstream.foreachRDD(rdd -> {
  rdd.foreachPartition(partitionOfRecords -> {
    // ConnectionPool is a static, lazily initialized pool of connections
    Connection connection = ConnectionPool.getConnection();
    while (partitionOfRecords.hasNext()) {
      connection.send(partitionOfRecords.next());
    }
    ConnectionPool.returnConnection(connection); // return to the pool for future reuse
  });
});
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
def sendPartition(iter):
    # ConnectionPool is a static, lazily initialized pool of connections
    connection = ConnectionPool.getConnection()
    for record in iter:
        connection.send(record)
    # return to the pool for future reuse
    ConnectionPool.returnConnection(connection)

dstream.foreachRDD(lambda rdd: rdd.foreachPartition(sendPartition))
{% endhighlight %}
</div>
</div>

请注意, 池中的连接应根据需要懒惰创建, 如果不使用一段时间, 则会超时.
这实现了最有效地将数据发送到外部系统.


##### 其他要记住的要点:
{:.no_toc}
- DStreams 通过输出操作进行延迟执行, 就像 RDD 由 RDD 操作懒惰地执行. 具体来说, DStream 输出操作中的 RDD 动作强制处理接收到的数据.因此, 如果您的应用程序没有任何输出操作, 或者具有 `dstream.foreachRDD()` 等输出操作, 而在其中没有任何 RDD 操作, 则不会执行任何操作.系统将简单地接收数据并将其丢弃.

- 默认情况下, 输出操作是 one-at-a-time 执行的. 它们按照它们在应用程序中定义的顺序执行.

***

## DataFrame 和 SQL 操作

您可以轻松地在流数据上使用 [DataFrames and SQL](sql-programming-guide.html) 和 SQL 操作. 您必须使用 StreamingContext 正在使用的 SparkContext 创建一个 SparkSession.此外, 必须这样做, 以便可以在 driver 故障时重新启动. 这是通过创建一个简单实例化的 SparkSession 单例实例来实现的.这在下面的示例中显示.它使用 DataFrames 和 SQL 来修改早期的字数 [示例以生成单词计数](#a-quick-example).将每个 RDD 转换为 DataFrame, 注册为临时表, 然后使用 SQL 进行查询.

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}

/** DataFrame operations inside your streaming program */

val words: DStream[String] = ...

words.foreachRDD { rdd =>

  // Get the singleton instance of SparkSession
  val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
  import spark.implicits._

  // Convert RDD[String] to DataFrame
  val wordsDataFrame = rdd.toDF("word")

  // Create a temporary view
  wordsDataFrame.createOrReplaceTempView("words")

  // Do word count on DataFrame using SQL and print it
  val wordCountsDataFrame = 
    spark.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
}

{% endhighlight %}

请参阅完整的 [源代码]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/SqlNetworkWordCount.scala).
</div>
<div data-lang="java" markdown="1">
{% highlight java %}

/** Java Bean class for converting RDD to DataFrame */
public class JavaRow implements java.io.Serializable {
  private String word;

  public String getWord() {
    return word;
  }

  public void setWord(String word) {
    this.word = word;
  }
}

...

/** DataFrame operations inside your streaming program */

JavaDStream<String> words = ... 

words.foreachRDD((rdd, time) -> {
  // Get the singleton instance of SparkSession
  SparkSession spark = SparkSession.builder().config(rdd.sparkContext().getConf()).getOrCreate();

  // Convert RDD[String] to RDD[case class] to DataFrame
  JavaRDD<JavaRow> rowRDD = rdd.map(word -> {
    JavaRow record = new JavaRow();
    record.setWord(word);
    return record;
  });
  DataFrame wordsDataFrame = spark.createDataFrame(rowRDD, JavaRow.class);

  // Creates a temporary view using the DataFrame
  wordsDataFrame.createOrReplaceTempView("words");

  // Do word count on table using SQL and print it
  DataFrame wordCountsDataFrame =
    spark.sql("select word, count(*) as total from words group by word");
  wordCountsDataFrame.show();
});
{% endhighlight %}

请参阅完整的 [源代码]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming/JavaSqlNetworkWordCount.java).
</div>
<div data-lang="python" markdown="1">
{% highlight python %}

# Lazily instantiated global instance of SparkSession
def getSparkSessionInstance(sparkConf):
    if ("sparkSessionSingletonInstance" not in globals()):
        globals()["sparkSessionSingletonInstance"] = SparkSession \
            .builder \
            .config(conf=sparkConf) \
            .getOrCreate()
    return globals()["sparkSessionSingletonInstance"]

...

# DataFrame operations inside your streaming program

words = ... # DStream of strings

def process(time, rdd):
    print("========= %s =========" % str(time))
    try:
        # Get the singleton instance of SparkSession
        spark = getSparkSessionInstance(rdd.context.getConf())

        # Convert RDD[String] to RDD[Row] to DataFrame
        rowRdd = rdd.map(lambda w: Row(word=w))
        wordsDataFrame = spark.createDataFrame(rowRdd)

        # Creates a temporary view using the DataFrame
        wordsDataFrame.createOrReplaceTempView("words")

        # Do word count on table using SQL and print it
        wordCountsDataFrame = spark.sql("select word, count(*) as total from words group by word")
        wordCountsDataFrame.show()
    except:
        pass

words.foreachRDD(process)
{% endhighlight %}

请参阅完整的 [源代码]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/sql_network_wordcount.py).

</div>
</div>

您还可以对来自不同线程的流数据（即异步运行的 StreamingContext ）上定义的表运行 SQL 查询.
只需确保您将 StreamingContext 设置为记住足够数量的流数据, 以便查询可以运行.
否则, 不知道任何异步 SQL 查询的 StreamingContext 将在查询完成之前删除旧的流数据.
例如, 如果要查询最后一个批次, 但是您的查询可能需要5分钟才能运行, 则可以调用 `streamingContext.remember(Minutes(5))` （以 Scala 或其他语言的等价物）.

有关DataFrames的更多信息, 请参阅 [DataFrames 和 SQL 指南](sql-programming-guide.html).

***

## MLlib 操作

您还可以轻松使用 [MLlib](ml-guide.html) 提供的机器学习算法.
首先, 有 streaming 机器学习算法（例如: [Streaming 线性回归](mllib-linear-methods.html#streaming-linear-regression), [Streaming KMeans](mllib-clustering.html#streaming-k-means) 等）, 其可以同时从 streaming 数据中学习, 并将该模型应用于 streaming 数据. 
除此之外, 对于更大类的机器学习算法, 您可以离线学习一个学习模型（即使用历史数据）, 然后将该模型在线应用于流数据.有关详细信息, 请参阅 [MLlib指南](ml-guide.html).

***

## 缓存 / 持久性

与 RDD 类似, DStreams 还允许开发人员将流的数据保留在内存中. 
也就是说, 在 DStream 上使用 `persist()` 方法会自动将该 DStream 的每个 RDD 保留在内存中. 
如果 DStream 中的数据将被多次计算（例如, 相同数据上的多个操作）, 这将非常有用. 
对于基于窗口的操作, 如 `reduceByWindow` 和 `reduceByKeyAndWindow` 以及基于状态的操作, 
如 `updateStateByKey`, 这是隐含的.因此, 基于窗口的操作生成的 DStream 会自动保存在内存中, 而不需要开发人员调用 `persist()`.

对于通过网络接收数据（例如: Kafka, Flume, sockets 等）的输入流, 默认持久性级别被设置为将数据复制到两个节点进行容错.

请注意, 与 RDD 不同, DStreams 的默认持久性级别将数据序列化在内存中. 
这在 [性能调优](#memory-tuning) 部分进一步讨论. 有关不同持久性级别的更多信息, 请参见 [Spark编程指南](programming-guide.html#rdd-persistence).

***

## Checkpointing

 streaming 应用程序必须 24/7 运行, 因此必须对应用逻辑无关的故障（例如, 系统故障, JVM 崩溃等）具有弹性. 为了可以这样做, Spark Streaming 需要 *checkpoint* 足够的信息到容错存储系统, 以便可以从故障中恢复.*checkpoint* 有两种类型的数据.

- *Metadata checkpointing* - 将定义 streaming 计算的信息保存到容错存储（如 HDFS）中.这用于从运行 streaming 应用程序的 driver 的节点的故障中恢复（稍后详细讨论）. 元数据包括:
  +  *Configuration* - 用于创建流应用程序的配置.
  +  *DStream operations* - 定义 streaming 应用程序的 DStream 操作集.
  +  *Incomplete batches* - 批量的job 排队但尚未完成.
- *Data checkpointing* - 将生成的 RDD 保存到可靠的存储.这在一些将多个批次之间的数据进行组合的 *状态* 变换中是必需的.在这种转换中, 生成的 RDD 依赖于先前批次的 RDD, 这导致依赖链的长度随时间而增加.为了避免恢复时间的这种无限增加（与依赖关系链成比例）, 有状态转换的中间 RDD 会定期 *checkpoint* 到可靠的存储（例如 HDFS）以切断依赖关系链.

总而言之, 元数据 checkpoint 主要用于从 driver 故障中恢复, 而数据或 RDD  checkpoint 对于基本功能（如果使用有状态转换）则是必需的.

#### 何时启用 checkpoint 
{:.no_toc}

对于具有以下任一要求的应用程序, 必须启用 checkpoint:

- *使用状态转换* - 如果在应用程序中使用 `updateStateByKey`或 `reduceByKeyAndWindow`（具有反向功能）, 则必须提供 checkpoint 目录以允许定期的 RDD checkpoint.
- *从运行应用程序的 driver 的故障中恢复* - 元数据 checkpoint 用于使用进度信息进行恢复.

请注意, 无需进行上述有状态转换的简单 streaming 应用程序即可运行, 无需启用 checkpoint. 
在这种情况下, 驱动器故障的恢复也将是部分的（一些接收但未处理的数据可能会丢失）. 
这通常是可以接受的, 许多运行 Spark Streaming 应用程序. 未来对非 Hadoop 环境的支持预计会有所改善.

#### 如何配置 checkpoint
{:.no_toc}
可以通过在保存 checkpoint 信息的容错, 可靠的文件系统（例如, HDFS, S3等）中设置目录来启用 checkpoint. 
这是通过使用 `streamingContext.checkpoint(checkpointDirectory)` 完成的. 这将允许您使用上述有状态转换.
另外, 如果要使应用程序从 driver 故障中恢复, 您应该重写 streaming 应用程序以具有以下行为.

  + 当程序第一次启动时, 它将创建一个新的 StreamingContext, 设置所有流, 然后调用 start().
  + 当程序在失败后重新启动时, 它将从 checkpoint 目录中的 checkpoint 数据重新创建一个 StreamingContext.

<div class="codetabs">
<div data-lang="scala" markdown="1">

使用 `StreamingContext.getOrCreate` 可以简化此行为. 这样使用如下.

{% highlight scala %}
// Function to create and setup a new StreamingContext
def functionToCreateContext(): StreamingContext = {
  val ssc = new StreamingContext(...)   // new context
  val lines = ssc.socketTextStream(...) // create DStreams
  ...
  ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
  ssc
}

// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start()
context.awaitTermination()
{% endhighlight %}

If the `checkpointDirectory` exists, then the context will be recreated from the checkpoint data.
If the directory does not exist (i.e., running for the first time),
then the function `functionToCreateContext` will be called to create a new
context and set up the DStreams. See the Scala example
[RecoverableNetworkWordCount]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala).
This example appends the word counts of network data into a file.

</div>
<div data-lang="java" markdown="1">

This behavior is made simple by using `JavaStreamingContext.getOrCreate`. This is used as follows.

{% highlight java %}
// Create a factory object that can create and setup a new JavaStreamingContext
JavaStreamingContextFactory contextFactory = new JavaStreamingContextFactory() {
  @Override public JavaStreamingContext create() {
    JavaStreamingContext jssc = new JavaStreamingContext(...);  // new context
    JavaDStream<String> lines = jssc.socketTextStream(...);     // create DStreams
    ...
    jssc.checkpoint(checkpointDirectory);                       // set checkpoint directory
    return jssc;
  }
};

// Get JavaStreamingContext from checkpoint data or create a new one
JavaStreamingContext context = JavaStreamingContext.getOrCreate(checkpointDirectory, contextFactory);

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start();
context.awaitTermination();
{% endhighlight %}

If the `checkpointDirectory` exists, then the context will be recreated from the checkpoint data.
If the directory does not exist (i.e., running for the first time),
then the function `contextFactory` will be called to create a new
context and set up the DStreams. See the Java example
[JavaRecoverableNetworkWordCount]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/java/org/apache/spark/examples/streaming/JavaRecoverableNetworkWordCount.java).
This example appends the word counts of network data into a file.

</div>
<div data-lang="python" markdown="1">

This behavior is made simple by using `StreamingContext.getOrCreate`. This is used as follows.

{% highlight python %}
# Function to create and setup a new StreamingContext
def functionToCreateContext():
    sc = SparkContext(...)  # new context
    ssc = StreamingContext(...)
    lines = ssc.socketTextStream(...)  # create DStreams
    ...
    ssc.checkpoint(checkpointDirectory)  # set checkpoint directory
    return ssc

# Get StreamingContext from checkpoint data or create a new one
context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext)

# Do additional setup on context that needs to be done,
# irrespective of whether it is being started or restarted
context. ...

# Start the context
context.start()
context.awaitTermination()
{% endhighlight %}

If the `checkpointDirectory` exists, then the context will be recreated from the checkpoint data.
If the directory does not exist (i.e., running for the first time),
then the function `functionToCreateContext` will be called to create a new
context and set up the DStreams. See the Python example
[recoverable_network_wordcount.py]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/python/streaming/recoverable_network_wordcount.py).
This example appends the word counts of network data into a file.

You can also explicitly create a `StreamingContext` from the checkpoint data and start the
 computation by using `StreamingContext.getOrCreate(checkpointDirectory, None)`.

</div>
</div>

除了使用 `getOrCreate` 之外, 还需要确保在失败时自动重新启动 driver 进程. 
这只能由用于运行应用程序的部署基础架构完成. 这在 [部署](#deploying-applications) 部分进一步讨论.

请注意, RDD 的 checkpoint 会导致保存到可靠存储的成本. 这可能会导致 RDD 得到 checkpoint 的批次的处理时间增加. 因此, 需要仔细设置 checkpoint 的间隔. 
在小批量大小（例如: 1秒）, 检查每个批次可能会显着降低操作吞吐量. 相反,  checkpoint 太少会导致谱系和任务大小增长, 这可能会产生不利影响. 
对于需要 RDD  checkpoint 的状态转换, 默认间隔是至少10秒的批间隔的倍数. 它可以通过使用 `dstream.checkpoint(checkpointInterval)` 进行设置. 通常, DStream 的5到10个滑动间隔的 checkpoint 间隔是一个很好的设置.

***

## Accumulators, Broadcast 变量, 和 Checkpoint

在Spark Streaming中, 无法从 checkpoint 恢复 [Accumulators](programming-guide.html#accumulators) 和 [Broadcast 变量](programming-guide.html#broadcast-variables) .
如果启用 checkpoint 并使用 [Accumulators](programming-guide.html#accumulators) 或 [Broadcast 变量](programming-guide.html#broadcast-variables) , 则必须为 [Accumulators](programming-guide.html#accumulators) 和 [Broadcast 变量](programming-guide.html#broadcast-variables) 创建延迟实例化的单例实例, 以便在 driver 重新启动失败后重新实例化. 
这在下面的示例中显示: 

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}

object WordBlacklist {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlacklist = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlacklist)
        }
      }
    }
    instance
  }
}

object DroppedWordsCounter {

  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("WordsInBlacklistCounter")
        }
      }
    }
    instance
  }
}

wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
  // Get or register the blacklist Broadcast
  val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
  // Get or register the droppedWordsCounter Accumulator
  val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
  // Use blacklist to drop words and use droppedWordsCounter to count them
  val counts = rdd.filter { case (word, count) =>
    if (blacklist.value.contains(word)) {
      droppedWordsCounter.add(count)
      false
    } else {
      true
    }
  }.collect().mkString("[", ", ", "]")
  val output = "Counts at time " + time + " " + counts
})

{% endhighlight %}

请参阅完整的 [源代码]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala).
</div>
<div data-lang="java" markdown="1">
{% highlight java %}

class JavaWordBlacklist {

  private static volatile Broadcast<List<String>> instance = null;

  public static Broadcast<List<String>> getInstance(JavaSparkContext jsc) {
    if (instance == null) {
      synchronized (JavaWordBlacklist.class) {
        if (instance == null) {
          List<String> wordBlacklist = Arrays.asList("a", "b", "c");
          instance = jsc.broadcast(wordBlacklist);
        }
      }
    }
    return instance;
  }
}

class JavaDroppedWordsCounter {

  private static volatile LongAccumulator instance = null;

  public static LongAccumulator getInstance(JavaSparkContext jsc) {
    if (instance == null) {
      synchronized (JavaDroppedWordsCounter.class) {
        if (instance == null) {
          instance = jsc.sc().longAccumulator("WordsInBlacklistCounter");
        }
      }
    }
    return instance;
  }
}

wordCounts.foreachRDD((rdd, time) -> {
  // Get or register the blacklist Broadcast
  Broadcast<List<String>> blacklist = JavaWordBlacklist.getInstance(new JavaSparkContext(rdd.context()));
  // Get or register the droppedWordsCounter Accumulator
  LongAccumulator droppedWordsCounter = JavaDroppedWordsCounter.getInstance(new JavaSparkContext(rdd.context()));
  // Use blacklist to drop words and use droppedWordsCounter to count them
  String counts = rdd.filter(wordCount -> {
    if (blacklist.value().contains(wordCount._1())) {
      droppedWordsCounter.add(wordCount._2());
      return false;
    } else {
      return true;
    }
  }).collect().toString();
  String output = "Counts at time " + time + " " + counts;
}

{% endhighlight %}

请参阅完整的 [源代码]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming/JavaRecoverableNetworkWordCount.java).
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
def getWordBlacklist(sparkContext):
    if ("wordBlacklist" not in globals()):
        globals()["wordBlacklist"] = sparkContext.broadcast(["a", "b", "c"])
    return globals()["wordBlacklist"]

def getDroppedWordsCounter(sparkContext):
    if ("droppedWordsCounter" not in globals()):
        globals()["droppedWordsCounter"] = sparkContext.accumulator(0)
    return globals()["droppedWordsCounter"]

def echo(time, rdd):
    # Get or register the blacklist Broadcast
    blacklist = getWordBlacklist(rdd.context)
    # Get or register the droppedWordsCounter Accumulator
    droppedWordsCounter = getDroppedWordsCounter(rdd.context)

    # Use blacklist to drop words and use droppedWordsCounter to count them
    def filterFunc(wordCount):
        if wordCount[0] in blacklist.value:
            droppedWordsCounter.add(wordCount[1])
            False
        else:
            True

    counts = "Counts at time %s %s" % (time, rdd.filter(filterFunc).collect())

wordCounts.foreachRDD(echo)

{% endhighlight %}

请参阅完整的 [源代码]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/recoverable_network_wordcount.py).

</div>
</div>

***

## 应用程序部署

本节讨论部署 Spark Streaming 应用程序的步骤.

### 要求
{:.no_toc}

要运行 Spark Streaming 应用程序, 您需要具备以下功能.

- *集群管理器集群* - 这是任何 Spark 应用程序的一般要求, 并在 [部署指南](cluster-overview.html) 中详细讨论.

- *打包应用程序 JAR* - 您必须将 streaming 应用程序编译为 JAR. 如果您正在使用 [`spark-submit`](submitting-applications.html) 启动应用程序, 则不需要在 JAR 中提供 Spark 和 Spark Streaming.但是, 如果您的应用程序使用高级资源（例如: Kafka, Flume）, 那么您将必须将他们链接的额外工件及其依赖项打包在用于部署应用程序的 JAR 中.例如, 使用 `KafkaUtils` 的应用程序必须在应用程序 JAR 中包含 `spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}}` 及其所有传递依赖关系.

- *为 executor 配置足够的内存* - 由于接收到的数据必须存储在内存中, 所以 executor 必须配置足够的内存来保存接收到的数据. 请注意, 如果您正在进行10分钟的窗口操作, 系统必须至少保留最近10分钟的内存中的数据. 因此, 应用程序的内存要求取决于其中使用的操作.

- *配置 checkpoint* - 如果 streaming 应用程序需要它, 则 Hadoop API 兼容容错存储（例如：HDFS, S3等）中的目录必须配置为 checkpoint 目录, 并且流程应用程序以 checkpoint 信息的方式编写 用于故障恢复. 有关详细信息, 请参阅 [checkpoint](#checkpointing) 部分.

- *配置应用程序 driver 的自动重新启动* - 要从 driver 故障自动恢复, 用于运行流应用程序的部署基础架构必须监视 driver 进程, 并在 driver 发生故障时重新启动 driver.不同的 [集群管理者](cluster-overview.html#cluster-manager-types) 有不同的工具来实现这一点.
    + *Spark Standalone* - 可以提交 Spark 应用程序 driver 以在Spark Standalone集群中运行（请参阅 [集群部署模式](spark-standalone.html#launching-spark-applications) ）, 即应用程序 driver 本身在其中一个工作节点上运行. 此外, 可以指示独立的群集管理器来监督 driver, 如果由于非零退出代码而导致 driver 发生故障, 或由于运行 driver 的节点发生故障, 则可以重新启动它. 有关详细信息, 请参阅 [Spark Standalone 指南]](spark-standalone.html) 中的群集模式和监督.
    + *YARN* - Yarn 支持类似的机制来自动重新启动应用程序.有关详细信息, 请参阅 YARN文档.
    + *Mesos* - [Marathon](https://github.com/mesosphere/marathon) 已被用来实现这一点与Mesos.

- *配置预写日志* - 自 Spark 1.2 以来, 我们引入了写入日志来实现强大的容错保证.如果启用, 则从 receiver 接收的所有数据都将写入配置 checkpoint 目录中的写入日志.这可以防止 driver 恢复时的数据丢失, 从而确保零数据丢失（在 [容错语义](#fault-tolerance-semantics) 部分中详细讨论）.可以通过将 [配置参数](configuration.html#spark-streaming) `spark.streaming.receiver.writeAheadLog.enable` 设置为 `true`来启用此功能.然而, 这些更强的语义可能以单个 receiver 的接收吞吐量为代价.通过 [并行运行更多的 receiver](#level-of-parallelism-in-data-receiving) 可以纠正这一点, 以增加总吞吐量.另外, 建议在启用写入日志时, 在日志已经存储在复制的存储系统中时, 禁用在 Spark 中接收到的数据的复制.这可以通过将输入流的存储级别设置为 `StorageLevel.MEMORY_AND_DISK_SER` 来完成.使用 S3（或任何不支持刷新的文件系统）写入日志时, 请记住启用 `spark.streaming.driver.writeAheadLog.closeFileAfterWrite` 和`spark.streaming.receiver.writeAheadLog.closeFileAfterWrite`.有关详细信息, 请参阅 [Spark Streaming配](configuration.html#spark-streaming).请注意, 启用 I/O 加密时, Spark 不会将写入写入日志的数据加密.如果需要对提前记录数据进行加密, 则应将其存储在本地支持加密的文件系统中.

- *设置最大接收速率* - 如果集群资源不够大, streaming 应用程序能够像接收到的那样快速处理数据, 则可以通过设置 记录/秒 的最大速率限制来对 receiver 进行速率限制. 请参阅 receiver 的 `spark.streaming.receiver.maxRate` 和用于 Direct Kafka 方法的 `spark.streaming.kafka.maxRatePerPartition` 的 [配置参数](configuration.html#spark-streaming). 在Spark 1.5中, 我们引入了一个称为背压的功能, 无需设置此速率限制, 因为Spark Streaming会自动计算速率限制, 并在处理条件发生变化时动态调整速率限制. 可以通过将 [配置参数](configuration.html#spark-streaming) `spark.streaming.backpressure.enabled` 设置为 `true` 来启用此 backpressure.

### 升级应用程序代码
{:.no_toc}

如果运行的 Spark Streaming 应用程序需要使用新的应用程序代码进行升级, 则有两种可能的机制.

- 升级后的 Spark Streaming 应用程序与现有应用程序并行启动并运行.一旦新的（接收与旧的数据相同的数据）已经升温并准备好黄金时段, 旧的可以被关掉.请注意, 这可以用于支持将数据发送到两个目的地（即较早和已升级的应用程序）的数据源.

- 现有应用程序正常关闭（请参阅 [`StreamingContext.stop(...)`](api/scala/index.html#org.apache.spark.streaming.StreamingContext) 或 [`JavaStreamingContext.stop(...)`](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html) 以获取正常的关闭选项）, 以确保已关闭的数据在关闭之前被完全处理.然后可以启动升级的应用程序, 这将从较早的应用程序停止的同一点开始处理.请注意, 只有在支持源端缓冲的输入源（如: Kafka 和 Flume）时才可以进行此操作, 因为数据需要在先前的应用程序关闭并且升级的应用程序尚未启动时进行缓冲.从升级前代码的早期 checkpoint 信息重新启动不能完成.checkpoint 信息基本上包含序列化的 Scala/Java/Python 对象, 并尝试使用新的修改的类反序列化对象可能会导致错误.在这种情况下, 可以使用不同的 checkpoint 目录启动升级的应用程序, 也可以删除以前的 checkpoint 目录.

***

## Monitoring Applications （监控应用程序）
除了 Spark 的 [monitoring capabilities（监控功能）](monitoring.html) , 还有其他功能特定于 Spark Streaming .当使用 StreamingContext 时, [Spark web UI](monitoring.html#web-interfaces) 显示一个额外的 `Streaming` 选项卡, 显示 running receivers （运行接收器）的统计信息（无论是 receivers （接收器）是否处于 active （活动状态）, 接收到的 records （记录）数,  receiver error （接收器错误）等）并完成 batches （批次）（batch processing times （批处理时间）,  queueing delays （排队延迟）等）.这可以用来监视 streaming application （流应用程序）的进度.

web UI 中的以下两个 metrics （指标）特别重要:

- *Processing Time （处理时间）* - 处理每 batch （批）数据的时间 .
- *Scheduling Delay （调度延迟）* - batch （批处理）在 queue （队列）中等待处理 previous batches （以前批次）完成的时间.

如果 batch processing time （批处理时间）始终 more than （超过） batch interval （批间隔） and/or queueing delay （排队延迟）不断增加, 表示系统是无法快速 process the batches （处理批次）, 并且正在 falling behind （落后）.
在这种情况下, 请考虑 [reducing （减少）](#reducing-the-batch-processing-times) batch processing time （批处理时间）.

Spark Streaming 程序的进展也可以使用 [StreamingListener](api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener) 接口, 这允许您获得 receiver status （接收器状态）和 processing times （处理时间）.请注意, 这是一个开发人员 API 并且将来可能会改善（即, 更多的信息报告）.

***************************************************************************************************
***************************************************************************************************

# Performance Tuning （性能调优）
在集群上的 Spark Streaming application 中获得最佳性能需要一些调整.本节介绍了可调整的多个 parameters （参数）和 configurations （配置）提高你的应用程序性能.在高层次上, 你需要考虑两件事情:

1. 通过有效利用集群资源,  Reducing the processing time of each batch of data （减少每批数据的处理时间）.

2. 设置正确的 batch size （批量大小）, 以便 batches of data （批量的数据）可以像 received （被接收）处理一样快（即 data processing （数据处理）与 data ingestion （数据摄取）保持一致）.

## Reducing the Batch Processing Times （减少批处理时间）
在 Spark 中可以进行一些优化, 以 minimize the processing time of
each batch （最小化每批处理时间）.这些已在 [Tuning Guide （调优指南）](tuning.html) 中详细讨论过.本节突出了一些最重要的.

### Level of Parallelism in Data Receiving （数据接收中的并行级别）
{:.no_toc}
通过网络接收数据（如Kafka, Flume, socket 等）需要 deserialized （反序列化）数据并存储在 Spark 中.如果数据接收成为系统的瓶颈, 那么考虑一下 parallelizing the data receiving （并行化数据接收）.注意每个 input DStream 创建接收 single stream of data （单个数据流）的 single receiver （单个接收器）（在 work machine 上运行）.
因此, 可以通过创建多个 input DStreams 来实现 Receiving multiple data streams （接收多个数据流）并配置它们以从 source(s) 接收 data stream （数据流）的 different partitions （不同分区）.例如, 接收 two topics of data （两个数据主题）的单个Kafka input DStream 可以分为两个 Kafka input streams （输入流）, 每个只接收一个 topic （主题）.这将运行两个 receivers （接收器）, 允许 in parallel （并行）接收数据, 从而提高 overall throughput （总体吞吐量）.这些 multiple
DStreams 可以 unioned （联合起来）创建一个 single DStream .然后 transformations （转化）为应用于 single input DStream 可以应用于 unified stream .如下这样做.

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
int numStreams = 5;
List<JavaPairDStream<String, String>> kafkaStreams = new ArrayList<>(numStreams);
for (int i = 0; i < numStreams; i++) {
  kafkaStreams.add(KafkaUtils.createStream(...));
}
JavaPairDStream<String, String> unifiedStream = streamingContext.union(kafkaStreams.get(0), kafkaStreams.subList(1, kafkaStreams.size()));
unifiedStream.print();
{% endhighlight %}
</div>
<div data-lang="python" markdown="1">
{% highlight python %}
numStreams = 5
kafkaStreams = [KafkaUtils.createStream(...) for _ in range (numStreams)]
unifiedStream = streamingContext.union(*kafkaStreams)
unifiedStream.pprint()
{% endhighlight %}
</div>
</div>

应考虑的另一个参数是 receiver's block interval （接收器的块间隔）, 这由[configuration parameter （配置参数）](configuration.html#spark-streaming) 的 `spark.streaming.blockInterval` 决定.对于大多数 receivers （接收器）, 接收到的数据 coalesced （合并）在一起存储在 Spark 内存之前的 blocks of data （数据块）.每个 batch （批次）中的 blocks （块）数确定将用于处理接收到的数据以 map-like （类似与 map 形式的） transformation （转换）的 task （任务）的数量.每个 receiver （接收器）每 batch （批次）的任务数量将是大约（ batch interval （批间隔）/ block interval （块间隔））.例如, 200 ms的 block interval （块间隔）每 2 秒 batches （批次）创建 10 个 tasks （任务）.如果 tasks （任务）数量太少（即少于每个机器的内核数量）, 那么它将无效, 因为所有可用的内核都不会被使用处理数据.要增加 given batch interval （给定批间隔）的 tasks （任务）数量, 请减少 block interval （块间​​隔）.但是, 推荐的 block interval （块间隔）最小值约为 50ms , 低于此任务启动开销可能是一个问题.

使用 multiple input streams （多个输入流）/ receivers （接收器）接收数据的替代方法是明确 repartition （重新分配） input data stream （输入数据流）（使用 `inputStream.repartition(<number of partitions>)` ）.
这会在 further processing （进一步处理）之前将 received batches of data （收到的批次数据） distributes （分发）到集群中指定数量的计算机.

### Level of Parallelism in Data Processing （数据处理中的并行度水平）
{:.no_toc}
如果在任何 computation （计算）阶段中使用 number of parallel tasks （并行任务的数量）, 则 Cluster resources （集群资源）可能未得到充分利用. 例如, 对于 distributed reduce （分布式 reduce）操作, 如 `reduceByKey` 和 `reduceByKeyAndWindow` , 默认并行任务的数量由 `spark.default.parallelism` [configuration property](configuration.html#spark-properties) 控制. 您
可以通过 parallelism （并行度）作为参数（见 [`PairDStreamFunctions`](api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions)
 文档 ）, 或设置 `spark.default.parallelism`
[configuration property](configuration.html#spark-properties) 更改默认值.

### Data Serialization （数据序列化）
{:.no_toc}
可以通过调优 serialization formats （序列化格式）来减少数据 serialization （序列化）的开销.在 streaming 的情况下, 有两种类型的数据被 serialized （序列化）.

* **Input data （输入数据）**: 默认情况下, 通过 Receivers 接收的 input data （输入数据）通过 [StorageLevel.MEMORY_AND_DISK_SER_2](api/scala/index.html#org.apache.spark.storage.StorageLevel$) 存储在 executors 的内存中.也就是说, 将数据 serialized （序列化）为 bytes （字节）以减少 GC 开销, 并复制以容忍 executor failures （执行器故障）.此外, 数据首先保留在内存中, 并且只有在内存不足以容纳 streaming computation （流计算）所需的所有输入数据时才会 spilled over （溢出）到磁盘.这个 serialization （序列化）显然具有开销 - receiver （接收器）必须使接收的数据 deserialize （反序列化）, 并使用 Spark 的 serialization format （序列化格式）重新序列化它.

* **Persisted RDDs generated by Streaming Operations （流式操作生成的持久 RDDs）**: 通过 streaming computations （流式计算）生成的 RDD 可能会持久存储在内存中.例如,  window operations （窗口操作）会将数据保留在内存中, 因为它们将被处理多次.但是, 与 [StorageLevel.MEMORY_ONLY](api/scala/index.html#org.apache.spark.storage.StorageLevel$) 的 Spark Core 默认情况不同, 通过流式计算生成的持久化 RDD 将以 [StorageLevel.MEMORY_ONLY_SER](api/scala/index.html#org.apache.spark.storage.StorageLevel$) （即序列化）, 以最小化 GC 开销.

在这两种情况下, 使用 Kryo serialization （Kryo 序列化）可以减少 CPU 和内存开销.有关详细信息, 请参阅 [Spark Tuning Guide](tuning.html#data-serialization) .对于 Kryo , 请考虑 registering custom classes , 并禁用对象引用跟踪（请参阅 [Configuration Guide](configuration.html#compression-and-serialization) 中的 Kryo 相关配置）.

在 streaming application 需要保留的数据量不大的特定情况下, 可以将数据（两种类型）作为 deserialized objects （反序列化对象）持久化, 而不会导致过多的 GC 开销.例如, 如果您使用几秒钟的 batch intervals （批次间隔）并且没有 window operations （窗口操作）, 那么可以通过明确地相应地设置 storage level （存储级别）来尝试禁用 serialization in persisted data （持久化数据中的序列化）.这将减少由于序列化造成的 CPU 开销, 潜在地提高性能, 而不需要太多的 GC 开销.

### Task Launching Overheads （任务启动开销）
{:.no_toc}
如果每秒启动的任务数量很高（比如每秒 50 个或更多）, 那么这个开销向 slaves 发送任务可能是重要的, 并且将难以实现 sub-second latencies （次要的延迟）.可以通过以下更改减少开销:

* **Execution mode （执行模式）**: 以 Standalone mode （独立模式）或 coarse-grained Mesos 模式运行 Spark 比 fine-grained Mesos 模式更好的任务启动时间.有关详细信息, 请参阅 [Running on Mesos guide](running-on-mesos.html) .

这些更改可能会将 batch processing time （批处理时间）缩短 100 毫秒, 从而允许 sub-second batch size （次秒批次大小）是可行的.

***

## Setting the Right Batch Interval （设置正确的批次间隔）
对于在集群上稳定地运行的 Spark Streaming application, 该系统应该能够处理数据尽可能快地被接收.换句话说, 应该处理批次的数据就像生成它们一样快.这是否适用于 application 可以在 [monitoring](#monitoring-applications) streaming web UI 中的 processing times 中被找到,  processing time （批处理处理时间）应小于 batch interval （批间隔）.

取决于 streaming computation （流式计算）的性质, 使用的 batch interval （批次间隔）可能对处理由应用程序持续一组固定的 cluster resources （集群资源）的数据速率有重大的影响.例如, 让我们考虑早期的 WordCountNetwork 示例.对于特定的 data rate （数据速率）, 系统可能能够跟踪每 2 秒报告 word counts （即 2 秒的 batch interval （批次间隔））, 但不能每 500 毫秒.因此, 需要设置 batch interval （批次间隔）, 使预期的数据速率在生产可以持续.

为您的应用程序找出正确的 batch size （批量大小）的一个好方法是使用进行测试 conservative batch interval （保守的批次间隔）（例如 5-10 秒）和 low data rate （低数据速率）.验证是否系统能够跟上 data rate （数据速率）, 可以检查遇到的 end-to-end delay （端到端延迟）的值通过每个 processed batch （处理的批次）（在 Spark driver log4j 日志中查找 "Total delay" , 或使用 [StreamingListener](api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener)
 接口）.
如果 delay （延迟）保持与 batch size （批量大小）相当, 那么系统是稳定的.除此以外, 如果延迟不断增加, 则意味着系统无法跟上, 因此不稳定.一旦你有一个 stable configuration （稳定的配置）的想法, 你可以尝试增加 data rate and/or 减少 batch size .请注意,  momentary increase （瞬时增加）由于延迟暂时增加只要延迟降低到 low value （低值）, 临时数据速率增加就可以很好（即, 小于 batch size （批量大小））.

***

## Memory Tuning （内存调优）
调整 Spark 应用程序的内存使用情况和 GC behavior 已经有很多的讨论在 [Tuning Guide](tuning.html#memory-tuning) 中.我们强烈建议您阅读一下.在本节中, 我们将在 Spark Streaming applications 的上下文中讨论一些 tuning parameters （调优参数）.

Spark Streaming application 所需的集群内存量在很大程度上取决于所使用的 transformations 类型.例如, 如果要在最近 10 分钟的数据中使用 window operation （窗口操作）, 那么您的集群应该有足够的内存来容纳内存中 10 分钟的数据.或者如果要使用大量 keys 的 `updateStateByKey` , 那么必要的内存将会很高.相反, 如果你想做一个简单的 map-filter-store 操作, 那么所需的内存就会很低.

一般来说, 由于通过 receivers （接收器）接收的数据与 StorageLevel.MEMORY_AND_DISK_SER_2 一起存储, 所以不适合内存的数据将会 spill over （溢出）到磁盘上.这可能会降低 streaming application （流式应用程序）的性能, 因此建议您提供足够的 streaming application （流量应用程序）所需的内存.最好仔细查看内存使用量并相应地进行估算. 

memory tuning （内存调优）的另一个方面是 garbage collection （垃圾收集）.对于需要低延迟的 streaming application , 由 JVM Garbage Collection 引起的大量暂停是不希望的.

有几个 parameters （参数）可以帮助您调整 memory usage （内存使用量）和 GC 开销:

* **Persistence Level of DStreams （DStreams 的持久性级别）**: 如前面在 [Data Serialization](#data-serialization) 部分中所述,  input data 和 RDD 默认保持为 serialized bytes （序列化字节）.与 deserialized persistence （反序列化持久性）相比, 这减少了内存使用量和 GC 开销.启用 Kryo serialization 进一步减少了 serialized sizes （序列化大小）和 memory usage （内存使用）.可以通过 compression （压缩）来实现内存使用的进一步减少（参见Spark配置 `spark.rdd.compress` ）, 代价是 CPU 时间.

* **Clearing old data （清除旧数据）**: 默认情况下, DStream 转换生成的所有 input data 和 persisted RDDs 将自动清除. Spark Streaming 决定何时根据所使用的 transformations （转换）来清除数据.例如, 如果您使用 10 分钟的 window operation （窗口操作）, 则 Spark Streaming 将保留最近 10 分钟的数据, 并主动丢弃旧数据.
数据可以通过设置 `streamingContext.remember` 保持更长的持续时间（例如交互式查询旧数据）.

* **CMS Garbage Collector （CMS垃圾收集器）**: 强烈建议使用 concurrent mark-and-sweep GC , 以保持 GC 相关的暂停始终如一.即使 concurrent GC 已知可以减少
系统的整体处理吞吐量, 其使用仍然建议实现更多一致的 batch processing times （批处理时间）.确保在 driver （使用 `--driver-java-options` 在 `spark-submit` 中 ）和 executors （使用 [Spark configuration](configuration.html#runtime-environment) `spark.executor.extraJavaOptions` ）中设置 CMS GC.

* **Other tips （其他提示）**: 为了进一步降低 GC 开销, 以下是一些更多的提示.
    - 使用 `OFF_HEAP` 存储级别的保持 RDDs .在 [Spark Programming Guide](programming-guide.html#rdd-persistence) 中查看更多详细信息.
    - 使用更小的 heap sizes 的 executors.这将降低每个 JVM heap 内的 GC 压力.

***

##### Important points to remember（要记住的要点）:
{:.no_toc}
- DStream 与 single receiver （单个接收器）相关联.为了获得读取并行性, 需要创建多个 receivers , 即 multiple DStreams .receiver 在一个 executor 中运行.它占据一个 core （内核）.确保在 receiver slots are booked 后有足够的内核进行处理, 即 `spark.cores.max` 应该考虑 receiver slots . receivers 以循环方式分配给 executors .

- 当从 stream source 接收到数据时, receiver 创建数据 blocks （块）.每个 blockInterval 毫秒生成一个新的数据块.在  N = batchInterval/blockInterval 的 batchInterval 期间创建 N 个数据块.这些块由当前 executor 的 BlockManager 分发给其他执行程序的 block managers .之后, 在驱动程序上运行的 Network Input Tracker （网络输入跟踪器）通知有关进一步处理的块位置

- 在驱动程序中为在 batchInterval 期间创建的块创建一个 RDD .在 batchInterval 期间生成的块是 RDD 的 partitions .每个分区都是一个 spark 中的 task. blockInterval == batchinterval 意味着创建 single partition （单个分区）, 并且可能在本地进行处理.

- 除非 non-local scheduling （非本地调度）进行, 否则块上的 map tasks （映射任务）将在 executors （接收 block, 复制块的另一个块）中进行处理.具有更大的 block interval （块间隔）意味着更大的块. `spark.locality.wait` 的高值增加了处理 local node （本地节点）上的块的机会.需要在这两个参数之间找到平衡, 以确保在本地处理较大的块.

- 而不是依赖于 batchInterval 和 blockInterval , 您可以通过调用 `inputDstream.repartition(n)` 来定义 number of partitions （分区数）.这样可以随机重新组合 RDD 中的数据, 创建 n 个分区.是的, 为了更大的 parallelism （并行性）.虽然是 shuffle 的代价. RDD 的处理由 driver's jobscheduler 作为一项工作安排.在给定的时间点, 只有一个 job 是 active 的.因此, 如果一个作业正在执行, 则其他作业将排队.

- 如果您有两个 dstream , 将会有两个 RDD 形成, 并且将创建两个将被安排在另一个之后的作业.为了避免这种情况, 你可以联合两个 dstream .这将确保为 dstream 的两个 RDD 形成一个 unionRDD .这个 unionRDD 然后被认为是一个 single job （单一的工作）.但 RDD 的 partitioning （分区）不受影响.

- 如果 batch processing time （批处理时间）超过 batchinterval （批次间隔）, 那么显然 receiver 的内存将会开始填满, 最终会抛出 exceptions （最可能是 BlockNotFoundException ）.目前没有办法暂停 receiver .使用 SparkConf 配置 `spark.streaming.receiver.maxRate` ,  receiver 的 rate 可以受到限制.


***************************************************************************************************
***************************************************************************************************

# Fault-tolerance Semantics （容错语义）
在本节中, 我们将讨论 Spark Streaming applications 在该 event 中的行为的失败.

## Background（背景）
{:.no_toc}
要了解 Spark Streaming 提供的语义, 请记住 Spark 的 RDD 的基本 fault-tolerance semantics （容错语义）.

1. RDD 是一个不可变的, 确定性地可重新计算的分布式数据集.每个RDD
记住在容错输入中使用的确定性操作的 lineage 数据集创建它.
1. 如果 RDD 的任何 partition 由于工作节点故障而丢失, 则该分区可以是
从 original fault-tolerant dataset （原始容错数据集）中使用业务流程重新计算.
1. 假设所有的 RDD transformations 都是确定性的, 最后的数据被转换, 无论 Spark 集群中的故障如何, RDD 始终是一样的.

Spark 运行在容错文件系统（如 HDFS 或 S3 ）中的数据上.因此, 从容错数据生成的所有 RDD 也都是容错的.但是, 这不是在大多数情况下, Spark Streaming 作为数据的情况通过网络接收（除非 `fileStream` 被使用）.为了为所有生成的 RDD 实现相同的 fault-tolerance properties （容错属性）, 接收的数据在集群中的工作节点中的多个 Spark executors 之间进行复制（默认 replication factor （备份因子）为 2）.这导致了发生故障时需要恢复的系统中的两种数据:

1. *Data received and replicated （数据接收和复制）* - 这个数据在单个工作节点作为副本的故障中幸存下来, 它存在于其他节点之一上.
1. *Data received but buffered for replication （接收数据但缓冲进行复制）* - 由于不复制, 恢复此数据的唯一方法是从 source 重新获取.

此外, 我们应该关注的有两种 failures:

1. *Failure of a Worker Node （工作节点的故障）* - 运行 executors 的任何工作节点都可能会故障, 并且这些节点上的所有内存中数据将丢失.如果任何 receivers 运行在失败节点, 则它们的 buffered （缓冲）数据将丢失.
1. *Failure of the Driver Node （Driver 节点的故障）* -  如果运行 Spark Streaming application 的 driver node 发生了故障, 那么显然 SparkContext 丢失了, 所有的 executors 和其内存中的数据也一起丢失了.

有了这个基础知识, 让我们了解 Spark Streaming 的 fault-tolerance semantics （容错语义）.

## Definitions （定义）
{:.no_toc}
streaming systems （流系统）的语义通常是通过系统可以处理每个记录的次数来捕获的.系统可以在所有可能的操作条件下提供三种类型的保证（尽管有故障等）.

1. *At most once （最多一次）*: 每个 record （记录）将被处理一次或根本不处理.
2. *At least once （至少一次）*: 每个 record （记录）将被处理一次或多次.这比*at-most once*, 因为它确保没有数据将丢失.但可能有重复.
3. *Exactly once（有且仅一次）*: 每个 record （记录） 将被精确处理一次 - 没有数据丢失, 数据不会被多次处理.这显然是三者的最强保证.

## Basic Semantics （基本语义）
{:.no_toc}
在任何 stream processing system （流处理系统）中, 广义上说, 处理数据有三个步骤.

1. *Receiving the data （接收数据）*: 使用 Receivers 或其他方式从数据源接收数据.

1. *Transforming the data （转换数据）*: 使用 DStream 和 RDD transformations 来 transformed （转换）接收到的数据.

1. *Pushing out the data （推出数据）*: 最终的转换数据被推出到 external systems （外部系统）, 如 file systems （文件系统）, databases （数据库）, dashboards （仪表板）等.

如果 streaming application 必须实现 end-to-end exactly-once guarantees （端到端的一次且仅一次性保证）, 那么每个步骤都必须提供 exactly-once guarantee .也就是说, 每个记录必须被精确地接收一次, 转换完成一次, 并被推送到下游系统一次.让我们在 Spark Streaming 的上下文中了解这些步骤的语义.

1. *Receiving the data （接收数据）*: 不同的 input sources 提供不同的保证.这将在下一小节中详细讨论.

1. *Transforming the data （转换数据）*: 所有已收到的数据都将被处理 _exactly once_ , 这得益于 RDD 提供的保证.即使存在故障, 只要接收到的输入数据可访问, 最终变换的 RDD 将始终具有相同的内容.

1. *Pushing out the data （推出数据）*: 默认情况下的输出操作确保  _at-least once_ 语义, 因为它取决于输出操作的类型（ idempotent （幂等））或 downstream system （下游系统）的语义（是否支持 transactions （事务））.但用户可以实现自己的事务机制来实现 _exactly-once_ 语义.这将在本节后面的更多细节中讨论.

## Semantics of Received Data （接收数据的语义）
{:.no_toc}
不同的 input sources （输入源）提供不同的保证, 范围从 _at-least once_ 到 _exactly once_ .

### With Files
{:.no_toc}
如果所有的 input data （输入数据）都已经存在于 fault-tolerant file system （容错文件系统）中 HDFS ,  Spark Streaming 可以随时从任何故障中恢复并处理所有数据.这给了 *exactly-once* 语义, 意味着无论什么故障, 所有的数据将被精确处理一次.

### With Receiver-based Sources （使用基于接收器的数据源）
{:.no_toc}
对于基于 receivers （接收器）的 input sources （输入源）, 容错语义取决于故障场景和接收器的类型.
正如我们 [earlier](#receiver-reliability) 讨论的, 有两种类型的 receivers （接收器）:

1. *Reliable Receiver （可靠的接收器）* - 这些 receivers （接收机）只有在确认收到的数据已被复制之后确认 reliable sources （可靠的源）.如果这样的接收器出现故障, source 将不会被接收对于 buffered (unreplicated) data （缓冲（未复制）数据）的确认.因此, 如果 receiver 是重新启动,  source 将重新发送数据, 并且不会由于故障而丢失数据.
1. *Unreliable Receiver （不可靠的接收器）* - 这样的接收器 *不会* 发送确认, 因此*可能* 丢失数据, 由于 worker 或 driver 故障.

根据使用的 receivers 类型, 我们实现以下语义.
如果 worker node 出现故障, 则 reliable receivers 没有数据丢失.unreliable
receivers , 收到但未复制的数据可能会丢失.如果 driver node 失败, 那么除了这些损失之外, 在内存中接收和复制的所有过去的数据将丢失.这将影响 stateful transformations （有状态转换）的结果.

为避免过去收到的数据丢失, Spark 1.2 引入了_write ahead logs_ 将接收到的数据保存到 fault-tolerant storage （容错存储）.用[write ahead logs enabled](#deploying-applications) 和 reliable receivers, 数据没有丢失.在语义方面, 它提供 at-least once guarantee （至少一次保证）.

下表总结了失败的语义:

<table class="table">
  <tr>
    <th style="width:30%">Deployment Scenario （部署场景）</th>
    <th>Worker Failure （Worker 故障）</th>
    <th>Driver Failure （Driver 故障）</th>
  </tr>
  <tr>
    <td>
      <i>Spark 1.1 或更早版本,</i> 或者<br/>
      <i>Spark 1.2 或者没有 write ahead logs 的更高的版本</i>
    </td>
    <td>
      Buffered data lost with unreliable receivers（unreliable receivers 的缓冲数据丢失）<br/>
      Zero data loss with reliable receivers （reliable receivers 的零数据丢失）<br/>
      At-least once semantics （至少一次性语义）
    </td>
    <td>
      Buffered data lost with unreliable receivers （unreliable receivers 的缓冲数据丢失）<br/>
      Past data lost with all receivers （所有的 receivers 的过去的数据丢失）<br/>
      Undefined semantics （未定义语义）
    </td>
  </tr>
  <tr>
    <td><i>Spark 1.2 或者带有 write ahead logs 的更高版本</i></td>
    <td>
        Zero data loss with reliable receivers（reliable receivers 的零数据丢失）<br/>
        At-least once semantics （至少一次性语义）
    </td>
    <td>
        Zero data loss with reliable receivers and files （reliable receivers 和 files 的零数据丢失）<br/>
        At-least once semantics （至少一次性语义）
    </td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

### With Kafka Direct API （使用 Kafka Direct API）
{:.no_toc}
在 Spark 1.3 中, 我们引入了一个新的 Kafka Direct API , 可以确保所有的 Kafka 数据都被 Spark Streaming exactly once （一次）接收.与此同时, 如果您实现 exactly-once output operation （一次性输出操作）, 您可以实现 end-to-end exactly-once guarantees （端到端的一次且仅一次性保证）.在 [Kafka Integration Guide](streaming-kafka-integration.html) 中进一步讨论了这种方法.

## Semantics of output operations （输出操作的语义）
{:.no_toc}
Output operations （输出操作）（如 `foreachRDD` ）具有 _at-least once_ 语义, 也就是说,  transformed data （变换后的数据）可能会不止一次写入 external entity （外部实体）在一个 worker 故障事件中.虽然这是可以接受的使用 `saveAs***Files`操作（因为文件将被相同的数据简单地覆盖） 保存到文件系统, 可能需要额外的努力来实现 exactly-once （一次且仅一次）语义.有两种方法.

- *Idempotent updates （幂等更新）*: 多次尝试总是写入相同的数据.例如,  `saveAs***Files` 总是将相同的数据写入生成的文件.

- *Transactional updates （事务更新）*: 所有更新都是事务性的, 以便更新完全按原子进行.这样做的一个方法如下.

    - 使用批处理时间（在 `foreachRDD` 中可用）和 RDD 的 partition index （分区索引）来创建 identifier （标识符）.该标识符唯一地标识 streaming application 中的 blob 数据.
    - 使用该 identifier （标识符）blob transactionally （blob 事务地）更新 external system （外部系统）（即, exactly once, atomically （一次且仅一次, 原子性地））.也就是说, 如果 identifier （标识符）尚未提交, 则以 atomically （原子方式）提交 partition data （分区数据）和 identifier （标识符）.否则, 如果已经提交, 请跳过更新.

          dstream.foreachRDD { (rdd, time) =>
            rdd.foreachPartition { partitionIterator =>
              val partitionId = TaskContext.get.partitionId()
              val uniqueId = generateUniqueId(time.milliseconds, partitionId)
              // use this uniqueId to transactionally commit the data in partitionIterator
            }
          }

***************************************************************************************************
***************************************************************************************************

# 快速链接
* 附加指南
    - [Kafka 集成指南](streaming-kafka-integration.html)
    - [Kinesis 集成指南](streaming-kinesis-integration.html)
    - [自定义 Receiver（接收器）指南](streaming-custom-receivers.html)
* 第三方 DStream 数据源可以在 [第三方项目](http://spark.apache.org/third-party-projects.html) 上查看.
* API 文档
  - Scala 文档
    * [StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) 和
  [DStream](api/scala/index.html#org.apache.spark.streaming.dstream.DStream)
    * [KafkaUtils](api/scala/index.html#org.apache.spark.streaming.kafka.KafkaUtils$),
    [FlumeUtils](api/scala/index.html#org.apache.spark.streaming.flume.FlumeUtils$),
    [KinesisUtils](api/scala/index.html#org.apache.spark.streaming.kinesis.KinesisUtils$),
  - Java 文档
    * [JavaStreamingContext](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html),
    [JavaDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html) 和
    [JavaPairDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html)
    * [KafkaUtils](api/java/index.html?org/apache/spark/streaming/kafka/KafkaUtils.html),
    [FlumeUtils](api/java/index.html?org/apache/spark/streaming/flume/FlumeUtils.html),
    [KinesisUtils](api/java/index.html?org/apache/spark/streaming/kinesis/KinesisUtils.html)
  - Python 文档
    * [StreamingContext](api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext) 和 [DStream](api/python/pyspark.streaming.html#pyspark.streaming.DStream)
    * [KafkaUtils](api/python/pyspark.streaming.html#pyspark.streaming.kafka.KafkaUtils)

* 更多的示例在 [Scala]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming)
  和 [Java]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/java/org/apache/spark/examples/streaming)
  和 [Python]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/python/streaming)
* 描述 Spark Streaming 的 [Paper](http://www.eecs.berkeley.edu/Pubs/TechRpts/2012/EECS-2012-259.pdf) 和 [video](http://youtu.be/g171ndOHgJ0).
