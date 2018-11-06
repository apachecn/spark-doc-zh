---
layout: global
title: Basic Statistics - RDD-based API
displayTitle: Basic Statistics - RDD-based API
---

* Table of contents
{:toc}


`\[
\newcommand{\R}{\mathbb{R}}
\newcommand{\E}{\mathbb{E}}
\newcommand{\x}{\mathbf{x}}
\newcommand{\y}{\mathbf{y}}
\newcommand{\wv}{\mathbf{w}}
\newcommand{\av}{\mathbf{\alpha}}
\newcommand{\bv}{\mathbf{b}}
\newcommand{\N}{\mathbb{N}}
\newcommand{\id}{\mathbf{I}}
\newcommand{\ind}{\mathbf{1}}
\newcommand{\0}{\mathbf{0}}
\newcommand{\unit}{\mathbf{e}}
\newcommand{\one}{\mathbf{1}}
\newcommand{\zero}{\mathbf{0}}
\]`

## Summary statistics
我们通过函数Statistics类的colStats为RDD[Vector]类型提供统计汇总信息。


<div class="codetabs">
<div data-lang="scala" markdown="1">

[`colStats()`](api/scala/index.html#org.apache.spark.mllib.stat.Statistics$) 返回
[`MultivariateStatisticalSummary`](api/scala/index.html#org.apache.spark.mllib.stat.MultivariateStatisticalSummary)的实例中包含最大、最小、平均值、方差、非零元和合计的统计信息,


请参照 [`MultivariateStatisticalSummary` Scala docs](api/scala/index.html#org.apache.spark.mllib.stat.MultivariateStatisticalSummary) 查看详细API.

{% include_example scala/org/apache/spark/examples/mllib/SummaryStatisticsExample.scala %}
</div>

<div data-lang="java" markdown="1">

[`colStats()`](api/java/org/apache/spark/mllib/stat/Statistics.html)  返回
[`MultivariateStatisticalSummary`](api/scala/index.html#org.apache.spark.mllib.sta
[`MultivariateStatisticalSummary`](api/java/org/apache/spark/mllib/stat/MultivariateStatisticalSummary.html)的实例中包含最大、最小、平均值、方差、非零元和合计的统计信息,


参照[`MultivariateStatisticalSummary` Java docs](api/java/org/apache/spark/mllib/stat/MultivariateStatisticalSummary.html) 查看详细API.

{% include_example java/org/apache/spark/examples/mllib/JavaSummaryStatisticsExample.java %}
</div>

<div data-lang="python" markdown="1">
[`colStats()`](api/python/pyspark.mllib.html#pyspark.mllib.stat.Statistics.colStats) 返回
[`MultivariateStatisticalSummary`](api/python/pyspark.mllib.html#pyspark.mllib.stat.MultivariateStatisticalSummary)的实例中包含最大、最小、平均值、方差、非零元和合计的统计信息.

参照 [`MultivariateStatisticalSummary` Python docs](api/python/pyspark.mllib.html#pyspark.mllib.stat.MultivariateStatisticalSummary) 查看详细API.

{% include_example python/mllib/summary_statistics_example.py %}
</div>

</div>

## Correlations 相关性

在汇总统计中计算两列数据的相关性是通用的操作。我们在`spark.mllib`提供了计算多列数据两两相关的灵活API.目前计算相关性的方法支持皮尔逊系数和斯皮尔曼系数。


<div class="codetabs">
<div data-lang="scala" markdown="1">
[`Statistics`](api/scala/index.html#org.apache.spark.mllib.stat.Statistics$)提供计算列与列之间相关性的方法。
根据输入的类型，两列`RDD[Double]` 还是一列 `RDD[Vector]`结果会分别对应`Double`或者是
`Matrix`

参照[`Statistics` Scala docs](api/scala/index.html#org.apache.spark.mllib.stat.Statistics$) 从API文档中获取详细信息。

{% include_example scala/org/apache/spark/examples/mllib/CorrelationsExample.scala %}
</div>

<div data-lang="java" markdown="1">
[`Statistics`](api/java/org/apache/spark/mllib/stat/Statistics.html)提供计算列与列之间相关性的方法。
根据输入的类型，两列`JavaDoubleRDD` 还是一列 `JavaRDD<Vector>`结果会分别对应`Double`或者是
`Matrix`

参照 [`Statistics` Java docs](api/java/org/apache/spark/mllib/stat/Statistics.html) 从API文档中获取详细信息.

{% include_example java/org/apache/spark/examples/mllib/JavaCorrelationsExample.java %}
</div>

<div data-lang="python" markdown="1">
[`Statistics`](api/python/pyspark.mllib.html#pyspark.mllib.stat.Statistics)提供计算列与列之间相关性的方法。
根据输入的类型，两列`RDD[Double]` 还是一列 `RDD[Vector]`结果会分别对应`Double`或者是`Matrix`

参照 [`Statistics` Python docs](api/python/pyspark.mllib.html#pyspark.mllib.stat.Statistics) 从API文档中获取详细信息.

{% include_example python/mllib/correlations_example.py %}
</div>

</div>

## Stratified sampling




与其他`spark.mllib`包中汇总统计的函数不同，`sampleByKey` and `sampleByKeyExact`等分层抽样方法可以在key-value 对的RDD上使用。
对于分层采样，可以将keys看做标签，value看做特征。例如key 可以是男女或者文档id，各自的value可以是人口统计的列表或者文档中的词汇表。
`sampleByKey`方法以掷硬币的方法决定是否进行采样，因此需要一次过滤完所有数据，并提供一个*期望*样本大小。`sampleByKeyExact`需要比`sampleByKey`中使用的每层简单随机抽样使用更多的资源，但提供在采样大小上将具有99.99%置信度的精确采样。
`sampleByKeyExact`目前在python api 中不支持。

<div class="codetabs">
<div data-lang="scala" markdown="1">
[`sampleByKeyExact()`](api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions) 允许用户精确采样
 $\lceil f_k \cdot n_k \rceil \, \forall k \in K$ 项,  $f_k$ 是key $k$ 期望的分数, $n_k$ 是key $k$ 的K-V对数量, $K$ key的集合。
 无放回抽样需要一次RDD上的附加过滤保证样本大小，而又放回抽样需要两个附加过滤。

{% include_example scala/org/apache/spark/examples/mllib/StratifiedSamplingExample.scala %}
</div>

<div data-lang="java" markdown="1">
[`sampleByKeyExact()`](api/java/org/apache/spark/api/java/JavaPairRDD.html) 允许用户精确采样
 $\lceil f_k \cdot n_k \rceil \, \forall k \in K$ 项,  $f_k$ 是key $k$ 期望的分数, $n_k$ 是key $k$ 的K-V对数量, $K$ key的集合。
 无放回抽样需要一次RDD上的附加过滤保证样本大小，而又放回抽样需要两个附加过滤。

{% include_example java/org/apache/spark/examples/mllib/JavaStratifiedSamplingExample.java %}
</div>
<div data-lang="python" markdown="1">
[`sampleByKey()`](api/python/pyspark.html#pyspark.RDD.sampleByKey) 允许用户精确采样
 $\lceil f_k \cdot n_k \rceil \, \forall k \in K$ 项,  $f_k$ 是key $k$ 期望的分数, $n_k$ 是key $k$ 的K-V对数量, $K$ key的集合。
 无放回抽样需要一次RDD上的附加过滤保证样本大小，而又放回抽样需要两个附加过滤。

*Note:* `sampleByKeyExact()` is currently not supported in Python.

{% include_example python/mllib/stratified_sampling_example.py %}
</div>

</div>

## Hypothesis testing
假设检验是用来判断结果是否具有统计学意义、这种结果是否偶然发生的有力工具。`spark.mllib`目前支持皮尔逊卡方( $\chi^2$)检验
用来验证拟合优度和独立性检验。输入类型决定了拟合优度或独立性测试是否进行。拟合优度要求输入类型必须是`Vector`，而独立性检验
要求输入必须是`Matrix`
`spark.mllib` 也支持输入类型`RDD[LabeledPoint]`以通过卡方检验独立性检测进行特征选择。

<div class="codetabs">
<div data-lang="scala" markdown="1">
[`Statistics`](api/scala/index.html#org.apache.spark.mllib.stat.Statistics$) 提供方法运行皮尔逊卡方检验。下面的例子展示了
如何运行和解释卡方检验。

{% include_example scala/org/apache/spark/examples/mllib/HypothesisTestingExample.scala %}
</div>

<div data-lang="java" markdown="1">
[`Statistics`](api/java/org/apache/spark/mllib/stat/Statistics.html)提供方法运行皮尔逊卡方检验。下面的例子展示了
如何运行和解释卡方检验。

参照 [`ChiSqTestResult` Java docs](api/java/org/apache/spark/mllib/stat/test/ChiSqTestResult.html) 从API 中获取详细信息.

{% include_example java/org/apache/spark/examples/mllib/JavaHypothesisTestingExample.java %}
</div>

<div data-lang="python" markdown="1">
[`Statistics`](api/python/index.html#pyspark.mllib.stat.Statistics$) 提供方法运行皮尔逊卡方检验。下面的例子展示了
如何运行和解释卡方检验。

参照 [`Statistics` Python docs](api/python/pyspark.mllib.html#pyspark.mllib.stat.Statistics) 从API 中获取详细信息.

{% include_example python/mllib/hypothesis_testing_example.py %}
</div>

</div>
另外，`spark.mllib`提供了检验符合某种分布的Kolmogorov Smirnov（KS）检验的1样本、2面的实现。
提供理论分布的名称（目前仅支持正太分布）和参数或者根据理论分布给出累计分布函数，用户可以检验他们的样本从该分布中抽取的原假设。
在这个用户对的正态分布的检验中并没有提供分布的参数，检验初始化标准正态分布并记录了适当的信息。
<div class="codetabs">
<div data-lang="scala" markdown="1">
[`Statistics`](api/scala/index.html#org.apache.spark.mllib.stat.Statistics$) 提供方法来执行
Kolmogorov Smirnov（KS）检验的1样本、2面的检验. 下面的例子展示了如何运行和理解假设检验。

参照 [`Statistics` Scala docs](api/scala/index.html#org.apache.spark.mllib.stat.Statistics$) 从API 中获取详细信息.

{% include_example scala/org/apache/spark/examples/mllib/HypothesisTestingKolmogorovSmirnovTestExample.scala %}
</div>

<div data-lang="java" markdown="1">
[`Statistics`](api/java/org/apache/spark/mllib/stat/Statistics.html) 提供方法来执行
Kolmogorov Smirnov（KS）检验的1样本、2面的检验. 下面的例子展示了如何运行和理解假设检验。

参照  [`Statistics` Java docs](api/java/org/apache/spark/mllib/stat/Statistics.html) 从API 中获取详细信息.

{% include_example java/org/apache/spark/examples/mllib/JavaHypothesisTestingKolmogorovSmirnovTestExample.java %}
</div>

<div data-lang="python" markdown="1">
[`Statistics`](api/python/pyspark.mllib.html#pyspark.mllib.stat.Statistics) 提供方法来执行
Kolmogorov Smirnov（KS）检验的1样本、2面的检验. 下面的例子展示了如何运行和理解假设检验。

参照  [`Statistics` Python docs](api/python/pyspark.mllib.html#pyspark.mllib.stat.Statistics) 从API 中获取详细信息.

{% include_example python/mllib/hypothesis_testing_kolmogorov_smirnov_test_example.py %}
</div>
</div>

### Streaming Significance Testing
`spark.mllib` 提供了流式测试的实现以支持分组测试用例。
这些测试用例用在Spark Streaming 的`DStream[(Boolean,Double)]`类型上；元组的第一个元素表示对照组(`false`) 或者实验组(`true`),元组的第二个元组是观测值
流式的显著性检验支持以下参数：

* `peacePeriod` - 忽略多少初始数据用来降低特殊值的影响。
* `windowSize` - The number of past batches to perform hypothesis 一个批量假设检验的数据量。设置为'0'将把之前所有批次的的数据累计处理。


<div class="codetabs">
<div data-lang="scala" markdown="1">
[`StreamingTest`](api/scala/index.html#org.apache.spark.mllib.stat.test.StreamingTest)
提供流式的假设检验.

{% include_example scala/org/apache/spark/examples/mllib/StreamingTestExample.scala %}
</div>

<div data-lang="java" markdown="1">
[`StreamingTest`](api/java/index.html#org.apache.spark.mllib.stat.test.StreamingTest)
提供流式的假设检验.

{% include_example java/org/apache/spark/examples/mllib/JavaStreamingTestExample.java %}
</div>
</div>


## Random data generation
随机数生成器是对随机数算法、原型法、性能测试非常有用的工具。`spark.mllib`支持根据给定的分布（如均匀分布、标准正态分布、泊松分布）提取独立同分布(i.i.d)的随机变量生成随机的RDD


<div class="codetabs">
<div data-lang="scala" markdown="1">
[`RandomRDDs`](api/scala/index.html#org.apache.spark.mllib.random.RandomRDDs$) 提供生成随机double类型的RDD或者向量类型的RDD的工厂方法。
下面的例子生成了一个随机double类型符合标准正态分布(0,1)的RDD然后将它转换为(1,4)的正态分布


Refer to the [`RandomRDDs` Scala docs](api/scala/index.html#org.apache.spark.mllib.random.RandomRDDs$) for details on the API.

{% highlight scala %}
import org.apache.spark.SparkContext
import org.apache.spark.mllib.random.RandomRDDs._

val sc: SparkContext = ...
//从标准正态分布`N(0, 1)`中提取100万独立同分布随机double变量生成10个分区的RDD
val u = normalRDD(sc, 1000000L, 10)
//用map 算子将RDD 随机数转换为符合正态分布`N(1, 4)`的RDD
val v = u.map(x => 1.0 + 2.0 * x)
{% endhighlight %}
</div>

<div data-lang="java" markdown="1">
[`RandomRDDs`](api/java/index.html#org.apache.spark.mllib.random.RandomRDDs) 提供生成随机double类型的RDD或者向量类型的RDD的工厂方法。
下面的例子生成了一个随机double类型符合标准正态分布(0,1)的RDD然后将它转换为(1,4)的正态分布。

Refer to the [`RandomRDDs` Java docs](api/java/org/apache/spark/mllib/random/RandomRDDs) for details on the API.

{% highlight java %}
import org.apache.spark.SparkContext;
import org.apache.spark.api.JavaDoubleRDD;
import static org.apache.spark.mllib.random.RandomRDDs.*;

JavaSparkContext jsc = ...

//从标准正态分布`N(0, 1)`中提取100万独立同分布随机double变量生成10个分区的RDD
JavaDoubleRDD u = normalJavaRDD(jsc, 1000000L, 10);
//用map 算子将RDD 随机数转换为符合正态分布`N(1, 4)`的RDD
JavaDoubleRDD v = u.mapToDouble(x -> 1.0 + 2.0 * x);
{% endhighlight %}
</div>

<div data-lang="python" markdown="1">
[`RandomRDDs`](api/python/pyspark.mllib.html#pyspark.mllib.random.RandomRDDs) 提供生成随机double类型的RDD或者向量类型的RDD的工厂方法。
下面的例子生成了一个随机double类型符合标准正态分布(0,1)的RDD然后将它转换为(1,4)的正态分布。

参照 [`RandomRDDs` Python docs](api/python/pyspark.mllib.html#pyspark.mllib.random.RandomRDDs) 从API 中获取详细信息。

{% highlight python %}
from pyspark.mllib.random import RandomRDDs

sc = ... # SparkContext

#从标准正态分布`N(0, 1)`中提取100万独立同分布随机double变量生成10个分区的RDD
u = RandomRDDs.normalRDD(sc, 1000000L, 10)
#用map 算子将RDD 随机数转换为符合正态分布`N(1, 4)`的RDD#
v = u.map(lambda x: 1.0 + 2.0 * x)
{% endhighlight %}
</div>
</div>

## Kernel density estimation

[核密度估计](https://en.wikipedia.org/wiki/Kernel_density_estimation) is a technique
useful for visualizing empirical probability distributions without requiring assumptions about the
particular distribution that the observed samples are drawn from. It computes an estimate of the
probability density function of a random variables, evaluated at a given set of points. It achieves
this estimate by expressing the PDF of the empirical distribution at a particular point as the
mean of PDFs of normal distributions centered around each of the samples.
核密度估计是一种对可视化经验概率分布 无需通过臆断从样本中得到特定分布。核密度估计是通过计算来估计随机变量的密度函数，通过给出
一个集合来评估结果。核密度估计通过特定点的概率密度函数。它通过在特定点表达经验分布的概率密度函数作为以每个样本为中心的正态分布的概率密度函数的平均值来实现这个估计

<div class="codetabs">

<div data-lang="scala" markdown="1">
[`KernelDensity`](api/scala/index.html#org.apache.spark.mllib.stat.KernelDensity) 提供了计算样本RDD的核密度估计方法。下面的例子展示了怎么实现它。

参照 [`KernelDensity` Scala docs](api/scala/index.html#org.apache.spark.mllib.stat.KernelDensity) 从API 中获取详细信息。

{% include_example scala/org/apache/spark/examples/mllib/KernelDensityEstimationExample.scala %}
</div>

<div data-lang="java" markdown="1">
[`KernelDensity`](api/java/index.html#org.apache.spark.mllib.stat.KernelDensity) 提供了计算样本RDD的核密度估计方法。下面的例子展示了怎么实现它。

参照 [`KernelDensity` Java docs](api/java/org/apache/spark/mllib/stat/KernelDensity.html) 从API 中获取详细信息。

{% include_example java/org/apache/spark/examples/mllib/JavaKernelDensityEstimationExample.java %}
</div>

<div data-lang="python" markdown="1">
[`KernelDensity`](api/python/pyspark.mllib.html#pyspark.mllib.stat.KernelDensity) 提供了计算样本RDD的核密度估计方法。下面的例子展示了怎么实现它。

参照 [`KernelDensity` Python docs](api/python/pyspark.mllib.html#pyspark.mllib.stat.KernelDensity) 从API 中获取详细信息。

{% include_example python/mllib/kernel_density_estimation_example.py %}
</div>

</div>
