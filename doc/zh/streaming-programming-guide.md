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

For ingesting data from sources like Kafka, Flume, and Kinesis that are not present in the Spark
Streaming core
 API, you will have to add the corresponding
artifact `spark-streaming-xyz_{{site.SCALA_BINARY_VERSION}}` to the dependencies. For example,
some of the common ones are as follows.
对于从现在没有在 Spark Streaming Core API 中的数据源获取数据，如 Kafka， Flume，Kinesis ，你必须添加相应的坐标  `spark-streaming-xyz_{{site.SCALA_BINARY_VERSION}}` 到依赖中.
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
Similar to that of RDDs, transformations allow the data from the input DStream to be modified.
DStreams support many of the transformations available on normal Spark RDD's.
Some of the common ones are as follows.

与 RDD 类似，transformation 允许从 input DStream 输入的数据做修改.
DStreams 支持很多在 RDD 中可用的 transformation 算子。一些常用的算子如下所示 : 

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

The update function will be called for each word, with `newValues` having a sequence of 1's (from
the `(word, 1)` pairs) and the `runningCount` having the previous count.

更新函数将会被每个单词调用，·newValues· 拥有一系列的 1（来自 (word, 1) pairs），runningCount 拥有之前的次数.

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

## Output Operations on DStreams
Output operations allow DStream's data to be pushed out to external systems like a database or a file systems.
Since the output operations actually allow the transformed data to be consumed by external systems,
they trigger the actual execution of all the DStream transformations (similar to actions for RDDs).
Currently, the following output operations are defined:

<table class="table">
<tr><th style="width:30%">Output Operation</th><th>Meaning</th></tr>
<tr>
  <td> <b>print</b>()</td>
  <td> Prints the first ten elements of every batch of data in a DStream on the driver node running
  the streaming application. This is useful for development and debugging.
  <br/>
  <span class="badge" style="background-color: grey">Python API</span> This is called
  <b>pprint()</b> in the Python API.
  </td>
</tr>
<tr>
  <td> <b>saveAsTextFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> Save this DStream's contents as text files. The file name at each batch interval is
  generated based on <i>prefix</i> and <i>suffix</i>: <i>"prefix-TIME_IN_MS[.suffix]"</i>. </td>
</tr>
<tr>
  <td> <b>saveAsObjectFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> Save this DStream's contents as <code>SequenceFiles</code> of serialized Java objects. The file
  name at each batch interval is generated based on <i>prefix</i> and
  <i>suffix</i>: <i>"prefix-TIME_IN_MS[.suffix]"</i>.
  <br/>
  <span class="badge" style="background-color: grey">Python API</span> This is not available in
  the Python API.
  </td>
</tr>
<tr>
  <td> <b>saveAsHadoopFiles</b>(<i>prefix</i>, [<i>suffix</i>]) </td>
  <td> Save this DStream's contents as Hadoop files. The file name at each batch interval is
  generated based on <i>prefix</i> and <i>suffix</i>: <i>"prefix-TIME_IN_MS[.suffix]"</i>.
  <br>
  <span class="badge" style="background-color: grey">Python API</span> This is not available in
  the Python API.
  </td>
</tr>
<tr>
  <td> <b>foreachRDD</b>(<i>func</i>) </td>
  <td> The most generic output operator that applies a function, <i>func</i>, to each RDD generated from
  the stream. This function should push the data in each RDD to an external system, such as saving the RDD to
  files, or writing it over the network to a database. Note that the function <i>func</i> is executed
  in the driver process running the streaming application, and will usually have RDD actions in it
  that will force the computation of the streaming RDDs.</td>
</tr>
<tr><td></td><td></td></tr>
</table>

### Design Patterns for using foreachRDD
{:.no_toc}
`dstream.foreachRDD` is a powerful primitive that allows data to be sent out to external systems.
However, it is important to understand how to use this primitive correctly and efficiently.
Some of the common mistakes to avoid are as follows.

Often writing data to external system requires creating a connection object
(e.g. TCP connection to a remote server) and using it to send data to a remote system.
For this purpose, a developer may inadvertently try creating a connection object at
the Spark driver, and then try to use it in a Spark worker to save records in the RDDs.
For example (in Scala),

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

This is incorrect as this requires the connection object to be serialized and sent from the
driver to the worker. Such connection objects are rarely transferable across machines. This
error may manifest as serialization errors (connection object not serializable), initialization
errors (connection object needs to be initialized at the workers), etc. The correct solution is
to create the connection object at the worker.

However, this can lead to another common mistake - creating a new connection for every record.
For example,

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

Typically, creating a connection object has time and resource overheads. Therefore, creating and
destroying a connection object for each record can incur unnecessarily high overheads and can
significantly reduce the overall throughput of the system. A better solution is to use
`rdd.foreachPartition` - create a single connection object and send all the records in a RDD
partition using that connection.

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

  This amortizes the connection creation overheads over many records.

Finally, this can be further optimized by reusing connection objects across multiple RDDs/batches.
One can maintain a static pool of connection objects than can be reused as
RDDs of multiple batches are pushed to the external system, thus further reducing the overheads.

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

Note that the connections in the pool should be lazily created on demand and timed out if not used for a while. This achieves the most efficient sending of data to external systems.


