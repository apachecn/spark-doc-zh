---
layout: global
title: Spark Streaming + Kafka 集成指南 (Kafka broker version 0.8.2.1 or higher)
---
本篇我们阐述如何配置 Spark Streaming 从 Kafka 接收数据.  有两种方法可以实现 : 老的方法是使用 Receiver 和 Kafka 的 high-level API（高级 API）来实现, 新方法则不需要使用 Receiver（从 Spark 1.3 中引入）. 它们有不同的 programming models（编程模型）, performance characteristics（性能特性）和 semantics guarantees（语义保证）, 所以请继续阅读关于它们更多的细节.  这两种方法都被认为是当前版本的 Spark 的稳定 API. 

## Approach 1: Receiver-based Approach （方法 1: 基于 Receiver 的方法）
此方法使用 Receiver 来接收数据.  Receiver 是使用 Kafka high-level consumer API 实现的.  与所有 Receiver 一样, Receiver 从 Kafka 接收的数据并存储在 Spark executor 中, 然后由 Spark Streaming 启动的作业处理数据. 

然而, 在默认配置下, 这种方法在程序失败时会丢失数据（请看 [receiver reliability （ receiver 的可靠性）](streaming-programming-guide.html#receiver-reliability) , 为了确保零数据丢失, 必须启用 Spark Streaming 中的 Write Ahead Logs（预写日志）（在 Spark 1.2 中引入）, 这会同步保存所有从 Kafka 接收的数据写入分布式文件系统（例如 HDFS）, 以便所有数据可以在故障时恢复. 有关 Write Ahead （预写日志）更多信息请参阅 streaming 编程指南中的 [Deploying section （应用程序部署）](streaming-programming-guide.html#deploying-applications) 一节. 

接着, 我们要讨论如何在你的 streaming application （streaming 应用程序）中使用这种方法. 

1. **Linking（依赖）:** 针对 Scala/Java 程序可以使用 SBT/Maven 包管理, 在包配置文件中加入以下 artifact（更多细节请查看 Spark 编程指南的 [Linking section （依赖章节）](streaming-programming-guide.html#linking)）. 

		groupId = org.apache.spark
		artifactId = spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}}
		version = {{site.SPARK_VERSION_SHORT}}

	针对 Python 应用程序, 在部署应用程序时, 必须添加上述库及其依赖关系.  请参阅下面的 *Deploying（部署）* 小节. 

