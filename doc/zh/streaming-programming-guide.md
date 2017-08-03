---
layout: global
displayTitle: Spark Streaming Programming Guide
title: Spark Streaming
description: Spark Streaming programming guide and tutorial for Spark SPARK_VERSION_SHORT
---

* This will become a table of contents (this text will be scraped).
{:toc}

# Overview
Spark Streaming is an extension of the core Spark API that enables scalable, high-throughput,
fault-tolerant stream processing of live data streams. Data can be ingested from many sources
like Kafka, Flume, Kinesis, or TCP sockets, and can be processed using complex
algorithms expressed with high-level functions like `map`, `reduce`, `join` and `window`.
Finally, processed data can be pushed out to filesystems, databases,
and live dashboards. In fact, you can apply Spark's
[machine learning](ml-guide.html) and
[graph processing](graphx-programming-guide.html) algorithms on data streams.

<p style="text-align: center;">
  <img
    src="img/streaming-arch.png"
    title="Spark Streaming architecture"
    alt="Spark Streaming"
    width="70%"
  />
</p>

Internally, it works as follows. Spark Streaming receives live input data streams and divides
the data into batches, which are then processed by the Spark engine to generate the final
stream of results in batches.

<p style="text-align: center;">
  <img src="img/streaming-flow.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="70%" />
</p>

Spark Streaming provides a high-level abstraction called *discretized stream* or *DStream*,
which represents a continuous stream of data. DStreams can be created either from input data
streams from sources such as Kafka, Flume, and Kinesis, or by applying high-level
operations on other DStreams. Internally, a DStream is represented as a sequence of
[RDDs](api/scala/index.html#org.apache.spark.rdd.RDD).

This guide shows you how to start writing Spark Streaming programs with DStreams. You can
write Spark Streaming programs in Scala, Java or Python (introduced in Spark 1.2),
all of which are presented in this guide.
You will find tabs throughout this guide that let you choose between code snippets of
different languages.

**Note:** There are a few APIs that are either different or not available in Python. Throughout this guide, you will find the tag <span class="badge" style="background-color: grey">Python API</span> highlighting these differences.

***************************************************************************************************

# A Quick Example
Before we go into the details of how to write your own Spark Streaming program,
let's take a quick look at what a simple Spark Streaming program looks like. Let's say we want to
count the number of words in text data received from a data server listening on a TCP
socket. All you need to
do is as follows.

<div class="codetabs">
<div data-lang="scala"  markdown="1" >
First, we import the names of the Spark Streaming classes and some implicit
conversions from StreamingContext into our environment in order to add useful methods to
other classes we need (like DStream). [StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) is the
main entry point for all streaming functionality. We create a local StreamingContext with two execution threads,  and a batch interval of 1 second.

{% highlight scala %}
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3

// Create a local StreamingContext with two working thread and batch interval of 1 second.
// The master requires 2 cores to prevent from a starvation scenario.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
{% endhighlight %}

Using this context, we can create a DStream that represents streaming data from a TCP
source, specified as hostname (e.g. `localhost`) and port (e.g. `9999`).

{% highlight scala %}
// Create a DStream that will connect to hostname:port, like localhost:9999
val lines = ssc.socketTextStream("localhost", 9999)
{% endhighlight %}

This `lines` DStream represents the stream of data that will be received from the data
server. Each record in this DStream is a line of text. Next, we want to split the lines by
space characters into words.

{% highlight scala %}
// Split each line into words
val words = lines.flatMap(_.split(" "))
{% endhighlight %}

`flatMap` is a one-to-many DStream operation that creates a new DStream by
generating multiple new records from each record in the source DStream. In this case,
each line will be split into multiple words and the stream of words is represented as the
`words` DStream.  Next, we want to count these words.

{% highlight scala %}
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
// Count each word in each batch
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print()
{% endhighlight %}

The `words` DStream is further mapped (one-to-one transformation) to a DStream of `(word,
1)` pairs, which is then reduced to get the frequency of words in each batch of data.
Finally, `wordCounts.print()` will print a few of the counts generated every second.

Note that when these lines are executed, Spark Streaming only sets up the computation it
will perform when it is started, and no real processing has started yet. To start the processing
after all the transformations have been setup, we finally call

{% highlight scala %}
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
{% endhighlight %}

The complete code can be found in the Spark Streaming example
[NetworkWordCount]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/NetworkWordCount.scala).
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