##### Other points to remember:
{:.no_toc}
- DStreams are executed lazily by the output operations, just like RDDs are lazily executed by RDD actions. Specifically, RDD actions inside the DStream output operations force the processing of the received data. Hence, if your application does not have any output operation, or has output operations like `dstream.foreachRDD()` without any RDD action inside them, then nothing will get executed. The system will simply receive the data and discard it.

- By default, output operations are executed one-at-a-time. And they are executed in the order they are defined in the application.

***

## DataFrame and SQL Operations
You can easily use [DataFrames and SQL](sql-programming-guide.html) operations on streaming data. You have to create a SparkSession using the SparkContext that the StreamingContext is using. Furthermore this has to done such that it can be restarted on driver failures. This is done by creating a lazily instantiated singleton instance of SparkSession. This is shown in the following example. It modifies the earlier [word count example](#a-quick-example) to generate word counts using DataFrames and SQL. Each RDD is converted to a DataFrame, registered as a temporary table and then queried using SQL.

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

See the full [source code]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/SqlNetworkWordCount.scala).
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

See the full [source code]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming/JavaSqlNetworkWordCount.java).
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

See the full [source code]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/sql_network_wordcount.py).

</div>
</div>

You can also run SQL queries on tables defined on streaming data from a different thread (that is, asynchronous to the running StreamingContext). Just make sure that you set the StreamingContext to remember a sufficient amount of streaming data such that the query can run. Otherwise the StreamingContext, which is unaware of the any asynchronous SQL queries, will delete off old streaming data before the query can complete. For example, if you want to query the last batch, but your query can take 5 minutes to run, then call `streamingContext.remember(Minutes(5))` (in Scala, or equivalent in other languages).

See the [DataFrames and SQL](sql-programming-guide.html) guide to learn more about DataFrames.

***