2. **Programming（编程）:** 在 streaming application 代码中 import `KafkaUtils` 包并且创建一个 input DStream. 

	<div class="codetabs">
	<div data-lang="scala" markdown="1">
		import org.apache.spark.streaming.kafka._

		val kafkaStream = KafkaUtils.createStream(streamingContext,
            [ZK quorum], [consumer group id], [per-topic number of Kafka partitions to consume])

    您还可以使用 `createStream` 指定 key（键）和 value（值）及其相应的 decoder（解码器）的类.  请参阅 [API 文档](api/scala/index.html#org.apache.spark.streaming.kafka.KafkaUtils$) 和 [示例]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/KafkaWordCount.scala). 
	</div>
	<div data-lang="java" markdown="1">
		import org.apache.spark.streaming.kafka.*;

		JavaPairReceiverInputDStream<String, String> kafkaStream =
			KafkaUtils.createStream(streamingContext,
            [ZK quorum], [consumer group id], [per-topic number of Kafka partitions to consume]);

    您还可以使用 `createStream` 指定 key（键）和 value（值）及其相应的 decoder（解码器）的类.  请参阅 [API 文档](api/java/index.html?org/apache/spark/streaming/kafka/KafkaUtils.html) 和 [示例]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming/JavaKafkaWordCount.java). 

	</div>
	<div data-lang="python" markdown="1">
		from pyspark.streaming.kafka import KafkaUtils

		kafkaStream = KafkaUtils.createStream(streamingContext, \
			[ZK quorum], [consumer group id], [per-topic number of Kafka partitions to consume])

	默认情况下, Python API 会将 Kafka 数据解码为 UTF8 编码的 strings .  您可以指定 custom decoding function （自定义解码函数）, 将 Kafka 记录中的 byte arrays （字节数组）解码为任意数据类型.  请参阅 [API 文档](api/python/pyspark.streaming.html#pyspark.streaming.kafka.KafkaUtils) 和 [示例]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/kafka_wordcount.py). 
	</div>
	</div>

	**注意事项:**

	- Kafka 中的 topic（主题）的 partition（分区）与在 Spark Streaming 中生成的 RDD 的分区无关.  因此, 增加 `KafkaUtils.createStream()` 中指定的 topic（主题）的 partition（分区）的数量只会增加使用单个 receiver（接收器）消费 topic（主题）的线程数.  它不会增加 Spark 在处理数据时的并行性. 有关详细信息, 请参阅主文档. 

	- 多个 Kafka input DStreams 可以用不同的 groups（组）和 topics（主题）来创建, 使得多个 receiver 可以并行接收数据. 

	- 如果在 HDFS 这种 replicated file system（副本型的文件系统）中启用了 Write Ahead Logs, 则接收的数据已经被 replicated（复制） 因此, 存储级别为 `StorageLevel.MEMORY_AND_DISK_SER`（即, 使用 `KafkaUtils.createStream(..., StorageLevel.MEMORY_AND_DISK_SER)` ）. 

3. **Deploying（部署 ）:** 与任何 Spark 应用程序一样, spark-submit 用于启动您的应用程序.  但是, Scala/Java 应用程序和 Python 应用程序的细节略有不同. 

	针对 Scala 和 Java 应用程序, 如果你使用 SBT 或 Maven 进行项目管理, 需要将 `spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}}` 及其依赖项打包到 JAR 中. 确保 `spark-core_{{site.SCALA_BINARY_VERSION}}` 和 `spark-streaming_{{site.SCALA_BINARY_VERSION}}` 被标记为 `provided` 依赖项, 因为它们本身就在 Spark 安装包中因此不需要打包. 然后运行 `spark-submit` 执行你的应用程序（请参阅 main programming guide （主编程指南）中的 [Deploying section （部署部分）](streaming-programming-guide.html#deploying-applications)）. 

	对于缺少 SBT/Maven 项目管理的 Python 应用程序, `spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}}` 及其依赖项可以使用 `--packages` 直接添加到 `spark-submit` 中（请参阅 [Application Submission Guide](submitting-applications.html)）, 如下,

	    ./bin/spark-submit --packages org.apache.spark:spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}}:{{site.SPARK_VERSION_SHORT}} ...

	或者, 您也可以从 [Maven repository](http://search.maven.org/#search|ga|1|a%3A%22spark-streaming-kafka-0-8-assembly_{{site.SCALA_BINARY_VERSION}}%22%20AND%20v%3A%22{{site.SPARK_VERSION_SHORT}}%22) 下载 `spark-streaming-kafka-0-8-assembly` 的 JAR 包, 并将其添 `spark-submit` 的 `--jars` 参数中. 

## Approach 2: Direct Approach (No Receivers) （方法 2 : 直接访问（无 Receivers））
这种新的无 Receiver 的 "direct（直接）" 方法已经在 Spark 1.3 中引入, 以确保更强的 end-to-end guarantees （端到端保证）.  代替使用 receiver（接收器）来接收数据, 该方法周期性地查询 Kafka 以获得每个 topic（主题）和 partition（分区）中最新的 offset（偏移量）, 并且定义每个批量中处理的 offset ranges（偏移范围）.  当启动作业时, Kafka 的 simple consumer API Kafka 用于从 kafka 中读取定义好 offset ranges（偏移范围）的数据（类似于从文件系统读取文件）. 请注意, 针对 Scala 和 Java API 此特性在 Spark 1.3 中就引入了, 针对 Python API 在 Spark 1.4 中开始引入. 

这种方法相较方法 1 有以下优点.

- *Simplified Parallelism（简化并行性）:* 无需创建多个输入 Kafka 流并联合它们.  使用 `directStream`, Spark Streaming 将创建与消费的 Kafka partition 一样多的 RDD 分区, 这将从 Kafka 并行读取数据.  因此, Kafka 和 RDD 分区之间存在一对一映射, 这更容易理解和调整. 

- *Efficiency（高效）:* 在第一种方法中实现 zero-data loss （零数据丢失）需要将数据存储在预写日志中, 该日志进一步复制数据.  这实际上是低效的, 因为数据有效地被复制两次 - 一次是 Kafka, 另一次是 Write Ahead Log.  第二种方法消除了问题, 因为没有接收器, 因此不需要 Write Ahead Logs.  只要您的 Kafka 有足够的保留时间消息可以从 Kafka 恢复. 

- *Exactly-once semantics（一次且仅一次语义）:* 第一种方法使用 Kafka 的 high level API（高级API）在 Zookeeper 中存储 consume 的 offset（偏移量）. 这是传统上消费 Kafka 数据的方式. 虽然这种方法（与预写日志结合）可以确保零数据丢失（即 at-least once 至少一次语义）, 但是在某些故障情况下, 一些 record（记录）很小的可能性会被消费两次. 这是因为 Spark Streaming 可靠接收的数据与 Zookeeper 跟踪的 offset（偏移）之间存在不一致. 因此, 在第二种方法中, 我们使用不依赖 Zookeeper 的 simple Kafka API. offset（偏移）由 Spark Streaming 在其 checkpoints（检查点）内进行跟踪. 这消除了 Spark Streaming 和 Zookeeper/Kafka 之间的不一致, 因此, 尽管出现故障, Spark Streaming 仍然有效地 exactly once（恰好一次）接收了每条 record（记录）. 为了实现输出结果的 exactly once（恰好一次）的语义, 将数据保存到外部数据存储的输出操作必须是幂等的, 或者是保存结果和 offset（偏移量）的原子事务（请参阅 [Semantics of output operations](streaming-programming-guide.html#semantics-of-output-operations) 获取更多信息）. 

注意, 这种方法的一个缺点是它不更新 Zookeeper 中的 offset（偏移）, 因此基于 Zookeeper 的 Kafka 监控工具将不会显示进度.  但是, 您可以在每个批处理中访问由此方法处理的 offset（偏移量）, 并自己更新 Zookeeper（请参阅下文）. 

接下来, 我们讨论如何在您的 streaming application 中使用此方法. 

1. **Linking（依赖）:** 针对 Scala/Java 程序可以使用 SBT/Maven 包管理, 在包配置文件中加入以下 artifact（更多细节请查看 Spark 编程指南的 [Linking section （依赖章节）](streaming-programming-guide.html#linking) ）. 

		groupId = org.apache.spark
		artifactId = spark-streaming-kafka-0-8_{{site.SCALA_BINARY_VERSION}}
		version = {{site.SPARK_VERSION_SHORT}}

2. **Programming（编程）:** 在 streaming application 代码中,  import `KafkaUtils` 包, 并创建如下的 input DStream . 

	<div class="codetabs">
	<div data-lang="scala" markdown="1">
		import org.apache.spark.streaming.kafka._

		val directKafkaStream = KafkaUtils.createDirectStream[
			[key class], [value class], [key decoder class], [value decoder class] ](
			streamingContext, [map of Kafka parameters], [set of topics to consume])

	您还可以将 `messageHandler` 参数传递给 `createDirectStream`, 以访问包含有关当前消息的元数据的 `MessageAndMetadata` , 并将其转换为任何所需类型.  请参阅 [API 文档](api/scala/index.html#org.apache.spark.streaming.kafka.KafkaUtils$) 和 [示例]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/scala/org/apache/spark/examples/streaming/DirectKafkaWordCount.scala) . 
	</div>
	<div data-lang="java" markdown="1">
		import org.apache.spark.streaming.kafka.*;

		JavaPairInputDStream<String, String> directKafkaStream =
			KafkaUtils.createDirectStream(streamingContext,
				[key class], [value class], [key decoder class], [value decoder class],
				[map of Kafka parameters], [set of topics to consume]);

	您还可以将 `messageHandler` 参数传递给 `createDirectStream`, 以访问包含有关当前消息的元数据的 `MessageAndMetadata` , 并将其转换为任何所需类型.  请参阅 [API 文档](api/java/index.html?org/apache/spark/streaming/kafka/KafkaUtils.html) 和 [示例]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/java/org/apache/spark/examples/streaming/JavaDirectKafkaWordCount.java) . 

	</div>
	<div data-lang="python" markdown="1">
		from pyspark.streaming.kafka import KafkaUtils
		directKafkaStream = KafkaUtils.createDirectStream(ssc, [topic], {"metadata.broker.list": brokers})

	您还可以将 `messageHandler` 参数传递给 `createDirectStream`, 以访问包含有关当前消息的元数据的 `MessageAndMetadata` , 并将其转换为任何所需类型. 默认情况下,  Python API 会将 Kafka 数据解码为 UTF8 编码的字符串.  您可以指定 custom decoding function （自定义解码函数）, 将 Kafka 记录中的字节数组解码为任意数据类型.  请参阅[API 文档](api/python/pyspark.streaming.html#pyspark.streaming.kafka.KafkaUtils) 和 [示例]({{site.SPARK_GITHUB_URL}}/blob/v{{site.SPARK_VERSION_SHORT}}/examples/src/main/python/streaming/direct_kafka_wordcount.py) . 
	</div>
	</div>

	在 Kafka 参数中, 您必须指定 `metadata.broker.list` 或 `bootstrap.servers`. 默认情况下, 它将从每个 Kafka partition 分区的最新 offset（偏移量）开始消费.  如果将 Kafka 参数中的配置 `auto.offset.reset` 设置为 `smallest（最小）`, 那么它将从 smallest offset （最小偏移）开始消费. 

	您还可以使用 `KafkaUtils.createDirectStream` 的其他变体开始从任何任意偏移量消费.  此外, 如果要访问每个批次中消费的 Kafka 偏移量, 可以执行以下操作

	<div class="codetabs">
	<div data-lang="scala" markdown="1">
		// Hold a reference to the current offset ranges, so it can be used downstream
		var offsetRanges = Array.empty[OffsetRange]

		directKafkaStream.transform { rdd =>
		  offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
		  rdd
		}.map {
                  ...
		}.foreachRDD { rdd =>
		  for (o <- offsetRanges) {
		    println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
		  }
		  ...
		}
	</div>
	<div data-lang="java" markdown="1">
		// Hold a reference to the current offset ranges, so it can be used downstream
		AtomicReference<OffsetRange[]> offsetRanges = new AtomicReference<>();

		directKafkaStream.transformToPair(rdd -> {
      OffsetRange[] offsets = ((HasOffsetRanges) rdd.rdd()).offsetRanges();
      offsetRanges.set(offsets);
      return rdd;
		}).map(
		  ...
		).foreachRDD(rdd -> {
      for (OffsetRange o : offsetRanges.get()) {
        System.out.println(
          o.topic() + " " + o.partition() + " " + o.fromOffset() + " " + o.untilOffset()
        );
      }
      ...
		});
	</div>
	<div data-lang="python" markdown="1">
		offsetRanges = []

		def storeOffsetRanges(rdd):
		    global offsetRanges
		    offsetRanges = rdd.offsetRanges()
		    return rdd

		def printOffsetRanges(rdd):
		    for o in offsetRanges:
		        print "%s %s %s %s" % (o.topic, o.partition, o.fromOffset, o.untilOffset)

		directKafkaStream \
		    .transform(storeOffsetRanges) \
		    .foreachRDD(printOffsetRanges)
	</div>
	</div>

	可以用这个更新 Zookeeper , 如果你想基于 Zookeeper 的 Kafka 监控工具显示 streaming application 进程的话. 

	注意, HasOffsetRanges 的类型转换只会在 directKafkaStream 调用的第一个方法中完成.  您可以使用 transform() 代替 foreachRDD() 作为第一个调用方法, 以便访问偏移量, 然后调用其他 Spark 方法.  然而, 请注意, RDD 分区和 Kafka partition 分区之间的一对一映射的关系, shuffle 或 repartition（重新分区）后不会保留.  例如 reduceByKey() 或 window() . 

	另一个要注意的是, 由于此方法不使用 Receiver（接收器）, 与 standard receiver-related （标准接收器相关）（即 `spark.streaming.receiver.*` 相关的 [配置](configuration.html) ）将不适用于通过此方法创建的 input DStreams. 而是使用 [配置](configuration.html) `spark.streaming.kafka.*` . 一个重要的配置项是 `spark.streaming.kafka.maxRatePerPartition` , 它是每个 Kafka partition 分区将被此 direct API 读取的最大速率（每秒消息数）. 

3. **Deploying（部署）:** 与第一种方法相同. 