If you have already [downloaded](index.html#downloading) and [built](index.html#building) Spark,
you can run this example as follows. You will first need to run Netcat
(a small utility found in most Unix-like systems) as a data server by using

{% highlight bash %}
$ nc -lk 9999
{% endhighlight %}

Then, in a different terminal, you can start the example by using

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


Then, any lines typed in the terminal running the netcat server will be counted and printed on
screen every second. It will look something like the following.

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

# Basic Concepts

Next, we move beyond the simple example and elaborate on the basics of Spark Streaming.

## Linking

Similar to Spark, Spark Streaming is available through Maven Central. To write your own Spark Streaming program, you will have to add the following dependency to your SBT or Maven project.

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

<table class="table">
<tr><th>Source</th><th>Artifact</th></tr>
<tr><td> Kafka </td><td> spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}} </td></tr>
<tr><td> Flume </td><td> spark-streaming-flume_{{site.SCALA_BINARY_VERSION}} </td></tr>
<tr><td> Kinesis<br/></td><td>spark-streaming-kinesis-asl_{{site.SCALA_BINARY_VERSION}} [Amazon Software License] </td></tr>
<tr><td></td><td></td></tr>
</table>

For an up-to-date list, please refer to the
[Maven repository](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.apache.spark%22%20AND%20v%3A%22{{site.SPARK_VERSION_SHORT}}%22)
for the full list of supported sources and artifacts.

***

## Initializing StreamingContext

To initialize a Spark Streaming program, a **StreamingContext** object has to be created which is the main entry point of all Spark Streaming functionality.

<div class="codetabs">
<div data-lang="scala" markdown="1">

A [StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) object can be created from a [SparkConf](api/scala/index.html#org.apache.spark.SparkConf) object.

{% highlight scala %}
import org.apache.spark._
import org.apache.spark.streaming._

val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))
{% endhighlight %}

