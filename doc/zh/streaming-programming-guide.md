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

# 接下来我们要了解什么
* Additional guides （附加指南）
    - [Kafka Integration Guide （Kafka 集成指南）](streaming-kafka-integration.html)
    - [Kinesis Integration Guide （Kinesis 集成指南）](streaming-kinesis-integration.html)
    - [Custom Receiver Guide （自定义接收器指南）](streaming-custom-receivers.html)
* 第三方 DStream 数据源可以在 [Third Party Projects](http://spark.apache.org/third-party-projects.html) 查看.
* API 文档
  - Scala 文档
    * [StreamingContext](api/scala/index.html#org.apache.spark.streaming.StreamingContext) and
  [DStream](api/scala/index.html#org.apache.spark.streaming.dstream.DStream)
    * [KafkaUtils](api/scala/index.html#org.apache.spark.streaming.kafka.KafkaUtils$),
    [FlumeUtils](api/scala/index.html#org.apache.spark.streaming.flume.FlumeUtils$),
    [KinesisUtils](api/scala/index.html#org.apache.spark.streaming.kinesis.KinesisUtils$),
  - Java 文档
    * [JavaStreamingContext](api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html),
    [JavaDStream](api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html) and
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
* [Paper](http://www.eecs.berkeley.edu/Pubs/TechRpts/2012/EECS-2012-259.pdf) 和 [video](http://youtu.be/g171ndOHgJ0) 描述 Spark Streaming.
