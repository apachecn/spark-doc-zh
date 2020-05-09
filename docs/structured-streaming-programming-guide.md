# 结构化流式编程指南

-  [概述](#概述)
-  [简单例子](#简单例子)
- [编程模型](#编程模型)
  - [基本概念](#基本概念)
  - [处理 `Event-time` 和 `Late Data`](#处理 `Event-time` 和 `Late Data`)
  - [ `fault-tolerance` 语义](#  `fault-tolerance` 语义)
- [使用 `Dataset` 和 `DataFrame` 的API](#使用 `Dataset` 和 `DataFrame` 的API)
  -  [创建流式 `DataFrame` 和流式 `Dataset`](#创建流式 `DataFrame` 和流式 `Dataset`)
    -  [输入源](#输入源)
    - [流式 `DataFrame/Dataset` 的模式推断和分区](#流式 `DataFrame/Dataset` 的模式推断和分区)
  - [对流式 `DataFrame/Dataset` 的操作](#对流式 `DataFrame/Dataset` 的操作)
    - [基本操作—— `Selection`,`Projection`,`Aggregation`](#基本操作—— `Selection`,`Projection`,`Aggregation`)
    - [`Event-time` 上的 `Window` 操作](#`Event-time` 上的 `Window` 操作)
      -  [处理 `Late Data` 和 `Watermark`](#处理 `Late Data` 和 `Watermark`)
    - [ `Join` 操作](#`Join 操作`)
      - [`Stream-static` 连接](#`Stream-static 连接`)
      - [`Stream-stream` 连接](#`Stream-stream` 连接)
        - [Watermark `Inner Join`](#Watermark `Inner Join`)
        -  [Watermark `Outer Join`](#Watermark `Outer Join`)
        - [支持指标 `Join` 流查询](#支持指标 `Join` 流查询)
    -  [Stream 重复数据删除(`Deduplication`)](Stream 重复数据删除(`Deduplication`))
    - [处理多个 `Watermark` 的策略](#处理多个 `Watermark` 的策略)
    - [任意的 `Stateful Operation`](#任意的 `Stateful Operation`)
    - [不支持的 `Operation`](#不支持的 `Operation`)
  - [开始 `Stream Query`](#开始 `Stream Query`)
    - [`Output` 模式](#`Output` 模式)
    - [`Output Sink`](#`Output Sink`)
      - [使用 `Foreach` 和 `ForeachBatch`](#使用 `Foreach` 和 `ForeachBatch`)
        - [`ForeachBatch`](#`ForeachBatch`)
        - [`Foreach`](#`Foreach`)
    -  [Trigger](#Trigger)
  - [`Stream` 管理查询](#`Stream` 管理查询)
  - [`Stream` 监控查询](#`Stream` 监控查询)
    - [读指标交互](#读指标交互)
    - [使用 Async API 编程输出指标报告](#使用 Async API 编程输出指标报表)
    - [使用 Dropwizard 输出指标报告](#使用 Dropwizard 输出指标报告)
  - [使用 `Checkpoint` 从失败中 `Recovery`](#使用 `Checkpoint` 从失败中恢复)
  - [`Stream` 查询中更改后的 `Recovery` 语义](#recovery-semantics-after-changes-in-a-streaming-query)
- [连续处理](#连续处理)
- [额外信息](#额外信息)


# 概述

结构化流是一个可伸缩的、 `fault-tolerance` 的流处理引擎，构建在Spark SQL引擎上。你可以用在静态数据上表示批处理计算的方式来表示流计算。Spark SQL引擎将负责增量地、连续地运行它，并在流数据不断到达时更新最终结果。你可以使用 `Scala`、`Java`、`Python`或`R`中的[Dataset/DataFrame API](sql-programming-guide-zh.md)来表示`stream aggregation`、`event-time window` 、`stream-to-batch join` 等。计算是在同一个优化的Spark SQL引擎上执行的。最后，系统通过 `checkpoint` 和 `write-ahead` 日志来确保 `end-to-end` `exactly-once`  `fault-tolerance` 保证。简而言之，结构化流提供了快速、可伸缩、`fault-tolerance`、`end-to-end`  `exactly-once` 流处理，而用户无需对流进行推断。

在内部，默认情况下，使用 `micro-batch` 引擎处理结构化流查询，`micro-batch` 引擎将数据流处理为一系列小批作业，从而实现低至100毫秒的 `end-to-end` 延迟和 `exactly-once` `fault-tolerance` 保证。但是，从Spark 2.3开始，我们引入了一种新的低延迟处理模式，称为 **连续处理(Continuous Processing)**，它可以在 `at-least-once` 保证的情况下实现低至1毫秒的 `end-to-end` 延迟。不需要更改查询中的 `Dataset` /`DataFrame` 操作，就可以根据应用程序需求选择模式。

在本指南中，我们将介绍编程模型和 API。我们将主要使用默认的 `micro-batch` 模型来解释这些概念，然后讨论连续处理模型。首先，让我们从一个结构化流查询的简单示例开始—一个流字数统计。



# 简单例子

假设你希望维护从监听 `TCP Socket` 的数据服务器接收到的文本数据的运行字数计数。让我们看看如何使用结构化流来表达这一点。你可以在Scala/Java/Python/R中看到完整的代码。如果你下载了Spark，你可以直接运行这个例子。在任何情况下，让我们一步一步地遍历这个示例，并了解它是如何工作的。首先，我们必须导入必要的类并创建一个本地SparkSession，这是与Spark相关的所有功能的起点。

- Scala

```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder
  .appName("StructuredNetworkWordCount")
  .getOrCreate()
  
import spark.implicits._
```

- Java

```java
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.sql.*;
import org.apache.spark.sql.streaming.StreamingQuery;

import java.util.Arrays;
import java.util.Iterator;

SparkSession spark = SparkSession
  .builder()
  .appName("JavaStructuredNetworkWordCount")
  .getOrCreate();
```



接下来，让我们创建一个 `stream DataFrame`，它表示从监听 `localhost:9999` 的服务器接收到的文本数据，并将 `DataFrame` 转换为 `word count` 计算。

- Scala

```scala
// Create DataFrame representing the stream of input lines from connection to localhost:9999
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

- Java

```java
// Create DataFrame representing the stream of input lines from connection to localhost:9999
Dataset<Row> lines = spark
  .readStream()
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load();

// Split the lines into words
Dataset<String> words = lines
  .as(Encoders.STRING())
  .flatMap((FlatMapFunction<String, String>) x -> Arrays.asList(x.split(" ")).iterator(), Encoders.STRING());

// Generate running word count
Dataset<Row> wordCounts = words.groupBy("value").count();
```

这 `lines ` DataFrame 表示一个包含流文本数据的无界表。该表包含一列名为“value”的字符串，流文本数据中的每一行都成为该表中的一行。请注意，由于我们只是在设置转换，还没有启动转换，所以目前还没有接收任何数据。接下来，我们使用以下命令将 DataFrame 转换为一个字符串 Dataset   `.as[String]`，这样我们就可以应用 `flatMap` 操作将每一行分割成多个单词。生成的单词 `Dataset` 包含所有单词。最后，我们定义了`wordCounts` DataFrame，方法是根据 `Dataset` 中的惟一值进行分组并计数。请注意，这是一个流DataFrame，它表示流的运行字数。

我们现在已经设置了对流数据的查询。剩下的就是实际开始接收数据并计算计数。为此，我们将它设置为在每次更新计数结果集(由 `outputMode("complete")` 指定)时将它们打印到控制台。然后使用 `start()` 启动流计算。

- Scala

```scala
// Start running the query that prints the running counts to the console
val query: StreamingQuery = wordCounts.writeStream
  .outputMode("complete")
  .format("console")
  .start()

query.awaitTermination()
```



- Java

```java
// Start running the query that printes the running counts to the console
StreamingQuery query = wordCounts.writeStream()
  .outPutMode("complete")
  .format("console")
  .start();

query.awaitTermination()
```



执行此代码后，流计算将在后台启动。`query` 对象是活动流查询的句柄，我们决定使用 `awaitTermination()` 来等待查询的终止，以防止在查询活动期间进程退出。

要实际执行此示例代码，你可以在自己的 [Spark应用程序](quick-start-zh.md#self-contained-applications)中编译代码，也可以在下载了Spark后[运行示例](index-zh.md#running-the-examples-and-shell)。在这里我们用下载的Spark 运行示例进行展示。你首先需要使用Netcat(在大多数类unix系统中可以找到的一个小实用程序)作为数据服务器运行

```bash
$ nc -lk 9999
```



然后，在另一个终端中，你可以使用以下命令启动示例

- Scala

```shell
$ ./bin/run-example org.apache.spark.examples.sql.streaming.StructuredNetworkWordCount localhost 9999
```



- Java

```java
$ ./bin/run-example org.apache.spark.examples.sql.streaming.JavaStructuredNetworkWordCount localhost 9999
```



然后，在运行 `netcat` 服务器的终端中键入的任何行都将被计数并在屏幕上每秒打印一次。它看起来就像下面这样。



```bash
# TERMINAL 1:
# Running Netcat

$ nc -lk 9999
apache spark
apache hadoop
```



- 下面以 Scala 为例（执行结果都一样，仅仅启动脚本命令行稍微不同而已）， 每秒打印一次，并且统计结果累计

```
# TERMINAL 2: RUNNING StructuredNetworkWordCount

$ ./bin/run-example org.apache.spark.examples.sql.streaming.StructuredNetworkWordCount localhost 9999

-------------------------------------------
Batch: 0
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
|apache|    1|
| spark|    1|
+------+-----+

-------------------------------------------
Batch: 1
-------------------------------------------
+------+-----+
| value|count|
+------+-----+
|apache|    2|
| spark|    1|
|hadoop|    1|
+------+-----+
...
```



# 编程模型

结构化流的关键思想是将实时数据流视为一个不断追加的表。这导致了一个新的流处理模型，它非常类似于批处理模型。你可以将流计算表示为标准的类似批处理的查询，就像在静态表上一样，而Spark将它作为 `unbound` 输入表上的*增量*查询来运行。让我们更详细地了解这个模型。

## 基本概念

将输入数据流视为“Input Table”。到达流上的每个数据项就像一个新行被追加到输入表中。

![Stream as a Table](img/structured-streaming-stream-as-a-table.png)

对输入的查询将生成“Result Table”。在每个触发间隔(例如，每1秒)，都会向输入表追加新行，从而最终更新结果表。无论何时更新结果表，我们都希望将更改后的结果行写入外部 `sink`。

![Model](img/structured-streaming-model.png)

“OutPut”被定义为写入外部存储器的内容。输出可以在不同的模式下定义:

- *`Complete Mode`* - 整个更新后的 `Result Table` 将被写入外部存储。由 `storage connector` 决定如何处理整个表的写入。
- *`Append Mode`* - 仅将新行添加到 `Result Table` ，因为最后一个 `trigger` 触发器将被写入外部存储。这仅适用于不希望更改结果表中现有行的查询。
- *`Update Mode`* - 仅将 `Result Table` 中自最后一个 `trigger` 触发器以来更新的行写入外部存储**(**从**Spark 2.1.1**开始可用**)**。注意，这与 `Complete Mode` 不同，因为该模式仅输出自上次触发器以来更改的行。如果查询不包含聚合，则其效果等同于追加模式。

注意，每种模式都适用于特定类型的查询。[稍后](#`Output` 模式)将对此进行详细讨论。

为了演示这个模型的使用，让我们在上面的[快速示例](#简单例子)上下文中理解这个模型。第一个 `lines` DataFrame是输入表，最后一行 `wordCounts` DataFrame是结果表。请注意，用于生成 `wordCounts` 的流行DataFrame上的查询与静态DataFrame上的查询完全相同。但是，当这个查询启动时，Spark将不断地检查来自 Socket 连接的新数据。如果有新数据，Spark将运行一个“incremental(增量)”查询，该查询将以前的运行计数与新数据组合起来，以计算更新的计数，如下所示。

![Model](img/structured-streaming-example-model.png)

**请注意，结构化流并不会物化整个表。**它从流数据源读取最新的可用数据，以增量方式处理它以更新结果，然后丢弃源数据。它只保留更新结果所需的最小中间状态数据(例如，前面示例中的中间计数)。

这个模型与许多其他流处理引擎有很大的不同。许多流系统要求用户自己维护运行的聚合，因此必须考虑 `fault-tolerance` 和数据一致性( `at-least-once` ，或 `at-most-once` ，或 `exactly-once` )。在这个模型中，当有新数据时，Spark负责更新结果表，从而使用户不必进行推理。作为一个示例，让我们看看这个模型如何处理基于 `event-time` 的处理和延迟到达的数据。



## 处理 `Event-time` 和 `Late Data`

 `event-time` 是嵌入到数据本身中的时间。对于许多应用程序，你可能希望对这个 `event-time` 进行操作。例如，如果你希望获得物联网设备每分钟生成的事件数，那么你可能希望使用数据生成时的时间(即数据中的 `event-time` )，而不是Spark接收它们的时间。这个 `event-time` 很自然地在这个模型中表示出来——来自设备的每个事件是表中的一行，而 `event-time` 是行中的列值。这使得基于窗口的聚合(例如每分钟事件数)成为 `event-time` 列上的一种特殊的分组和聚合类型——每个时间窗口都是一个组，每一行都可以属于多个窗口/组。因此，可以在静态 `Dataset` (例如，从收集的设备事件日志)和数据流上一致地定义这种基于 `event-time` 窗口的聚合查询，这使得用户使用地更加轻松。

此外，此模型自然会根据 `event-time` 处理比预期晚到达的数据。因为Spark正在更新结果表，所以它可以完全控制在出现延迟数据时更新旧的聚合，以及清理旧的聚合以限制中间状态数据的大小。从<u>**Spark 2.1**</u>开始，我们支持 `Watermark` ，允许用户指定延迟数据的阈值，允许引擎相应地清理旧状态。稍后将在窗口操作部分更详细地解释这些操作。



##  `fault-tolerance` 语义

提供 `end-to-end`  `Exactly-once` 语义是结构化流设计背后的关键目标之一。为了实现这一点，我们设计了结构化的流源、`Sink` 和执行引擎来可靠地跟踪处理的确切进度，以便它能够通过重新启动和/或重新处理来处理任何类型的故障。假设每个流源都有偏移量(类似于Kafka偏移量或Kinesis序列号)来跟踪流中的读取位置。该引擎使用 `checkpoint` 和 `write-ahead` 日志来记录每个触发器中正在处理的数据的偏移范围。`Stream sink` 被设计成处理后处理的幂等性。通过使用可重放的源和幂等 Sink，结构化流可以确保在任何失败情况下 `end-to-end` 的 `Exactly-once` 语义。



# 使用 `Dataset` 和 `DataFrame` 的API

自Spark 2.0以来，DataFrames和 `Dataset` 可以表示静态的有界数据，也可以表示流式的无界数据。与静态 `Dataset` /DataFrames类似，你可以使用公共入口点SparkSession (Scala/Java/Python/R docs)从流数据源创建流 `Dataset` / `Dataset` ，并对它们应用与静态 `Dataset` / `Dataset` 相同的操作。如果你不熟悉 `Dataset` / `Dataset` ，强烈建议你使用 [DataFrame/Dataset编程指南](sql-programming-guide-zh.md)熟悉它们。



## 创建流式 `DataFrame` 和流式 `Dataset`

流数据流可以通过 `SparkSession.readStream()` 返回的  `DataStreamReader` 接口([Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.streaming.DataStreamReader)/[Java]([Java](http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/DataStreamReader.html))/[Python](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.streaming.DataStreamReader)文档)创建。在[R](http://spark.apache.org/docs/latest/api/R/read.stream.html)中，使用 `read.stream()` 方法。与创建静态DataFrame的read接口类似，你可以指定源数据格式、模式、选项等的详细信息。

### 输入源

有一些 `built-in` 的资源。

- `File source` - 以数据流的形式读取目录中写入的文件。支持的文件格式有 `text`，`csv`, `json`,  `orc`, `parquet`。请参阅 `DataStreamReader` 接口的文档，以获得 `up-to-date` 的列表和每种文件格式支持的选项。请注意，文件必须自动地放在给定的目录中，在大多数文件系统中，可以通过文件移动操作来实现.
- `Kafka source` - 从Kafka读取数据。它与`Kafka broker` 版本0.10.0或更高版本兼容。有关更多细节，请参阅[Kafka集成指南](http://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)。
-  `Socket source(用于测试)`- 从 Socket 连接读取 UTF8文本数据。监听位于 Driver 的 Socket 服务。注意，这应该只用于测试，因为它不提供 `end-to-end` 的 `fault-tolerance` 保证。
- `Rate source(用于测试)` - 以每秒指定的行数生成数据，每个输出行包含一个 `timestamp` 和 `value`。其中`timestamp` 是包含消息分派时间的 `Timestamp` 类型，而 `value` 是包含消息计数的 `Long` 类型，从第一行的0开始。此源代码用于测试和 `benchmark`(基准测试)。

有些源不是 `fault-tolerance` 的，因为它们不保证在发生故障后可以使用 `checkpoint` 偏移量重新播放数据。请参阅前面有关 [`fault-tolerance` 语义](#`fault-tolerance` 语义)的部分。以下是Spark中所有源的详细信息。



<table class="table">
<tbody>
<tr>
<th>Source</th>
<th>Options</th>
<th>Fault-tolerant</th>
<th>Notes</th>
</tr>
<tr>
<td><strong>File source</strong></td>
<td><code>path</code>: 到输入目录的路径，对所有文件格式都通用。<br /><code>maxFilesPerTrigger</code>: 每个 `trigger` 中要考虑的新文件的最大数量 (默认: no max)<br /><code>latestFirst</code>: 是否先处理最新的新文件，在有大量积压文件时非常有用 (默认: false)<br /><code>fileNameOnly</code>: 是否仅根据文件名而不是完整路径检查新文件 (默认: false). 将这个值设置为 `true`，以下文件将被视为相同的文件，因为它们的文件名为“dataset.txt”，都是一样的:<br />"file:///dataset.txt"<br />"s3://a/dataset.txt"<br />"s3n://a/b/dataset.txt"<br />"s3a://a/b/c/dataset.txt"<br /><br /><br />有关文件格式特定的选项，请参阅&nbsp;<code>DataStreamReader</code>&nbsp;(<a href="http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.streaming.DataStreamReader">Scala</a>/<a href="http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/DataStreamReader.html">Java</a>/<a href="http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.streaming.DataStreamReader">Python</a>/<a href="http://spark.apache.org/docs/latest/api/R/read.stream.html">R</a>). E.g. "parquet" 格式选项 请参见&nbsp;<code>DataStreamReader.parquet()</code>.<br /><br />I此外，还有一些会话配置会影响某些文件格式。请参见&nbsp;<a href="sql-programming-guide-zh.md">SQL Programming Guide</a>&nbsp;更多详细信息. E.g., "parquet", 请参见&nbsp;<a href="sql-data-sources-parquet-zh.md#配置">Parquet 配置</a>&nbsp;部分.</td>
<td>Yes</td>
<td>支持glob路径，但不支持多个逗号分隔的 paths/globs。</td>
</tr>
<tr>
<td><strong>Socket Source</strong></td>
<td><code>host</code>: 连接的主机名，必须指定<br /><code>port</code>: 连接端口，必须指定</td>
<td>No</td>
<td>&nbsp;</td>
</tr>
<tr>
<td><strong>Rate Source</strong></td>
<td><code>rowsPerSecond</code>&nbsp;(e.g. 100, 默认: 1): 每秒应该生成多少行。<br /><br /><code>rampUpTime</code>&nbsp;(e.g. 5s, 默认: 0s): 在生成数据速度上升之前需要多长时间&nbsp;<code>rowsPerSecond</code>. 使用比秒更细的粒度将被截断为整数秒。<br /><br /><code>numPartitions</code>&nbsp;(e.g. 10, 默认: Spark 默认并行度): 生成数据行的分区数<br /><br />Source 将尽所能的达到&nbsp;<code>rowsPerSecond</code>, 但是查询可能受到资源限制，并且&nbsp;<code>numPartitions</code>&nbsp;可以调整，以帮助达到所需的速度。</td>
<td>Yes</td>
<td>&nbsp;</td>
</tr>
<tr>
<td><strong>Kafka Source</strong></td>
<td>See the&nbsp;<a href="structured-streaming-kafka-integration-zh.md">Kafka Integration Guide</a>.</td>
<td>Yes</td>
</tr>
</tbody>
</table>



这里有一些例子。

- Scala

```scala
val spark: SparkSession = ...

// Read text from socket
val socketDF:Dataset[Row] = spark
  .readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()

socketDF.isStreaming    // Returns True for DataFrames that have streaming sources

socketDF.printSchema

// Read all the csv files written atomically in a directory
val userSchema: StructType = new StructType().add("name", "string").add("age", "integer")
val csvDF: Dataset<Row> = spark
  .readStream
  .option("sep", ";")
  .schema(userSchema)      // Specify schema of the csv files
  .csv("/path/to/directory")    // Equivalent to format("csv").load("/path/to/directory")
```



- Java

```java
SparkSession spark = ...

// Read text from socket
Dataset<Row> socketDF = spark
  .readStream()
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load();

socketDF.isStreaming();    // Returns True for DataFrames that have streaming sources

socketDF.printSchema();

// Read all the csv files written atomically in a directory
StructType userSchema = new StructType().add("name", "string").add("age", "integer");
Dataset<Row> csvDF = spark
  .readStream()
  .option("sep", ";")
  .schema(userSchema)      // Specify schema of the csv files
  .csv("/path/to/directory");    // Equivalent to format("csv").load("/path/to/directory")
```



这些示例生成无类型的流式数据流，这意味着在编译时不检查DataFrame的模式，只在提交查询时检查。有些操作，如map、flatMap等，需要在编译时知道类型。为此，可以使用与静态DataFrame相同的方法将这些无类型的流数据流转换为有类型的流 `Dataset` 。有关更多细节，请参阅[SQL编程指南](sql-programming-guide-zh.md)。另外，关于支持的stream source 的更多细节将在本文后面讨论。



### 流式 `DataFrame/Dataset` 的模式推断和分区

默认情况下，来自基于文件的源的结构化流要求你指定模式，而不是依赖Spark自动推断它。这个限制确保流查询使用一致的模式，即使在失败的情况下也是如此。对于特殊用例，可以通过将  `spark.sql.streaming.schemaInference` 设置为 `true` 来重新启用模式推断。

当存在名为 `/key=value/` 的子目录时，分区发现确实会发生，并且列表将自动递归到这些目录中。如果这些列出现在用户提供的模式中，则 `Spark` 将根据正在读取的文件的路径填充它们。组成分区方案的目录必须在查询启动时出现，并且必须保持静态。例如， 当 `/data/year=2015/` 是可以添加 `/data/year=2016/`，但是更改分区列是无效的(即通过创建目录 `/data/date=2016-04-17/`)。



## 对流式 `DataFrame/Dataset` 的操作

你可以在 `DataFrame` 和 `Dataset` 上应用所有类型的操作——从无类型的，类似sql的操作(例如 `select`, `where`, `groupBy`)，到类型化的类似 `RDD`  的操作(例如 `map`,  `filter`, `flatMap`)。有关更多细节，请参阅SQL编程指南。让我们来看几个可以使用的示例操作。

### 基本操作—— `Selection`,`Projection`,`Aggregation`

数据流支持DataFrame/Dataset上的大多数常见操作。本节后面将讨论少数不受支持的操作。

- Scala

```scala
case class DeviceData(device: String, deviceType: String, signal: Double, time: DateTime)

val df: DataFrame = ... // streaming DataFrame with IOT device data with schema { device: string, deviceType: string, signal: double, time: string }
val ds: Dataset[DeviceData] = df.as[DeviceData]    // streaming Dataset with IOT device data

// Select the devices which have signal more than 10
df.select("device").where("signal > 10")      // using untyped APIs   
ds.filter(_.signal > 10).map(_.device)         // using typed APIs

// Running count of the number of updates for each device type
df.groupBy("deviceType").count()                          // using untyped API

// Running average signal for each device type
import org.apache.spark.sql.expressions.scalalang.typed
ds.groupByKey(_.deviceType).agg(typed.avg(_.signal))    // using typed API
```



- Java

```java
import org.apache.spark.api.java.function.*;
import org.apache.spark.sql.*;
import org.apache.spark.sql.expressions.javalang.typed;
import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder;

public class DeviceData {
  private String device;
  private String deviceType;
  private Double signal;
  private java.sql.Date time;
  ...
  // Getter and setter methods for each field
}

Dataset<Row> df = ...;    // streaming DataFrame with IOT device data with schema { device: string, type: string, signal: double, time: DateType }
Dataset<DeviceData> ds = df.as(ExpressionEncoder.javaBean(DeviceData.class)); // streaming Dataset with IOT device data

// Select the devices which have signal more than 10
df.select("device").where("signal > 10"); // using untyped APIs
ds.filter((FilterFunction<DeviceData>) value -> value.getSignal() > 10)
  .map((MapFunction<DeviceData, String>) value -> value.getDevice(), Encoders.STRING());

// Running count of the number of updates for each device type
df.groupBy("deviceType").count(); // using untyped API

// Running average signal for each device type
ds.groupByKey((MapFunction<DeviceData, String>) value -> value.getDeviceType(), Encoders.STRING())
  .agg(typed.avg((MapFunction<DeviceData, Double>) value -> value.getSignal()));
```



你还可以将一个流式DataFrame/Dataset注册为一个临时视图，然后在其上应用SQL命令。

- Scala

```scala
df.createOrReplaceTempView("updates")
spark.sql("select count(*) from updates")  // returns another streaming DF
```



- Java

```java
df.createOrReplaceTempView("updates");
spark.sql("select count(*) from updates");  // returns another streaming DF
```



注意，你可以使用 `df.isStreaming` 来识别一个DataFrame/Dataset是否有流式数据。

- Scala

```scala
df.isStreaming
```

- Java

```java
df.isStreaming()
```



### `Event-time` 上的 `Window` 操作 

滑动 `event-time` 窗口上的聚合与结构化流非常简单，非常类似于分组聚合。在分组聚合中，为用户指定的分组列中的每个惟一值维护聚合值(例如计数)。对于基于窗口的聚合，将为每个窗口维护聚合值，其中包含一行的 `event-time` 。让我们用一个例子来理解它。

想象一下，我们的 [快速示例](#简单例子) 被修改了，流现在包含了行以及该行生成的时间。我们想要计算10分钟内窗口内的单词数，而不是运行单词计数，每5分钟更新一次。也就是说，在10分钟窗口内收到的单词数为12:00 - 12:10、12:05 - 12:15、12:10 - 12:20等。请注意，12:00 - 12:10表示在12:00之后但在12:10之前到达的数据。现在，考虑12:07收到的一个单词。此字应增加与两个窗口12:00 - 12:10和12:05 - 12:15对应的计数。因此计数将被分组键(即单词)和窗口(可以从 `event-time` 计算)索引。

结果表将类似如下所示。

![Window Operations](img/structured-streaming-window.png)

由于这种窗口操作类似于分组，因此在代码中可以使用groupBy()和window()操作来表示窗口聚合。在下面的Scala/Java/Python示例中，你可以看到完整的代码。

- Scala

```scala
import spark.implicits._

val words = ... // streaming DataFrame of schema { timestamp: Timestamp, word: String }

// Group the data by window and word and compute the count of each group
val windowedCounts = words.groupBy(
  window($"timestamp", "10 minutes", "5 minutes"),
  $"word"
).count()
```

- Java

```java
Dataset<Row> words = ... // streaming DataFrame of schema { timestamp: Timestamp, word: String }

// Group the data by window and word and compute the count of each group
Dataset<Row> windowedCounts = words.groupBy(
  functions.window(words.col("timestamp"), "10 minutes", "5 minutes"),
  words.col("word")
).count();
```



#### 处理 `Late Data` 和 `Watermark`

现在，考虑如果其中一个事件延迟, 应用程序会发生什么。例如，在12:04生成的单词(即 `event-time` )可以在12:11被应用程序接收。应用程序应该使用时间12:04而不是12:11来更新窗口12:00 - 12:10的旧计数。这在我们的基于窗口的分组中很自然地发生——结构化流可以在很长一段时间内保持部分聚合的中间状态，以便延期到达数据可以正确地更新旧窗口的聚合值，如下所示。

![Handling Late Data](img/structured-streaming-late-data.png)

但是，为了连续几天运行这个查询，系统必须限制它在内存中积累的中间状态的数量。这意味着系统需要知道何时可以将旧的聚合从内存状态中删除，因为应用程序将不再接收该聚合的最新数据。为了实现这一点，在Spark 2.1中，我们引入了 `Watermark` ，它允许引擎自动跟踪数据中的当前 `event-time` ，并尝试相应地清除旧状态。你可以通过指定 `event-time` 列和数据在 `event-time` 方面的预期延迟的阈值来定义查询的 `Watermark` 。 对于在T时结束的特定窗口，引擎将维护状态并允许延期数据更新状态，直到(`max event time seen by the engine - late threshold > T`)。换句话说，阈值内的延迟数据将被聚合，但是超过阈值的数据将开始被删除(有关确切的保证，请参阅[后面](#semantic-guarantees-of-aggregation-with-watermarking)的部分)。让我们用一个例子来理解这一点。我们可以使用如下所示的 `withWatermark()` 在前面的示例中轻松定义 `Watermark` 。

- Scala

```scala
import spark.implicits._

val words = ... // streaming DataFrame of schema { timestamp: Timestamp, word: String }

// Group the data by window and word and compute the count of each group
val windowedCounts = words
    .withWatermark("timestamp", "10 minutes")
    .groupBy(
        window($"timestamp", "10 minutes", "5 minutes"),
        $"word")
    .count()
```



- Java

```java
Dataset<Row> words = ... // streaming DataFrame of schema { timestamp: Timestamp, word: String }

// Group the data by window and word and compute the count of each group
Dataset<Row> windowedCounts = words
    .withWatermark("timestamp", "10 minutes")
    .groupBy(
        window(col("timestamp"), "10 minutes", "5 minutes"),
        col("word"))
    .count();
```



在本例中，我们将查询的 `Watermark` 定义为“时间戳”列的值，并将“10分钟”定义为允许数据延迟的阈值。如果此查询以更新输出模式运行(稍后将在[输出模式](#`Output` 模式)一节中讨论)，则引擎将不断更新结果表中的窗口计数，直到该窗口比 `Watermark` 的时间更早，这比“时间戳”列中的当前 `event-time` 晚10分钟。这是一个例子。

![Watermarking in Update Mode](img/structured-streaming-watermark-update-mode.png)

如图所示，引擎跟踪的最大 `event-time` 为蓝色虚线，每个触发器开始处的 `Watermark` 设置为(`max event time - '10 mins'`)红线。例如，当引擎观察数据(`12:14,dog`)时，它将下一个触发器的 `Watermark` 设置为12:04。这个 `Watermark` 让引擎保持中间状态额外10分钟，允许延迟到达数据被计数。例如，数据(`12:09,cat`)出现故障和延迟，并且落在窗口 `12:00 - 12:10` 和 `12:05 - 12:15`。由于它仍然在触发器中的 `Watermark` 12:04 时间后面，因此引擎仍然将中间计数作为状态维护，并正确更新相关窗口的计数。但是，当 `Watermark` 被更新到12:11时，窗口(`12:00 - 12:10`)的中间状态被清除，所有后续数据(例如(`12:04, donkey`))被认为“too late”，因此被忽略。注意，在每个 `trigger` 之后，更新计数(即紫色行)被写入 `sink` 作为    `trigger` 输出，这是由更新模式决定的。

某些 `Sink` (例如文件)可能不支持更新模式所需的细粒度 (`fine-grain`) 更新。为了与它们一起工作，我们还支持 `Append Mode`，其中只将最终计数写入 `sink`。如下图所示。

注意，在非流 `Dataset` 上使用带 `Watermark` 是不允许的。由于 `Watermark` 不应该以任何方式影响任何批量查询，我们将直接忽略它。

![Watermarking in Append Mode](img/structured-streaming-watermark-append-mode.png)

与前面的更新模式类似，引擎维护每个窗口的中间计数。但是，部分计数不会更新到结果表，也不会写入sink。引擎等待“10分钟”以计算最后的日期，然后删除窗口< `Watermark` 的中间状态，并将最后的计数追加到 `Result Table`/`sink`。例如，窗口 `12:00 - 12:10` 的最终计数仅在 `Watermark` 更新到12:11之后才追加到结果表。

**用于 `Watermark` 的条件为清除聚集状态**

需要注意的是，为了清除聚合查询中的状态，必须满足以下条件(*从Spark 2.1.1开始，以后可能会更改*)。

- **Output 模式必须是 `Append` 或 `Update `**： `Complete mode` 需要保存所有聚合数据，因此不能使用 `Watermark` 来删除中间状态。有关每个输出模式的语义的详细说明，请参阅[输出模式](#`Output` 模式)一节。
- 聚合必须具有 `event-time` 列，或 `event-time` 列上的 `window`。
- 必须在与聚合中使用的时间戳列相同的列上调用  `withWatermark` 。例如 `df.withWatermark("time", "1 min").groupBy("time2").count()` 在 `Append` 输出模式下无效，因为 `Watermark` 是在与聚合列不同的列上定义的。
- 要使用 `watermark` 细节，必须在聚合之前调用 `withWatermark`。例如, `df.groupBy("time").count().withWatermark("time", "1 min")` 是无效的 `Append` 输出模式。

**带 `Watermark`聚合语义保证 **

-  `Watermark` 延迟(与 `Watermark` 一起设置)为“2小时”，保证引擎不会丢弃任何延迟小于2小时的数据。换句话说，任何在 `event-time` 上少于2小时之前处理的最新数据都将被聚合。

- 然而，这种保证只在一个方向上是严格的。延迟超过2小时的数据不保证被删除;它可能聚合，也可能不聚合。数据越是延迟，引擎处理它的可能性就越小。

  

### `Join` 操作

结构化流支持将 Stream `Dataset` `DataFrame` 与静态 `Dataset`/`DataFrame` 以及另一个流  `Dataset` / `DataFrame` 连接起来。Stream Join 的结果是增量生成的，类似于前一节中流聚合的结果。在本节中，我们将探讨在上述情况下支持哪种类型的连接(即 `inner`、`outer` 等)。请注意，在所有受支持的连接类型中，与流 `Dataset` `/DataFrame` 的连接的结果与和流中包含相同数据的静态 `Dataset` `/DataFrame`  的连接的结果完全相同。



### `Stream-static` 连接

自从在Spark 2.0中引入以来，结构化流就支持流和静态`Dataset` `/DataFrame` 之间的连接(`inner join` 和某种类型的 `outer join` )。下面是一个简单的例子。

- Scala

```scala
val staticDf = spark.read. ...
val streamingDf = spark.readStream. ...

streamingDf.join(staticDf, "type")          // inner equi-join with a static DF
streamingDf.join(staticDf, "type", "right_join")  // right outer join with a static DF  
```



- Java

```java
Dataset<Row> staticDf = spark.read(). ...;
Dataset<Row> streamingDf = spark.readStream(). ...;
streamingDf.join(staticDf, "type");         // inner equi-join with a static DF
streamingDf.join(staticDf, "type", "right_join");  // right outer join with a static DF
```



注意，`Stream-static` 连接不是有状态的，因此不需要状态管理。但是，目前还不支持一些类型的`stream-static` `outer join`。这些都列在这个 [连接部分的末尾](#支持指标 `Join` 流查询)。



### `Stream-stream` 连接

在Spark 2.3中，我们添加了对 `stream-stream` 连接的支持，也就是说，你可以连接两个流 `Dataset`/ `DataFrame`。在两个数据流之间生成连接结果的挑战在于，在任何时候，对于连接的两边， `Dataset` 的视图都是不完整的，这使得在输入之间查找匹配项变得更加困难。从一个输入流接收的任何行可以与从另一个输入流接收的任何将来的、尚未接收的行匹配。因此，对于这两个输入流，我们将过去的输入缓冲为流状态，这样我们就可以将每个未来的输入与过去的输入匹配起来，从而生成连接的结果。此外，与流聚合类似，我们自动处理延迟的、无序(`out-of-order`)的数据，并可以使用 `Watermark` 限制状态。让我们讨论一下受支持的不同类型的 `stream-stream` 连接以及如何使用它们。



####  Watermark `Inner Join`

支持任何类型列上的 `inner join` 以及任何类型的连接条件。但是，随着流的运行，流状态的大小将无限地增长，因为必须保存所有过去的输入，因为任何新的输入都可以与过去的任何输入匹配。为了避免无界状态，你必须定义附加的连接条件，使不确定的旧输入无法与未来的输入匹配，因此可以从状态中清除。换句话说，你必须在联接中执行以下附加步骤。

- 在两个输入上定义 `Watermark` 延迟，以便引擎知道延迟的程度(类似于流聚合)
- 在两个输入之间定义一个 `event-time` 约束，这样引擎就可以找出一个输入的旧行什么时候不需要(即不满足时间约束)与另一个输入匹配。这个约束可以用以下两种方式之一进行定义：
  - 时间范围连接条件(例如 `JOIN ON leftTime BETWEEN rightTime AND rightTime + INTERVAL 1 HOUR`)
  - `event-time` 窗口连接 (例如…`JOIN ON leftTimeWindow = rightTimeWindow`)。

让我们用一个例子来理解这一点。

假设我们想要加入一个广告印象流(当一个广告出现时)和另一个用户点击广告流，以关联什么时候印象导致了可货币化的点击。要允许在这个`stream-stream` 连接中进行状态清理，你必须如下所示指定 `Watermark` 延迟和时间约束。

1. `Watermark` 延迟: 例如，在 `event-time` 内，广告显示和相应的点击可能分别延迟2小时和3小时。
2. `event-time` 范围条件: 例如，在相应的印象之后0秒到1小时的时间范围内可以发生一次单击。

代码看起来是这样的。

- Scala

```scala
import org.apache.spark.sql.functions.expr

val impressions = spark.readStream. ...
val clicks = spark.readStream. ...

// Apply watermarks on event-time columns
val impressionsWithWatermark = impressions.withWatermark("impressionTime", "2 hours")
val clicksWithWatermark = clicks.withWatermark("clickTime", "3 hours")

// Join with event-time constraints
impressionsWithWatermark.join(
  clicksWithWatermark,
  expr("""
    clickAdId = impressionAdId AND
    clickTime >= impressionTime AND
    clickTime <= impressionTime + interval 1 hour
    """)
)
```



- Java

```java
import static org.apache.spark.sql.functions.expr

Dataset<Row> impressions = spark.readStream(). ...
Dataset<Row> clicks = spark.readStream(). ...

// Apply watermarks on event-time columns
Dataset<Row> impressionsWithWatermark = impressions.withWatermark("impressionTime", "2 hours");
Dataset<Row> clicksWithWatermark = clicks.withWatermark("clickTime", "3 hours");

// Join with event-time constraints
impressionsWithWatermark.join(
  clicksWithWatermark,
  expr(
    "clickAdId = impressionAdId AND " +
    "clickTime >= impressionTime AND " +
    "clickTime <= impressionTime + interval 1 hour ")
);
```



**带 `Watermark`的 `stream-stream` inner join 语义保证**

这与[带 `Watermark`聚合语义保证](#带 `Watermark`聚合语义保证)类似。 `Watermark` 延迟“2小时”保证了引擎不会丢失任何延迟小于2小时的数据。但是延迟超过2小时的数据可能会被处理，也可能不会被处理。

####  Watermark `Outer Join`

虽然 `Watermark` + `event-time` 约束对于 `inner join` 是可选的，但是对于左右 `outer join` 必须指定它们。这是因为，为了在外部连接中生成NULL结果，引擎必须知道输入行在将来什么时候与任何东西都不匹配。因此，必须指定 `Watermark` + `event-time` 约束才能生成正确的结果。因此，带有 `outer join` 的查询与前面的广告货币化示例非常相似，不同之处是有一个额外的参数将其指定为外连接。

- Scala

```scala
impressionsWithWatermark.join(
  clicksWithWatermark,
  expr("""
    clickAdId = impressionAdId AND
    clickTime >= impressionTime AND
    clickTime <= impressionTime + interval 1 hour
    """),
  joinType = "leftOuter"      // can be "inner", "leftOuter", "rightOuter"
 )
```



- Java

```java
impressionsWithWatermark.join(
  clicksWithWatermark,
  expr(
    "clickAdId = impressionAdId AND " +
    "clickTime >= impressionTime AND " +
    "clickTime <= impressionTime + interval 1 hour "),
  "leftOuter"                 // can be "inner", "leftOuter", "rightOuter"
);
```



**带 Watermark Outer Join Stream-Stream 语义保证**

关于 `Watermark` 延迟和是否删除数据，`Output join` 与 `Inner join` 具有相同的语义保证。

**警告**： 

关于如何生成外部结果，有几个重要的特征需要注意。

- *外部 `NULL` 结果将产生一个延迟，该延迟取决于指定的 `Watermark` 延迟和时间范围条件。* 这是因为引擎必须等待很长时间，以确保没有匹配上，将来也不会有更多的匹配。

- 在目前的 `micro-match` 引擎实现中， `Watermark` 是在一个`micro-match` 结束时进行的，下一个微批使用更新后的 `Watermark` 来清理状态并输出外部结果。因为我们只在需要处理新数据时才触发 `micro-batch` ，所以如果流中没有接收到新数据，则外部结果的生成可能会延迟。简而言之，如果连接的两个输入流中的任何一个在一段时间内都没有接收到数据，则外部(左或右两种情况)输出可能会延迟。

  

#### 支持指标 `Join` 流查询

<table class="table">
<tbody>
<tr>
<th>Left Input</th>
<th>Right Input</th>
<th>Join Type</th>
<th>&nbsp;</th>
</tr>
<tr>
<td>Static</td>
<td>Static</td>
<td>All types</td>
<td>支持，因为它不是一个数据 <code>stream</code>，即使它可以出现在一个 <code>stream</code> 查询操作</td>
</tr>
<tr>
<td rowspan="4">Stream</td>
<td rowspan="4">Static</td>
<td>Inner</td>
<td>支持，无状态</td>
</tr>
<tr>
<td>Left Outer</td>
<td>支持，无状态</td>
</tr>
<tr>
<td>Right Outer</td>
<td>不支持</td>
</tr>
<tr>
<td>Full Outer</td>
<td>不支持</td>
</tr>
<tr>
<td rowspan="4">Static</td>
<td rowspan="4">Stream</td>
<td>Inner</td>
<td>支持，无状态</td>
</tr>
<tr>
<td>Left Outer</td>
<td>不支持</td>
</tr>
<tr>
<td>Right Outer</td>
<td>支持，无状态</td>
</tr>
<tr>
<td>Full Outer</td>
<td>不支持</td>
</tr>
<tr>
<td rowspan="4">Stream</td>
<td rowspan="4">Stream</td>
<td>Inner</td>
<td>支持, 可选指定两边的 <code>watermark</code> + <code>时间</code>限制的状态清除</td>
</tr>
<tr>
<td>Left Outer</td>
<td>有条件支持，必须指定<code>watermark</code> 在右边+ <code>time</code> 限制正确的结果，可选指定<code>watermark</code> 在左边的所有状态清除</td>
</tr>
<tr>
<td>Right Outer</td>
<td>有条件支持, 必须指定<code>watermark</code> 在左边+ <code>time</code> 限制正确的结果，可选指定<code>watermark</code> 在右边的所有状态清除</td>
</tr>
<tr>
<td>Full Outer</td>
<td>不支持</td>
</tr>
</tbody>
</table>



关于支持的连接的更多细节:

- 连接可以级联，也就是说，可以执行 `df1.join(df2, ...).join(df3, ...).join(df4, ....)`.

- 从Spark 2.4开始，你只能在查询处于 `Append` 输出模式时使用连接。还不支持其他输出模式。

- 从Spark 2.4开始，你不能在联接之前使用其他 `non-map-like` 的操作。以下是一些不能使用的例子。
  - 不能在连接之前使用流聚合。

  - 在连接之前，不能在 `Update` 模式中使用 `mapGroupsWithState` 和 `flatMapGroupsWithState`。

    

## Stream 重复数据删除(`Deduplication`)

你可以使用事件中的唯一标识符来删除流中的重复数据记录。这与使用唯一标识符列的静态重复数据删除完全相同。查询将存储来自以前记录的必要数据量，以便可以过滤重复的记录。与聚合类似，可以使用也可以不使用 `Watermark` 来进行重复数据的删除 。

- 使用 `Watermark`  : 如果重复记录到达的时间有上限，那么你可以在 `event-time` 列上定义 `Watermark` ，并使用 `guid` 和 `event-time` 列来删除重复记录。该查询将使用 `Watermark` 从过去的记录中删除旧的状态数据，这些数据不再期望得到任何重复。这限制了查询必须维护的状态的数量。
- 无 `Watermark` ： 由于对重复记录何时到达没有限制，查询回将所有过去记录的数据存储在状态信息中。



- Scala

```scala
val streamingDf = spark.readStream. ...  // columns: guid, eventTime, ...

// Without watermark using guid column
streamingDf.dropDuplicates("guid")

// With watermark using guid and eventTime columns
streamingDf
  .withWatermark("eventTime", "10 seconds")
  .dropDuplicates("guid", "eventTime")
```



- Java

```java
Dataset<Row> streamingDf = spark.readStream(). ...;  // columns: guid, eventTime, ...

// Without watermark using guid column
streamingDf.dropDuplicates("guid");

// With watermark using guid and eventTime columns
streamingDf
  .withWatermark("eventTime", "10 seconds")
  .dropDuplicates("guid", "eventTime");
```



### 处理多个 `Watermark` 的策略

一个流查询可以有多个统一或连接在一起的输入流。每个输入流都有一个不同的延期数据阈值，需要对有状态操作进行容忍。在每个输入流上使用 `withWatermarks("eventTime", delay)` 的阈值来指定这些阈值。例如，考虑一个具有 `inputStream1` 和  `inputStream1` 之间的 `stream-stream` 连接的查询。

```scala
inputStream1.withWatermark("eventTime1", "1 hour")
  .join(
    inputStream2.withWatermark("eventTime2", "2 hours"),
    joinCondition)
```



在执行查询时，结构化流分别跟踪每个输入流中看到的最大 `event-time` ，根据相应的延迟计算 `Watermark` ，并选择单个全局 `Watermark` 用于有状态操作。默认情况下，选择最小值作为全局 `Watermark` ，因为它可以确保当一个流落后于其他流(例如，其中一个流由于上游失败而停止接收数据)时，不会因为太晚而意外丢失数据。换句话说，全局 `Watermark` 将以最慢流的速度安全地移动，查询输出也将相应地延迟。

然而，在某些情况下，你可能希望获得更快的结果，即使这意味着从最慢的流中删除数据。自从 Spark 2.4 以来，你可以通过设置SQL配置 `spark.sql.streaming.multipleWatermarkPolicy` 为 `max`(默认值： `min`) 来设置多重 `Watermark` 策略，选择最大值作为全局 `Watermark` 。这让全局 `Watermark` 移动的速度最快的流。然而，作为一个副作用，来自较慢的流的数据将被积极地丢弃。因此，请明智地使用此配置。

### 任意的 `Stateful Operation`

许多场景需要比聚合更高级的有状态操作。例如，在许多场景中，必须从事件的数据流跟踪会话。要进行这种会话，必须将任意类型的数据保存为状态，并在每个触发器中使用数据流事件对状态执行任意操作。从Spark 2.2开始，这可以使用 `mapGroupsWithState` 操作和功能更强大的 `flatMapGroupsWithState` 操作来完成。这两个操作都允许你在分组 `Dataset` 上应用用户定义的代码来更新用户定义的状态。要了解更多的具体细节，请看API文档([Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.streaming.GroupState)/[Java](http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/GroupState.html))和示例([Scala](https://github.com/apache/spark/blob/v2.4.5/examples/src/main/scala/org/apache/spark/examples/sql/streaming/StructuredSessionization.scala)/[Java](https://github.com/apache/spark/blob/v2.4.5/examples/src/main/java/org/apache/spark/examples/sql/streaming/JavaStructuredSessionization.java))。

### 不支持的 `Operation`

有一些DataFrame/Dataset操作是流数据不支持的。其中一些如下。

- 流 `Dataset` 中还不支持多个流聚合(即流  DF 上的聚合链)。
- 在 Dataset 流中， 不支持 `Limit` 和 获取开始N行（类似 topN）操作
- 在 Dataset 流中， 不支持 `Distinct` 操作
-  Dataset 流仅在聚合之后并在 `Compleate Output Mode` 下，才支持 `Sort` 操作。
-  `Dataset` 流， 不支持少数类型的 `outer join`。有关更多详细信息，请参阅[支持指标 `Join` 流查询](#支持指标 `Join` 流查询)。

此外，有些 `Dataset` 方法在`Dataset` 流上不支持的。它们是将立即运行查询并返回结果的操作，这在`Dataset` 流上是没有意义的。相反，这些功能可以通过显式地启动一个流查询来实现(参见下一节)。

- `count()` -  `Dataset` 流不能返回单个 `count`。相反，使用 `ds.groupBy().count()`，它返回一个包含正在运行的计数的流 `Dataset` 。
- `foreach()` - 使用 `ds.writeStream.foreach(...)` (参见下一节) 代替。
- `show()` - 使用控制台 `sink` 代替(参见下一节)。

如果你尝试任何这些操作，你将看到一个 `AnalysisException`，如“ `Dataset` / `DataFrame` 流中， 操作 `XYZ` 是不支持的”。虽然其中一些可能会在未来的Spark版本中得到支持，但其他一些则很难在流数据上有效地实现。例如，不支持对输入流进行排序，因为它需要跟踪流中收到的所有数据。因此，这基本上很难有效执行。



## 开始 `Stream Query`

一旦定义了 `DataFrame/Dataset` 的最终结果，剩下的就是开始流计算了。为此，必须使用通过 `Dataset.writeStream()` 返回的 `DataStreamWriter` ([Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.streaming.DataStreamWriter)/[Java](http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/DataStreamWriter.html)/[Python](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.streaming.DataStreamWriter)文档)。你必须在这个接口中指定以下内容的一个或多个。

- 输出 `sink` 的详细信息**:**数据格式、位置等。

- *输出模式* :  指定写入输出 `sink` 的内容。

- *query 名称*:  可选, 指定 query 的唯一名称以进行标识。

- *触发间隔* ： 可选， 指定触发间隔。如果没有指定，系统将在前面的处理完成后立即检查新数据的可用性。如果由于前一个处理未完成而错过了触发时间，则系统将立即触发处理。

- *`checkpoint` 位置*： 对于一些可以保证 `end-to-end` `fault-tolerance` 的输出 `sink`，指定系统将写入所有 `checkpoint` 信息的位置。这应该是 `HDFS-compatible` 的 `fault-tolerance` 文件系统中的一个目录。下一节将更详细地讨论 `checkpoint` 的语义。

  

### `Output` 模式

有几种类型的输出模式。

- **Append mode(默认）** - 这是默认模式， (因最后一个触发器（trigger）会输出到 `sink`) 只有新行才会添加到 `Result Table`。这只适用于那些添加到结果表的行永远不会改变的查询。因此，此模式保证每个行只输出一次(假设 `fault-tolerance` sink)。例如，只有 `select`、`where`、`map`、`flatMap`、`filter`、`join`等查询支持 Append 模式。
- `Complete mode` - 每个触发器触发后， 整个 `Result Table` 会输出到 `sink`。 `Complete mode ` 支持聚合查询
- `Update 模式` - (Spark 2.1.1 版本后可用) 只有在最后一个触发器之后， 更新的结果表中的行才会输出到 Sink。更多信息将在未来的版本中添加。

不同类型的流 Query 支持不同的输出模式。这是兼容性选项。



<table class="table">
<tbody>
<tr>
<th>Query Type</th>
<th>&nbsp;</th>
<th>Supported Output Modes</th>
<th>Notes</th>
</tr>
<tr>
<td rowspan="2">Queries with aggregation</td>
<td>Aggregation on event-time with watermark</td>
<td>Append, Update, Complete</td>
<td>Append mode uses watermark to drop old aggregation state. But the output of a windowed aggregation is delayed the late threshold specified in `withWatermark()` as by the modes semantics, rows can be added to the Result Table only once after they are finalized (i.e. after watermark is crossed). See the&nbsp;<a href="http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#handling-late-data-and-watermarking">Late Data</a>&nbsp;section for more details.<br /><br />Update mode uses watermark to drop old aggregation state.<br /><br />Complete mode does not drop old aggregation state since by definition this mode preserves all data in the Result Table.</td>
</tr>
<tr>
<td>Other aggregations</td>
<td>Complete, Update</td>
<td>Since no watermark is defined (only defined in other category), old aggregation state is not dropped.<br /><br />Append mode is not supported as aggregates can update thus violating the semantics of this mode.</td>
</tr>
<tr>
<td colspan="2">Queries with&nbsp;<code>mapGroupsWithState</code></td>
<td>Update</td>
<td>&nbsp;</td>
</tr>
<tr>
<td rowspan="2">Queries with&nbsp;<code>flatMapGroupsWithState</code></td>
<td>Append operation mode</td>
<td>Append</td>
<td>Aggregations are allowed after&nbsp;<code>flatMapGroupsWithState</code>.</td>
</tr>
<tr>
<td>Update operation mode</td>
<td>Update</td>
<td>Aggregations not allowed after&nbsp;<code>flatMapGroupsWithState</code>.</td>
</tr>
<tr>
<td colspan="2">Queries with&nbsp;<code>joins</code></td>
<td>Append</td>
<td>Update and Complete mode not supported yet. See the&nbsp;<a href="http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#support-matrix-for-joins-in-streaming-queries">support matrix in the Join Operations section</a>&nbsp;for more details on what types of joins are supported.</td>
</tr>
<tr>
<td colspan="2">Other queries</td>
<td>Append, Update</td>
<td>Complete mode not supported as it is infeasible to keep all unaggregated data in the Result Table.</td>
</tr>
</tbody>
</table>



### `Output Sink`

有几种类型的 `build-in` 输出 `sink`。

- **File sink**: 将输出保存到一个目录中

```scala
writeStream
    .format("parquet")        // can be "orc", "json", "csv", etc.
    .option("path", "path/to/destination/dir")
    .start()
```



- **Kafka sink** : `kafka ` 中存储一个或多个 `topic` 的输出。

```scala
writeStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
    .option("topic", "updates")
    .start()
```



- **Foreach sink** - 对输出中的记录执行任意计算。有关更多细节，请参阅本节后面的内容。

```scala
writeStream
    .foreach(...)
    .start()
```



- **Console sink (for debugging)** -每次触发时将输出打印到控制台/ stdout。 都支持“Append ”和“Complete ”输出模式。 这应该用于低数据量的调试目的，因为在每次触发后，整个输出被收集并存储在驱动程序的内存中。

```scala
writeStream
    .format("console")
    .start()
```

-  **Memory sink (for debugging)** - 输出作为内存表存储在内存中。 都支持“Append ”和“Complete ”输出模式。 由于整个输出被收集并存储在驱动程序的内存中，所以应用于低数据量的调试目的。 因此，请谨慎使用

```scala
writeStream
    .format("memory")
    .queryName("tableName")
    .start()
```



某些 `sink` 不 `fault-tolerance`，因为它们不保证输出的持久性，仅用于调试目的。 请参阅上一节关于 [fault-tolerance semantic`](#容错语义) 的部分。 以下是Spark中所有 `Sink` 的详细信息。



<table class="table">
<tbody>
<tr>
<th>Sink</th>
<th>Supported Output Modes</th>
<th>Options</th>
<th>Fault-tolerant</th>
<th>Notes</th>
</tr>
<tr>
<td><strong>File Sink</strong></td>
<td>Append</td>
<td><code>path</code>: path to the output directory, must be specified.<br /><br />For file-format-specific options, see the related methods in DataFrameWriter (<a href="http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameWriter">Scala</a>/<a href="http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/DataFrameWriter.html">Java</a>/<a href="http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrameWriter">Python</a>/<a href="http://spark.apache.org/docs/latest/api/R/write.stream.html">R</a>). E.g. for "parquet" format options see&nbsp;<code>DataFrameWriter.parquet()</code></td>
<td>Yes (exactly-once)</td>
<td>Supports writes to partitioned tables. Partitioning by time may be useful.</td>
</tr>
<tr>
<td><strong>Kafka Sink</strong></td>
<td>Append, Update, Complete</td>
<td>See the&nbsp;<a href="structured-streaming-kafka-integration-zh.md">Kafka Integration Guide</a></td>
<td>Yes (at-least-once)</td>
<td>More details in the&nbsp;<a href="structured-streaming-kafka-integration-zh.md">Kafka Integration Guide</a></td>
</tr>
<tr>
<td><strong>Foreach Sink</strong></td>
<td>Append, Update, Complete</td>
<td>None</td>
<td>Yes (at-least-once)</td>
<td>More details in the&nbsp;<a href="#使用 `Foreach` 和 `ForeachBatch`">next section</a></td>
</tr>
<tr>
<td><strong>ForeachBatch Sink</strong></td>
<td>Append, Update, Complete</td>
<td>None</td>
<td>Depends on the implementation</td>
<td>More details in the&nbsp;<a href="#使用 `Foreach` 和 `ForeachBatch`">next section</a></td>
</tr>
<tr>
<td><strong>Console Sink</strong></td>
<td>Append, Update, Complete</td>
<td><code>numRows</code>: Number of rows to print every trigger (default: 20)<br /><code>truncate</code>: Whether to truncate the output if too long (default: true)</td>
<td>No</td>
<td>&nbsp;</td>
</tr>
<tr>
<td><strong>Memory Sink</strong></td>
<td>Append, Complete</td>
<td>None</td>
<td>No. But in Complete Mode, restarted query will recreate the full table.</td>
<td>Table name is the query name.</td>
</tr>
</tbody>
</table>
<p>&nbsp;</p>



请注意，您必须调用 `start()`  来实际启动查询的执行。 这将返回一个 `StreamingQuery` 对象，它是连续运行执行的 `handle`。 您可以使用此对象来管理查询，我们将在下一小节中讨论。 现在，让我们通过几个例子了解所有这些。



- Scala

  ```scala
  // ========== DF with no aggregations ==========
  val noAggDF = deviceDataDf.select("device").where("signal > 10")   
  
  // Print new data to console
  noAggDF
    .writeStream
    .format("console")
    .start()
  
  // Write new data to Parquet files
  noAggDF
    .writeStream
    .format("parquet")
    .option("checkpointLocation", "path/to/checkpoint/dir")
    .option("path", "path/to/destination/dir")
    .start()
  
  // ========== DF with aggregation ==========
  val aggDF = df.groupBy("device").count()
  
  // Print updated aggregations to console
  aggDF
    .writeStream
    .outputMode("complete")
    .format("console")
    .start()
  
  // Have all the aggregates in an in-memory table
  aggDF
    .writeStream
    .queryName("aggregates")    // this query name will be the table name
    .outputMode("complete")
    .format("memory")
    .start()
  
  spark.sql("select * from aggregates").show()   // interactively query in-memory table
  ```

- Java

  ```java
  // ========== DF with no aggregations ==========
  Dataset<Row> noAggDF = deviceDataDf.select("device").where("signal > 10");
  
  // Print new data to console
  noAggDF
    .writeStream()
    .format("console")
    .start();
  
  // Write new data to Parquet files
  noAggDF
    .writeStream()
    .format("parquet")
    .option("checkpointLocation", "path/to/checkpoint/dir")
    .option("path", "path/to/destination/dir")
    .start();
  
  // ========== DF with aggregation ==========
  Dataset<Row> aggDF = df.groupBy("device").count();
  
  // Print updated aggregations to console
  aggDF
    .writeStream()
    .outputMode("complete")
    .format("console")
    .start();
  
  // Have all the aggregates in an in-memory table
  aggDF
    .writeStream()
    .queryName("aggregates")    // this query name will be the table name
    .outputMode("complete")
    .format("memory")
    .start();
  
  spark.sql("select * from aggregates").show();   // interactively query in-memory table
  ```



##### 使用 `Foreach` 和 `ForeachBatch`

`foreach` 和 `foreachBatch` 操作允许你对流查询的输出应用任意操作和写入逻辑。它们的使用场景略有不同:

- `foreach` - 允许在每一行上执行自定义写逻辑，

- `foreachBatch` - 允许在每个微批的输出上执行任意操作和自定义逻辑。让我们更详细地了解它们的用法。

  

**ForeachBatch**

`foreachBatch(...)` 允许你指定一个函数，该函数在流查询的每个 `micro-batch` 的输出数据上执行。从Spark 2.4开始，Scala、Java和Python都支持这个特性。它有一个 `DataFrame`或 `Dataset` 的两个参数:`

- `micro-batch` 输出数据
- `micro-batch` 唯一 ID

- Scala

```scala
streamingDF.writeStream.foreachBatch {
  (batchDF: DataFrame, batchId: Long) => 
  // Transform and write batchDF
}.start()
```



- Java

```java
streamingDatasetOfString.writeStream().foreachBatch(
  new VoidFunction2<Dataset<String>, Long> {
    public void call(Dataset<String> dataset, Long batchId) {
      // Transform and write batchDF
    }    
  }
).start();

streamingDatasetOfString.writeStream().foreachBatch (
  new VoidFunction2<Dataset<String>, Long> {
    public void call (Dataset<String> dataset, Long batchId) {
      // Transform and write batchDF
    }
  }
).start();
```



使用 `foreachBatch` ，你可以执行以下操作。

- **重用现有的批处理数据源** - 对于许多存储系统，可能还没有可用的流 `Sink`，但可能已经存在用于批处理查询的数据写入器。使用 `foreachBatch`，你可以在每个 `micro-batch` 的输出上使用批数据写入器。

- **写入多个位置**  - 如果你想将流查询的输出写入多个位置，那么你可以简单地多次写入输出`DataFrame/Dataset` 。但是，每次写操作都会导致重新计算输出数据(包括可能的重新读取输入数据)。为了避免重复计算，你应该缓存输出的DataFrame/Dataset，将其写入多个位置，然后释放它。这是一个 `outline`(轮廓，大纲)。

```scala
streamingDF.writeStream.foreachBatch { (batchDF: DataFrame, batchId: Long) =>
  batchDF.persist()
  batchDF.write.format(...).save(...)  // location 1
  batchDF.write.format(...).save(...)  // location 2
  batchDF.unpersist()
}
```



- **应用额外的 DataFrame操作** - 在流 `DataFrame` 中不支持许多 `DataFrame`和 `Dataset` 操作，因为在这些情况下 `Spark` 不支持生成增量计划。使用 `foreachBatch`，你可以在每个 `micro-batch` 输出上应用这些操作。但是，你必须自己考虑执行该操作的 `end-to-end` 语义。

  

**注意:**

- 默认情况下，`foreachBatch` 只提供 `at-least-once` 写保证。但是，你可以使用提供给函数的 `batchId` 来重复输出，并获得 `exactly-once` 的保证。

- `foreachBatch` 不能使用连续处理模式，因为它基本上依赖于流查询的 `micro-batch` 执行。如果以连续模式写入数据，则使用 `foreach`。

  

**Foreach**

如果foreachBatch不能满足要求(例如，对应的批数据写入器不存在，或者是连续处理模式)，那么你可以使用foreach来表达你的自定义写入器逻辑。具体来说，你可以通过将数据写入逻辑划分为三种方法来表达:open、process和close。从Spark 2.4开始，foreach就可以在Scala、Java和Python中使用。

- 在Scala中，你必须扩展类 `ForeachWriter(docs)`

```scala
streamingDatasetOfString.writeStream.foreach(
  new ForeachWriter[String] {

    def open(partitionId: Long, version: Long): Boolean = {
      // Open connection
    }

    def process(record: String): Unit = {
      // Write string to connection
    }

    def close(errorOrNull: Throwable): Unit = {
      // Close the connection
    }
  }
).start()
```



- 在 Java 中，你必须扩展类 `ForeachWriter(docs)`

```java
streamingDatasetOfString.writeStream().foreach(
  new ForeachWriter[String] {

    @Override public boolean open(long partitionId, long version) {
      // Open connection
    }

    @Override public void process(String record) {
      // Write string to connection
    }

    @Override public void close(Throwable errorOrNull) {
      // Close the connection
    }
  }
).start();
```



**执行语义** 当流查询开始时，`Spark` 以如下方式调用函数或对象的方法:

- 此对象的单个副本负责查询中单个任务生成的所有数据。换句话说，一个实例负责处理以分布式方式生成的数据的一个分区。

- 此对象必须是可序列化的，因为每个任务将获得所提供对象的一个新的序列化-反序列化副本。因此，强烈建议对写入数据进行任何初始化(例如。在调用open()方法之后完成(打开连接或启动事务)，这意味着任务已经准备好生成数据。

- 这些方法的生命周期如下:

  - 对于带有 `partition_id` 的每个分区:
    - 对于带有 `epoch_id` 的流数据的每  `batch/epoch`:
      - 方法 `open(partitionId, epochId)` 被调用。
      - 如果 `open(...)` 返回 true，那么对于 `partition` 和 `batch/epoch` 中的每一行，都将调用方法 `process(row)`。
      - 方法 `close(error)` 是在处理行时发生错误(如果有的话)时调用的。

- `close()` 方法(如果存在)在open()方法存在并成功返回时调用(与返回值无关)，除非 `JVM`或 `Python` 进程中途崩溃。

- **Note**: Spark不保证相同的输出(partitionId, epochId)，所以不能实现重复数据删除 (partitionId, epochId) 。例如，source由于某些原因提供了不同数量的分区，Spark optimization改变了分区数量，等等。详见[SPARK-28650](https://issues.apache.org/jira/browse/SPARK-28650)。如果你需要重复数据删除，试试 `foreachBatch`。

  

### Trigger

流查询的 `trigger` 设置定义了流数据处理的时间, 该查询是作为具有固定批处理间隔的 `micro-batch` 查询执行，还是作为连续处理查询执行。下面是支持的不同类型的 `trigger`。

<table class="table">
<tbody>
<tr>
<th>Trigger Type</th>
<th>Description</th>
</tr>
<tr>
<td><em>unspecified (default)</em></td>
<td>If no trigger setting is explicitly specified, then by default, the query will be executed in micro-batch mode, where micro-batches will be generated as soon as the previous micro-batch has completed processing.</td>
</tr>
<tr>
<td><strong>Fixed interval micro-batches</strong></td>
<td>The query will be executed with micro-batches mode, where micro-batches will be kicked off at the user-specified intervals.
<ul>
<li>If the previous micro-batch completes within the interval, then the engine will wait until the interval is over before kicking off the next micro-batch.</li>
<li>If the previous micro-batch takes longer than the interval to complete (i.e. if an interval boundary is missed), then the next micro-batch will start as soon as the previous one completes (i.e., it will not wait for the next interval boundary).</li>
<li>If no new data is available, then no micro-batch will be kicked off.</li>
</ul>
</td>
</tr>
<tr>
<td><strong>One-time micro-batch</strong></td>
<td>The query will execute *only one* micro-batch to process all the available data and then stop on its own. This is useful in scenarios you want to periodically spin up a cluster, process everything that is available since the last period, and then shutdown the cluster. In some case, this may lead to significant cost savings.</td>
</tr>
<tr>
<td><strong>Continuous with fixed checkpoint interval</strong><br /><em>(experimental)</em></td>
<td>The query will be executed in the new low-latency, continuous processing mode. Read more about this in the&nbsp;<a href="http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#continuous-processing">Continuous Processing section</a>&nbsp;below.</td>
</tr>
</tbody>
</table>

下面是一些代码示例。

- Scala

```scala
import org.apache.spark.sql.streaming.Trigger

// Default trigger (runs micro-batch as soon as it can)
df.writeStream
  .format("console")
  .start()

// ProcessingTime trigger with two-seconds micro-batch interval
df.writeStream
  .format("console")
  .trigger(Trigger.ProcessingTime("2 seconds"))
  .start()

// One-time trigger
df.writeStream
  .format("console")
  .trigger(Trigger.Once())
  .start()

// Continuous trigger with one-second checkpointing interval
df.writeStream
  .format("console")
  .trigger(Trigger.Continuous("1 second"))
  .start()
```



- Java

```java
import org.apache.spark.sql.streaming.Trigger

// Default trigger (runs micro-batch as soon as it can)
df.writeStream
  .format("console")
  .start();

// ProcessingTime trigger with two-seconds micro-batch interval
df.writeStream
  .format("console")
  .trigger(Trigger.ProcessingTime("2 seconds"))
  .start();

// One-time trigger
df.writeStream
  .format("console")
  .trigger(Trigger.Once())
  .start();

// Continuous trigger with one-second checkpointing interval
df.writeStream
  .format("console")
  .trigger(Trigger.Continuous("1 second"))
  .start();
```



## `Stream` 管理查询

启动查询时创建的 `StreamingQuery` 对象可用于监视和管理查询。



- Scala

```scala
val query = df.writeStream.format("console").start()   // get the query object

query.id          // get the unique identifier of the running query that persists across restarts from checkpoint data

query.runId       // get the unique id of this run of the query, which will be generated at every start/restart

query.name        // get the name of the auto-generated or user-specified name

query.explain()   // print detailed explanations of the query

query.stop()      // stop the query

query.awaitTermination()   // block until query is terminated, with stop() or with error

query.exception       // the exception if the query has been terminated with error

query.recentProgress  // an array of the most recent progress updates for this query

query.lastProgress    // the most recent progress update of this streaming query
```



- Java

```java
StreamingQuery query = df.writeStream().format("console").start();   // get the query object

query.id();          // get the unique identifier of the running query that persists across restarts from checkpoint data

query.runId();       // get the unique id of this run of the query, which will be generated at every start/restart

query.name();        // get the name of the auto-generated or user-specified name

query.explain();   // print detailed explanations of the query

query.stop();      // stop the query

query.awaitTermination();   // block until query is terminated, with stop() or with error

query.exception();       // the exception if the query has been terminated with error

query.recentProgress();  // an array of the most recent progress updates for this query

query.lastProgress();    // the most recent progress update of this streaming query
```



你可以在单个SparkSession中启动任意数量的查询。它们将同时运行，共享集群资源。你可以使用`sparkSession.streams()` 来获得可用于管理当前活动查询的`StreamingQueryManager ([Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.streaming.StreamingQueryManager)/[Java](http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/StreamingQueryManager.html)/[Python](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.streaming.StreamingQueryManager)文档)。



- Scala

```scala
val spark: SparkSession = ...

spark.streams.active    // get the list of currently active streaming queries

spark.streams.get(id)   // get a query object by its unique id

spark.streams.awaitAnyTermination()   // block until any one of them terminates
```



- Java

```java
SparkSession spark = ...

spark.streams().active();    // get the list of currently active streaming queries

spark.streams().get(id);   // get a query object by its unique id

spark.streams().awaitAnyTermination();   // block until any one of them terminates
```



## `Stream` 监控查询

有多种方法可以监视 `active` 的流查询。你可以使用Spark的 `Dropwizard` 指标支持将指标推送到外部系统，或者以编程方式访问它们。



### 读指标交互

你能使用`streamingQuery.lastProgress()` 和 `streamingQuery.status()` 直接地获取到 `active ` 查询的当前状态和指标。 `streamingQuery.lastProgress()` 在 `Scala` 和 `Java` 中返回一个 `StreamingQueryProgress` 对象，在 `Python` 中返回一个具有相同字段的字典。它包含关于流的最后一个 `trigger` 中所取得进展的所有信息-处理了什么数据、处理速率、延迟等等。还有 `streamingQuery.recentProgress`返回最近几个进展的数组。

此外，`streamingQuery.status()`  在Scala和Java中返回一个 `StreamingQueryStatus` 对象，在 `Python` 中返回一个具有相同字段的字典。它提供了关于查询正在立即执行的操作的信息-触发器是否活动、数据是否正在处理等等。

这里有几个例子。



- Scala 

```scala
val query: StreamingQuery = ...

println(query.lastProgress)

/* Will print something like the following.

{
  "id" : "ce011fdc-8762-4dcb-84eb-a77333e28109",
  "runId" : "88e2ff94-ede0-45a8-b687-6316fbef529a",
  "name" : "MyQuery",
  "timestamp" : "2016-12-14T18:45:24.873Z",
  "numInputRows" : 10,
  "inputRowsPerSecond" : 120.0,
  "processedRowsPerSecond" : 200.0,
  "durationMs" : {
    "triggerExecution" : 3,
    "getOffset" : 2
  },
  "eventTime" : {
    "watermark" : "2016-12-14T18:45:24.873Z"
  },
  "stateOperators" : [ ],
  "sources" : [ {
    "description" : "KafkaSource[Subscribe[topic-0]]",
    "startOffset" : {
      "topic-0" : {
        "2" : 0,
        "4" : 1,
        "1" : 1,
        "3" : 1,
        "0" : 1
      }
    },
    "endOffset" : {
      "topic-0" : {
        "2" : 0,
        "4" : 115,
        "1" : 134,
        "3" : 21,
        "0" : 534
      }
    },
    "numInputRows" : 10,
    "inputRowsPerSecond" : 120.0,
    "processedRowsPerSecond" : 200.0
  } ],
  "sink" : {
    "description" : "MemorySink"
  }
}
*/


println(query.status)

/*  Will print something like the following.
{
  "message" : "Waiting for data to arrive",
  "isDataAvailable" : false,
  "isTriggerActive" : false
}
*/
```



- Java

```java
StreamingQuery query = ...

System.out.println(query.lastProgress());
/* Will print something like the following.

{
  "id" : "ce011fdc-8762-4dcb-84eb-a77333e28109",
  "runId" : "88e2ff94-ede0-45a8-b687-6316fbef529a",
  "name" : "MyQuery",
  "timestamp" : "2016-12-14T18:45:24.873Z",
  "numInputRows" : 10,
  "inputRowsPerSecond" : 120.0,
  "processedRowsPerSecond" : 200.0,
  "durationMs" : {
    "triggerExecution" : 3,
    "getOffset" : 2
  },
  "eventTime" : {
    "watermark" : "2016-12-14T18:45:24.873Z"
  },
  "stateOperators" : [ ],
  "sources" : [ {
    "description" : "KafkaSource[Subscribe[topic-0]]",
    "startOffset" : {
      "topic-0" : {
        "2" : 0,
        "4" : 1,
        "1" : 1,
        "3" : 1,
        "0" : 1
      }
    },
    "endOffset" : {
      "topic-0" : {
        "2" : 0,
        "4" : 115,
        "1" : 134,
        "3" : 21,
        "0" : 534
      }
    },
    "numInputRows" : 10,
    "inputRowsPerSecond" : 120.0,
    "processedRowsPerSecond" : 200.0
  } ],
  "sink" : {
    "description" : "MemorySink"
  }
}
*/


System.out.println(query.status());
/*  Will print something like the following.
{
  "message" : "Waiting for data to arrive",
  "isDataAvailable" : false,
  "isTriggerActive" : false
}
*/
```



### 使用 Async API 编程输出指标报告

你还可以通过附加 `StreamingQueryListener` ([Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.streaming.StreamingQueryListener)/[Java](http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/streaming/StreamingQueryListener.html)文档)来异步监控与 `SparkSession` 相关的所有查询。一旦你将定制的 `StreamingQueryListener` 对象与 `sparkSession.streams.attachListener()` 连接起来，你将在查询开始和停止时以及在 `active` 查询中取得进展时获得回调。举个例子，

- Scala

```scala
val spark: SparkSession = ...

spark.streams.addListener(new StreamingQueryListener() {
    override def onQueryStarted(queryStarted: QueryStartedEvent): Unit = {
        println("Query started: " + queryStarted.id)
    }
    override def onQueryTerminated(queryTerminated: QueryTerminatedEvent): Unit = {
        println("Query terminated: " + queryTerminated.id)
    }
    override def onQueryProgress(queryProgress: QueryProgressEvent): Unit = {
        println("Query made progress: " + queryProgress.progress)
    }
})
```



- Java 

```java
SparkSession spark = ...

spark.streams().addListener(new StreamingQueryListener() {
    @Override
    public void onQueryStarted(QueryStartedEvent queryStarted) {
        System.out.println("Query started: " + queryStarted.id());
    }
    @Override
    public void onQueryTerminated(QueryTerminatedEvent queryTerminated) {
        System.out.println("Query terminated: " + queryTerminated.id());
    }
    @Override
    public void onQueryProgress(QueryProgressEvent queryProgress) {
        System.out.println("Query made progress: " + queryProgress.progress());
    }
});
```



### 使用 Dropwizard 输出指标报告

Spark支持使用[Dropwizard库](monitoring-zh.md#指标)报告指标。为了能够报告结构化流查询的指标，你必须在 `SparkSession`中显式地启用 `spark.sql.streaming.metricsEnabled`。

- Scala

```scala
spark.conf.set("spark.sql.streaming.metricsEnabled", "true")
// or
spark.sql("SET spark.sql.streaming.metricsEnabled=true")
```



- Java

```java
spark.conf().set("spark.sql.streaming.metricsEnabled", "true");
// or
spark.sql("SET spark.sql.streaming.metricsEnabled=true");
```

在启用此配置之后，所有在 `SparkSession` 中启动的查询都将通过 `Dropwizard` 向任何已配置的接收端报告指标(例如 `Ganglia`、`Graphite` 、`JMX`等)。



### 使用 `Checkpoint` 从失败中 `Recovery`

在出现故障或有意关闭的情况下，你可以恢复前一个查询的进度和状态，并继续它停止的地方。这是使用 `checkpoint` 和 `write-ahead` 日志完成的。你可以使用 `checkpoint` 位置配置一个查询，该查询将保存所有的进度信息(即每个 `triggle` 中处理的偏移范围)和运行聚合(例如quick `word count` 示例中)到 `checkpoint` 位置。这个 `checkpoint` 位置必须是HDFS兼容的文件系统中的一个路径，并且可以在[启动查询](##开始 `Stream Query`)时在`DataStreamWriter` 中设置为一个选项。

- Scala

```scala
aggDF
  .writeStream
  .outputMode("complete")
  .option("checkpointLocation", "path/to/HDFS/dir")
  .format("memory")
  .start()
```



- Java

```java
aggDF
  .writeStream()
  .outputMode("complete")
  .option("checkpointLocation", "path/to/HDFS/dir")
  .format("memory")
  .start();
```



## `Stream` 查询中更改后的 `Recovery` 语义

在从同一 `checkpoint` 位置重新启动之间，允许流查询中的哪些更改是有限制的。以下是几种不允许的更改，或者这些更改的效果没有得到很好的定义。All:

- 允许的术语意味着你可以执行指定的更改，但是其效果的语义是否定义良好取决于查询和更改。
- 术语 <u>不允许</u> 意味着你不应该进行指定的更改，因为重新启动的查询可能会失败，出现不可预知的错误。`sdf` 表示一个使用 `sparkSession.readStream` 生成的流数据流。

**类型的变化**

- 输入源的数量或类型**(**即不同的源**)**的改变**:**这是不允许的。
- **输入源参数的更改**: 是否允许这样做以及更改的语义是否定义良好取决于源和查询。这里有几个例子。
  - 允许添加/删除/修改速率限制: `spark.readStream.format("kafka").option("subscribe", "topic")` 到 `spark.readStream.format("kafka").option("subscribe", "topic").option("maxOffsetsPerTrigger", ...)`
  - 通常不允许更改订阅的 `topic`/`file`，因为结果是不可预测的:  `spark.readStream.format("kafka").option("subscribe", "topic")` 到`spark.readStream.format("kafka").option("subscribe", "newTopic")` 

- *`output sink`类型的更改**:**允许在几个特定的接收器组合之间进行更改。*这需要在个案的基础上加以核实。这里有几个例子。

  - 允许 `File sink` 更改为 `Kafka sink` : Kafka 仅会接收到新数据

  - `Kafka sink` 更改为 `File sink` 是不允许的

  - `Kafka sink` 可改为 `foreach` ，反之亦然。

    

- *输出 `sink` 参数的更改**:**是否允许这样做，以及更改的语义是否定义良好取决于接收器和查询。*这里有几个例子:
  - 不允许 `file sink` 更改输出目录： `sdf.writeStream.format("parquet").option("path", "/somePath")` to `sdf.writeStream.format("parquet").option("path", "/anotherPath")`
    - 允许更改输出 `topic`: `sdf.writeStream.format("kafka").option("topic", "someTopic")` 到 `sdf.writeStream.format("kafka").option("topic", "anotherTopic")`
  - 允许对 `user-define` 的 `foreach sink` (即ForeachWriter代码)进行更改，但是更改的语义取决于代码。
- *改变 `projection`/ `filter` / `map-like` 操作*:某些情况下是允许的。例如:
  - 允许添加/删除 `filter`:`sdf.selectExpr("a")` 到 `sdf.where(...).selectExpr("a").filter(...)` 
  - 允许更改具有相同输出模式的 `projection`: `sdf.selectExpr("stringColumn AS json").writeStream` 到 `sdf.selectExpr("anotherStringColumn AS json").writeStream`
  - 有条件地允许更改具有不同输出模式的 `projection`:  仅仅当 `output sink` 允许 Schema  `a` 到 `b`更改时， `sdf.selectExpr("a").writeStream` 到 `sdf.selectExpr("b").writeStream` 才会被允许。

- `stateful` 操作中的更改**:**流查询中的一些操作需要维护状态数据，以便持续更新结果。结构化流自动将状态数据 `checkpoint` 到 `fault-tolerance` 存储(例如，HDFS、AWS S3、Azure Blob存储)，并在重启后恢复。但是，这假设状态数据的模式在重启时保持不变。这意味着在重新启动之间不允许对流查询的有状态操作进行任何更改(即添加、删除或模式修改)。以下是有状态操作的列表，为了确保状态恢复，在重启之间不应该更改这些操作的模式:
  - *Streaming aggregation**:**例如，**`sdf.groupBy("a").agg(...)`.**。*不允许对分组键或聚合的数量或类型进行任何更改。
  - *流式重复数据删除**:**例如，**sdf.dropDuplicates("a")**。*不允许对分组键或聚合的数量或类型进行任何更改。
  - `stream-stream` join:  例如： `sdf1.join(sdf2, ...)` (即。两个输入都是用sparkSession.readStream生成的)。不允许更改模式或 `equi-joining` 列。不允许更改连接类型(outer 或 inner)。`join condition` 中的其他更改定义不清。
- *任意有状态操作**:**例如，`sdf.groupByKey(...).mapGroupsWithState(...)` 或 `sdf.groupByKey(...).flatMapGroupsWithState(...)`。 不允许对用户定义状态的模式和超时类型进行任何更改。允许在 `user-defined` 的 `state-mapping` 函数中进行任何更改，但是更改的语义效果取决于用户定义的逻辑。如果你真的希望支持状态模式更改，那么可以使用支持模式迁移的 `encode`/ `decode`方案显式地将复杂的状态数据结构 `encode`/`decode` 为字节。例如，如果你将你的状态保存为avro编码的字节，那么你可以自由地更改查询之间的Avro-state-schema，因为二进制状态将始终成功地恢复。
- 

# 连续处理

## [实验]

**连续处理是** Spark 2.3中引入的一种新的实验性流执行模式，它支持低****(~1 ms)****的 `end-to-end` 延迟，并保证 `at-least-once`  `fault-tolerance` 。与默认的 `micro-batch` 引擎相比，默认的 `micro-batch` 引擎只能实现一次精确的保证，但最多只能实现约100ms的延迟。对于某些类型的查询(下面将讨论)，你可以在不修改应用程序逻辑的情况下选择执行它们的模式(即不更改 `DataFrame`/`Dataset`操作)。

要在连续处理模式下运行受支持的查询，只需指定一个连续触发器，并将所需的 `checkpoint` 间隔作为参数。例如,

- Scala

```scala
import org.apache.spark.sql.streaming.Trigger

spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("subscribe", "topic1")
  .load()
  .selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("topic", "topic1")
  .trigger(Trigger.Continuous("1 second"))  // only change in query
  .start()
```



- Java 

```java
import org.apache.spark.sql.streaming.Trigger;

spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("subscribe", "topic1")
  .load()
  .selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
  .writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("topic", "topic1")
  .trigger(Trigger.Continuous("1 second"))  // only change in query
  .start();
```



 `checkpoint` 间隔为1秒，这意味着连续处理引擎将每秒钟记录查询的进度。生成的 `checkpoint` 采用与 `micro-batch` 引擎兼容的格式，因此可以使用任何触发器重新启动任何查询。例如，使用 `micro-batch` 模式启动的受支持查询可以在连续模式中重新启动，反之亦然。注意，无论何时切换到连续模式，都至少会得到一次 `fault-tolerance` 保证。



## 支持查询

从Spark 2.4开始，在连续处理模式中只支持以下类型的查询。

- *Operations*: 在连续模式下只支持 map-like 的  Dataset/DataFrame  操作，即只支持projection(select**、**map**、**flatMap**、**mapPartition 等 )和 selection (where**、**filter 等)。
  - 除了聚合函数(因为聚合还不受支持), `current_timestamp()` 和`current_date()`(使用时间的确定性计算很有挑战性)之外 所有的 SQL 函数都支持。

- Source :

  -  Kafka source:所有选项都支持。
  - Rate source :  适合测试。只有 `numPartitions` 和 `rowsPerSecond` 在连续模式下受支持

- Sink: 
  - `Kafka sink`:支持所有选项。

  - `Memory sink`:很适合调试。

  - `Console sink`:很适合调试。支持所有选项。请注意，控制台将在连续触发器中指定的每个 `checkpoint` 间隔打印。

  有关 `input source` 和 `output sink` 的详细信息，请参阅[输入源](#输入源)和[输出接收](#`Output Sink`)部分。虽然 `Console sink` 有利于测试、 `end-to-end`  `low-latency`  处理可以是最好的观察与Kafka 的 `source` 和 `sink`,这允许 `engine` 过程中,并在输入 `topic`中输入数据可用后的几毫秒内使结果在输出 `topic`中可用.

  

## 警告

- 持续处理引擎启动多个长时间运行的任务，这些任务不断地从源读取数据、处理数据并不断地向 sink写入数据。查询所需的任务数量取决于查询可以并行地从源读取多少个 `partition`。因此，在开始连续处理查询之前，你必须确保集群中有足够多的 `core` 来并行处理所有任务。例如，如果你从一个有10个 `partiton`的Kafka `topic` 中读取数据，那么集群必须至少有10 core，以便查询取得进展。
- 停止一个连续的处理流可能会产生虚假的任务终止警告。这些都可以安全忽略。
- 目前没有失败任务的自动重试。任何失败都将导致停止查询，并且需要从 `checkpoint` 手动重新启动查询。
- 

# 额外信息

**进一步的阅读**

- 查看并运行[Scala](https://github.com/apache/spark/tree/v2.4.5/examples/src/main/scala/org/apache/spark/examples/sql/streaming)/[Java](https://github.com/apache/spark/tree/v2.4.5/examples/src/main/java/org/apache/spark/examples/sql/streaming)/[Python](https://github.com/apache/spark/tree/v2.4.5/examples/src/main/python/sql/streaming)/[R](https://github.com/apache/spark/tree/v2.4.5/examples/src/main/r/streaming)示例。
  - [关于如何运行Spark示例的说明](index-zh.md#running-the-examples-and-shell)

- 阅读关于与Kafka集成的[结构化流Kafka集成指南](structured-streaming-kafka-integration-zh.md)

- 请参阅Spark SQL编程指南中有关使用数据流/ `Dataset` 的更多详细信息

- 第三方博客文章

  - [Apache Spark 2.1中带结构化流的实时流ETL (Databricks博客)](https://databricks.com/blog/2017/01/19/real-time-streaming-etl-structured-streaming-apache-spark-2-1.html)

  - [Apache Spark的结构化流(Databricks博客)中与Apache Kafka的实时 `end-to-end` 集成](https://databricks.com/blog/2017/04/04/real-time-end-to-end-integration-with-apache-kafka-in-apache-sparks-structured-streaming.html)
  - [Apache Spark的结构化流(Databricks博客)中的 `event-time` 聚合和 `Watermark`](https://databricks.com/blog/2017/05/08/event-time-aggregation-watermarking-apache-sparks-structured-streaming.html)

**会谈**

- 2017 欧洲 Spark 峰会
  - `Easy`，`Scalable`， `fault-tolerance` 流处理与 `Structured Streaming` 在Apache Spark -第1部分[slide/video](https://databricks.com/session/easy-scalable-fault-tolerant-stream-processing-with-structured-streaming-in-apache-spark)，第2部分 [slides/video]https://databricks.com/session/easy-scalable-fault-tolerant-stream-processing-with-structured-streaming-in-apache-spark-continues)
  - 深入研究 `stateful Stream Process`的`Structure stream`- [slide/video](https://databricks.com/session/deep-dive-into-stateful-stream-processing-in-structured-streaming)
- 2016年，Spark 峰会
- 对 `Structure Stream` 的深入研究——[slide/video](https://spark-summit.org/2016/events/a-deep-dive-into-structured-streaming/)

 