The `appName` parameter is a name for your application to show on the cluster UI.
`master` is a [Spark, Mesos or YARN cluster URL](submitting-applications.html#master-urls),
or a special __"local[\*]"__ string to run in local mode. In practice, when running on a cluster,
you will not want to hardcode `master` in the program,
but rather [launch the application with `spark-submit`](submitting-applications.html) and
receive it there. However, for local testing and unit tests, you can pass "local[\*]" to run Spark Streaming
in-process (detects the number of cores in the local system). Note that this internally creates a [SparkContext](api/scala/index.html#org.apache.spark.SparkContext) (starting point of all Spark functionality) which can be accessed as `ssc.sparkContext`.

The batch interval must be set based on the latency requirements of your application
and available cluster resources. See the [Performance Tuning](#setting-the-right-batch-interval)
section for more details.

A `StreamingContext` object can also be created from an existing `SparkContext` object.

{% highlight scala %}
import org.apache.spark.streaming._

val sc = ...                // existing SparkContext
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

After a context is defined, you have to do the following.

1. Define the input sources by creating input DStreams.
1. Define the streaming computations by applying transformation and output operations to DStreams.
1. Start receiving data and processing it using `streamingContext.start()`.
1. Wait for the processing to be stopped (manually or due to any error) using `streamingContext.awaitTermination()`.
1. The processing can be manually stopped using `streamingContext.stop()`.

##### Points to remember:
{:.no_toc}
- Once a context has been started, no new streaming computations can be set up or added to it.
- Once a context has been stopped, it cannot be restarted.
- Only one StreamingContext can be active in a JVM at the same time.
- stop() on StreamingContext also stops the SparkContext. To stop only the StreamingContext, set the optional parameter of `stop()` called `stopSparkContext` to false.
- A SparkContext can be re-used to create multiple StreamingContexts, as long as the previous StreamingContext is stopped (without stopping the SparkContext) before the next StreamingContext is created.

***

## Discretized Streams (DStreams)
**Discretized Stream** or **DStream** is the basic abstraction provided by Spark Streaming.
It represents a continuous stream of data, either the input data stream received from source,
or the processed data stream generated by transforming the input stream. Internally,
a DStream is represented by a continuous series of RDDs, which is Spark's abstraction of an immutable,
distributed dataset (see [Spark Programming Guide](programming-guide.html#resilient-distributed-datasets-rdds) for more details). Each RDD in a DStream contains data from a certain interval,
as shown in the following figure.

<p style="text-align: center;">
  <img src="img/streaming-dstream.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="70%" />
</p>

Any operation applied on a DStream translates to operations on the underlying RDDs. For example,
in the [earlier example](#a-quick-example) of converting a stream of lines to words,
the `flatMap` operation is applied on each RDD in the `lines` DStream to generate the RDDs of the
 `words` DStream. This is shown in the following figure.

<p style="text-align: center;">
  <img src="img/streaming-dstream-ops.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="70%" />
</p>


These underlying RDD transformations are computed by the Spark engine. The DStream operations
hide most of these details and provide the developer with a higher-level API for convenience.
These operations are discussed in detail in later sections.

***

## Input DStreams and Receivers
Input DStreams are DStreams representing the stream of input data received from streaming
sources. In the [quick example](#a-quick-example), `lines` was an input DStream as it represented
the stream of data received from the netcat server. Every input DStream
(except file stream, discussed later in this section) is associated with a **Receiver**
([Scala doc](api/scala/index.html#org.apache.spark.streaming.receiver.Receiver),
[Java doc](api/java/org/apache/spark/streaming/receiver/Receiver.html)) object which receives the
data from a source and stores it in Spark's memory for processing.

Spark Streaming provides two categories of built-in streaming sources.

- *Basic sources*: Sources directly available in the StreamingContext API.
  Examples: file systems, and socket connections.
- *Advanced sources*: Sources like Kafka, Flume, Kinesis, etc. are available through
  extra utility classes. These require linking against extra dependencies as discussed in the
  [linking](#linking) section.

We are going to discuss some of the sources present in each category later in this section.

Note that, if you want to receive multiple streams of data in parallel in your streaming
application, you can create multiple input DStreams (discussed
further in the [Performance Tuning](#level-of-parallelism-in-data-receiving) section). This will
create multiple receivers which will simultaneously receive multiple data streams. But note that a
Spark worker/executor is a long-running task, hence it occupies one of the cores allocated to the
Spark Streaming application. Therefore, it is important to remember that a Spark Streaming application
needs to be allocated enough cores (or threads, if running locally) to process the received data,
as well as to run the receiver(s).

##### Points to remember
{:.no_toc}

- When running a Spark Streaming program locally, do not use "local" or "local[1]" as the master URL.
  Either of these means that only one thread will be used for running tasks locally. If you are using
  an input DStream based on a receiver (e.g. sockets, Kafka, Flume, etc.), then the single thread will
  be used to run the receiver, leaving no thread for processing the received data. Hence, when
  running locally, always use "local[*n*]" as the master URL, where *n* > number of receivers to run
  (see [Spark Properties](configuration.html#spark-properties) for information on how to set
  the master).

- Extending the logic to running on a cluster, the number of cores allocated to the Spark Streaming
  application must be more than the number of receivers. Otherwise the system will receive data, but
  not be able to process it.

### Basic Sources
{:.no_toc}

We have already taken a look at the `ssc.socketTextStream(...)` in the [quick example](#a-quick-example)
which creates a DStream from text
data received over a TCP socket connection. Besides sockets, the StreamingContext API provides
methods for creating DStreams from files as input sources.

- **File Streams:** For reading data from files on any file system compatible with the HDFS API (that is, HDFS, S3, NFS, etc.), a DStream can be created as:

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

	Spark Streaming will monitor the directory `dataDirectory` and process any files created in that directory (files written in nested directories not supported). Note that

     + The files must have the same data format.
     + The files must be created in the `dataDirectory` by atomically *moving* or *renaming* them into
     the data directory.
     + Once moved, the files must not be changed. So if the files are being continuously appended, the new data will not be read.

	For simple text files, there is an easier method `streamingContext.textFileStream(dataDirectory)`. And file streams do not require running a receiver, hence does not require allocating cores.

	<span class="badge" style="background-color: grey">Python API</span> `fileStream` is not available in the Python API, only	`textFileStream` is	available.

- **Streams based on Custom Receivers:** DStreams can be created with data streams received through custom receivers. See the [Custom Receiver
  Guide](streaming-custom-receivers.html) for more details.

- **Queue of RDDs as a Stream:** For testing a Spark Streaming application with test data, one can also create a DStream based on a queue of RDDs, using `streamingContext.queueStream(queueOfRDDs)`. Each RDD pushed into the queue will be treated as a batch of data in the DStream, and processed like a stream.

For more details on streams from sockets and files, see the API documentations of the relevant functions in
[StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) for
Scala, [JavaStreamingContext](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html)
for Java, and [StreamingContext](api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext) for Python.

### Advanced Sources
{:.no_toc}

<span class="badge" style="background-color: grey">Python API</span> As of Spark {{site.SPARK_VERSION_SHORT}},
out of these sources, Kafka, Kinesis and Flume are available in the Python API.

This category of sources require interfacing with external non-Spark libraries, some of them with
complex dependencies (e.g., Kafka and Flume). Hence, to minimize issues related to version conflicts
of dependencies, the functionality to create DStreams from these sources has been moved to separate
libraries that can be [linked](#linking) to explicitly when necessary.

Note that these advanced sources are not available in the Spark shell, hence applications based on
these advanced sources cannot be tested in the shell. If you really want to use them in the Spark
shell you will have to download the corresponding Maven artifact's JAR along with its dependencies
and add it to the classpath.

Some of these advanced sources are as follows.

- **Kafka:** Spark Streaming {{site.SPARK_VERSION_SHORT}} is compatible with Kafka broker versions 0.8.2.1 or higher. See the [Kafka Integration Guide](streaming-kafka-integration.html) for more details.

- **Flume:** Spark Streaming {{site.SPARK_VERSION_SHORT}} is compatible with Flume 1.6.0. See the [Flume Integration Guide](streaming-flume-integration.html) for more details.

- **Kinesis:** Spark Streaming {{site.SPARK_VERSION_SHORT}} is compatible with Kinesis Client Library 1.2.1. See the [Kinesis Integration Guide](streaming-kinesis-integration.html) for more details.

### Custom Sources
{:.no_toc}

<span class="badge" style="background-color: grey">Python API</span> This is not yet supported in Python.

Input DStreams can also be created out of custom data sources. All you have to do is implement a
user-defined **receiver** (see next section to understand what that is) that can receive data from
the custom sources and push it into Spark. See the [Custom Receiver
Guide](streaming-custom-receivers.html) for details.

### Receiver Reliability
{:.no_toc}

There can be two kinds of data sources based on their *reliability*. Sources
(like Kafka and Flume) allow the transferred data to be acknowledged. If the system receiving
data from these *reliable* sources acknowledges the received data correctly, it can be ensured
that no data will be lost due to any kind of failure. This leads to two kinds of receivers:

1. *Reliable Receiver* - A *reliable receiver* correctly sends acknowledgment to a reliable
  source when the data has been received and stored in Spark with replication.
1. *Unreliable Receiver* - An *unreliable receiver* does *not* send acknowledgment to a source. This can be used for sources that do not support acknowledgment, or even for reliable sources when one does not want or need to go into the complexity of acknowledgment.

The details of how to write a reliable receiver are discussed in the
[Custom Receiver Guide](streaming-custom-receivers.html).

***

## Transformations on DStreams
Similar to that of RDDs, transformations allow the data from the input DStream to be modified.
DStreams support many of the transformations available on normal Spark RDD's.
Some of the common ones are as follows.

<table class="table">
<tr><th style="width:25%">Transformation</th><th>Meaning</th></tr>
<tr>
  <td> <b>map</b>(<i>func</i>) </td>
  <td> Return a new DStream by passing each element of the source DStream through a
  function <i>func</i>. </td>
</tr>
<tr>
  <td> <b>flatMap</b>(<i>func</i>) </td>
  <td> Similar to map, but each input item can be mapped to 0 or more output items. </td>
</tr>
<tr>
  <td> <b>filter</b>(<i>func</i>) </td>
  <td> Return a new DStream by selecting only the records of the source DStream on which
  <i>func</i> returns true. </td>
</tr>
<tr>
  <td> <b>repartition</b>(<i>numPartitions</i>) </td>
  <td> Changes the level of parallelism in this DStream by creating more or fewer partitions. </td>
</tr>
<tr>
  <td> <b>union</b>(<i>otherStream</i>) </td>
  <td> Return a new DStream that contains the union of the elements in the source DStream and
  <i>otherDStream</i>. </td>
</tr>
<tr>
  <td> <b>count</b>() </td>
  <td> Return a new DStream of single-element RDDs by counting the number of elements in each RDD
   of the source DStream. </td>
</tr>
<tr>
  <td> <b>reduce</b>(<i>func</i>) </td>
  <td> Return a new DStream of single-element RDDs by aggregating the elements in each RDD of the
  source DStream using a function <i>func</i> (which takes two arguments and returns one).
  The function should be associative and commutative so that it can be computed in parallel. </td>
</tr>
<tr>
  <td> <b>countByValue</b>() </td>
  <td> When called on a DStream of elements of type K, return a new DStream of (K, Long) pairs
  where the value of each key is its frequency in each RDD of the source DStream.  </td>
</tr>
<tr>
  <td> <b>reduceByKey</b>(<i>func</i>, [<i>numTasks</i>]) </td>
  <td> When called on a DStream of (K, V) pairs, return a new DStream of (K, V) pairs where the
  values for each key are aggregated using the given reduce function. <b>Note:</b> By default,
  this uses Spark's default number of parallel tasks (2 for local mode, and in cluster mode the number
  is determined by the config property <code>spark.default.parallelism</code>) to do the grouping.
  You can pass an optional <code>numTasks</code> argument to set a different number of tasks.</td>
</tr>
<tr>
  <td> <b>join</b>(<i>otherStream</i>, [<i>numTasks</i>]) </td>
  <td> When called on two DStreams of (K, V) and (K, W) pairs, return a new DStream of (K, (V, W))
  pairs with all pairs of elements for each key. </td>
</tr>
<tr>
  <td> <b>cogroup</b>(<i>otherStream</i>, [<i>numTasks</i>]) </td>
  <td> When called on a DStream of (K, V) and (K, W) pairs, return a new DStream of
  (K, Seq[V], Seq[W]) tuples.</td>
</tr>
<tr>
  <td> <b>transform</b>(<i>func</i>) </td>
  <td> Return a new DStream by applying a RDD-to-RDD function to every RDD of the source DStream.
  This can be used to do arbitrary RDD operations on the DStream. </td>
</tr>
<tr>
  <td> <b>updateStateByKey</b>(<i>func</i>) </td>
  <td> Return a new "state" DStream where the state for each key is updated by applying the
  given function on the previous state of the key and the new values for the key. This can be
  used to maintain arbitrary state data for each key.</td>
</tr>
<tr><td></td><td></td></tr>
</table>

A few of these transformations are worth discussing in more detail.

#### UpdateStateByKey Operation
{:.no_toc}
The `updateStateByKey` operation allows you to maintain arbitrary state while continuously updating
it with new information. To use this, you will have to do two steps.

1. Define the state - The state can be an arbitrary data type.
1. Define the state update function - Specify with a function how to update the state using the
previous state and the new values from an input stream.

In every batch, Spark will apply the state  update function for all existing keys, regardless of whether they have new data in a batch or not. If the update function returns `None` then the key-value pair will be eliminated.

Let's illustrate this with an example. Say you want to maintain a running count of each word
seen in a text data stream. Here, the running count is the state and it is an integer. We
define the update function as:

<div class="codetabs">
<div data-lang="scala" markdown="1">

{% highlight scala %}
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}
{% endhighlight %}

This is applied on a DStream containing words (say, the `pairs` DStream containing `(word,
1)` pairs in the [earlier example](#a-quick-example)).

{% highlight scala %}
val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
{% endhighlight %}

The update function will be called for each word, with `newValues` having a sequence of 1's (from
the `(word, 1)` pairs) and the `runningCount` having the previous count.

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

Note that using `updateStateByKey` requires the checkpoint directory to be configured, which is
discussed in detail in the [checkpointing](#checkpointing) section.


#### Transform Operation
{:.no_toc}
The `transform` operation (along with its variations like `transformWith`) allows
arbitrary RDD-to-RDD functions to be applied on a DStream. It can be used to apply any RDD
operation that is not exposed in the DStream API.
For example, the functionality of joining every batch in a data stream
with another dataset is not directly exposed in the DStream API. However,
you can easily use `transform` to do this. This enables very powerful possibilities. For example,
one can do real-time data cleaning by joining the input data stream with precomputed
spam information (maybe generated with Spark as well) and then filtering based on it.

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

Note that the supplied function gets called in every batch interval. This allows you to do
time-varying RDD operations, that is, RDD operations, number of partitions, broadcast variables,
etc. can be changed between batches.

#### Window Operations
{:.no_toc}
Spark Streaming also provides *windowed computations*, which allow you to apply
transformations over a sliding window of data. The following figure illustrates this sliding
window.

<p style="text-align: center;">
  <img src="img/streaming-dstream-window.png"
       title="Spark Streaming data flow"
       alt="Spark Streaming"
       width="60%" />
</p>

As shown in the figure, every time the window *slides* over a source DStream,
the source RDDs that fall within the window are combined and operated upon to produce the
RDDs of the windowed DStream. In this specific case, the operation is applied over the last 3 time
units of data, and slides by 2 time units. This shows that any window operation needs to
specify two parameters.

 * <i>window length</i> - The duration of the window (3 in the figure).
 * <i>sliding interval</i> - The interval at which the window operation is performed (2 in
 the figure).

These two parameters must be multiples of the batch interval of the source DStream (1 in the
figure).

Let's illustrate the window operations with an example. Say, you want to extend the
[earlier example](#a-quick-example) by generating word counts over the last 30 seconds of data,
every 10 seconds. To do this, we have to apply the `reduceByKey` operation on the `pairs` DStream of
`(word, 1)` pairs over the last 30 seconds of data. This is done using the
operation `reduceByKeyAndWindow`.

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

Some of the common window operations are as follows. All of these operations take the
said two parameters - <i>windowLength</i> and <i>slideInterval</i>.

<table class="table">
<tr><th style="width:25%">Transformation</th><th>Meaning</th></tr>
<tr>
  <td> <b>window</b>(<i>windowLength</i>, <i>slideInterval</i>) </td>
  <td> Return a new DStream which is computed based on windowed batches of the source DStream.
  </td>
</tr>
<tr>
  <td> <b>countByWindow</b>(<i>windowLength</i>, <i>slideInterval</i>) </td>
  <td> Return a sliding window count of elements in the stream.
  </td>
</tr>
<tr>
  <td> <b>reduceByWindow</b>(<i>func</i>, <i>windowLength</i>, <i>slideInterval</i>) </td>
  <td> Return a new single-element stream, created by aggregating elements in the stream over a
  sliding interval using <i>func</i>. The function should be associative and commutative so that it can be computed
  correctly in parallel.
  </td>
</tr>
<tr>
  <td> <b>reduceByKeyAndWindow</b>(<i>func</i>, <i>windowLength</i>, <i>slideInterval</i>,
  [<i>numTasks</i>]) </td>
  <td> When called on a DStream of (K, V) pairs, returns a new DStream of (K, V)
  pairs where the values for each key are aggregated using the given reduce function <i>func</i>
  over batches in a sliding window. <b>Note:</b> By default, this uses Spark's default number of
  parallel tasks (2 for local mode, and in cluster mode the number is determined by the config
  property <code>spark.default.parallelism</code>) to do the grouping. You can pass an optional
  <code>numTasks</code> argument to set a different number of tasks.
  </td>
</tr>
<tr>
  <td> <b>reduceByKeyAndWindow</b>(<i>func</i>, <i>invFunc</i>, <i>windowLength</i>,
  <i>slideInterval</i>, [<i>numTasks</i>]) </td>
  <td markdown="1"> A more efficient version of the above <code>reduceByKeyAndWindow()</code> where the reduce
  value of each window is calculated incrementally using the reduce values of the previous window.
  This is done by reducing the new data that enters the sliding window, and "inverse reducing" the
  old data that leaves the window. An example would be that of "adding" and "subtracting" counts
  of keys as the window slides. However, it is applicable only to "invertible reduce functions",
  that is, those reduce functions which have a corresponding "inverse reduce" function (taken as
  parameter <i>invFunc</i>). Like in <code>reduceByKeyAndWindow</code>, the number of reduce tasks
  is configurable through an optional argument. Note that [checkpointing](#checkpointing) must be
  enabled for using this operation.
</td>
</tr>
<tr>
  <td> <b>countByValueAndWindow</b>(<i>windowLength</i>,
  <i>slideInterval</i>, [<i>numTasks</i>]) </td>
  <td> When called on a DStream of (K, V) pairs, returns a new DStream of (K, Long) pairs where the
  value of each key is its frequency within a sliding window. Like in
  <code>reduceByKeyAndWindow</code>, the number of reduce tasks is configurable through an
  optional argument.
</td>
</tr>
<tr><td></td><td></td></tr>
</table>

#### Join Operations
{:.no_toc}
Finally, its worth highlighting how easily you can perform different kinds of joins in Spark Streaming.


##### Stream-stream joins
{:.no_toc}
Streams can be very easily joined with other streams.

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
Here, in each batch interval, the RDD generated by `stream1` will be joined with the RDD generated by `stream2`. You can also do `leftOuterJoin`, `rightOuterJoin`, `fullOuterJoin`. Furthermore, it is often very useful to do joins over windows of the streams. That is pretty easy as well. 

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
This has already been shown earlier while explain `DStream.transform` operation. Here is yet another example of joining a windowed stream with a dataset.

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

In fact, you can also dynamically change the dataset you want to join against. The function provided to `transform` is evaluated every batch interval and therefore will use the current dataset that `dataset` reference points to.

The complete list of DStream transformations is available in the API documentation. For the Scala API,
see [DStream](api/scala/index.html#org.apache.spark.streaming.dstream.DStream)
and [PairDStreamFunctions](api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions).
For the Java API, see [JavaDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html)
and [JavaPairDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html).
For the Python API, see [DStream](api/python/pyspark.streaming.html#pyspark.streaming.DStream).

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
