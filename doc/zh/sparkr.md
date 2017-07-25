---
layout: global
displayTitle: SparkR (R on Spark)
title: SparkR (R on Spark)
---

* This will become a table of contents (this text will be scraped).
{:toc}

# 概述

SparkR 是一个 R package, 它提供了一个轻量级的前端以从 R 中使用 Apache Spark.
在 Spark {{site.SPARK_VERSION}} 中, SparkR 提供了一个分布式的 data frame, 它实现了像 selection, filtering, aggregation etc 一系列所支持的操作.（[dplyr](https://github.com/hadley/dplyr) 与 R data frames 相似) ）, 除了可用于海量数据上之外. SparkR 还支持使用 MLlib 来进行分布式的 machine learning（机器学习）.

# SparkDataFrame

SparkDataFrame 是一个分布式的, 将数据映射到有名称的 colums（列）的集合. 在概念上
相当于关系数据库中的 `table` 表或 R 中的 data frame，但在该引擎下有更多的优化.
SparkDataFrames 可以从各种来源构造，例如:
结构化的数据文件，Hive 中的表，外部数据库或现有的本地 R data frames.

All of the examples on this page use sample data included in R or the Spark distribution and can be run using the `./bin/sparkR` shell.

## 启动: SparkSession

<div data-lang="r"  markdown="1">
SparkR 的入口点是 `SparkSession`, 它会连接您的 R 程序到 Spark 集群中.
您可以使用 `sparkR.session` 来创建 `SparkSession`, 并传递诸如应用程序名称, 依赖的任何 spark 软件包等选项, 等等. 
此外，还可以通过 `SparkSession` 来与 `SparkDataFrames` 一起工作。 如果您正在使用 `sparkR` shell，那么 `SparkSession` 应该已经被创建了，你不需要再调用 `sparkR.session`.

<div data-lang="r" markdown="1">
{% highlight r %}
sparkR.session()
{% endhighlight %}
</div>

## 从 RStudio 来启动

您可以从 RStudio 中来启动 SparkR.
您可以从 RStudio, R shell, Rscript 或者 R IDEs 中连接你的 R 程序到 Spark 集群中去.
要开始, 确保已经在环境变量中设置好 SPARK_HOME (您可以检测下 [Sys.getenv](https://stat.ethz.ch/R-manual/R-devel/library/base/html/Sys.getenv.html)), 加载 SparkR package, 并且像下面一样调用 `sparkR.session`. 
它将检测 Spark 的安装, 并且, 如果没有发现, 它将自动的下载并且缓存起来. 当然，您也可以手动的运行 `install.spark`.

为了调用 `sparkR.session`, 您也可以指定某些 Spark driver 的属性.
通常哪些 [应用程序属性](configuration.html#application-properties) 和
[运行时环境](configuration.html#runtime-environment) 不能以编程的方式来设置, 这是因为 driver 的 JVM 进程早就已经启动了, 在这种情况下 SparkR 会帮你做好准备.
要设置它们, 可以像在 `sparkConfig` 参数中的其它属性一样传递它们到 `sparkR.session()` 中去.

<div data-lang="r" markdown="1">
{% highlight r %}
if (nchar(Sys.getenv("SPARK_HOME")) < 1) {
  Sys.setenv(SPARK_HOME = "/home/spark")
}
library(SparkR, lib.loc = c(file.path(Sys.getenv("SPARK_HOME"), "R", "lib")))
sparkR.session(master = "local[*]", sparkConfig = list(spark.driver.memory = "2g"))
{% endhighlight %}
</div>

下面的 Spark driver 属性可以 从 RStudio 的 sparkR.session 的 sparkConfig 中进行设置:

<table class="table">
  <tr><th>Property Name<（属性名称）</th><th>Property group（属性分组）</th><th><code>spark-submit</code> equivalent</th></tr>
  <tr>
    <td><code>spark.master</code></td>
    <td>Application Properties</td>
    <td><code>--master</code></td>
  </tr>
  <tr>
    <td><code>spark.yarn.keytab</code></td>
    <td>Application Properties</td>
    <td><code>--keytab</code></td>
  </tr>
  <tr>
    <td><code>spark.yarn.principal</code></td>
    <td>Application Properties</td>
    <td><code>--principal</code></td>
  </tr>
  <tr>
    <td><code>spark.driver.memory</code></td>
    <td>Application Properties</td>
    <td><code>--driver-memory</code></td>
  </tr>
  <tr>
    <td><code>spark.driver.extraClassPath</code></td>
    <td>Runtime Environment</td>
    <td><code>--driver-class-path</code></td>
  </tr>
  <tr>
    <td><code>spark.driver.extraJavaOptions</code></td>
    <td>Runtime Environment</td>
    <td><code>--driver-java-options</code></td>
  </tr>
  <tr>
    <td><code>spark.driver.extraLibraryPath</code></td>
    <td>Runtime Environment</td>
    <td><code>--driver-library-path</code></td>
  </tr>
</table>

</div>

## 创建 SparkDataFrames

有了一个 `SparkSession` 之后, 可以从一个本地的 R data frame, [Hive 表](sql-programming-guide.html#hive-tables), 或者其它的  [data sources](sql-programming-guide.html#data-sources) 中来创建 `SparkDataFrame` 应用程序. 

### 从本地的 data frames 来创建 SparkDataFrames

要创建一个 data frame 最简单的方式是去转换一个本地的 R data frame 成为一个 SparkDataFrame.
我们明确的使用 `as.DataFrame` 或 `createDataFrame` 并且经过本地的 R data frame 中以创建一个 SparkDataFrame.
例如, 下面的例子基于 R 中已有的 `faithful` 来创建一个 `SparkDataFrame`.

<div data-lang="r"  markdown="1">
{% highlight r %}
df <- as.DataFrame(faithful)

# 展示第一个 SparkDataFrame 的内容
head(df)
##  eruptions waiting
##1     3.600      79
##2     1.800      54
##3     3.333      74

{% endhighlight %}
</div>

### 从 Data Sources（数据源）创建 SparkDataFrame

SparkR 支持通过 `SparkDataFrame` 接口对各种 data sources（数据源）进行操作.
本节介绍使用数据源加载和保存数据的常见方法.
您可以查看 Spark Sql 编程指南的 [specific options](sql-programming-guide.html#manually-specifying-options) 部分以了解更多可用于内置的 data sources（数据源）内容.

从数据源创建 SparkDataFrames 常见的方法是 `read.df`.
此方法将加载文件的路径和数据源的类型，并且将自动使用当前活动的 SparkSession.
SparkR 天生就支持读取 JSON, CSV 和 Parquet 文件, 并且通过可靠来源的软件包 [第三方项目](http://spark.apache.org/third-party-projects.html), 您可以找到 Avro 等流行文件格式的 data source connectors（数据源连接器）.
可以用 `spark-submit` 或 `sparkR` 命令指定 `--packages` 来添加这些包, 或者在交互式 R shell 或从 RStudio 中使用`sparkPackages` 参数初始化 `SparkSession`.

<div data-lang="r" markdown="1">
{% highlight r %}
sparkR.session(sparkPackages = "com.databricks:spark-avro_2.11:3.0.0")
{% endhighlight %}
</div>

We can see how to use data sources using an example JSON input file. Note that the file that is used here is _not_ a typical JSON file. Each line in the file must contain a separate, self-contained valid JSON object. For more information, please see [JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/). As a consequence, a regular multi-line JSON file will most often fail.

我们可以看看如何使用 JSON input file 的例子来使用数据源.
注意, 这里使用的文件是 _not_ 一个经典的 JSON 文件.
文件中的每行都必须包含一个单独的，独立的有效的JSON对象


<div data-lang="r"  markdown="1">
{% highlight r %}
people <- read.df("./examples/src/main/resources/people.json", "json")
head(people)
##  age    name
##1  NA Michael
##2  30    Andy
##3  19  Justin

# SparkR 自动从 JSON 文件推断出 schema（模式）
printSchema(people)
# root
#  |-- age: long (nullable = true)
#  |-- name: string (nullable = true)

# 同样, 使用  read.json 读取多个文件
people <- read.json(c("./examples/src/main/resources/people.json", "./examples/src/main/resources/people2.json"))

{% endhighlight %}
</div>

该 data sources API 原生支持 CSV 格式的 input files（输入文件）. 要了解更多信息请参阅 SparkR [read.df](api/R/read.df.html) API 文档.

<div data-lang="r"  markdown="1">
{% highlight r %}
df <- read.df(csvPath, "csv", header = "true", inferSchema = "true", na.strings = "NA")

{% endhighlight %}
</div>

该 data sources API 也可用于将 SparkDataFrames 存储为多个 file formats（文件格式）.
例如, 我们可以使用 `write.df` 把先前的示例的 SparkDataFrame 存储为一个 Parquet 文件.

<div data-lang="r"  markdown="1">
{% highlight r %}
write.df(people, path = "people.parquet", source = "parquet", mode = "overwrite")
{% endhighlight %}
</div>

### 从 Hive tables 来创建 SparkDataFrame

您也可以从 Hive tables（表）来创建 SparkDataFrames.
为此，我们需要创建一个具有 Hive 支持的 SparkSession，它可以访问 Hive MetaStore 中的 tables（表）.
请注意, Spark 应该使用 [Hive support](building-spark.html#building-with-hive-and-jdbc-support) 来构建，更多细节可以在  [SQL 编程指南](sql-programming-guide.html#starting-point-sparksession) 中查阅.

<div data-lang="r" markdown="1">
{% highlight r %}
sparkR.session()

sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)")
sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")

# Queries can be expressed in HiveQL.
results <- sql("FROM src SELECT key, value")

# results is now a SparkDataFrame
head(results)
##  key   value
## 1 238 val_238
## 2  86  val_86
## 3 311 val_311

{% endhighlight %}
</div>

## SparkDataFrame 操作

SparkDataFrames 支持一些用于结构化数据处理的 functions（函数）.
这里我们包括一些基本的例子，一个完整的列表可以在 [API](api/R/index.html) 文档中找到:

### Selecting rows（行）, columns（列）

<div data-lang="r"  markdown="1">
{% highlight r %}
# Create the SparkDataFrame
df <- as.DataFrame(faithful)

# 获取关于 SparkDataFrame 基础信息
df
## SparkDataFrame[eruptions:double, waiting:double]

# Select only the "eruptions" column
head(select(df, df$eruptions))
##  eruptions
##1     3.600
##2     1.800
##3     3.333

# You can also pass in column name as strings
head(select(df, "eruptions"))

# Filter the SparkDataFrame to only retain rows with wait times shorter than 50 mins
head(filter(df, df$waiting < 50))
##  eruptions waiting
##1     1.750      47
##2     1.750      47
##3     1.867      48

{% endhighlight %}

</div>

### Grouping, Aggregation（分组, 聚合）

SparkR data frames 支持一些常见的, 用于在 grouping（分组）数据后进行 aggregate（聚合）的函数.
例如, 我们可以在 `faithful` dataset 中计算 `waiting` 时间的直方图, 如下所示.

<div data-lang="r"  markdown="1">
{% highlight r %}

# We use the `n` operator to count the number of times each waiting time appears
head(summarize(groupBy(df, df$waiting), count = n(df$waiting)))
##  waiting count
##1      70     4
##2      67     1
##3      69     2

# We can also sort the output from the aggregation to get the most common waiting times
waiting_counts <- summarize(groupBy(df, df$waiting), count = n(df$waiting))
head(arrange(waiting_counts, desc(waiting_counts$count)))
##   waiting count
##1      78    15
##2      83    14
##3      81    13

{% endhighlight %}
</div>

### Operating on Columns（列上的操作）

SparkR 还提供了一些可以直接应用于列进行数据处理和 aggregatation（聚合）的函数.
下面的例子展示了使用基本的算术函数.

<div data-lang="r"  markdown="1">
{% highlight r %}

# Convert waiting time from hours to seconds.
# Note that we can assign this to a new column in the same SparkDataFrame
df$waiting_secs <- df$waiting * 60
head(df)
##  eruptions waiting waiting_secs
##1     3.600      79         4740
##2     1.800      54         3240
##3     3.333      74         4440

{% endhighlight %}
</div>

### 应用 User-Defined Function（UDF 用户自定义函数）
在 SparkR 中, 我们支持几种 User-Defined Functions:

#### Run a given function on a large dataset using `dapply` or `dapplyCollect`

##### dapply
应用一个 function（函数）到 `SparkDataFrame` 的每个 partition（分区）.
应用于 `SparkDataFrame` 每个 partition（分区）的 function（函数）应该只有一个参数, 它中的 `data.frame` 对应传递的每个分区.
函数的输出应该是一个 `data.frame`.
Schema 指定生成的 `SparkDataFrame` row format.
它必须匹配返回值的 [data types](#data-type-mapping-between-r-and-spark).

<div data-lang="r"  markdown="1">
{% highlight r %}

# Convert waiting time from hours to seconds.
# Note that we can apply UDF to DataFrame.
schema <- structType(structField("eruptions", "double"), structField("waiting", "double"),
                     structField("waiting_secs", "double"))
df1 <- dapply(df, function(x) { x <- cbind(x, x$waiting * 60) }, schema)
head(collect(df1))
##  eruptions waiting waiting_secs
##1     3.600      79         4740
##2     1.800      54         3240
##3     3.333      74         4440
##4     2.283      62         3720
##5     4.533      85         5100
##6     2.883      55         3300
{% endhighlight %}
</div>

##### dapplyCollect
像 `dapply` 那样, 应用一个函数到 `SparkDataFrame` 的每个分区并且手机返回结果.
函数的输出应该是一个 `data.frame`.
但是, 不需要传递 Schema.
注意, 如果运行在所有分区上的函数的输出不能 pulled（拉）到 driver 的内存中过去, 则 `dapplyCollect` 会失败.

<div data-lang="r"  markdown="1">
{% highlight r %}

# Convert waiting time from hours to seconds.
# Note that we can apply UDF to DataFrame and return a R's data.frame
ldf <- dapplyCollect(
         df,
         function(x) {
           x <- cbind(x, "waiting_secs" = x$waiting * 60)
         })
head(ldf, 3)
##  eruptions waiting waiting_secs
##1     3.600      79         4740
##2     1.800      54         3240
##3     3.333      74         4440

{% endhighlight %}
</div>

#### Run a given function on a large dataset grouping by input column(s) and using `gapply` or `gapplyCollect`（在一个大的 dataset 上通过 input colums（输入列）来进行 grouping（分组）并且使用 `gapply` or `gapplyCollect` 来运行一个指定的函数）

##### gapply
应用给一个函数到 `SparkDataFrame` 的每个 group.
该函数被应用到 `SparkDataFrame` 的每个 group, 并且应该只有两个参数: grouping key 和 R `data.frame` 对应的 key.
该 groups 从 `SparkDataFrame` 的 columns（列）中选择.
函数的输出应该是 `data.frame`.
Schema 指定生成的 `SparkDataFrame` row format.
它必须在 Spark [data types 数据类型](#data-type-mapping-between-r-and-spark) 的基础上表示 R 函数的输出 schema（模式）.
用户可以设置返回的 `data.frame` 列名.

<div data-lang="r"  markdown="1">
{% highlight r %}

# Determine six waiting times with the largest eruption time in minutes.
schema <- structType(structField("waiting", "double"), structField("max_eruption", "double"))
result <- gapply(
    df,
    "waiting",
    function(key, x) {
        y <- data.frame(key, max(x$eruptions))
    },
    schema)
head(collect(arrange(result, "max_eruption", decreasing = TRUE)))

##    waiting   max_eruption
##1      64       5.100
##2      69       5.067
##3      71       5.033
##4      87       5.000
##5      63       4.933
##6      89       4.900
{% endhighlight %}
</div>

##### gapplyCollect
像 `gapply` 那样, 将函数应用于 `SparkDataFrame` 的每个分区，并将结果收集回 R data.frame.
函数的输出应该是一个 `data.frame`.
但是，不需要传递 schema（模式）.
请注意，如果在所有分区上运行的 UDF 的输出无法 pull（拉）到 driver 的内存, 那么 `gapplyCollect` 可能会失败.

<div data-lang="r"  markdown="1">
{% highlight r %}

# Determine six waiting times with the largest eruption time in minutes.
result <- gapplyCollect(
    df,
    "waiting",
    function(key, x) {
        y <- data.frame(key, max(x$eruptions))
        colnames(y) <- c("waiting", "max_eruption")
        y
    })
head(result[order(result$max_eruption, decreasing = TRUE), ])

##    waiting   max_eruption
##1      64       5.100
##2      69       5.067
##3      71       5.033
##4      87       5.000
##5      63       4.933
##6      89       4.900

{% endhighlight %}
</div>

#### 使用 `spark.lapply` 分发运行一个本地的 R 函数

##### spark.lapply
类似于本地 R 中的 `lapply`, `spark.lapply` 在元素列表中运行一个函数，并使用 Spark 分发计算.
以类似于 `doParallel` 或 `lapply` 的方式应用于列表的元素.
所有计算的结果应该放在一台机器上.
如果不是这样, 他们可以像 `df < - createDataFrame(list)` 这样做, 然后使用 `dapply`.

<div data-lang="r"  markdown="1">
{% highlight r %}
# Perform distributed training of multiple models with spark.lapply. Here, we pass
# a read-only list of arguments which specifies family the generalized linear model should be.
families <- c("gaussian", "poisson")
train <- function(family) {
  model <- glm(Sepal.Length ~ Sepal.Width + Species, iris, family = family)
  summary(model)
}
# Return a list of model's summaries
model.summaries <- spark.lapply(families, train)

# Print the summary of each model
print(model.summaries)

{% endhighlight %}
</div>

## SparkR 中运行 SQL 查询
A SparkDataFrame can also be registered as a temporary view in Spark SQL and that allows you to run SQL queries over its data.
The `sql` function enables applications to run SQL queries programmatically and returns the result as a `SparkDataFrame`.

<div data-lang="r"  markdown="1">
{% highlight r %}
# Load a JSON file
people <- read.df("./examples/src/main/resources/people.json", "json")

# Register this SparkDataFrame as a temporary view.
createOrReplaceTempView(people, "people")

# SQL statements can be run by using the sql method
teenagers <- sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")
head(teenagers)
##    name
##1 Justin

{% endhighlight %}
</div>

# 机器学习

## 算法

SparkR 现支持下列机器学习算法:

#### 分类

* [`spark.logit`](api/R/spark.logit.html): [`逻辑回归 Logistic Regression
`](ml-classification-regression.html#logistic-regression)
* [`spark.mlp`](api/R/spark.mlp.html): [`多层感知 (MLP)`](ml-classification-regression.html#multilayer-perceptron-classifier)
* [`spark.naiveBayes`](api/R/spark.naiveBayes.html): [`朴素贝叶斯`](ml-classification-regression.html#naive-bayes)
* [`spark.svmLinear`](api/R/spark.svmLinear.html): [`线性支持向量机`](ml-classification-regression.html#linear-support-vector-machine)

#### 回归

* [`spark.survreg`](api/R/spark.survreg.html): [`加速失败时间生存模型 Accelerated Failure Time (AFT) Survival  Model`](ml-classification-regression.html#survival-regression)
* [`spark.glm`](api/R/spark.glm.html) or [`glm`](api/R/glm.html): [`广义线性模型 Generalized Linear Model (GLM)`](ml-classification-regression.html#generalized-linear-regression)
* [`spark.isoreg`](api/R/spark.isoreg.html): [`保序回归`](ml-classification-regression.html#isotonic-regression)

#### 树

* [`spark.gbt`](api/R/spark.gbt.html): `梯度提升树 for` [`回归`](ml-classification-regression.html#gradient-boosted-tree-regression) `and` [`分类`](ml-classification-regression.html#gradient-boosted-tree-classifier)
* [`spark.randomForest`](api/R/spark.randomForest.html): `随机森林 for` [`回归`](ml-classification-regression.html#random-forest-regression) `and` [`分类`](ml-classification-regression.html#random-forest-classifier)

#### 聚类

* [`spark.bisectingKmeans`](api/R/spark.bisectingKmeans.html): [`二分k均值`](ml-clustering.html#bisecting-k-means)
* [`spark.gaussianMixture`](api/R/spark.gaussianMixture.html): [`高斯混合模型 (GMM)`](ml-clustering.html#gaussian-mixture-model-gmm)
* [`spark.kmeans`](api/R/spark.kmeans.html): [`K-Means`](ml-clustering.html#k-means)
* [`spark.lda`](api/R/spark.lda.html): [`隐含狄利克雷分布 (LDA)`](ml-clustering.html#latent-dirichlet-allocation-lda)

#### 协同过滤

* [`spark.als`](api/R/spark.als.html): [`交替最小二乘 (ALS)`](ml-collaborative-filtering.html#collaborative-filtering)

#### 频繁模式挖掘

* [`spark.fpGrowth`](api/R/spark.fpGrowth.html) : [`FP-growth`](ml-frequent-pattern-mining.html#fp-growth)

#### 统计

* [`spark.kstest`](api/R/spark.kstest.html): `柯尔莫哥洛夫-斯米尔诺夫检验`

SparkR 底层实现使用 MLlib 来训练模型. 有关示例代码，请参阅MLlib用户指南的相应章节.
用户可以调用`summary`输出拟合模型的摘要, 利用模型对数据进行[预测](api/R/predict.html), 并且使用 [write.ml](api/R/write.ml.html)/[read.ml](api/R/read.ml.html) 来 保存/加载拟合的模型 .
SparkR 支持对模型拟合使用部分R的公式运算符, 包括 ‘~’, ‘.’, ‘:’, ‘+’, 和 ‘-‘.


## 模型持久化

下面的例子展示了SparkR如何 保存/加载 机器学习模型.
{% include_example read_write r/ml/ml.R %}

# R和Spark之间的数据类型映射
<table class="table">
<tr><th>R</th><th>Spark</th></tr>
<tr>
  <td>byte</td>
  <td>byte</td>
</tr>
<tr>
  <td>integer</td>
  <td>integer</td>
</tr>
<tr>
  <td>float</td>
  <td>float</td>
</tr>
<tr>
  <td>double</td>
  <td>double</td>
</tr>
<tr>
  <td>numeric</td>
  <td>double</td>
</tr>
<tr>
  <td>character</td>
  <td>string</td>
</tr>
<tr>
  <td>string</td>
  <td>string</td>
</tr>
<tr>
  <td>binary</td>
  <td>binary</td>
</tr>
<tr>
  <td>raw</td>
  <td>binary</td>
</tr>
<tr>
  <td>logical</td>
  <td>boolean</td>
</tr>
<tr>
  <td><a href="https://stat.ethz.ch/R-manual/R-devel/library/base/html/DateTimeClasses.html">POSIXct</a></td>
  <td>timestamp</td>
</tr>
<tr>
  <td><a href="https://stat.ethz.ch/R-manual/R-devel/library/base/html/DateTimeClasses.html">POSIXlt</a></td>
  <td>timestamp</td>
</tr>
<tr>
  <td><a href="https://stat.ethz.ch/R-manual/R-devel/library/base/html/Dates.html">Date</a></td>
  <td>date</td>
</tr>
<tr>
  <td>array</td>
  <td>array</td>
</tr>
<tr>
  <td>list</td>
  <td>array</td>
</tr>
<tr>
  <td>env</td>
  <td>map</td>
</tr>
</table>

# Structured Streaming

SparkR 支持 Structured Streaming API (测试阶段). Structured Streaming 是一个 构建于SparkSQL引擎之上的易拓展、可容错的流式处理引擎. 更多信息请参考 R API [Structured Streaming Programming Guide](structured-streaming-programming-guide.html)

# R 函数名冲突

当在R中加载或引入(attach)一个新package时, 可能会发生函数名[冲突](https://stat.ethz.ch/R-manual/R-devel/library/base/html/library.html),一个函数掩盖了另一个函数

下列函数是被SparkR所掩盖的:

<table class="table">
  <tr><th>被掩盖函数</th><th>如何获取</th></tr>
  <tr>
    <td><code>cov</code> in <code>package:stats</code></td>
    <td><code><pre>stats::cov(x, y = NULL, use = "everything",
           method = c("pearson", "kendall", "spearman"))</pre></code></td>
  </tr>
  <tr>
    <td><code>filter</code> in <code>package:stats</code></td>
    <td><code><pre>stats::filter(x, filter, method = c("convolution", "recursive"),
              sides = 2, circular = FALSE, init)</pre></code></td>
  </tr>
  <tr>
    <td><code>sample</code> in <code>package:base</code></td>
    <td><code>base::sample(x, size, replace = FALSE, prob = NULL)</code></td>
  </tr>
</table>

由于SparkR的一部分是在`dplyr`软件包上建模的，因此SparkR中的某些函数与`dplyr`中同名. 根据两个包的加载顺序, 后加载的包会掩盖先加载的包的部分函数. 在这种情况下, 可以在函数名前指定包名前缀, 例如: `SparkR::cume_dist(x)` or `dplyr::cume_dist(x)`.

你可以在 R 中使用[`search()`](https://stat.ethz.ch/R-manual/R-devel/library/base/html/search.html)检查搜索路径


# 迁移指南

## SparkR 1.5.x 升级至 1.6.x

 - 在Spark 1.6.0 之前, 写入模式默认值为 `append`. 在 Spark 1.6.0 改为 `error` 匹配 Scala API.
 - SparkSQL 将R 中的 `NA` 转换为 `null`,反之亦然.

## SparkR 1.6.x 升级至 2.0

 -  `table` 方法已经移除并替换为 `tableToDF`.
 - 类 `DataFrame` 已改名为 `SparkDataFrame` 避免名称冲突.
 - Spark的 `SQLContext` 和 `HiveContext` 已经过时并替换为 `SparkSession`. 相应的摒弃 `sparkR.init()`而通过调用 `sparkR.session()` 来实例化SparkSession. 一旦实例化完成, 当前的SparkSession即可用于SparkDataFrame 操作(注释:spark2.0开始所有的driver实例通过sparkSession来进行构建).
 - `sparkR.session` 不支持 `sparkExecutorEnv` 参数.要为executors设置环境，请使用前缀"spark.executorEnv.VAR_NAME"设置Spark配置属性，例如"spark.executorEnv.PATH", 
 -`sqlContext` 不再需要下列函数: `createDataFrame`, `as.DataFrame`, `read.json`, `jsonFile`, `read.parquet`, `parquetFile`, `read.text`, `sql`, `tables`, `tableNames`, `cacheTable`, `uncacheTable`, `clearCache`, `dropTempTable`, `read.df`, `loadDF`, `createExternalTable`.
 -  `registerTempTable` 方法已经过期并且替换为`createOrReplaceTempView`.
 -  `dropTempTable` 方法已经过期并且替换为 `dropTempView`.
 - `sc` SparkContext 参数不再需要下列函数: `setJobGroup`, `clearJobGroup`, `cancelJobGroup`

## 升级至 SparkR 2.1.0

 - `join` 不再执行笛卡尔积计算, 使用 `crossJoin` 来进行笛卡尔积计算.

## 升级至 SparkR 2.2.0

 - `createDataFrame` 和 `as.DataFrame` 添加`numPartitions`参数. 数据分割时, 分区位置计算已经与scala计算相一致.
 - 方法 `createExternalTable` 已经过期并且替换为`createTable`. 可以调用这两种方法来创建外部或托管表. 已经添加额外的 catalog 方法.
 - 默认情况下，derby.log现在已保存到`tempdir()`目录中. 当实例化SparkSession且选项enableHiveSupport 为TRUE,会创建derby.log .
 - 更正`spark.lda` 错误设置优化器的bug.
 - 更新模型概况输出 `coefficients` as `matrix`. 更新的模型概况包括 `spark.logit`, `spark.kmeans`, `spark.glm`. `spark.gaussianMixture` 的模型概况已经添加对数概度(log-likelihood)  `loglik`.
