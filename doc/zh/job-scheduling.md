---
layout: global
title:  作业调度
---

* This will become a table of contents (this text will be scraped).
{:toc}

# 概述

Spark 有好几计算资源调度的方式. 首先,回忆一下 [集群模式概述](cluster-overview.html), 每个Spark 应用(包含一个SparkContext实例)中运行了一些其独占的执行器(executor)进程. 集群管理器提供了Spark 应用之间的资源调度[scheduling across applications](#scheduling-across-applications). 其次,
在各个Spark应用内部,各个线程可能并发地通过action算子提交多个Spark作业(job).如果你的应用服务于网络请求，那这种情况是很常见的. 在Spark应用内部(对应同一个SparkContext)各个作业之间，Spark默认FIFO调度，同时也可以支持公平调度 [fair scheduler](#scheduling-within-an-application).

# 跨应用调度

如果在集群上运行,每个Spark应用都会SparkContext获得一批独占的执行器JVM,来运行其任务并存储数据. 如果有多个用户共享集群,那么会有很多资源分配相关的选项,如何设计还取觉于具体的集群管理器.

对Spark所支持的各个集群管理器而言,最简单的的资源分配,就是静态划分.这种方式就意味着,每个Spark应用都是设定一个最大可用资源总量,并且该应用在整个生命周期内都会占住这个资源.这种方式在 Spark's独立部署 [standalone](spark-standalone.html)
和 [YARN](running-on-yarn.html)调度,以及Mesos粗粒度模式下都可用.[coarse-grained Mesos mode](running-on-mesos.html#mesos-run-modes) .
Resource allocation can be configured as follows, based on the cluster type:

* **Standalone mode:** 默认情况下,Spark应用在独立部署的集群中都会以FIFO(first-in-first-out)模式顺序提交运行,并且每个spark应用都会占用集群中所有可用节点.不过你可以通过设置spark.cores.max或者spark.deploy.defaultCores来限制单个应用所占用的节点个数.最后,除了可以控制对CPU的使用数量之外，还可以通过spark.executor.memory来控制各个应用的内存占用量. 
* **Mesos:** 在Mesos中要使用静态划分的话,需要将spark.mesos.coarse设为true,同样，你也需要配置spark.cores.max来控制各个应用的CPU总数，以及spark.executor.memory来控制各个应用的内存占用.
* **YARN:** 在YARN中需要使用 --num-executors 选项来控制Spark应用在集群中分配的执行器的个数.对于单个执行器(executor)所占用的资源,可以使用 --executor-memory和--executor-cores来控制.
Mesos上还有一种动态共享CPU的方式。在这种模式下,每个Spark应用的内存占用仍然是固定且独占的(仍由spark.exexcutor.memory决定),但是如果该Spark应用没有在某个机器上执行任务的话,那么其它应用可以占用该机器上的CPU。这种模式对集群中有大量不是很活跃应用的场景非常有效，例如：集群中有很多不同用户的Spark shell session.但这种模式不适用于低延时的场景,因为当Spark应用需要使用CPU的时候,可能需要等待一段时间才能取得对CPU的使用权。要使用这种模式,只需要在mesos://URL上设置spark.mesos.coarse属性为false即可。

值得注意的是,目前还没有任何一种资源分配模式支持跨Spark应用的内存共享。如果你想通过这种方式共享数据,我们建议你可以单独使用一个服务(例如：alluxio),这样就能实现多应用访问同一个RDD的数据。

## 动态资源分配

Spark 提供了一种基于负载来动态调节Spark应用资源占用的机制。这意味着，你的应用会在资源空闲的时间将其释放给集群，需要时再重新申请。这一特性在多个应用Spark集群资源的情况下特别有用.

这个特性默认是禁止的，但是在所有的粗粒度集群管理器上都是可用的,如：i.e.
独立部署模式[standalone mode](spark-standalone.html), [YARN mode](running-on-yarn.html), and
粗粒度模式[Mesos coarse-grained mode](running-on-mesos.html#mesos-run-modes).

### 配置和部署

要使用这一特性有两个前提条件。首先，你的应用必须设置spark.dynamicAllocation.enabled为true。其次，你必须在每个节点上启动external shuffle service,并将spark.shuffle.service.enabled设为true。external shuffle service 的目的是在移除executor的时候，能够保留executor输出的shuffle文件(本文后续有更新的描述
[below](job-scheduling.html#graceful-decommission-of-executors)). 启用external shuffle service 的方式在各个集群管理器上各不相同：

在Spark独立部署的集群中,你只需要在worker启动前设置spark.shuffle.service.enabled为true即可。

在Mesos粗粒度模式下,你需要在各个节点上运行$SPARK_HOME/sbin/start-mesos-shuffle-service.sh 并设置 spark.shuffle.service.enabled为true即可. 例如, 你可以在Marathon来启用这一功能。

在YARN模式下，需要按以下步骤在各个NodeManager上启动： [here](running-on-yarn.html#configuring-the-external-shuffle-service).

所有其它的配置都是可选的，在spark.dynamicAllocation.*和spark.shuffle.service.*这两个命明空间下有更加详细的介绍
[configurations page](configuration.html#dynamic-allocation).

### 资源分配策略

总体上来说,Spark应该在执行器空闲时将其关闭，而在后续要用时再申请。因为没有一个固定的方法,可以预测一个执行器在后续是否马上会被分配去执行任务，或者一个新分配的执行器实际上是空闲的，所以我们需要一个试探性的方法，来决定是否申请或是移除一个执行器。

#### 请求策略

一个启用了动态分配的Spark应用会有等待任务需要调度的时候，申请额外的执行器。在这种情况下，必定意味着已有的执行器已经不足以同时执行所有未完成的任务。

Spark会分轮次来申请执行器。实际的资源申请，会在任务挂起spark.dynamicAllocation.schedulerBacklogTimeout秒后首次触发，其后如果等待队列中仍有挂起的任务，则每过spark.dynamicAllocation.sustainedSchedulerBacklogTimeout秒后触发一次资源申请。另外，每一轮申请的执行器个数以指数形式增长。例如：一个Spark应用可能在首轮申请1个执行器，后续的轮次申请个数可能是2个、4个、8个....。

采用指数级增长策略的原因有两个：第一，对于任何一个Spark应用如果只需要多申请少数几个执行器的话，那么必须非常谨慎的启动资源申请，这和TCP慢启动有些类似；第二，如果一旦Spark应用确实需要申请多个执行器的话，那么可以确保其所需的计算资源及时增长。

#### 移除策略

移除执行器的策略就简单得多了。Spark应用会在某个执行器空闲超过 spark.dynamicAllocation.executorIdleTimeout秒后将其删除，在大多数情况下，执行器的移除条件和申请条件都是互斥的，也就是说，执行器在有等待执行任务挂起时，不应该空闲。

### 优雅的关闭Executor(执行器)

非动态分配模式下，执行器可能的退出原因有执行失败或是相关Spark应用已经退出。不管是哪种原因，执行器的所有状态都已经不再需要，可以丢弃掉。但是在动态分配的情况下，执行器有可能在Spark应用运行期间被移除。这时候，如果Spark应用尝试去访问该执行器存储的状态，就必须重算这一部分数据。因此，Spark需要一种机制，能够优雅的关闭执行器，同时还保留其状态数据。

这种需求对于混洗操作尤其重要。混洗过程中，Spark 执行器首先将 map 输出写到本地磁盘，同时执行器本身又是一个文件服务器，这样其他执行器就能够通过该执行器获得对应的 map 结果数据。一旦有某些任务执行时间过长，动态分配有可能在混洗结束前移除任务异常的执行器，而这些被移除的执行器对应的数据将会被重新计算，但这些重算其实是不必要的。

要解决这一问题，就需要用到 external shuffle service ，该服务在 Spark 1.2 引入。该服务在每个节点上都会启动一个不依赖于任何 Spark 应用或执行器的独立进程。一旦该服务启用，Spark 执行器不再从各个执行器上获取 shuffle 文件，转而从这个 service 获取。这意味着，任何执行器输出的混洗状态数据都可能存留时间比对应的执行器进程还长。

除了混洗文件之外，执行器也会在磁盘或者内存中缓存数。一旦执行器被移除，其缓存数据将无法访问。这个问题目前还没有解决。或许在未来的版本中，可能会采用外部混洗服务类似的方法，将缓存数据保存在堆外存储中以解决这一问题。 

# 应用内调度

在指定的 Spark 应用内部（对应同一 SparkContext 实例），多个线程可能并发地提交 Spark 作业（job）。在本节中，作业（job）是指，由 Spark action 算子（如 : collect）触发的一系列计算任务的集合。Spark 调度器是完全线程安全的，并且能够支持 Spark 应用同时处理多个请求（比如 : 来自不同用户的查询）。 

默认，Spark 应用内部使用 FIFO 调度策略。每个作业被划分为多个阶段（stage）（例如 : map 阶段和 reduce 阶段），第一个作业在其启动后会优先获取所有的可用资源，然后是第二个作业再申请，再第三个……。如果前面的作业没有把集群资源占满，则后续的作业可以立即启动运行，否则，后提交的作业会有明显的延迟等待。

不过从 Spark 0.8 开始，Spark 也能支持各个作业间的公平（Fair）调度。公平调度时，Spark 以轮询的方式给每个作业分配资源，因此所有的作业获得的资源大体上是平均分配。这意味着，即使有大作业在运行，小的作业再提交也能立即获得计算资源而不是等待前面的作业结束，大大减少了延迟时间。这种模式特别适合于多用户配置。
要启用公平调度器，只需设置一下 SparkContext 中 spark.scheduler.mode 属性为 FAIR 即可 : 

{% highlight scala %}
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.set("spark.scheduler.mode", "FAIR")
val sc = new SparkContext(conf)
{% endhighlight %}

## 公平调度资源池
公平调度器还可以支持将作业分组放入资源池（pool），然后给每个资源池配置不同的选项（如 : 权重）。这样你就可以给一些比较重要的作业创建一个“高优先级”资源池，或者你也可以把每个用户的作业分到一组，这样一来就是各个用户平均分享集群资源，而不是各个作业平分集群资源。Spark 公平调度的实现方式基本都是模仿 [Hadoop Fair Scheduler](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/FairScheduler.html). 来实现的。 

默认情况下，新提交的作业都会进入到默认资源池中，不过作业对应于哪个资源池，可以在提交作业的线程中用 SparkContext.setLocalProperty 设定 spark.scheduler.pool 属性。示例代码如下 : 

{% highlight scala %}
// Assuming sc is your SparkContext variable
sc.setLocalProperty("spark.scheduler.pool", "pool1")
{% endhighlight %}

一旦设好了局部属性，所有该线程所提交的作业（即 : 在该线程中调用action算子，如 : RDD.save/count/collect 等）都会使用这个资源池。这个设置是以线程为单位保存的，你很容易实现用同一线程来提交同一用户的所有作业到同一个资源池中。同样，如果需要清除资源池设置，只需在对应线程中调用如下代码 : 

{% highlight scala %}
sc.setLocalProperty("spark.scheduler.pool", null)
{% endhighlight %}

## 资源池默认行为

默认地，各个资源池之间平分整个集群的资源（包括 default 资源池），但在资源池内部，默认情况下，作业是 FIFO 顺序执行的。举例来说，如果你为每个用户创建了一个资源池，那么久意味着各个用户之间共享整个集群的资源，但每个用户自己提交的作业是按顺序执行的，而不会出现后提交的作业抢占前面作业的资源。

## 配置资源池属性

资源池的属性需要通过配置文件来指定。每个资源池都支持以下3个属性 : 

* `schedulingMode`: 可以是 FIFO 或 FAIR，控制资源池内部的作业是如何调度的。
* `weight`:  控制资源池相对其他资源池，可以分配到资源的比例。默认所有资源池的 weight 都是 1。如果你将某个资源池的 weight 设为 2，那么该资源池中的资源将是其他池子的2倍。如果将 weight 设得很高，如 1000，可以实现资源池之间的调度优先级 – 也就是说，weight=1000 的资源池总能立即启动其对应的作业。
* `minShare`:  除了整体 weight 之外，每个资源池还能指定一个最小资源分配值（CPU 个数），管理员可能会需要这个设置。公平调度器总是会尝试优先满足所有活跃（active）资源池的最小资源分配值，然后再根据各个池子的 weight 来分配剩下的资源。因此，minShare 属性能够确保每个资源池都能至少获得一定量的集群资源。minShare 的默认值是 0。

资源池属性是一个 XML 文件，可以基于 conf/fairscheduler.xml.template 修改，然后在 [SparkConf](configuration.html#spark-properties). 的 spark.scheduler.allocation.file 属性指定文件路径：

{% highlight scala %}
conf.set("spark.scheduler.allocation.file", "/path/to/file")
{% endhighlight %}

资源池 XML 配置文件格式如下，其中每个池子对应一个 <pool> 元素，每个资源池可以有其独立的配置 : 

{% highlight xml %}
<?xml version="1.0"?>
<allocations>
  <pool name="production">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>2</minShare>
  </pool>
  <pool name="test">
    <schedulingMode>FIFO</schedulingMode>
    <weight>2</weight>
    <minShare>3</minShare>
  </pool>
</allocations>
{% endhighlight %}

完整的例子可以参考 conf/fairscheduler.xml.template。注意，没有在配置文件中配置的资源池都会使用默认配置（schedulingMode : FIFO，weight : 1，minShare : 0）。
