---
layout: global
displayTitle: SparkR (R on Spark)
title: SparkR (R on Spark)
---

* This will become a table of contents (this text will be scraped).
{:toc}

# Overview
SparkR is an R package that provides a light-weight frontend to use Apache Spark from R.
In Spark {{site.SPARK_VERSION}}, SparkR provides a distributed data frame implementation that
supports operations like selection, filtering, aggregation etc. (similar to R data frames,
[dplyr](https://github.com/hadley/dplyr)) but on large datasets. SparkR also supports distributed
machine learning using MLlib.

# SparkDataFrame

A SparkDataFrame is a distributed collection of data organized into named columns. It is conceptually
equivalent to a table in a relational database or a data frame in R, but with richer
optimizations under the hood. SparkDataFrames can be constructed from a wide array of sources such as:
structured data files, tables in Hive, external databases, or existing local R data frames.

All of the examples on this page use sample data included in R or the Spark distribution and can be run using the `./bin/sparkR` shell.

## Starting Up: SparkSession

<div data-lang="r"  markdown="1">
The entry point into SparkR is the `SparkSession` which connects your R program to a Spark cluster.
You can create a `SparkSession` using `sparkR.session` and pass in options such as the application name, any spark packages depended on, etc. Further, you can also work with SparkDataFrames via `SparkSession`. If you are working from the `sparkR` shell, the `SparkSession` should already be created for you, and you would not need to call `sparkR.session`.

<div data-lang="r" markdown="1">
{% highlight r %}
sparkR.session()
{% endhighlight %}
</div>

## Starting Up from RStudio

You can also start SparkR from RStudio. You can connect your R program to a Spark cluster from
RStudio, R shell, Rscript or other R IDEs. To start, make sure SPARK_HOME is set in environment
(you can check [Sys.getenv](https://stat.ethz.ch/R-manual/R-devel/library/base/html/Sys.getenv.html)),
load the SparkR package, and call `sparkR.session` as below. It will check for the Spark installation, and, if not found, it will be downloaded and cached automatically. Alternatively, you can also run `install.spark` manually.

In addition to calling `sparkR.session`,
 you could also specify certain Spark driver properties. Normally these
[Application properties](configuration.html#application-properties) and
[Runtime Environment](configuration.html#runtime-environment) cannot be set programmatically, as the
driver JVM process would have been started, in this case SparkR takes care of this for you. To set
them, pass them as you would other configuration properties in the `sparkConfig` argument to
`sparkR.session()`.

<div data-lang="r" markdown="1">
{% highlight r %}
if (nchar(Sys.getenv("SPARK_HOME")) < 1) {
  Sys.setenv(SPARK_HOME = "/home/spark")
}
library(SparkR, lib.loc = c(file.path(Sys.getenv("SPARK_HOME"), "R", "lib")))
sparkR.session(master = "local[*]", sparkConfig = list(spark.driver.memory = "2g"))
{% endhighlight %}
</div>

The following Spark driver properties can be set in `sparkConfig` with `sparkR.session` from RStudio:

<table class="table">
  <tr><th>Property Name</th><th>Property group</th><th><code>spark-submit</code> equivalent</th></tr>
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

## Creating SparkDataFrames
With a `SparkSession`, applications can create `SparkDataFrame`s from a local R data frame, from a [Hive table](sql-programming-guide.html#hive-tables), or from other [data sources](sql-programming-guide.html#data-sources).

### From local data frames
The simplest way to create a data frame is to convert a local R data frame into a SparkDataFrame. Specifically we can use `as.DataFrame` or `createDataFrame` and pass in the local R data frame to create a SparkDataFrame. As an example, the following creates a `SparkDataFrame` based using the `faithful` dataset from R.

<div data-lang="r"  markdown="1">
{% highlight r %}
df <- as.DataFrame(faithful)

# Displays the first part of the SparkDataFrame
head(df)
##  eruptions waiting
##1     3.600      79
##2     1.800      54
##3     3.333      74

{% endhighlight %}
</div>

### From Data Sources

SparkR supports operating on a variety of data sources through the `SparkDataFrame` interface. This section describes the general methods for loading and saving data using Data Sources. You can check the Spark SQL programming guide for more [specific options](sql-programming-guide.html#manually-specifying-options) that are available for the built-in data sources.

The general method for creating SparkDataFrames from data sources is `read.df`. This method takes in the path for the file to load and the type of data source, and the currently active SparkSession will be used automatically.
SparkR supports reading JSON, CSV and Parquet files natively, and through packages available from sources like [Third Party Projects](http://spark.apache.org/third-party-projects.html), you can find data source connectors for popular file formats like Avro. These packages can either be added by
specifying `--packages` with `spark-submit` or `sparkR` commands, or if initializing SparkSession with `sparkPackages` parameter when in an interactive R shell or from RStudio.

<div data-lang="r" markdown="1">
{% highlight r %}
sparkR.session(sparkPackages = "com.databricks:spark-avro_2.11:3.0.0")
{% endhighlight %}
</div>

We can see how to use data sources using an example JSON input file. Note that the file that is used here is _not_ a typical JSON file. Each line in the file must contain a separate, self-contained valid JSON object. For more information, please see [JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/). As a consequence, a regular multi-line JSON file will most often fail.

<div data-lang="r"  markdown="1">
{% highlight r %}
people <- read.df("./examples/src/main/resources/people.json", "json")
head(people)
##  age    name
##1  NA Michael
##2  30    Andy
##3  19  Justin

# SparkR automatically infers the schema from the JSON file
printSchema(people)
# root
#  |-- age: long (nullable = true)
#  |-- name: string (nullable = true)

# Similarly, multiple files can be read with read.json
people <- read.json(c("./examples/src/main/resources/people.json", "./examples/src/main/resources/people2.json"))

{% endhighlight %}
</div>

The data sources API natively supports CSV formatted input files. For more information please refer to SparkR [read.df](api/R/read.df.html) API documentation.

<div data-lang="r"  markdown="1">
{% highlight r %}
df <- read.df(csvPath, "csv", header = "true", inferSchema = "true", na.strings = "NA")

{% endhighlight %}
</div>

The data sources API can also be used to save out SparkDataFrames into multiple file formats. For example we can save the SparkDataFrame from the previous example
to a Parquet file using `write.df`.

<div data-lang="r"  markdown="1">
{% highlight r %}
write.df(people, path = "people.parquet", source = "parquet", mode = "overwrite")
{% endhighlight %}
</div>

### From Hive tables

You can also create SparkDataFrames from Hive tables. To do this we will need to create a SparkSession with Hive support which can access tables in the Hive MetaStore. Note that Spark should have been built with [Hive support](building-spark.html#building-with-hive-and-jdbc-support) and more details can be found in the [SQL programming guide](sql-programming-guide.html#starting-point-sparksession). In SparkR, by default it will attempt to create a SparkSession with Hive support enabled (`enableHiveSupport = TRUE`).

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

## SparkDataFrame Operations

SparkDataFrames support a number of functions to do structured data processing.
Here we include some basic examples and a complete list can be found in the [API](api/R/index.html) docs:

### Selecting rows, columns

<div data-lang="r"  markdown="1">
{% highlight r %}
# Create the SparkDataFrame
df <- as.DataFrame(faithful)

# Get basic information about the SparkDataFrame
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

### Grouping, Aggregation

SparkR data frames support a number of commonly used functions to aggregate data after grouping. For example we can compute a histogram of the `waiting` time in the `faithful` dataset as shown below

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

### Operating on Columns

SparkR also provides a number of functions that can directly applied to columns for data processing and during aggregation. The example below shows the use of basic arithmetic functions.

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

### Applying User-Defined Function
In SparkR, we support several kinds of User-Defined Functions:

#### Run a given function on a large dataset using `dapply` or `dapplyCollect`

##### dapply
Apply a function to each partition of a `SparkDataFrame`. The function to be applied to each partition of the `SparkDataFrame`
and should have only one parameter, to which a `data.frame` corresponds to each partition will be passed. The output of function should be a `data.frame`. Schema specifies the row format of the resulting a `SparkDataFrame`. It must match to [data types](#data-type-mapping-between-r-and-spark) of returned value.

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
Like `dapply`, apply a function to each partition of a `SparkDataFrame` and collect the result back. The output of function
should be a `data.frame`. But, Schema is not required to be passed. Note that `dapplyCollect` can fail if the output of UDF run on all the partition cannot be pulled to the driver and fit in driver memory.

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

#### Run a given function on a large dataset grouping by input column(s) and using `gapply` or `gapplyCollect`

##### gapply
Apply a function to each group of a `SparkDataFrame`. The function is to be applied to each group of the `SparkDataFrame` and should have only two parameters: grouping key and R `data.frame` corresponding to
that key. The groups are chosen from `SparkDataFrame`s column(s).
The output of function should be a `data.frame`. Schema specifies the row format of the resulting
`SparkDataFrame`. It must represent R function's output schema on the basis of Spark [data types](#data-type-mapping-between-r-and-spark). The column names of the returned `data.frame` are set by user.

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
Like `gapply`, applies a function to each partition of a `SparkDataFrame` and collect the result back to R data.frame. The output of the function should be a `data.frame`. But, the schema is not required to be passed. Note that `gapplyCollect` can fail if the output of UDF run on all the partition cannot be pulled to the driver and fit in driver memory.

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

#### Run local R functions distributed using `spark.lapply`

##### spark.lapply
Similar to `lapply` in native R, `spark.lapply` runs a function over a list of elements and distributes the computations with Spark.
Applies a function in a manner that is similar to `doParallel` or `lapply` to elements of a list. The results of all the computations
should fit in a single machine. If that is not the case they can do something like `df <- createDataFrame(list)` and then use
`dapply`

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

## Running SQL Queries from SparkR
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