## MLlib Operations
You can also easily use machine learning algorithms provided by [MLlib](ml-guide.html). First of all, there are streaming machine learning algorithms (e.g. [Streaming Linear Regression](mllib-linear-methods.html#streaming-linear-regression), [Streaming KMeans](mllib-clustering.html#streaming-k-means), etc.) which can simultaneously learn from the streaming data as well as apply the model on the streaming data. Beyond these, for a much larger class of machine learning algorithms, you can learn a learning model offline (i.e. using historical data) and then apply the model online on streaming data. See the [MLlib](ml-guide.html) guide for more details.

***

## Caching / Persistence
Similar to RDDs, DStreams also allow developers to persist the stream's data in memory. That is,
using the `persist()` method on a DStream will automatically persist every RDD of that DStream in
memory. This is useful if the data in the DStream will be computed multiple times (e.g., multiple
operations on the same data). For window-based operations like `reduceByWindow` and
`reduceByKeyAndWindow` and state-based operations like `updateStateByKey`, this is implicitly true.
Hence, DStreams generated by window-based operations are automatically persisted in memory, without
the developer calling `persist()`.

For input streams that receive data over the network (such as, Kafka, Flume, sockets, etc.), the
default persistence level is set to replicate the data to two nodes for fault-tolerance.

Note that, unlike RDDs, the default persistence level of DStreams keeps the data serialized in
memory. This is further discussed in the [Performance Tuning](#memory-tuning) section. More
information on different persistence levels can be found in the [Spark Programming Guide](programming-guide.html#rdd-persistence).

***

## Checkpointing
A streaming application must operate 24/7 and hence must be resilient to failures unrelated
to the application logic (e.g., system failures, JVM crashes, etc.). For this to be possible,
Spark Streaming needs to *checkpoint* enough information to a fault-
tolerant storage system such that it can recover from failures. There are two types of data
that are checkpointed.

- *Metadata checkpointing* - Saving of the information defining the streaming computation to
  fault-tolerant storage like HDFS. This is used to recover from failure of the node running the
  driver of the streaming application (discussed in detail later). Metadata includes:
  +  *Configuration* - The configuration that was used to create the streaming application.
  +  *DStream operations* - The set of DStream operations that define the streaming application.
  +  *Incomplete batches* - Batches whose jobs are queued but have not completed yet.
- *Data checkpointing* - Saving of the generated RDDs to reliable storage. This is necessary
  in some *stateful* transformations that combine data across multiple batches. In such
  transformations, the generated RDDs depend on RDDs of previous batches, which causes the length
  of the dependency chain to keep increasing with time. To avoid such unbounded increases in recovery
   time (proportional to dependency chain), intermediate RDDs of stateful transformations are periodically
  *checkpointed* to reliable storage (e.g. HDFS) to cut off the dependency chains.

To summarize, metadata checkpointing is primarily needed for recovery from driver failures,
whereas data or RDD checkpointing is necessary even for basic functioning if stateful
transformations are used.

#### When to enable Checkpointing
{:.no_toc}

Checkpointing must be enabled for applications with any of the following requirements:

- *Usage of stateful transformations* - If either `updateStateByKey` or `reduceByKeyAndWindow` (with
  inverse function) is used in the application, then the checkpoint directory must be provided to
  allow for periodic RDD checkpointing.
- *Recovering from failures of the driver running the application* - Metadata checkpoints are used
   to recover with progress information.

Note that simple streaming applications without the aforementioned stateful transformations can be
run without enabling checkpointing. The recovery from driver failures will also be partial in
that case (some received but unprocessed data may be lost). This is often acceptable and many run
Spark Streaming applications in this way. Support for non-Hadoop environments is expected
to improve in the future.

#### How to configure Checkpointing
{:.no_toc}

Checkpointing can be enabled by setting a directory in a fault-tolerant,
reliable file system (e.g., HDFS, S3, etc.) to which the checkpoint information will be saved.
This is done by using `streamingContext.checkpoint(checkpointDirectory)`. This will allow you to
use the aforementioned stateful transformations. Additionally,
if you want to make the application recover from driver failures, you should rewrite your
streaming application to have the following behavior.

  + When the program is being started for the first time, it will create a new StreamingContext,
    set up all the streams and then call start().
  + When the program is being restarted after failure, it will re-create a StreamingContext
    from the checkpoint data in the checkpoint directory.

<div class="codetabs">
<div data-lang="scala" markdown="1">

This behavior is made simple by using `StreamingContext.getOrCreate`. This is used as follows.

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

In addition to using `getOrCreate` one also needs to ensure that the driver process gets
restarted automatically on failure. This can only be done by the deployment infrastructure that is
used to run the application. This is further discussed in the
[Deployment](#deploying-applications) section.

Note that checkpointing of RDDs incurs the cost of saving to reliable storage.
This may cause an increase in the processing time of those batches where RDDs get checkpointed.
Hence, the interval of
checkpointing needs to be set carefully. At small batch sizes (say 1 second), checkpointing every
batch may significantly reduce operation throughput. Conversely, checkpointing too infrequently
causes the lineage and task sizes to grow, which may have detrimental effects. For stateful
transformations that require RDD checkpointing, the default interval is a multiple of the
batch interval that is at least 10 seconds. It can be set by using
`dstream.checkpoint(checkpointInterval)`. Typically, a checkpoint interval of 5 - 10 sliding intervals of a DStream is a good setting to try.

***

## Accumulators, Broadcast Variables, and Checkpoints

[Accumulators](programming-guide.html#accumulators) and [Broadcast variables](programming-guide.html#broadcast-variables) cannot be recovered from checkpoint in Spark Streaming. If you enable checkpointing and use [Accumulators](programming-guide.html#accumulators) or [Broadcast variables](programming-guide.html#broadcast-variables) as well, you'll have to create lazily instantiated singleton instances for [Accumulators](programming-guide.html#accumulators) and [Broadcast variables](programming-guide.html#broadcast-variables) so that they can be re-instantiated after the driver restarts on failure. This is shown in the following example.

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

See the full [source code]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala).
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

See the full [source code]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming/JavaRecoverableNetworkWordCount.java).
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

See the full [source code]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/recoverable_network_wordcount.py).

</div>
</div>

***

## Deploying Applications
This section discusses the steps to deploy a Spark Streaming application.

### Requirements
{:.no_toc}

To run a Spark Streaming applications, you need to have the following.

- *Cluster with a cluster manager* - This is the general requirement of any Spark application,
  and discussed in detail in the [deployment guide](cluster-overview.html).

- *Package the application JAR* - You have to compile your streaming application into a JAR.
  If you are using [`spark-submit`](submitting-applications.html) to start the
  application, then you will not need to provide Spark and Spark Streaming in the JAR. However,
  if your application uses [advanced sources](#advanced-sources) (e.g. Kafka, Flume),
  then you will have to package the extra artifact they link to, along with their dependencies,
  in the JAR that is used to deploy the application. For example, an application using `KafkaUtils`
  will have to include `spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}}` and all its
  transitive dependencies in the application JAR.

- *Configuring sufficient memory for the executors* - Since the received data must be stored in
  memory, the executors must be configured with sufficient memory to hold the received data. Note
  that if you are doing 10 minute window operations, the system has to keep at least last 10 minutes
  of data in memory. So the memory requirements for the application depends on the operations
  used in it.

- *Configuring checkpointing* - If the stream application requires it, then a directory in the
  Hadoop API compatible fault-tolerant storage (e.g. HDFS, S3, etc.) must be configured as the
  checkpoint directory and the streaming application written in a way that checkpoint
  information can be used for failure recovery. See the [checkpointing](#checkpointing) section
  for more details.

- *Configuring automatic restart of the application driver* - To automatically recover from a
  driver failure, the deployment infrastructure that is
  used to run the streaming application must monitor the driver process and relaunch the driver
  if it fails. Different [cluster managers](cluster-overview.html#cluster-manager-types)
  have different tools to achieve this.
    + *Spark Standalone* - A Spark application driver can be submitted to run within the Spark
      Standalone cluster (see
      [cluster deploy mode](spark-standalone.html#launching-spark-applications)), that is, the
      application driver itself runs on one of the worker nodes. Furthermore, the
      Standalone cluster manager can be instructed to *supervise* the driver,
      and relaunch it if the driver fails either due to non-zero exit code,
      or due to failure of the node running the driver. See *cluster mode* and *supervise* in the
      [Spark Standalone guide](spark-standalone.html) for more details.
    + *YARN* - Yarn supports a similar mechanism for automatically restarting an application.
      Please refer to YARN documentation for more details.
    + *Mesos* - [Marathon](https://github.com/mesosphere/marathon) has been used to achieve this
      with Mesos.

- *Configuring write ahead logs* - Since Spark 1.2,
  we have introduced _write ahead logs_ for achieving strong
  fault-tolerance guarantees. If enabled,  all the data received from a receiver gets written into
  a write ahead log in the configuration checkpoint directory. This prevents data loss on driver
  recovery, thus ensuring zero data loss (discussed in detail in the
  [Fault-tolerance Semantics](#fault-tolerance-semantics) section). This can be enabled by setting
  the [configuration parameter](configuration.html#spark-streaming)
  `spark.streaming.receiver.writeAheadLog.enable` to `true`. However, these stronger semantics may
  come at the cost of the receiving throughput of individual receivers. This can be corrected by
  running [more receivers in parallel](#level-of-parallelism-in-data-receiving)
  to increase aggregate throughput. Additionally, it is recommended that the replication of the
  received data within Spark be disabled when the write ahead log is enabled as the log is already
  stored in a replicated storage system. This can be done by setting the storage level for the
  input stream to `StorageLevel.MEMORY_AND_DISK_SER`. While using S3 (or any file system that
  does not support flushing) for _write ahead logs_, please remember to enable
  `spark.streaming.driver.writeAheadLog.closeFileAfterWrite` and
  `spark.streaming.receiver.writeAheadLog.closeFileAfterWrite`. See
  [Spark Streaming Configuration](configuration.html#spark-streaming) for more details.
  Note that Spark will not encrypt data written to the write ahead log when I/O encryption is
  enabled. If encryption of the write ahead log data is desired, it should be stored in a file
  system that supports encryption natively.

- *Setting the max receiving rate* - If the cluster resources is not large enough for the streaming
  application to process data as fast as it is being received, the receivers can be rate limited
  by setting a maximum rate limit in terms of records / sec.
  See the [configuration parameters](configuration.html#spark-streaming)
  `spark.streaming.receiver.maxRate` for receivers and `spark.streaming.kafka.maxRatePerPartition`
  for Direct Kafka approach. In Spark 1.5, we have introduced a feature called *backpressure* that
  eliminate the need to set this rate limit, as Spark Streaming automatically figures out the
  rate limits and dynamically adjusts them if the processing conditions change. This backpressure
  can be enabled by setting the [configuration parameter](configuration.html#spark-streaming)
  `spark.streaming.backpressure.enabled` to `true`.

### Upgrading Application Code
{:.no_toc}

If a running Spark Streaming application needs to be upgraded with new
application code, then there are two possible mechanisms.

- The upgraded Spark Streaming application is started and run in parallel to the existing application.
Once the new one (receiving the same data as the old one) has been warmed up and is ready
for prime time, the old one be can be brought down. Note that this can be done for data sources that support
sending the data to two destinations (i.e., the earlier and upgraded applications).

- The existing application is shutdown gracefully (see
[`StreamingContext.stop(...)`](api/scala/index.html#org.apache.spark.streaming.StreamingContext)
or [`JavaStreamingContext.stop(...)`](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html)
for graceful shutdown options) which ensure data that has been received is completely
processed before shutdown. Then the
upgraded application can be started, which will start processing from the same point where the earlier
application left off. Note that this can be done only with input sources that support source-side buffering
(like Kafka, and Flume) as data needs to be buffered while the previous application was down and
the upgraded application is not yet up. And restarting from earlier checkpoint
information of pre-upgrade code cannot be done. The checkpoint information essentially
contains serialized Scala/Java/Python objects and trying to deserialize objects with new,
modified classes may lead to errors. In this case, either start the upgraded app with a different
checkpoint directory, or delete the previous checkpoint directory.

***

## Monitoring Applications
Beyond Spark's [monitoring capabilities](monitoring.html), there are additional capabilities
specific to Spark Streaming. When a StreamingContext is used, the
[Spark web UI](monitoring.html#web-interfaces) shows
an additional `Streaming` tab which shows statistics about running receivers (whether
receivers are active, number of records received, receiver error, etc.)
and completed batches (batch processing times, queueing delays, etc.). This can be used to
monitor the progress of the streaming application.

The following two metrics in web UI are particularly important:

- *Processing Time* - The time to process each batch of data.
- *Scheduling Delay* - the time a batch waits in a queue for the processing of previous batches
  to finish.

If the batch processing time is consistently more than the batch interval and/or the queueing
delay keeps increasing, then it indicates that the system is
not able to process the batches as fast they are being generated and is falling behind.
In that case, consider
[reducing](#reducing-the-batch-processing-times) the batch processing time.

The progress of a Spark Streaming program can also be monitored using the
[StreamingListener](api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener) interface,
which allows you to get receiver status and processing times. Note that this is a developer API
and it is likely to be improved upon (i.e., more information reported) in the future.

***************************************************************************************************
***************************************************************************************************

# Performance Tuning
Getting the best performance out of a Spark Streaming application on a cluster requires a bit of
tuning. This section explains a number of the parameters and configurations that can be tuned to
improve the performance of you application. At a high level, you need to consider two things:

1. Reducing the processing time of each batch of data by efficiently using cluster resources.

2. Setting the right batch size such that the batches of data can be processed as fast as they
  	are received (that is, data processing keeps up with the data ingestion).

## Reducing the Batch Processing Times
There are a number of optimizations that can be done in Spark to minimize the processing time of
each batch. These have been discussed in detail in the [Tuning Guide](tuning.html). This section
highlights some of the most important ones.

### Level of Parallelism in Data Receiving
{:.no_toc}
Receiving data over the network (like Kafka, Flume, socket, etc.) requires the data to be deserialized
and stored in Spark. If the data receiving becomes a bottleneck in the system, then consider
parallelizing the data receiving. Note that each input DStream
creates a single receiver (running on a worker machine) that receives a single stream of data.
Receiving multiple data streams can therefore be achieved by creating multiple input DStreams
and configuring them to receive different partitions of the data stream from the source(s).
For example, a single Kafka input DStream receiving two topics of data can be split into two
Kafka input streams, each receiving only one topic. This would run two receivers,
allowing data to be received in parallel, thus increasing overall throughput. These multiple
DStreams can be unioned together to create a single DStream. Then the transformations that were
being applied on a single input DStream can be applied on the unified stream. This is done as follows.

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

Another parameter that should be considered is the receiver's block interval,
which is determined by the [configuration parameter](configuration.html#spark-streaming)
`spark.streaming.blockInterval`. For most receivers, the received data is coalesced together into
blocks of data before storing inside Spark's memory. The number of blocks in each batch
determines the number of tasks that will be used to process 
the received data in a map-like transformation. The number of tasks per receiver per batch will be
approximately (batch interval / block interval). For example, block interval of 200 ms will
create 10 tasks per 2 second batches. If the number of tasks is too low (that is, less than the number
of cores per machine), then it will be inefficient as all available cores will not be used to
process the data. To increase the number of tasks for a given batch interval, reduce the
block interval. However, the recommended minimum value of block interval is about 50 ms,
below which the task launching overheads may be a problem.

An alternative to receiving data with multiple input streams / receivers is to explicitly repartition
the input data stream (using `inputStream.repartition(<number of partitions>)`).
This distributes the received batches of data across the specified number of machines in the cluster
before further processing.

### Level of Parallelism in Data Processing
{:.no_toc}
Cluster resources can be under-utilized if the number of parallel tasks used in any stage of the
computation is not high enough. For example, for distributed reduce operations like `reduceByKey`
and `reduceByKeyAndWindow`, the default number of parallel tasks is controlled by
the `spark.default.parallelism` [configuration property](configuration.html#spark-properties). You
can pass the level of parallelism as an argument (see
[`PairDStreamFunctions`](api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions)
documentation), or set the `spark.default.parallelism`
[configuration property](configuration.html#spark-properties) to change the default.

### Data Serialization
{:.no_toc}
The overheads of data serialization can be reduced by tuning the serialization formats. In the case of streaming, there are two types of data that are being serialized.

* **Input data**: By default, the input data received through Receivers is stored in the executors' memory with [StorageLevel.MEMORY_AND_DISK_SER_2](api/scala/index.html#org.apache.spark.storage.StorageLevel$). That is, the data is serialized into bytes to reduce GC overheads, and replicated for tolerating executor failures. Also, the data is kept first in memory, and spilled over to disk only if the memory is insufficient to hold all of the input data necessary for the streaming computation. This serialization obviously has overheads -- the receiver must deserialize the received data and re-serialize it using Spark's serialization format. 

* **Persisted RDDs generated by Streaming Operations**: RDDs generated by streaming computations may be persisted in memory. For example, window operations persist data in memory as they would be processed multiple times. However, unlike the Spark Core default of [StorageLevel.MEMORY_ONLY](api/scala/index.html#org.apache.spark.storage.StorageLevel$), persisted RDDs generated by streaming computations are persisted with [StorageLevel.MEMORY_ONLY_SER](api/scala/index.html#org.apache.spark.storage.StorageLevel$) (i.e. serialized) by default to minimize GC overheads.

In both cases, using Kryo serialization can reduce both CPU and memory overheads. See the [Spark Tuning Guide](tuning.html#data-serialization) for more details. For Kryo, consider registering custom classes, and disabling object reference tracking (see Kryo-related configurations in the [Configuration Guide](configuration.html#compression-and-serialization)).

In specific cases where the amount of data that needs to be retained for the streaming application is not large, it may be feasible to persist data (both types) as deserialized objects without incurring excessive GC overheads. For example, if you are using batch intervals of a few seconds and no window operations, then you can try disabling serialization in persisted data by explicitly setting the storage level accordingly. This would reduce the CPU overheads due to serialization, potentially improving performance without too much GC overheads.

### Task Launching Overheads
{:.no_toc}
If the number of tasks launched per second is high (say, 50 or more per second), then the overhead
of sending out tasks to the slaves may be significant and will make it hard to achieve sub-second
latencies. The overhead can be reduced by the following changes:

* **Execution mode**: Running Spark in Standalone mode or coarse-grained Mesos mode leads to
  better task launch times than the fine-grained Mesos mode. Please refer to the
  [Running on Mesos guide](running-on-mesos.html) for more details.

These changes may reduce batch processing time by 100s of milliseconds,
thus allowing sub-second batch size to be viable.

***

## Setting the Right Batch Interval
For a Spark Streaming application running on a cluster to be stable, the system should be able to
process data as fast as it is being received. In other words, batches of data should be processed
as fast as they are being generated. Whether this is true for an application can be found by
[monitoring](#monitoring-applications) the processing times in the streaming web UI, where the batch
processing time should be less than the batch interval.

Depending on the nature of the streaming
computation, the batch interval used may have significant impact on the data rates that can be
sustained by the application on a fixed set of cluster resources. For example, let us
consider the earlier WordCountNetwork example. For a particular data rate, the system may be able
to keep up with reporting word counts every 2 seconds (i.e., batch interval of 2 seconds), but not
every 500 milliseconds. So the batch interval needs to be set such that the expected data rate in
production can be sustained.

A good approach to figure out the right batch size for your application is to test it with a
conservative batch interval (say, 5-10 seconds) and a low data rate. To verify whether the system
is able to keep up with the data rate, you can check the value of the end-to-end delay experienced
by each processed batch (either look for "Total delay" in Spark driver log4j logs, or use the
[StreamingListener](api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener)
interface).
If the delay is maintained to be comparable to the batch size, then system is stable. Otherwise,
if the delay is continuously increasing, it means that the system is unable to keep up and it
therefore unstable. Once you have an idea of a stable configuration, you can try increasing the
data rate and/or reducing the batch size. Note that a momentary increase in the delay due to
temporary data rate increases may be fine as long as the delay reduces back to a low value
(i.e., less than batch size).

***

## Memory Tuning
Tuning the memory usage and GC behavior of Spark applications has been discussed in great detail
in the [Tuning Guide](tuning.html#memory-tuning). It is strongly recommended that you read that. In this section, we discuss a few tuning parameters specifically in the context of Spark Streaming applications.

The amount of cluster memory required by a Spark Streaming application depends heavily on the type of transformations used. For example, if you want to use a window operation on the last 10 minutes of data, then your cluster should have sufficient memory to hold 10 minutes worth of data in memory. Or if you want to use `updateStateByKey` with a large number of keys, then the necessary memory  will be high. On the contrary, if you want to do a simple map-filter-store operation, then the necessary memory will be low.

In general, since the data received through receivers is stored with StorageLevel.MEMORY_AND_DISK_SER_2, the data that does not fit in memory will spill over to the disk. This may reduce the performance of the streaming application, and hence it is advised to provide sufficient memory as required by your streaming application. Its best to try and see the memory usage on a small scale and estimate accordingly. 

Another aspect of memory tuning is garbage collection. For a streaming application that requires low latency, it is undesirable to have large pauses caused by JVM Garbage Collection. 

There are a few parameters that can help you tune the memory usage and GC overheads:

* **Persistence Level of DStreams**: As mentioned earlier in the [Data Serialization](#data-serialization) section, the input data and RDDs are by default persisted as serialized bytes. This reduces both the memory usage and GC overheads, compared to deserialized persistence. Enabling Kryo serialization further reduces serialized sizes and memory usage. Further reduction in memory usage can be achieved with compression (see the Spark configuration `spark.rdd.compress`), at the cost of CPU time.

* **Clearing old data**: By default, all input data and persisted RDDs generated by DStream transformations are automatically cleared. Spark Streaming decides when to clear the data based on the transformations that are used. For example, if you are using a window operation of 10 minutes, then Spark Streaming will keep around the last 10 minutes of data, and actively throw away older data. 
Data can be retained for a longer duration (e.g. interactively querying older data) by setting `streamingContext.remember`.

* **CMS Garbage Collector**: Use of the concurrent mark-and-sweep GC is strongly recommended for keeping GC-related pauses consistently low. Even though concurrent GC is known to reduce the
overall processing throughput of the system, its use is still recommended to achieve more
consistent batch processing times. Make sure you set the CMS GC on both the driver (using `--driver-java-options` in `spark-submit`) and the executors (using [Spark configuration](configuration.html#runtime-environment) `spark.executor.extraJavaOptions`).

* **Other tips**: To further reduce GC overheads, here are some more tips to try.
    - Persist RDDs using the `OFF_HEAP` storage level. See more detail in the [Spark Programming Guide](programming-guide.html#rdd-persistence).
    - Use more executors with smaller heap sizes. This will reduce the GC pressure within each JVM heap.

***

##### Important points to remember:
{:.no_toc}
- A DStream is associated with a single receiver. For attaining read parallelism multiple receivers i.e. multiple DStreams need to be created. A receiver is run within an executor. It occupies one core. Ensure that there are enough cores for processing after receiver slots are booked i.e. `spark.cores.max` should take the receiver slots into account. The receivers are allocated to executors in a round robin fashion.

- When data is received from a stream source, receiver creates blocks of data.  A new block of data is generated every blockInterval milliseconds. N blocks of data are created during the batchInterval where N = batchInterval/blockInterval. These blocks are distributed by the BlockManager of the current executor to the block managers of other executors. After that, the Network Input Tracker running on the driver is informed about the block locations for further processing.

- An RDD is created on the driver for the blocks created during the batchInterval. The blocks generated during the batchInterval are partitions of the RDD. Each partition is a task in spark. blockInterval== batchinterval would mean that a single partition is created and probably it is processed locally.

- The map tasks on the blocks are processed in the executors (one that received the block, and another where the block was replicated) that has the blocks irrespective of block interval, unless non-local scheduling kicks in.
Having bigger blockinterval means bigger blocks. A high value of `spark.locality.wait` increases the chance of processing a block on the local node. A balance needs to be found out between these two parameters to ensure that the bigger blocks are processed locally.

- Instead of relying on batchInterval and blockInterval, you can define the number of partitions by calling `inputDstream.repartition(n)`. This reshuffles the data in RDD randomly to create n number of partitions. Yes, for greater parallelism. Though comes at the cost of a shuffle. An RDD's processing is scheduled by driver's jobscheduler as a job. At a given point of time only one job is active. So, if one job is executing the other jobs are queued.

- If you have two dstreams there will be two RDDs formed and there will be two jobs created which will be scheduled one after the another. To avoid this, you can union two dstreams. This will ensure that a single unionRDD is formed for the two RDDs of the dstreams. This unionRDD is then considered as a single job. However the partitioning of the RDDs is not impacted.

- If the batch processing time is more than batchinterval then obviously the receiver's memory will start filling up and will end up in throwing exceptions (most probably BlockNotFoundException). Currently there is  no way to pause the receiver. Using SparkConf configuration `spark.streaming.receiver.maxRate`, rate of receiver can be limited.


***************************************************************************************************
***************************************************************************************************

# Fault-tolerance Semantics
In this section, we will discuss the behavior of Spark Streaming applications in the event
of failures. 

## Background
{:.no_toc}
To understand the semantics provided by Spark Streaming, let us remember the basic fault-tolerance semantics of Spark's RDDs.

1. An RDD is an immutable, deterministically re-computable, distributed dataset. Each RDD
remembers the lineage of deterministic operations that were used on a fault-tolerant input
dataset to create it.
1. If any partition of an RDD is lost due to a worker node failure, then that partition can be
re-computed from the original fault-tolerant dataset using the lineage of operations.
1. Assuming that all of the RDD transformations are deterministic, the data in the final transformed
   RDD will always be the same irrespective of failures in the Spark cluster.

Spark operates on data in fault-tolerant file systems like HDFS or S3. Hence,
all of the RDDs generated from the fault-tolerant data are also fault-tolerant. However, this is not
the case for Spark Streaming as the data in most cases is received over the network (except when
`fileStream` is used). To achieve the same fault-tolerance properties for all of the generated RDDs,
the received data is replicated among multiple Spark executors in worker nodes in the cluster
(default replication factor is 2). This leads to two kinds of data in the
system that need to recovered in the event of failures:

1. *Data received and replicated* - This data survives failure of a single worker node as a copy
  of it exists on one of the other nodes.
1. *Data received but buffered for replication* - Since this is not replicated,
   the only way to recover this data is to get it again from the source.

Furthermore, there are two kinds of failures that we should be concerned about:

1. *Failure of a Worker Node* - Any of the worker nodes running executors can fail,
   and all in-memory data on those nodes will be lost. If any receivers were running on failed
   nodes, then their buffered data will be lost.
1. *Failure of the Driver Node* - If the driver node running the Spark Streaming application
   fails, then obviously the SparkContext is lost, and all executors with their in-memory
   data are lost.

With this basic knowledge, let us understand the fault-tolerance semantics of Spark Streaming.

## Definitions
{:.no_toc}
The semantics of streaming systems are often captured in terms of how many times each record can be processed by the system. There are three types of guarantees that a system can provide under all possible operating conditions (despite failures, etc.)

1. *At most once*: Each record will be either processed once or not processed at all.
2. *At least once*: Each record will be processed one or more times. This is stronger than *at-most once* as it ensure that no data will be lost. But there may be duplicates.
3. *Exactly once*: Each record will be processed exactly once - no data will be lost and no data will be processed multiple times. This is obviously the strongest guarantee of the three.

## Basic Semantics
{:.no_toc}
In any stream processing system, broadly speaking, there are three steps in processing the data.

1. *Receiving the data*: The data is received from sources using Receivers or otherwise.

1. *Transforming the data*: The received data is transformed using DStream and RDD transformations.

1. *Pushing out the data*: The final transformed data is pushed out to external systems like file systems, databases, dashboards, etc.

If a streaming application has to achieve end-to-end exactly-once guarantees, then each step has to provide an exactly-once guarantee. That is, each record must be received exactly once, transformed exactly once, and pushed to downstream systems exactly once. Let's understand the semantics of these steps in the context of Spark Streaming.

1. *Receiving the data*: Different input sources provide different guarantees. This is discussed in detail in the next subsection.

1. *Transforming the data*: All data that has been received will be processed _exactly once_, thanks to the guarantees that RDDs provide. Even if there are failures, as long as the received input data is accessible, the final transformed RDDs will always have the same contents.

1. *Pushing out the data*: Output operations by default ensure _at-least once_ semantics because it depends on the type of output operation (idempotent, or not) and the semantics of the downstream system (supports transactions or not). But users can implement their own transaction mechanisms to achieve _exactly-once_ semantics. This is discussed in more details later in the section.

## Semantics of Received Data
{:.no_toc}
Different input sources provide different guarantees, ranging from _at-least once_ to _exactly once_. Read for more details.

### With Files
{:.no_toc}
If all of the input data is already present in a fault-tolerant file system like
HDFS, Spark Streaming can always recover from any failure and process all of the data. This gives
*exactly-once* semantics, meaning all of the data will be processed exactly once no matter what fails.

### With Receiver-based Sources
{:.no_toc}
For input sources based on receivers, the fault-tolerance semantics depend on both the failure
scenario and the type of receiver.
As we discussed [earlier](#receiver-reliability), there are two types of receivers:

1. *Reliable Receiver* - These receivers acknowledge reliable sources only after ensuring that
  the received data has been replicated. If such a receiver fails, the source will not receive
  acknowledgment for the buffered (unreplicated) data. Therefore, if the receiver is
  restarted, the source will resend the data, and no data will be lost due to the failure.
1. *Unreliable Receiver* - Such receivers do *not* send acknowledgment and therefore *can* lose
  data when they fail due to worker or driver failures.

Depending on what type of receivers are used we achieve the following semantics.
If a worker node fails, then there is no data loss with reliable receivers. With unreliable
receivers, data received but not replicated can get lost. If the driver node fails,
then besides these losses, all of the past data that was received and replicated in memory will be
lost. This will affect the results of the stateful transformations.

To avoid this loss of past received data, Spark 1.2 introduced _write
ahead logs_ which save the received data to fault-tolerant storage. With the [write ahead logs
enabled](#deploying-applications) and reliable receivers, there is zero data loss. In terms of semantics, it provides an at-least once guarantee. 

The following table summarizes the semantics under failures:

<table class="table">
  <tr>
    <th style="width:30%">Deployment Scenario</th>
    <th>Worker Failure</th>
    <th>Driver Failure</th>
  </tr>
  <tr>
    <td>
      <i>Spark 1.1 or earlier,</i> OR<br/>
      <i>Spark 1.2 or later without write ahead logs</i>
    </td>
    <td>
      Buffered data lost with unreliable receivers<br/>
      Zero data loss with reliable receivers<br/>
      At-least once semantics
    </td>
    <td>
      Buffered data lost with unreliable receivers<br/>
      Past data lost with all receivers<br/>
      Undefined semantics
    </td>
  </tr>
  <tr>
    <td><i>Spark 1.2 or later with write ahead logs</i></td>
    <td>
        Zero data loss with reliable receivers<br/>
        At-least once semantics
    </td>
    <td>
        Zero data loss with reliable receivers and files<br/>
        At-least once semantics
    </td>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

### With Kafka Direct API
{:.no_toc}
In Spark 1.3, we have introduced a new Kafka Direct API, which can ensure that all the Kafka data is received by Spark Streaming exactly once. Along with this, if you implement exactly-once output operation, you can achieve end-to-end exactly-once guarantees. This approach is further discussed in the [Kafka Integration Guide](streaming-kafka-integration.html).

## Semantics of output operations
{:.no_toc}
Output operations (like `foreachRDD`) have _at-least once_ semantics, that is, 
the transformed data may get written to an external entity more than once in
the event of a worker failure. While this is acceptable for saving to file systems using the
`saveAs***Files` operations (as the file will simply get overwritten with the same data),
additional effort may be necessary to achieve exactly-once semantics. There are two approaches.

- *Idempotent updates*: Multiple attempts always write the same data. For example, `saveAs***Files` always writes the same data to the generated files.

- *Transactional updates*: All updates are made transactionally so that updates are made exactly once atomically. One way to do this would be the following.

    - Use the batch time (available in `foreachRDD`) and the partition index of the RDD to create an identifier. This identifier uniquely identifies a blob data in the streaming application.
    - Update external system with this blob transactionally (that is, exactly once, atomically) using the identifier. That is, if the identifier is not already committed, commit the partition data and the identifier atomically. Else, if this was already committed, skip the update.

          dstream.foreachRDD { (rdd, time) =>
            rdd.foreachPartition { partitionIterator =>
              val partitionId = TaskContext.get.partitionId()
              val uniqueId = generateUniqueId(time.milliseconds, partitionId)
              // use this uniqueId to transactionally commit the data in partitionIterator
            }
          }

***************************************************************************************************
***************************************************************************************************

# Where to Go from Here
* Additional guides
    - [Kafka Integration Guide](streaming-kafka-integration.html)
    - [Kinesis Integration Guide](streaming-kinesis-integration.html)
    - [Custom Receiver Guide](streaming-custom-receivers.html)
* Third-party DStream data sources can be found in [Third Party Projects](http://spark.apache.org/third-party-projects.html)
* API documentation
  - Scala docs
    * [StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) and
  [DStream](api/scala/index.html#org.apache.spark.streaming.dstream.DStream)
    * [KafkaUtils](api/scala/index.html#org.apache.spark.streaming.kafka.KafkaUtils$),
    [FlumeUtils](api/scala/index.html#org.apache.spark.streaming.flume.FlumeUtils$),
    [KinesisUtils](api/scala/index.html#org.apache.spark.streaming.kinesis.KinesisUtils$),
  - Java docs
    * [JavaStreamingContext](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html),
    [JavaDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html) and
    [JavaPairDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html)
    * [KafkaUtils](api/java/index.html?org/apache/spark/streaming/kafka/KafkaUtils.html),
    [FlumeUtils](api/java/index.html?org/apache/spark/streaming/flume/FlumeUtils.html),
    [KinesisUtils](api/java/index.html?org/apache/spark/streaming/kinesis/KinesisUtils.html)
  - Python docs
    * [StreamingContext](api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext) and [DStream](api/python/pyspark.streaming.html#pyspark.streaming.DStream)
    * [KafkaUtils](api/python/pyspark.streaming.html#pyspark.streaming.kafka.KafkaUtils)

* More examples in [Scala]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming)
  and [Java]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/java/org/apache/spark/examples/streaming)
  and [Python]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/python/streaming)
* [Paper](http://www.eecs.berkeley.edu/Pubs/TechRpts/2012/EECS-2012-259.pdf) and [video](http://youtu.be/g171ndOHgJ0) describing Spark Streaming.
