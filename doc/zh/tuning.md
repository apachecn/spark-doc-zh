---
layout: global
displayTitle: Tuning Spark
title: Tuning
description: Tuning and performance optimization guide for Spark SPARK_VERSION_SHORT
---

* This will become a table of contents (this text will be scraped).
{:toc}

由于大多数 Spark 计算的内存性质， Spark 程序可能由集群中的任何资源（ CPU ，网络带宽或内存）导致瓶颈。
通常情况下，如果数据有合适的内存，瓶颈就是网络带宽，但有时您还需要进行一些调整，例如 [以序列化形式存储 RDD ](programming-guide.html#rdd-persistence)来减少内存的使用。
本指南将涵盖两个主要的主题：数据序列化，这对于良好的网络性能至关重要，并且还可以减少内存使用和内存优化。 我们选几个较小的主题进行展开。

# 数据序列化

序列化在任何分布式应用程序的性能中起着重要的作用。
很慢的将对象序列化或消费大量字节的格式将会大大减慢计算速度。
通常，这可能是您优化 Spark 应用程序的第一件事。
 Spark 宗旨在于方便和性能之间取得一个平衡（允许您使用操作中的任何 Java 类型）。 它提供了两种序列化库：

* [Java serialization](http://docs.oracle.com/javase/6/docs/api/java/io/Serializable.html):
  默认情况下，使用 Java `ObjectOutputStream` 框架的 Spark 序列化对象，并且可以与您创建的任何实现 [`java.io.Serializable`](http://docs.oracle.com/javase/6/docs/api/java/io/Serializable.html) 的类一起使用。
  您还可以通过扩展 [`java.io.Externalizable`](http://docs.oracle.com/javase/6/docs/api/java/io/Externalizable.html) 来更紧密地控制序列化的性能。
   Java 序列化是灵活的，但通常相当缓慢，并导致许多类的大型序列化格式。
* [Kryo serialization](https://github.com/EsotericSoftware/kryo): 
   Spark 也可以使用 Kryo 库（版本2）来更快地对对象进行序列化。 Kryo 比 Java 序列化（通常高达10x）要快得多，而且更紧凑，但并不支持所有的 `Serializable` 类型，并且需要先*注册*您将在程序中使用的类以获得最佳性能。

您可以通过使用 [SparkConf](configuration.html#spark-properties) 初始化作业 并进行调用来切换到使用 Kryo `conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")`。此设置配置用于不仅在工作节点之间进行洗牌数据的串行器，而且还将 RDD 序列化到磁盘。 Kryo 不是默认的唯一原因是因为自定义注册要求，但我们建议您尝试在任何网络密集型应用程序。自从 Spark 2.0.0 以来，我们在使用简单类型，简单类型的数组或字符串类型对RDD进行混洗时，内部使用 Kryo serializer 。

Spark 自动包含 Kryo 序列化器，用于 [Twitter chill](https://github.com/twitter/chill) 中 AllScalaRegistrar 涵盖的许多常用的核心 Scala 类。

要使用 Kryo 注册自己的自定义类，请使用该 `registerKryoClasses` 方法。

{% highlight scala %}
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)
{% endhighlight %}

所述 [Kryo 文档](https://github.com/EsotericSoftware/kryo) 描述了更先进的注册选项，如添加自定义序列的代码。

如果您的对象很大，您可能还需要增加 `spark.kryoserializer.buffer` [配置](configuration.html#compression-and-serialization)。该值需要足够大才能容纳您将序列化的最大对象。

最后，如果您没有注册自定义类， Kryo 仍然可以工作，但它必须存储每个对象的完整类名称，这是浪费的。

# 内存调优

有三个方面的考虑在调整内存使用：该*量*的存储你的对象所使用的（你可能希望你的整个数据集，以适应在内存中），则*成本*访问这些对象，并且开销*垃圾收集*（如果你有高成交物品条款）。

默认情况下， Java 对象可以快速访问，但可以轻松地消耗比其字段中的 "raw" 数据多2-5倍的空间。这是由于以下几个原因：

* 每个不同的 Java 对象都有一个 "object header" ，它大约是16个字节，包含一个指向它的类的指针。对于一个数据很少的对象（比如说一个`Int`字段），这可以比数据大。
* Java `String` 在原始字符串数据上具有大约40字节的开销（因为它们存储在 `Char` 数组中并保留额外的数据，例如长度），并且由于 UTF-16 的内部使用而将每个字符存储为*两个*字节 `String` 编码。因此，一个10个字符的字符串可以容易地消耗60个字节。
* 公共收集类，例如 `HashMap` 和 `LinkedList` ，使用链接的数据结构，其中每个条目（例如： `Map.Entry` ）存在"包装器"对象。该对象不仅具有 header ，还包括指针（通常为8个字节）到列表中的下一个对象。
* 原始类型的集合通常将它们存储为"盒装"对象，例如： `java.lang.Integer`。

本节将从 Spark 的内存管理概述开始，然后讨论用户可以采取的具体策略，以便在他/她的应用程序中更有效地使用内存。具体来说，我们将描述如何确定对象的内存使用情况，以及如何改进数据结构，或通过以串行格式存储数据。然后我们将介绍调整 Spark 的缓存大小和 Java 垃圾回收器。

## 内存管理概述

Spark 中的内存使用大部分属于两类：执行和存储。执行存储器是指用于以混洗，连接，排序和聚合计算的存储器，而存储内存是指用于在集群中缓存和传播内部数据的内存。在 Spark 中，执行和存储共享一个统一的区域（M）。当没有使用执行存储器时，存储器可以获取所有可用的存储器，反之亦然。如果需要，执行可以驱逐存储，但只有在总存储内存使用量低于某个阈值（R）之前。换句话说， `R` 描述 `M` 缓存块永远不会被驱逐的区域。由于实施的复杂性，存储不得驱逐执行。

该设计确保了几个理想的性能。首先，不使用缓存的应用程序可以将整个空间用于执行，从而避免不必要的磁盘泄漏。第二，使用缓存的应用程序可以保留最小的存储空间（R），其中数据块不受驱逐。最后，这种方法为各种工作负载提供了合理的开箱即用性能，而不需要用户内部如何分配内存的专业知识。

虽然有两种相关配置，但典型用户不需要调整它们，因为默认值适用于大多数工作负载：

* `spark.memory.fraction` 表示大小 `M`（JVM堆空间 - 300MB）（默认为0.6）的一小部分。剩余的空间（40％）保留用于用户数据结构，Spark中的内部元数据，并且在稀疏和异常大的记录的情况下保护OOM错误。
* `spark.memory.storageFraction` 表示大小 `R` 为 `M` （默认为0.5）的一小部分。 `R` 是 `M` 缓存块中的缓存被执行驱逐的存储空间。

`spark.memory.fraction` 应该设置值，以便在 JVM 的旧版或"终身"版本中舒适地适应这一堆堆空间。有关详细信息，请参阅下面高级 GC 调优的讨论。

## 确定内存消耗

大小数据集所需的内存消耗量的最佳方式是创建 RDD ，将其放入缓存中，并查看 Web UI 中的“存储”页面。该页面将告诉您 RDD 占用多少内存。

为了估计特定对象的内存消耗，使用 `SizeEstimator` 的 `estimate` 方法这是用于与不同的数据布局试验修剪内存使用情况，以及确定的空间的广播变量将占据每个执行器堆的量是有用的。

## 调整数据结构

减少内存消耗的第一种方法是避免添加开销的 Java 功能，例如基于指针的数据结构和包装对象。有几种方法可以做到这一点：

1. 将数据结构设计为偏好对象数组和原始类型，而不是标准的 Java 或 Scala 集合类（例如： `HashMap` ）。该 [fastutil](http://fastutil.di.unimi.it) 库提供方便的集合类基本类型是与 Java 标准库兼容。
2. 尽可能避免使用很多小对象和指针的嵌套结构。
3. 考虑使用数字 ID 或枚举对象而不是键的字符串。
4. 如果您的 RAM 小于32 GB，请设置 JVM 标志 `-XX:+UseCompressedOops` ，使指针为4个字节而不是8个字节。您可以添加这些选项 [`spark-env.sh`](configuration.html#environment-variables)。

## 序列化 RDD 存储

当您的对象仍然太大而无法有效存储，尽管这种调整，减少内存使用的一个更简单的方法是以序列化形式存储它们，使用 [RDD 持久性 API](programming-guide.html#rdd-persistence) 中的序列化 StorageLevel ，例如： `MEMORY_ONLY_SER` 。 Spark 将会将每个 RDD 分区存储为一个大字节数组。以序列化形式存储数据的唯一缺点是访问时间较短，因为必须对每个对象进行反序列化。如果您想以序列化形式缓存数据，我们强烈建议[使用 Kryo](#data-serialization) ，因为它导致比 Java 序列化更小的尺寸（而且肯定比原 Java 对象）更小。

## 垃圾收集调整

当您的程序存储的 RDD 有很大的"流失"时， JVM 垃圾收集可能是一个问题。（程序中通常没有问题，只读一次 RDD ，然后在其上运行许多操作）。
当 Java 需要驱逐旧对象为新的对象腾出空间时，需要跟踪所有 Java 对象并找到未使用的。要记住的要点是，垃圾收集的成本与 Java 对象的数量成正比，因此使用较少对象的数据结构（例如： `Ints` 数组，而不是 `LinkedList` ）大大降低了此成本。
一个更好的方法是如上所述以序列化形式持久化对象：现在每个 RDD 分区只有一个对象（一个字节数组）。
在尝试其他技术之前，如果 GC 是一个问题，首先要使用[序列化缓存](#serialized-rdd-storage)。

由于任务的工作记忆（运行任务所需的空间）和缓存在节点上的 RDD 之间的干扰， GC 也可能是一个问题。我们将讨论如何控制分配给RDD缓存的空间来减轻这一点。

**测量 GC 的影响**

GC 调整的第一步是收集关于垃圾收集发生频率和GC花费的时间的统计信息。这可以通过添加 `-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps` 到 Java 选项来完成。（有关将 Java 选项传递给 Spark 作业的信息，请参阅[配置指南](configuration.html#Dynamically-Loading-Spark-Properties)）下次运行 Spark 作业时，每当发生垃圾回收时，都会看到在工作日志中打印的消息。请注意，这些日志将在您的群集的工作节点上（ `stdout` 在其工作目录中的文件中），而不是您的驱动程序。

**高级 GC 优化**

为了进一步调整垃圾收集，我们首先需要了解一些关于 JVM 内存管理的基本信息：

* Java堆空间分为两个区域 Young 和 Old 。 Young 一代的目的是持有短命的物体，而 Old 一代的目标是使用寿命更长的物体。

* Young 一代进一步分为三个区域[ Eden ， Survivor1 ， Survivor2 ]。

* 垃圾收集过程的简化说明：当 Eden 已满时， Eden 上运行了一个小型 GC ，并将 Eden 和 Survivor1 中存在的对象复制到 Survivor2 。幸存者地区被交换。如果一个对象足够老，或者 Survivor2 已满，则会移动到 Old 。最后，当 Old 接近满时，一个完整的 GC 被调用。

Spark 中 GC 调优的目的是确保只有长寿命的 RDD 存储在 Old 版本中，并且 Young 版本的大小足够存储短命期的对象。这将有助于避免使用完整的 GC 来收集任务执行期间创建的临时对象。可能有用的一些步骤是：

* 通过收集 GC 统计信息来检查垃圾收集是否太多。如果在任务完成之前多次调用完整的 GC ，这意味着没有足够的可用于执行任务的内存。

* 如果太小的集合太多，而不是很多主要的 GC ，为 Eden 分配更多的内存将会有所帮助。您可以将 Eden 的大小设置为对每个任务需要多少内存的估计。如果确定 Eden 的大小 `E` ，那么您可以使用该选项设置年轻一代的大小 `-Xmn=4/3*E` 。（按比例增加4/3是考虑幸存者地区使用的空间。）

* 在打印的 GC 统计信息中，如果 OldGen 接近于满，则通过降低减少用于缓存的内存量 `spark.memory.fraction` ; 缓存较少的对象比减慢任务执行更好。或者，考虑减少年轻一代的大小。这意味着 `-Xmn` 如果您将其设置为如上所述降低。如果没有，请尝试更改 JVM `NewRatio` 参数的值。许多 JVM 默认为2，这意味着 Old 版本占据堆栈的2/3。它应该足够大，使得该分数超过 `spark.memory.fraction`。

* 尝试使用 G1GC 垃圾回收器 `-XX:+UseG1GC`。在垃圾收集是瓶颈的一些情况下，它可以提高性能。请注意，大执行人堆大小，可能重要的是增加了 [G1 区域大小](https://blogs.oracle.com/g1gc/entry/g1_gc_tuning_a_case) 与 `-XX:G1HeapRegionSize`。

* 例如，如果您的任务是从 HDFS 读取数据，则可以使用从 HDFS 读取的数据块的大小来估计任务使用的内存量。请注意，解压缩块的大小通常是块大小的2或3倍。所以如果我们希望有3或4个任务的工作空间，而 HDFS 块的大小是128MB，我们可以估计 Eden 的大小`4*3*128MB`。

* 监控垃圾收集的频率和时间如何随着新设置的变化而变化。

我们的经验表明， GC 调整的效果取决于您的应用程序和可用的内存量。有[更多的优化选项](http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html) 在线描述，但在较高的水平，管理完整的 GC 如何经常发生可以减少开销帮助。

可以通过`spark.executor.extraJavaOptions`在作业的配置中设置来指定执行器的 GC 调整标志。

# 其他注意事项

## 并行度水平

集群不会被充分利用，除非您将每个操作的并行级别设置得足够高。自动星火设置的 "地图" 任务的数量根据其大小对每个文件运行（尽管你可以通过可选的参数来控制它 `SparkContext.textFile` ，等等），以及用于分布式"减少"操作，如： `groupByKey` 和 `reduceByKey` ，它采用了最大父 RDD 的分区数。您可以将并行级别作为第二个参数传递（请参阅 [`spark.PairRDDFunctions`](api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions) 文档），或者将 config 属性设置 `spark.default.parallelism` 为更改默认值。一般来说，我们建议您的群集中每个 CPU 内核有2-3个任务。

## 减少任务的内存使用

有时，您将得到一个 OutOfMemoryError ，因为您的 RDD 不适合内存，而是因为您的其中一个任务的工作集（如其中一个 reduce 任务`groupByKey`）太大。 Spark 的 shuffle 操作（`sortByKey`， `groupByKey`， `reduceByKey`， `join`，等）建立每个任务中的哈希表来进行分组，而这往往是很大的。这里最简单的解决方案是*增加并行级别*，以便每个任务的输入集都更小。 Spark 可以有效地支持短达200 ms 的任务，因为它可以将多个任务中的一个执行者JVM重用，并且任务启动成本低，因此您可以将并行级别安全地提高到集群中的核心数量。

## 广播大的变量

使用 可用的[广播功能](programming-guide.html#broadcast-variables) `SparkContext` 可以大大减少每个序列化任务的大小，以及在群集上启动作业的成本。如果您的任务使用其中的驱动程序中的任何大对象（例如：静态查找表），请考虑将其变为广播变量。 Spark 打印主机上每个任务的序列化大小，因此您可以查看该任务以决定您的任务是否过大; 一般任务大于20 KB大概值得优化。

## 数据本地化

数据本地化可能会对 Spark job 的性能产生重大影响。如果数据和在其上操作的代码在一起，则计算往往是快速的。但如果代码和数据分开，则必须移动到另一个。通常，代码大小远小于数据，因此将数据代码从一个地方寄送到另一个地方比一大块数据更快。 Spark 围绕数据局部性的一般原则构建其调度。

数据本地化是指数据和代码处理有多近。根据数据的当前位置有几个地方级别。从最近到最远的顺序：

- `PROCESS_LOCAL` 数据与运行代码在同一个 JVM 中。这是可能的最好的地方
- `NODE_LOCAL` 数据在同一个节点上。示例可能在同一节点上的 HDFS 或同一节点上的另一个执行程序中。这比 `PROCESS_LOCAL` 因为数据必须在进程之间移动慢一些
- `NO_PREF` 数据从任何地方同样快速访问，并且没有本地偏好
- `RACK_LOCAL` 数据位于同一机架上的服务器上。数据位于同一机架上的不同服务器上，因此需要通过网络发送，通常通过单个交换机发送
- `ANY` 数据在网络上的其他地方，而不在同一个机架中

Spark 喜欢将所有 task 安排在最佳的本地级别，但这并不总是可能的。在任何空闲 executor 中没有未处理数据的情况下， Spark 将切换到较低的本地级别。有两个选项： a ）等待一个繁忙的 CPU 释放在相同服务器上的数据上启动任务，或者 b ）立即在更远的地方启动一个新的任务，需要在那里移动数据。

Spark 通常做的是等待一个繁忙的 CPU 释放的希望。一旦超时，它将开始将数据从远处移动到可用的 CPU 。每个级别之间的回退等待超时可以在一个参数中单独配置或全部配置; 有关详细信息，请参阅[配置页面](configuration.html#scheduling) `spark.locality` 上的 参数。如果您的 task 很长，并且本地化差，您应该增加这些设置，但默认值通常会很好。

# 概要
这是一个简短的指南，指出调整 Spark 应用程序时应该了解的主要问题 - 最重要的是数据序列化和内存调整。对于大多数程序，以序列化形式切换到 Kryo 序列化和持久化数据将会解决大多数常见的性能问题。随时在 [Spark 邮件列表](https://spark.apache.org/community.html) 中询问有关其他调优最佳做法的信息。
