---
layout: global
title: "MLlib: 主要指南"
displayTitle: "机器学习库 (MLlib) 指南"
---

MLib 是 Spark 的机器学习（ML）库。其目标是使实用的机器学习具有可扩展性并且变得容易。在较高的水平上，它提供了以下工具：

* ML Algorithms （ML 算法）: 常用的学习算法，如分类，回归，聚类和协同过滤
* Featurization （特征）: 特征提取，变换，降维和选择
* Pipelines （管道）: 用于构建，评估和调整 ML Pipelines 的工具
* Persistence （持久性）: 保存和加载算法，模型和 Pipelines
* Utilities （实用）: 线性代数，统计学，数据处理等

# 公告: 基于 DataFrame 的 API 是主要的 API

**MLlib 的基于 RDD 的 API 现在处于维护状态。**

从 Spark 2.0 开始， `spark.mllib` 包中的基于 [RDD](programming-guide.html#resilient-distributed-datasets-rdds) 的 API 已经进入了维护模式。Spark 的主要的机器学习 API 现在是 `spark.ml` 包中的基于 [DataFrame](sql-programming-guide.html) 的 API 。

*有什么影响？*

* MLlib 仍将支持基于 RDD 的 API ，在 `spark.mllib` 中有 bug 修复。
* MLlib 不会为基于 RDD 的 API 添加新功能。
* 在 Spark 2.x 发行版本中， MLlib 将向基于 DataFrames 的 API 添加功能，以达到与基于 RDD 的 API 的功能奇偶校验。
* 在达到功能奇偶校验（大概估计为 Spark 2.3 ）之后，基于 RDD 的 API 将被弃用。
* 预计将在 Spark 3.0 中删除基于 RDD 的 API 。

*为什么 MLlib 切换到基于 DataFrame 的 API ？*

* DataFrames 提供比 RDD 更加用户友好的 API 。 DataFrames 的许多好处包括 Spark Datasources，SQL/DataFrame 查询，Tungsten 和 Catalyst 优化以及跨语言的统一 API 。
* 用于 MLlib 的基于 DataFrame 的 API 为 ML algorithms （ML 算法）和跨多种语言提供了统一的 API 。
* DataFrames 便于实际的 ML Pipelines （ML 管道），特别是 feature transformations （特征转换）。有关详细信息，请参阅 [Pipelines 指南](ml-pipeline.html) 。

*什么是 "Spark ML" ?*

* "Spark ML" 不是官方名称，但偶尔用于引用基于 MLlib DataFrame 的 API 。这主要是由于基于 DataFrame 的 API 使用的 `org.apache.spark.ml` Scala 包名称，以及我们最初使用的 "Spark ML Pipelines" 术语来强调 pipeline （管道）概念。
  
*MLlib 是否被弃用了？*

* 不是。 MLlib 包括基于 RDD 的 API 和基于 DataFrame 的 API 。基于 RDD 的 API 现在处于维护模式。但是，API 和 MLlib 都不会被弃用。

# 依赖

MLlib 使用线性代数包 [Breeze](http://www.scalanlp.org/) ，这取决于 [netlib-java](https://github.com/fommil/netlib-java) 进行优化数字处理。如果 native libraries[^1] （本地库）在运行时不可用，您将看到一条警告消息并且 pure JVM 将使用实现。

由于运行时专用二进制文件的许可问题，我们不包括 `netlib-java` 的本机默认情况下代理。
要配置 `netlib-java` / Breeze 以使用系统优化的二进制文件，包括 `com.github.fommil.netlib：all：1.1.2` （或者用 `-Pnetlib-lgpl` 构建 Spark ）作为你项目的依赖并参阅 [netlib-java](https://github.com/fommil/netlib-java) 文档了解为您的平台的额外安装说明。

要在 Python 中使用 MLlib ，您将需要 [NumPy](http://www.numpy.org) 1.4 版本或者更高版本。

[^1]: 要了解更多关于本地系统优化的好处和背景，您可能希望观看 Sam Halliday 的 ScalaX 讨论关于 [Scala 中的高性能线性代数](http://fommil.github.io/scalax14/#/) 。

# 2.2 中的亮点

下面的列表突出显示了在 Spark 发行的 `2.2` 版本中添加到 MLlib 的一些新功能和增强功能：

* [`ALS`](ml-collaborative-filtering.html) methods for _top-k_ recommendations for all
 users or items, matching the functionality in `mllib`
 ([SPARK-19535](https://issues.apache.org/jira/browse/SPARK-19535)).
 Performance was also improved for both `ml` and `mllib`
 ([SPARK-11968](https://issues.apache.org/jira/browse/SPARK-11968) and
 [SPARK-20587](https://issues.apache.org/jira/browse/SPARK-20587))
* [`Correlation`](ml-statistics.html#correlation) and
 [`ChiSquareTest`](ml-statistics.html#hypothesis-testing) stats functions for `DataFrames`
 ([SPARK-19636](https://issues.apache.org/jira/browse/SPARK-19636) and
 [SPARK-19635](https://issues.apache.org/jira/browse/SPARK-19635))
* [`FPGrowth`](ml-frequent-pattern-mining.html#fp-growth) algorithm for frequent pattern mining
 ([SPARK-14503](https://issues.apache.org/jira/browse/SPARK-14503))
* `GLM` now supports the full `Tweedie` family
 ([SPARK-18929](https://issues.apache.org/jira/browse/SPARK-18929))
* [`Imputer`](ml-features.html#imputer) feature transformer to impute missing values in a dataset
 ([SPARK-13568](https://issues.apache.org/jira/browse/SPARK-13568))
* [`LinearSVC`](ml-classification-regression.html#linear-support-vector-machine)
 for linear Support Vector Machine classification
 ([SPARK-14709](https://issues.apache.org/jira/browse/SPARK-14709))
* Logistic regression now supports constraints on the coefficients during training
 ([SPARK-20047](https://issues.apache.org/jira/browse/SPARK-20047))

# Migration guide （迁移指南）

MLlib 正在积极发展。标记为 `Experimental`/`DeveloperApi` 的 APIs 可能会在将来的版本中更改，下面的迁移指南将解释发行版本之间的所有更改。

## 从 2.1 到 2.2

### 重大改变

没有重大改变。

### behavior 的弃用和更改

**弃用**

没有弃用。

**behavior 的更改**

* [SPARK-19787](https://issues.apache.org/jira/browse/SPARK-19787):对于`ALS.train` 方法 （标记为 `DeveloperApi` ）， `regParam` 的默认值从 `1.0` 更改为 `0.1` 。
 **注意** 这样做不会影响 `ALS` Estimator or Model （估计器或模型），也不影响 MLlib 的 `ALS` 类。
* [SPARK-14772](https://issues.apache.org/jira/browse/SPARK-14772):修复了 `Param.copy` 方法的 Python 和 Scala API 之间的不一致。
* [SPARK-11569](https://issues.apache.org/jira/browse/SPARK-11569): `StringIndexer`现在处理 `NULL` 值，就像 unseen values 一样。 以前是 exception 将始终抛出 `handleInvalid` 参数的设置。
  
## 以前的 Spark 版本

以前的迁移指南已经在 [这个页面](ml-migration-guides.html) 进行了归档。

---
