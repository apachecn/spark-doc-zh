---
layout: global
title: Frequent Pattern Mining - RDD-based API
displayTitle: Frequent Pattern Mining - RDD-based API
---

挖掘频繁项，频繁项集，子序列或者是其他子结构 通常是分析大型数据集的第一步，这已经是数据挖掘多年的积极研究课题。有关 更多信息，请参考维基百科的[关联规则学习](http://en.wikipedia.org/wiki/Association_rule_learning)。 `spark.mllib` 提供 FP-growth 的并行实现，这是一种普遍的挖掘频繁项集的算法。

## FP-growth

FP-growth 算法是由韩家炜在 Han et al., Mining frequent patterns without candidate generation 提出的，“FP”代表频繁模式的意思。该算法的基本思路如下，给予一个事务数据库，FP-Growth 算法的第一步是每一项出现的次数，通过最小支持度进行筛选确定频繁项。不同于另一种关联算法Apriori 算法，FP-growth 算法第二步是通过产生后缀树( FP-tree )来存储所有的事务数据库中的项，而不是像 Apriori 算法那样花费大量的内存去产生候选项集。然后，通过遍历 FP-Tree 可以挖掘出频繁项集。在 spark.mllib 中，我们实现了并行的 FP-growth 算法叫做 PFP，正如论文Li et al., PFP: Parallel FP-growth for query recommendation 中描述的，PFP 将基于相同后缀的事务分发到相同的机器上，因此相比的单台机器的实现，这样有更好的扩展性。我们推荐用户读 Li et al., PFP: Parallel FP-growth for query recommendation这篇论文去理解更多的信息。
spark mllib 中 FP-growth 算法的实现要使用到以下两个超参数:
      • minSupport (最小支持度):一个项集被认为是频繁项集的最小支持度。例如，如果一个项总共有5个事务的数据集中，出现了3次，那么它的支持度为3/5=0.6
      • numPartitions (分区数):分区的个数，同于将事务分配到几个分区。
例子

文中描述了 FP-growth  算法， [Han et al., 挖掘频繁模式，无候选人生成](http://dx.doi.org/10.1145/335191.335372) ，其中 "FP" 代表频繁模式。给定交易数据集，FP-growth 的第一步是计算项目频率并识别频繁项。与同样目的设计的类似 [Apriori-like](http://en.wikipedia.org/wiki/Apriori_algorithm) 的算法不同， FP-growth 的第二步使用后缀树 (FP-tree) 结构来编码事务，而不会显式生成候选集，生成的代价通常很高。第二步之后，可以从 FP-tree 中提取频繁项集。在 `spark.mllib` 我们中，我们实现了一个名为 PFP 的 FP-growth 并行版本，在 [Li et al., PFP: Parallel FP-growth for query recommendation](http://dx.doi.org/10.1145/1454008.1454027) 中描述。PFP 根据事务后缀分配生长 FP-tree 的工作，因此比单机实现更具可扩展性。我们将用户参考论文了解更多细节。

`spark.mllib` 的 FP-growth 实现需要以下 (hyper-) 参数：

* minSupport (最小支持度)：项集的最小支持被识别为频繁。例如，如果一个项出现 3/5 的交易，它支持 3/5=0.6。
* numPartitions (分区数)：用于分发工作的分区数。

**示例：**

<div class="codetabs">
<div data-lang="scala" markdown="1">

[`FPGrowth`](api/scala/index.html#org.apache.spark.mllib.fpm.FPGrowth) 实现 FP-growth 算法。它采用事务的 RDD ，每个事务是 泛型类型的数组的 `Array`。 使用事务调用 `FPGrowth.run` 返回一个[`FPGrowthModel`](api/scala/index.html#org.apache.spark.mllib.fpm.FPGrowthModel) ，它存储具有其频率的频繁项集。
以下示例说明了如何从事务中挖掘频繁项集和关联规则（请参阅 [关联规则](mllib-frequent-pattern-mining.html#association-rules) 的详细信息）。

有关 API 的详细信息，请参阅 [`FPGrowth` Scala 文档](api/scala/index.html#org.apache.spark.mllib.fpm.FPGrowth)。

{% include_example scala/org/apache/spark/examples/mllib/SimpleFPGrowth.scala %}

</div>

<div data-lang="java" markdown="1">

[`FPGrowth`](api/java/org/apache/spark/mllib/fpm/FPGrowth.html) 实现 FP-growth 算法。 它采用 `JavaRDD` 的事务，其中每个事务是泛型类型的项的`Iterable` 。 使用事务调用 `FPGrowth.run` 返回一个 [`FPGrowthModel`](api/java/org/apache/spark/mllib/fpm/FPGrowthModel.html)，它存储具有其频率的频繁项集。 以下示例说明了如何从事务中挖掘频繁项集和关联规则（请参阅 [关联规则](mllib-frequent-pattern-mining.html#association-rules)  的详细信息）。

有关API的详细信息，请参阅 [`FPGrowth` Java 文档](api/java/org/apache/spark/mllib/fpm/FPGrowth.html)。

{% include_example java/org/apache/spark/examples/mllib/JavaSimpleFPGrowth.java %}

</div>

<div data-lang="python" markdown="1">

[`FPGrowth`](api/python/pyspark.mllib.html#pyspark.mllib.fpm.FPGrowth) 实现 FP-growth 算法。 它需要事务的 `RDD` ，其中每个事务是通用类型的项目的`List`。 使用事务调用 `FPGrowth.train` 返回一个 [`FPGrowthModel`](api/python/pyspark.mllib.html#pyspark.mllib.fpm.FPGrowthModel) ，它存储具有其频率的频繁项集。

有关API的更多详细信息，请参阅[`FPGrowth` Python 文档](api/python/pyspark.mllib.html#pyspark.mllib.fpm.FPGrowth)。

{% include_example python/mllib/fpgrowth_example.py %}

</div>

</div>

## 关联规则

<div class="codetabs">
<div data-lang="scala" markdown="1">

[AssociationRules](api/scala/index.html#org.apache.spark.mllib.fpm.AssociationRules) 实现了一个并行规则生成算法，用于构建具有单个项目作为结果的规则。

有关API的详细信息，请参阅 [`AssociationRules` Scala 文档](api/java/org/apache/spark/mllib/fpm/AssociationRules.html)。

{% include_example scala/org/apache/spark/examples/mllib/AssociationRulesExample.scala %}

</div>

<div data-lang="java" markdown="1">

[AssociationRules](api/java/org/apache/spark/mllib/fpm/AssociationRules.html) 实现了一个并行规则生成算法，用于构建具有单个项目作为结果的规则。
有关API的详细信息，请参阅 [`AssociationRules` Java 文档](api/java/org/apache/spark/mllib/fpm/AssociationRules.html)。

{% include_example java/org/apache/spark/examples/mllib/JavaAssociationRulesExample.java %}

</div>
</div>

## PrefixSpan

PrefixSpan 是一种序列模式挖掘算法， 具体描述见论文 [Pei et al., Mining Sequential Patterns by Pattern-Growth: The
PrefixSpan Approach](http://dx.doi.org/10.1109%2FTKDE.2004.77)。我们推荐读者去读相关的论文去更加深入的理解的序列模式挖掘问题。

`spark.mllib` 的 PrefixSpan 实现需要以下参数：

* minSupport (最小支持度)：最小支持度要求能够反映频繁序列模式。
* maxPatternLength(最大频繁序列的长度)：频繁顺序模式的最大长度。超过此长度的任何频繁模式将不包括在结果中。
* maxLocalProjDBSize (最大单机投影数据库的项数)：在投影数据库的本地迭代处理之前，前缀投影数据库中允许的最大项数量开始。应该根据你的执行者的大小来调整这个参数。

**示例：**

以下示例说明了在序列上运行的 PrefixSpan（使用与 Pei 等同的符号）：

~~~
  <(12)3>
  <1(32)(12)>
  <(12)5>
  <6>
~~~

<div class="codetabs">
<div data-lang="scala" markdown="1">

[`PrefixSpan`](api/scala/index.html#org.apache.spark.mllib.fpm.PrefixSpan) 实现 PrefixSpan 算法。 调用 `PrefixSpan.run` 返回一个 [`PrefixSpanModel`](api/scala/index.html#org.apache.spark.mllib.fpm.PrefixSpanModel)，用于存储具有其频率的频繁序列。

有关API的详细信息，请参阅 [`PrefixSpan` Scala 文档](api/scala/index.html#org.apache.spark.mllib.fpm.PrefixSpan) 和 [`PrefixSpanModel` Scala 文档](api/scala/index.html#org.apache.spark.mllib.fpm.PrefixSpanModel)。

{% include_example scala/org/apache/spark/examples/mllib/PrefixSpanExample.scala %}

</div>

<div data-lang="java" markdown="1">

[`PrefixSpan`](api/java/org/apache/spark/mllib/fpm/PrefixSpan.html) 实现 PrefixSpan 算法。 调用  `PrefixSpan.run`  返回一个 [`PrefixSpanModel`](api/java/org/apache/spark/mllib/fpm/PrefixSpanModel.html) ，用于存储具有其频率的频繁序列。

有关API的详细信息，请参阅 [`PrefixSpan` Java 文档](api/java/org/apache/spark/mllib/fpm/PrefixSpan.html) 和 [`PrefixSpanModel` Java 文档](api/java/org/apache/spark/mllib/fpm/PrefixSpanModel.html)。

{% include_example java/org/apache/spark/examples/mllib/JavaPrefixSpanExample.java %}

</div>
</div>

