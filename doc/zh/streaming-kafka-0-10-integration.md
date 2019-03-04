---
layout: global
title: Spark Streaming + Kafka 集成指南 (Kafka版本0.10.0或更高版本)
---

Kafka 0.10的Spark Streaming集成在设计上类似于0.8 [Direct Stream approach](streaming-kafka-0-8-integration.html#approach-2-direct-approach-no-receivers)。 它提供简单的并行性，Kafka分区和Spark分区之间的1：1对应，以及访问偏移和元数据。 然而，因为新的集成使用了 [新的Kafka消费者API](http://kafka.apache.org/documentation.html#newconsumerapi) 而不是简单的API，所以在使用上有显着的差异。 此版本的集成被标记为实验性的，因此API可能会更改。

### 链接
对于使用SBT/Maven项目定义的Scala/Java应用程序，将流应用程序与以下artifact链接 (有关详细信息，请参阅主编程指南中的[链接部分](streaming-programming-guide.html#linking))。

	groupId = org.apache.spark
	artifactId = spark-streaming-kafka-0-10_{{site.SCALA_BINARY_VERSION}}
	version = {{site.SPARK_VERSION_SHORT}}

**不要** 在org.apache.kafka artifact上手动添加依赖项（例如kafka-clients）。 spark-streaming-kafka-0-10 artifact已经具有适当的传递依赖性，不同的版本可能在难以诊断的方式上不兼容。

### 创建 Direct Stream
 请注意，导入的命名空间包括版本org.apache.spark.streaming.kafka010

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe

val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "localhost:9092,anotherhost:9092",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "use_a_separate_group_id_for_each_stream",
  "auto.offset.reset" -> "latest",
  "enable.auto.commit" -> (false: java.lang.Boolean)
)

val topics = Array("topicA", "topicB")
val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

stream.map(record => (record.key, record.value))
{% endhighlight %}
Each item in the stream is a [ConsumerRecord](http://kafka.apache.org/0100/javadoc/org/apache/kafka/clients/consumer/ConsumerRecord.html)
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
import java.util.*;
import org.apache.spark.SparkConf;
import org.apache.spark.TaskContext;
import org.apache.spark.api.java.*;
import org.apache.spark.api.java.function.*;
import org.apache.spark.streaming.api.java.*;
import org.apache.spark.streaming.kafka010.*;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;
import scala.Tuple2;

Map<String, Object> kafkaParams = new HashMap<>();
kafkaParams.put("bootstrap.servers", "localhost:9092,anotherhost:9092");
kafkaParams.put("key.deserializer", StringDeserializer.class);
kafkaParams.put("value.deserializer", StringDeserializer.class);
kafkaParams.put("group.id", "use_a_separate_group_id_for_each_stream");
kafkaParams.put("auto.offset.reset", "latest");
kafkaParams.put("enable.auto.commit", false);

Collection<String> topics = Arrays.asList("topicA", "topicB");

JavaInputDStream<ConsumerRecord<String, String>> stream =
  KafkaUtils.createDirectStream(
    streamingContext,
    LocationStrategies.PreferConsistent(),
    ConsumerStrategies.<String, String>Subscribe(topics, kafkaParams)
  );

stream.mapToPair(record -> new Tuple2<>(record.key(), record.value()));
{% endhighlight %}
</div>
</div>

有关可能的kafkaParams，请参阅 [Kafka consumer配置文件]   (http://kafka.apache.org/documentation.html#newconsumerconfigs)。
如果您的Spark批处理持续时间大于默认的Kafka心跳会话超时 (30 秒)，请适当增加 heartbeat.interval.ms 和 session.timeout.ms。 对于大于5分钟的批次，这将需要更改代理上的 group.max.session.timeout.ms。
请注意，示例将enable.auto.commit设置为false， 有关讨论，请参阅下面的 [存储偏移](streaming-kafka-0-10-integration.html#storing-offsets) 。

### LocationStrategies
新的Kafka consumer API会将消息预取到缓冲区中。因此，出于性能原因，Spark集成保持缓存消费者对执行者（而不是为每个批次重新创建它们）是重要的，并且更喜欢在具有适当消费者的主机位置上调度分区。   

在大多数情况下，您应该使用  `LocationStrategies.PreferConsistent`如上所示。 这将在可用的执行器之间均匀分配分区。  如果您的执行程序与Kafka代理所在的主机相同，请使用 `PreferBrokers`，这将更喜欢在该分区的Kafka leader上安排分区。 最后，如果您在分区之间的负载有显着偏差，请使用  `PreferFixed`。这允许您指定分区到主机的显式映射 (任何未指定的分区将使用一致的位置)。

消费者的缓存的默认最大大小为64。 如果您希望处理超过 (64 * 执行程序数) Kafka分区，则可以通过以下方式更改此设置 `spark.streaming.kafka.consumer.cache.maxCapacity`.

如果您想禁用Kafka使用者的缓存，则可以将`spark.streaming.kafka.consumer.cache.enabled` 设置为 `false`。 可能需要禁用缓存来解决SPARK-19185中描述的问题。 SPARK-19185解决后，该属性可能会在更高版本的Spark中删除。

缓存由topicpartition和group.id键入， 因此对每个调用者使用一个 **单独** `group.id` 进行 `createDirectStream`.


### ConsumerStrategies
新的Kafka consumer API有许多不同的方式来指定主题，其中一些需要相当多的post-object-instantiation设置。 `ConsumerStrategies` 提供了一种抽象，允许Spark即使在从检查点重新启动后也能获得正确配置的消费者。

`ConsumerStrategies.Subscribe`，如上所示，允许您订阅固定的主题集合。 `SubscribePattern`允许您使用正则表达式来指定感兴趣的主题。 注意，与0.8集成不同，在运行流期间使用 `Subscribe` 或  `SubscribePattern` 应该响应添加分区。 最后， `Assign` 允许您指定固定的分区集合。 所有三个策略都有重载的构造函数，允许您指定特定分区的起始偏移量。

如果您具有上述选项不满足的特定用户设置需求，则 `ConsumerStrategy` 是可以扩展的公共类。

### 创建RDD
如果您有一个更适合批处理的用例，则可以为定义的偏移量范围创建RDD。

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
// Import dependencies and create kafka params as in Create Direct Stream above

val offsetRanges = Array(
  // topic, partition, inclusive starting offset, exclusive ending offset
  OffsetRange("test", 0, 0, 100),
  OffsetRange("test", 1, 0, 100)
)

val rdd = KafkaUtils.createRDD[String, String](sparkContext, kafkaParams, offsetRanges, PreferConsistent)
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
// Import dependencies and create kafka params as in Create Direct Stream above

OffsetRange[] offsetRanges = {
  // topic, partition, inclusive starting offset, exclusive ending offset
  OffsetRange.create("test", 0, 0, 100),
  OffsetRange.create("test", 1, 0, 100)
};

JavaRDD<ConsumerRecord<String, String>> rdd = KafkaUtils.createRDD(
  sparkContext,
  kafkaParams,
  offsetRanges,
  LocationStrategies.PreferConsistent()
);
{% endhighlight %}
</div>
</div>

注意，你不能使用 `PreferBrokers`，因为没有流没有驱动程序端消费者为你自动查找代理元数据。 如果需要，请使用 `PreferFixed` 查找自己的元数据。

### 获取偏移量

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    val o: OffsetRange = offsetRanges(TaskContext.get.partitionId)
    println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
  }
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
stream.foreachRDD(rdd -> {
  OffsetRange[] offsetRanges = ((HasOffsetRanges) rdd.rdd()).offsetRanges();
  rdd.foreachPartition(consumerRecords -> {
    OffsetRange o = offsetRanges[TaskContext.get().partitionId()];
    System.out.println(
      o.topic() + " " + o.partition() + " " + o.fromOffset() + " " + o.untilOffset());
  });
});
{% endhighlight %}
</div>
</div>

注意类型转换 `HasOffsetRanges` 只会成功，如果是在第一个方法中调用的结果 `createDirectStream`，不是后来一系列的方法。 请注意，RDD分区和Kafka分区之间的一对一映射在任何随机或重新分区的方法后不会保留， 例如 reduceByKey() 或 window()。

### 存储偏移量
在失败的情况下的Kafka交付语义取决于如何和何时存储偏移。 Spark 输出操作是 [at-least-once](streaming-programming-guide.html#semantics-of-output-operations)。 因此，如果你想要一个完全一次的语义的等价物，你必须在一个等幂输出之后存储偏移，或者在一个原子事务中存储偏移和输出。使用这种集成，您有3个选项，按照可靠性 (和代码复杂性)的增加，如何存储偏移。

#### 检查点
如果启用Spark [checkpointing](streaming-programming-guide.html#checkpointing)，偏移将存储在检查点中。 这很容易实现，但有缺点。你的输出操作必须是幂等的，因为你会得到重复的输出; 事务不是一个选项。此外，如果应用程序代码已更改，您将无法从检查点恢复。 对于计划升级，您可以通过与旧代码同时运行新代码来缓解这种情况 (因为输出必须是幂等的，它们不应该冲突)。但对于需要更改代码的意外故障，您将丢失数据，除非您有其他方法来识别已知的良好起始偏移。

#### kafka
Kafka有一个偏移提交API，将偏移存储在特殊的Kafka主题中。 默认情况下，新消费者将定期自动提交偏移量。 这几乎肯定不是你想要的，因为消费者成功轮询的消息可能还没有导致Spark输出操作，导致未定义的语义。这就是为什么上面的流示例将 "enable.auto.commit" 设置为false的原因。但是，您可以在使用 `commitAsync` 存储了输出后，向Kafka提交偏移量。与检查点相比，Kafka是一个耐用的存储，而不管您的应用程序代码的更改。 然而，Kafka不是事务性的，所以你的输出必须仍然是幂等的。

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  // some time later, after outputs have completed
  stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
{% endhighlight %}
As with HasOffsetRanges, the cast to CanCommitOffsets will only succeed if called on the result of createDirectStream, not after transformations.  The commitAsync call is threadsafe, but must occur after outputs if you want meaningful semantics.
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
stream.foreachRDD(rdd -> {
  OffsetRange[] offsetRanges = ((HasOffsetRanges) rdd.rdd()).offsetRanges();

  // some time later, after outputs have completed
  ((CanCommitOffsets) stream.inputDStream()).commitAsync(offsetRanges);
});
{% endhighlight %}
</div>
</div>

#### 自己的数据存储
对于支持事务的数据存储，即使在故障情况下，也可以在同一事务中保存偏移量作为结果，以保持两者同步。如果您仔细检查重复或跳过的偏移范围，则回滚事务可防止重复或丢失的邮件影响结果。这给出了恰好一次语义的等价物。也可以使用这种策略甚至对于聚合产生的输出，聚合通常很难使幂等。

<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
// The details depend on your data store, but the general idea looks like this

// begin from the the offsets committed to the database
val fromOffsets = selectOffsetsFromYourDatabase.map { resultSet =>
  new TopicPartition(resultSet.string("topic"), resultSet.int("partition")) -> resultSet.long("offset")
}.toMap

val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Assign[String, String](fromOffsets.keys.toList, kafkaParams, fromOffsets)
)

stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  val results = yourCalculation(rdd)

  // begin your transaction

  // update results
  // update offsets where the end of existing offsets matches the beginning of this batch of offsets
  // assert that offsets were updated correctly

  // end your transaction
}
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
// The details depend on your data store, but the general idea looks like this

// begin from the the offsets committed to the database
Map<TopicPartition, Long> fromOffsets = new HashMap<>();
for (resultSet : selectOffsetsFromYourDatabase)
  fromOffsets.put(new TopicPartition(resultSet.string("topic"), resultSet.int("partition")), resultSet.long("offset"));
}

JavaInputDStream<ConsumerRecord<String, String>> stream = KafkaUtils.createDirectStream(
  streamingContext,
  LocationStrategies.PreferConsistent(),
  ConsumerStrategies.<String, String>Assign(fromOffsets.keySet(), kafkaParams, fromOffsets)
);

stream.foreachRDD(rdd -> {
  OffsetRange[] offsetRanges = ((HasOffsetRanges) rdd.rdd()).offsetRanges();
  
  Object results = yourCalculation(rdd);

  // begin your transaction

  // update results
  // update offsets where the end of existing offsets matches the beginning of this batch of offsets
  // assert that offsets were updated correctly

  // end your transaction
});
{% endhighlight %}
</div>
</div>

### SSL / TLS
新的Kafka消费者 [支持 SSL](http://kafka.apache.org/documentation.html#security_ssl)。要启用它，请在传递到 `createDirectStream` / `createRDD` 之前适当地设置kafkaParams。注意，这只适用于Spark和Kafka代理之间的通信; 您仍然有责任单独保证 Spark [securing](security.html)节点间通信。


<div class="codetabs">
<div data-lang="scala" markdown="1">
{% highlight scala %}
val kafkaParams = Map[String, Object](
  // the usual params, make sure to change the port in bootstrap.servers if 9092 is not TLS
  "security.protocol" -> "SSL",
  "ssl.truststore.location" -> "/some-directory/kafka.client.truststore.jks",
  "ssl.truststore.password" -> "test1234",
  "ssl.keystore.location" -> "/some-directory/kafka.client.keystore.jks",
  "ssl.keystore.password" -> "test1234",
  "ssl.key.password" -> "test1234"
)
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
Map<String, Object> kafkaParams = new HashMap<String, Object>();
// the usual params, make sure to change the port in bootstrap.servers if 9092 is not TLS
kafkaParams.put("security.protocol", "SSL");
kafkaParams.put("ssl.truststore.location", "/some-directory/kafka.client.truststore.jks");
kafkaParams.put("ssl.truststore.password", "test1234");
kafkaParams.put("ssl.keystore.location", "/some-directory/kafka.client.keystore.jks");
kafkaParams.put("ssl.keystore.password", "test1234");
kafkaParams.put("ssl.key.password", "test1234");
{% endhighlight %}
</div>
</div>

### 部署

与任何Spark应用程序一样， `spark-submit`用于启动应用程序。

对于Scala和Java应用程序，如果您使用SBT或Maven进行项目管理，则将程序包 `spark-streaming-kafka-0-10_{{site.SCALA_BINARY_VERSION}}` 及其依赖项包含到应用程序JAR中。 确保 `spark-core_{{site.SCALA_BINARY_VERSION}}` 和 `spark-streaming_{{site.SCALA_BINARY_VERSION}}` 被标记为 `provided` 依赖关系，因为它们已经存在于Spark安装中。 然后使用`spark-submit` 启动应用程序 (参阅主程序指南中的 [Deploying section](streaming-programming-guide.html#deploying-applications)).

