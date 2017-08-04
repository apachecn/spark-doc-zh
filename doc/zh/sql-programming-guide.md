---
layout: global
displayTitle: Spark SQL, DataFrames and Datasets Guide
title: Spark SQL and DataFrames
---

* This will become a table of contents (this text will be scraped).
{:toc}

# Overview

Spark SQL 是 Spark 处理结构化数据的一个模块.与基础的 Spark RDD API 不同, Spark SQL 提供了查询结构化数据及计算结果等信息的接口.在内部, Spark SQL 使用这个额外的信息去执行额外的优化.有几种方式可以跟 Spark SQL 进行交互, 包括 SQL 和 Dataset API.当使用相同执行引擎进行计算时, 无论使用哪种 API / 语言都可以快速的计算.这种统一意味着开发人员能够在基于提供最自然的方式来表达一个给定的 transformation API 之间实现轻松的来回切换不同的 .

该页面所有例子使用的示例数据都包含在 Spark 的发布中, 并且可以使用 `spark-shell`, `pyspark` shell, 或者 `sparkR` shell来运行.


## SQL

Spark SQL 的功能之一是执行 SQL 查询.Spark SQL 也能够被用于从已存在的 Hive 环境中读取数据.更多关于如何配置这个特性的信息, 请参考 [Hive 表](#hive-tables) 这部分. 当以另外的编程语言运行SQL  时, 查询结果将以 [Dataset/DataFrame](#datasets-and-dataframes)的形式返回.您也可以使用 [命令行](#running-the-spark-sql-cli)或者通过 [JDBC/ODBC](#running-the-thrift-jdbcodbc-server)与 SQL 接口交互.

## Datasets and DataFrames

一个 Dataset 是一个分布式的数据集合
Dataset 是在 Spark 1.6 中被添加的新接口, 它提供了 RDD 的优点（强类型化, 能够使用强大的 lambda 函数）与Spark SQL执行引擎的优点.一个 Dataset 可以从 JVM 对象来 [构造](#creating-datasets) 并且使用转换功能（map, flatMap, filter, 等等）.
Dataset API 在[Scala][scala-datasets] 和
[Java][java-datasets]是可用的.Python 不支持 Dataset API.但是由于 Python 的动态特性, 许多 Dataset API 的优点已经可用了 (也就是说, 你可能通过 name 天生的`row.columnName`属性访问一行中的字段).这种情况和 R 相似.

一个 DataFrame 是一个 *Dataset* 组成的指定列.它的概念与一个在关系型数据库或者在 R/Python 中的表是相等的,  但是有很多优化. DataFrames 可以从大量的 [sources](#data-sources) 中构造出来, 比如: 结构化的文本文件, Hive中的表, 外部数据库, 或者已经存在的 RDDs.
DataFrame API 可以在 Scala,
Java, [Python](api/python/pyspark.sql.html#pyspark.sql.DataFrame), 和 [R](api/R/index.html)中实现.
在 Scala 和 Java中, 一个 DataFrame 所代表的是一个多个 `Row`（行）的的 Dataset（数据集合）.
在 [the Scala API][scala-datasets]中, `DataFrame` 仅仅是一个 `Dataset[Row]`类型的别名.
然而, 在 [Java API][java-datasets]中, 用户需要去使用 `Dataset<Row>` 去代表一个 `DataFrame`.

[scala-datasets]: api/scala/index.html#org.apache.spark.sql.Dataset
[java-datasets]: api/java/index.html?org/apache/spark/sql/Dataset.html

在此文档中, 我们将常常会引用 Scala/Java Datasets 的 `Row`s 作为 DataFrames.

# 开始入门

## 起始点: SparkSession

<div class="codetabs">
<div data-lang="scala"  markdown="1">

Spark SQL中所有功能的入口点是 [`SparkSession`](api/scala/index.html#org.apache.spark.sql.SparkSession) 类. 要创建一个 `SparkSession`, 仅使用 `SparkSession.builder()`就可以了:

{% include_example init_session scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java" markdown="1">

Spark SQL中所有功能的入口点是 [`SparkSession`](api/java/index.html#org.apache.spark.sql.SparkSession) 类. 要创建一个 `SparkSession`, 仅使用 `SparkSession.builder()`就可以了:

{% include_example init_session java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>

<div data-lang="python"  markdown="1">

Spark SQL中所有功能的入口点是 [`SparkSession`](api/python/pyspark.sql.html#pyspark.sql.SparkSession) 类. 要穿件一个 `SparkSession`, 仅使用 `SparkSession.builder`就可以了:

{% include_example init_session python/sql/basic.py %}
</div>

<div data-lang="r"  markdown="1">

Spark SQL中所有功能的入口点是 [`SparkSession`](api/R/sparkR.session.html) 类. 要初始化一个基本的 `SparkSession`, 仅调用 `sparkR.session()`即可:

{% include_example init_session r/RSparkSQLExample.R %}

注意第一次调用时, `sparkR.session()` 初始化一个全局的 `SparkSession` 单实例, 并且总是返回一个引用此实例, 可以连续的调用. 通过这种方式, 用户仅需要创建一次 `SparkSession` , 然后像 `read.df` SparkR函数就能够立即获取全局的实例,用户不需要再 `SparkSession` 之间进行实例的传递.
</div>
</div>

Spark 2.0 中的`SparkSession` 为 Hive 特性提供了内嵌的支持, 包括使用 HiveQL 编写查询的能力, 访问 Hive UDF,以及从 Hive 表中读取数据的能力.为了使用这些特性, 你不需要去有一个已存在的 Hive 设置.

## 创建 DataFrames

<div class="codetabs">
<div data-lang="scala"  markdown="1">
在一个 `SparkSession`中, 应用程序可以从一个 [已经存在的 `RDD`](#interoperating-with-rdds),
从hive表, 或者从 [Spark数据源](#data-sources)中创建一个DataFrames.

举个例子, 下面就是基于一个JSON文件创建一个DataFrame:

{% include_example create_df scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java" markdown="1">
在一个 `SparkSession`中, 应用程序可以从一个 [已经存在的 `RDD`](#interoperating-with-rdds),
从hive表, 或者从 [Spark数据源](#data-sources)中创建一个DataFrames.

举个例子, 下面就是基于一个JSON文件创建一个DataFrame:

{% include_example create_df java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>

<div data-lang="python"  markdown="1">
在一个 `SparkSession`中, 应用程序可以从一个 [已经存在的 `RDD`](#interoperating-with-rdds),
从hive表, 或者从 [Spark数据源](#data-sources)中创建一个DataFrames.

举个例子, 下面就是基于一个JSON文件创建一个DataFrame:

{% include_example create_df python/sql/basic.py %}
</div>

<div data-lang="r"  markdown="1">
在一个 `SparkSession`中, 应用程序可以从一个本地的R frame 数据,
从hive表, 或者从[Spark数据源](#data-sources).

举个例子, 下面就是基于一个JSON文件创建一个DataFrame:

{% include_example create_df r/RSparkSQLExample.R %}

</div>
</div>


## 无类型的Dataset操作 (aka DataFrame 操作)

DataFrames 提供了一个特定的语法用在 [Scala](api/scala/index.html#org.apache.spark.sql.Dataset), [Java](api/java/index.html?org/apache/spark/sql/Dataset.html), [Python](api/python/pyspark.sql.html#pyspark.sql.DataFrame) and [R](api/R/SparkDataFrame.html)中机构化数据的操作.

正如上面提到的一样, Spark 2.0中, DataFrames在Scala 和 Java API中, 仅仅是多个 `Row`s的Dataset. 这些操作也参考了与强类型的Scala/Java Datasets中的"类型转换" 对应的"无类型转换" .

这里包括一些使用 Dataset 进行结构化数据处理的示例 :

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example untyped_ops scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}

能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/scala/index.html#org.apache.spark.sql.Dataset).

除了简单的列引用和表达式之外, DataFrame 也有丰富的函数库, 包括 string 操作, date 算术, 常见的 math 操作以及更多.可用的完整列表请参考  [DataFrame 函数指南](api/scala/index.html#org.apache.spark.sql.functions$).
</div>

<div data-lang="java" markdown="1">

{% include_example untyped_ops java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}

为了能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/java/org/apache/spark/sql/Dataset.html).

除了简单的列引用和表达式之外, DataFrame 也有丰富的函数库, 包括 string 操作, date 算术, 常见的 math 操作以及更多.可用的完整列表请参考  [DataFrame 函数指南](api/java/org/apache/spark/sql/functions.html).
</div>

<div data-lang="python"  markdown="1">
在Python中, 可以通过(`df.age`) 或者(`df['age']`)来获取DataFrame的列. 虽然前者便于交互式操作, 但是还是建议用户使用后者, 这样不会破坏列名, 也能引用DataFrame的类.

{% include_example untyped_ops python/sql/basic.py %}
为了能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/python/pyspark.sql.html#pyspark.sql.DataFrame).

除了简单的列引用和表达式之外, DataFrame 也有丰富的函数库, 包括 string 操作, date 算术, 常见的 math 操作以及更多.可用的完整列表请参考  [DataFrame 函数指南](api/python/pyspark.sql.html#module-pyspark.sql.functions).

</div>

<div data-lang="r"  markdown="1">

{% include_example untyped_ops r/RSparkSQLExample.R %}

为了能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/R/index.html).

除了简单的列引用和表达式之外, DataFrame 也有丰富的函数库, 包括 string 操作, date 算术, 常见的 math 操作以及更多.可用的完整列表请参考  [DataFrame 函数指南](api/R/SparkDataFrame.html).

</div>

</div>

## Running SQL Queries Programmatically

<div class="codetabs">
<div data-lang="scala"  markdown="1">
`SparkSession`的`sql`函数可以使应用以编程的没事运行SQL查询并且返回一个`DataFrame`.

{% include_example run_sql scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java" markdown="1">
`SparkSession`的`sql`函数可以使应用以编程的没事运行SQL查询并且返回一个`Dataset<Row>`.

{% include_example run_sql java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>

<div data-lang="python"  markdown="1">
`SparkSession`的`sql`函数可以使应用以编程的没事运行SQL查询并且返回一个`DataFrame`.
{% include_example run_sql python/sql/basic.py %}
</div>

<div data-lang="r"  markdown="1">
`SparkSession`的`sql`函数可以使应用以编程的没事运行SQL查询并且返回一个`DataFrame`.

{% include_example run_sql r/RSparkSQLExample.R %}

</div>
</div>


## 全局临时视图

Spark SQL中的临时视图是session级别的, 也就是会随着session的消失而消失. 如果你想让一个临时视图在所有session中相互传递并且可用, 直到Spark 应用退出, 你可以建立一个全局的临时视图.全局的临时视图存在于系统数据库 `global_temp`中, 我们必须加上库名去引用它, 比如. `SELECT * FROM global_temp.view1`.

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example global_temp_view scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java" markdown="1">
{% include_example global_temp_view java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example global_temp_view python/sql/basic.py %}
</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}

CREATE GLOBAL TEMPORARY VIEW temp_view AS SELECT a + 1, b * 2 FROM tbl

SELECT * FROM global_temp.temp_view

{% endhighlight %}

</div>
</div>


## 创建Datasets

Dataset 与 RDD 相似, 然而, 并不是使用 Java 序列化或者 Kryo [编码器](api/scala/index.html#org.apache.spark.sql.Encoder) 来序列化用于处理或者通过网络进行传输的对象. 虽然编码器和标准的序列化都负责将一个对象序列化成字节, 编码器是动态生成的代码, 并且使用了一种允许 Spark 去执行许多像 filtering, sorting 以及 hashing 这样的操作, 不需要将字节反序列化成对象的格式.

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example create_ds scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java" markdown="1">
{% include_example create_ds java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>
</div>

## RDD的互操作性

Spark SQL 支持两种不同的方法用于转换已存在的 RDD 成为 Dataset.第一种方法是使用反射去推断一个包含指定的对象类型的 RDD 的 Schema.在你的 Spark 应用程序中当你已知 Schema 时这个基于方法的反射可以让你的代码更简洁.

第二种用于创建 Dataset 的方法是通过一个允许你构造一个 Schema 然后把它应用到一个已存在的 RDD 的编程接口.然而这种方法更繁琐, 当列和它们的类型知道运行时都是未知时它允许你去构造 Dataset.

### 使用反射推断Schema
<div class="codetabs">

<div data-lang="scala"  markdown="1">

Spark SQL 的 Scala 接口支持自动转换一个包含 case classes 的 RDD 为 DataFrame.Case class 定义了表的 Schema.Case class 的参数名使用反射读取并且成为了列名.Case class 也可以是嵌套的或者包含像 `Seq` 或者 `Array` 这样的复杂类型.这个 RDD 能够被隐式转换成一个 DataFrame 然后被注册为一个表.表可以用于后续的 SQL 语句.

{% include_example schema_inferring scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java"  markdown="1">

Spark SQL 支持一个[JavaBeans]的RDD(http://stackoverflow.com/questions/3295496/what-is-a-javabean-exactly)自动转换为一个DataFrame.
`BeanInfo`利用反射定义表的schema. 目前Spark SQL不支持含有`Map`的JavaBeans. 但是支持嵌套`List`或者 `Array`JavaBeans . 
你可以通过创建一个有getters和setters的序列化的类来创建一个JavaBean.

{% include_example schema_inferring java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>

<div data-lang="python"  markdown="1">

Spark SQL能够把RDD 转换为一个DataFrame, 并推断其类型. 这些行由一系列key/value键值对组成. key值代表了表的列名,类型按抽样推断整个数据集, 同样的也适用于JSON文件.

{% include_example schema_inferring python/sql/basic.py %}
</div>

</div>

### 以编程的方式指定Schema

<div class="codetabs">

<div data-lang="scala"  markdown="1">

当 case class 不能够在执行之前被定义（例如, records 记录的结构在一个 string 字符串中被编码了, 或者一个 text 文本 dataset 将被解析并且不同的用户投影的字段是不一样的）.一个 `DataFrame` 可以使用下面的三步以编程的方式来创建.

1. 从原始的 RDD 创建 RDD 的 `Row`（行）;
2. Step 1 被创建后, 创建 Schema 表示一个 `StructType` 匹配 RDD 中的 `Row`（行）的结构.
3. 通过 `SparkSession` 提供的 `createDataFrame` 方法应用 Schema 到 RDD 的 RowS（行）.

例如:

{% include_example programmatic_schema scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java"  markdown="1">

When JavaBean classes cannot be defined ahead of time (for example,
the structure of records is encoded in a string, or a text dataset will be parsed and
fields will be projected differently for different users),
a `Dataset<Row>` can be created programmatically with three steps.

1. Create an RDD of `Row`s from the original RDD;
2. Create the schema represented by a `StructType` matching the structure of
`Row`s in the RDD created in Step 1.
3. Apply the schema to the RDD of `Row`s via `createDataFrame` method provided
by `SparkSession`.

For example:

{% include_example programmatic_schema java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>

<div data-lang="python"  markdown="1">

当一个字典不能被提前定义 (例如,记录的结构是在一个字符串中, 抑或一个文本中解析, 被不同的用户所属),
一个 `DataFrame` 可以通过以下3步来创建.

1. RDD从原始的RDD穿件一个RDD的toples或者一个列表;
2. Step 1 被创建后, 创建 Schema 表示一个 `StructType` 匹配 RDD 中的结构.
3. 通过 `SparkSession` 提供的 `createDataFrame` 方法应用 Schema 到 RDD .

For example:

{% include_example programmatic_schema python/sql/basic.py %}
</div>

</div>

## Aggregations

The [built-in DataFrames functions](api/scala/index.html#org.apache.spark.sql.functions$) provide common
aggregations such as `count()`, `countDistinct()`, `avg()`, `max()`, `min()`, etc.
While those functions are designed for DataFrames, Spark SQL also has type-safe versions for some of them in
[Scala](api/scala/index.html#org.apache.spark.sql.expressions.scalalang.typed$) and
[Java](api/java/org/apache/spark/sql/expressions/javalang/typed.html) to work with strongly typed Datasets.
Moreover, users are not limited to the predefined aggregate functions and can create their own.

### Untyped User-Defined Aggregate Functions

<div class="codetabs">

<div data-lang="scala"  markdown="1">

Users have to extend the [UserDefinedAggregateFunction](api/scala/index.html#org.apache.spark.sql.expressions.UserDefinedAggregateFunction)
abstract class to implement a custom untyped aggregate function. For example, a user-defined average
can look like:

{% include_example untyped_custom_aggregation scala/org/apache/spark/examples/sql/UserDefinedUntypedAggregation.scala%}
</div>

<div data-lang="java"  markdown="1">

{% include_example untyped_custom_aggregation java/org/apache/spark/examples/sql/JavaUserDefinedUntypedAggregation.java%}
</div>

</div>

### Type-Safe User-Defined Aggregate Functions

User-defined aggregations for strongly typed Datasets revolve around the [Aggregator](api/scala/index.html#org.apache.spark.sql.expressions.Aggregator) abstract class.
For example, a type-safe user-defined average can look like:
<div class="codetabs">

<div data-lang="scala"  markdown="1">

{% include_example typed_custom_aggregation scala/org/apache/spark/examples/sql/UserDefinedTypedAggregation.scala%}
</div>

<div data-lang="java"  markdown="1">

{% include_example typed_custom_aggregation java/org/apache/spark/examples/sql/JavaUserDefinedTypedAggregation.java%}
</div>

</div>

# Data Sources （数据源）

Spark SQL 支持通过 DataFrame 接口对各种 data sources （数据源）进行操作.
DataFrame 可以使用 relational transformations （关系转换）操作, 也可用于创建 temporary view （临时视图）.
将 DataFrame 注册为 temporary view （临时视图）允许您对其数据运行 SQL 查询. 本节
描述了使用 Spark Data Sources 加载和保存数据的一般方法, 然后涉及可用于 built-in data sources （内置数据源）的 specific options （特定选项）.

## Generic Load/Save Functions （通用 加载/保存 功能）

在最简单的形式中, 默认数据源（`parquet`, 除非另有配置 `spark.sql.sources.default` ）将用于所有操作.

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example generic_load_save_functions scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example generic_load_save_functions java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">

{% include_example generic_load_save_functions python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">

{% include_example generic_load_save_functions r/RSparkSQLExample.R %}

</div>
</div>

### Manually Specifying Options （手动指定选项）

您还可以 manually specify （手动指定）将与任何你想传递给 data source 的其他选项一起使用的 data source . Data sources 由其 fully qualified name （完全限定名称）（即 `org.apache.spark.sql.parquet` ）, 但是对于 built-in sources （内置的源）, 你也可以使用它们的 shortnames （短名称）（`json`, `parquet`, `jdbc`, `orc`, `libsvm`, `csv`, `text`）.从任何 data source type （数据源类型）加载 DataFrames 可以使用此 syntax （语法）转换为其他类型.

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example manual_load_options scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example manual_load_options java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example manual_load_options python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">
{% include_example manual_load_options r/RSparkSQLExample.R %}
</div>
</div>

### Run SQL on files directly （直接在文件上运行 SQL）

不使用读取 API 将文件加载到 DataFrame 并进行查询, 也可以直接用 SQL 查询该文件.

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example direct_sql scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example direct_sql java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example direct_sql python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">
{% include_example direct_sql r/RSparkSQLExample.R %}

</div>
</div>

### Save Modes （保存模式）

Save operations （保存操作）可以选择使用 `SaveMode` , 它指定如何处理现有数据如果存在的话. 重要的是要意识到, 这些 save modes （保存模式）不使用任何 locking （锁定）并且不是 atomic （原子）. 另外, 当执行 `Overwrite` 时, 数据将在新数据写出之前被删除.

<table class="table">
<tr><th>Scala/Java</th><th>Any Language</th><th>Meaning</th></tr>
<tr>
  <td><code>SaveMode.ErrorIfExists</code> (default)</td>
  <td><code>"error"</code> (default)</td>
  <td>
    将 DataFrame 保存到 data source （数据源）时, 如果数据已经存在, 则会抛出异常.
  </td>
</tr>
<tr>
  <td><code>SaveMode.Append</code></td>
  <td><code>"append"</code></td>
  <td>
    将 DataFrame 保存到 data source （数据源）时, 如果 data/table 已存在, 则 DataFrame 的内容将被 append （附加）到现有数据中.
  </td>
</tr>
<tr>
  <td><code>SaveMode.Overwrite</code></td>
  <td><code>"overwrite"</code></td>
  <td>
    Overwrite mode （覆盖模式）意味着将 DataFrame 保存到 data source （数据源）时, 如果 data/table 已经存在, 则预期 DataFrame 的内容将 overwritten （覆盖）现有数据.
  </td>
</tr>
<tr>
  <td><code>SaveMode.Ignore</code></td>
  <td><code>"ignore"</code></td>
  <td>
    Ignore mode （忽略模式）意味着当将 DataFrame 保存到 data source （数据源）时, 如果数据已经存在, 则保存操作预期不会保存 DataFrame 的内容, 并且不更改现有数据. 这与 SQL 中的<code> CREATE TABLE IF NOT EXISTS </code> 类似.
  </td>
</tr>
</table>

### Saving to Persistent Tables （保存到持久表）

`DataFrames` 也可以使用 `saveAsTable` 命令作为 persistent tables （持久表）保存到 Hive metastore 中. 请注意, existing Hive deployment （现有的 Hive 部署）不需要使用此功能. Spark 将为您创建默认的 local Hive metastore （本地 Hive metastore）（使用 Derby ）. 与 `createOrReplaceTempView` 命令不同,  `saveAsTable` 将 materialize （实现） DataFrame 的内容, 并创建一个指向 Hive metastore 中数据的指针. 即使您的 Spark 程序重新启动,  Persistent tables （持久性表）仍然存在, 因为您保持与同一个 metastore 的连接. 可以通过使用表的名称在 `SparkSession` 上调用 `table` 方法来创建 persistent tabl （持久表）的 DataFrame .

对于 file-based （基于文件）的 data source （数据源）, 例如 text, parquet, json等, 您可以通过 `path` 选项指定 custom table path （自定义表路径）, 例如 `df.write.option("path", "/some/path").saveAsTable("t")` . 当表被 dropped （删除）时, custom table path （自定义表路径）将不会被删除, 并且表数据仍然存在. 如果未指定自定义表路径, Spark 将把数据写入 warehouse directory （仓库目录）下的默认表路径. 当表被删除时, 默认的表路径也将被删除.

从 Spark 2.1 开始, persistent datasource tables （持久性数据源表）将 per-partition metadata （每个分区元数据）存储在 Hive metastore 中. 这带来了几个好处:

- 由于 metastore 只能返回查询的必要 partitions （分区）, 因此不再需要将第一个查询上的所有 partitions discovering 到表中.
- Hive DDLs 如 `ALTER TABLE PARTITION ... SET LOCATION` 现在可用于使用 Datasource API 创建的表.

请注意, 创建 external datasource tables （外部数据源表）（带有 `path` 选项）的表时, 默认情况下不会收集 partition information （分区信息）. 要 sync （同步） metastore 中的分区信息, 可以调用 `MSCK REPAIR TABLE` .

### Bucketing, Sorting and Partitioning （分桶, 排序和分区）

对于 file-based data source （基于文件的数据源）, 也可以对 output （输出）进行 bucket 和 sort 或者 partition .
Bucketing 和 sorting 仅适用于 persistent tables :

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% include_example write_sorting_and_bucketing scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example write_sorting_and_bucketing java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example write_sorting_and_bucketing python/sql/datasource.py %}
</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}

CREATE TABLE users_bucketed_by_name(
  name STRING,
  favorite_color STRING,
  favorite_numbers array<integer>
) USING parquet 
CLUSTERED BY(name) INTO 42 BUCKETS;

{% endhighlight %}

</div>

</div>

在使用 Dataset API 时, partitioning 可以同时与 `save` 和 `saveAsTable` 一起使用.


<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% include_example write_partitioning scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example write_partitioning java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example write_partitioning python/sql/datasource.py %}
</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}

CREATE TABLE users_by_favorite_color(
  name STRING, 
  favorite_color STRING,
  favorite_numbers array<integer>
) USING csv PARTITIONED BY(favorite_color);

{% endhighlight %}

</div>

</div>

可以为 single table （单个表）使用 partitioning 和 bucketing:

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% include_example write_partition_and_bucket scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example write_partition_and_bucket java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example write_partition_and_bucket python/sql/datasource.py %}
</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}

CREATE TABLE users_bucketed_and_partitioned(
  name STRING,
  favorite_color STRING,
  favorite_numbers array<integer>
) USING parquet 
PARTITIONED BY (favorite_color)
CLUSTERED BY(name) SORTED BY (favorite_numbers) INTO 42 BUCKETS;

{% endhighlight %}

</div>

</div>

`partitionBy` 创建一个 directory structure （目录结构）, 如 [Partition Discovery](#partition-discovery) 部分所述.
因此, 对 cardinality （基数）较高的 columns 的适用性有限. 相反, `bucketBy` 可以在固定数量的 buckets 中分配数据, 并且可以在 a number of unique values is unbounded （多个唯一值无界时）使用数据.

## Parquet Files

[Parquet](http://parquet.io) 是许多其他数据处理系统支持的 columnar format （柱状格式）.
Spark SQL 支持读写 Parquet 文件, 可自动保留 schema of the original data （原始数据的模式）. 当编写 Parquet 文件时, 出于兼容性原因, 所有 columns 都将自动转换为可空.

### Loading Data Programmatically （以编程的方式加载数据）

使用上面例子中的数据:

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% include_example basic_parquet_example scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example basic_parquet_example java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">

{% include_example basic_parquet_example python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">

{% include_example basic_parquet_example r/RSparkSQLExample.R %}

</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}

CREATE TEMPORARY VIEW parquetTable
USING org.apache.spark.sql.parquet
OPTIONS (
  path "examples/src/main/resources/people.parquet"
)

SELECT * FROM parquetTable

{% endhighlight %}

</div>

</div>

### Partition Discovery （分区发现）

Table partitioning （表分区）是在像 Hive 这样的系统中使用的常见的优化方法. 在 partitioned table （分区表）中, 数据通常存储在不同的目录中, partitioning column values encoded （分区列值编码）在每个 partition directory （分区目录）的路径中. Parquet data source （Parquet 数据源）现在可以自动 discover （发现）和 infer （推断）分区信息. 例如, 我们可以使用以下 directory structure （目录结构）将所有以前使用的 population data （人口数据）存储到 partitioned table （分区表）中, 其中有两个额外的列 `gender` 和 `country` 作为 partitioning columns （分区列）:

{% highlight text %}

path
└── to
    └── table
        ├── gender=male
        │   ├── ...
        │   │
        │   ├── country=US
        │   │   └── data.parquet
        │   ├── country=CN
        │   │   └── data.parquet
        │   └── ...
        └── gender=female
            ├── ...
            │
            ├── country=US
            │   └── data.parquet
            ├── country=CN
            │   └── data.parquet
            └── ...

{% endhighlight %}

通过将 `path/to/table` 传递给 `SparkSession.read.parquet` 或 `SparkSession.read.load` , Spark SQL 将自动从路径中提取 partitioning information （分区信息）.
现在返回的 DataFrame 的 schema （模式）变成:

{% highlight text %}

root
|-- name: string (nullable = true)
|-- age: long (nullable = true)
|-- gender: string (nullable = true)
|-- country: string (nullable = true)

{% endhighlight %}

请注意, 会自动 inferred （推断） partitioning columns （分区列）的 data types （数据类型）.目前, 支持 numeric data types （数字数据类型）和 string type （字符串类型）.有些用户可能不想自动推断 partitioning columns （分区列）的数据类型.对于这些用例,  automatic type inference （自动类型推断）可以由 `spark.sql.sources.partitionColumnTypeInference.enabled` 配置, 默认为 `true` .当禁用 type inference （类型推断）时, string type （字符串类型）将用于 partitioning columns （分区列）.

从 Spark 1.6.0 开始, 默认情况下, partition discovery （分区发现）只能找到给定路径下的 partitions （分区）.对于上述示例, 如果用户将 `path/to/table/gender=male` 传递给 `SparkSession.read.parquet` 或 `SparkSession.read.load` , 则 `gender` 将不被视为 partitioning column （分区列）.如果用户需要指定 partition discovery （分区发现）应该开始的基本路径, 则可以在数据源选项中设置 `basePath`.例如, 当 `path/to/table/gender=male` 是数据的路径并且用户将 `basePath` 设置为 `path/to/table/`, `gender` 将是一个 partitioning column （分区列）.

### Schema Merging （模式合并）

像 ProtocolBuffer ,  Avro 和 Thrift 一样, Parquet 也支持 schema evolution （模式演进）. 用户可以从一个 simple schema （简单的架构）开始, 并根据需要逐渐向 schema 添加更多的 columns （列）. 以这种方式, 用户可能会使用不同但相互兼容的 schemas 的 multiple Parquet files （多个 Parquet 文件）. Parquet data source （Parquet 数据源）现在能够自动检测这种情况并 merge （合并）所有这些文件的 schemas .

由于 schema merging （模式合并）是一个 expensive operation （相对昂贵的操作）, 并且在大多数情况下不是必需的, 所以默认情况下从 1.5.0 开始. 你可以按照如下的方式启用它:

1. 读取 Parquet 文件时, 将 data source option （数据源选项） `mergeSchema` 设置为 `true` （如下面的例子所示）, 或
2. 将 global SQL option （全局 SQL 选项） `spark.sql.parquet.mergeSchema` 设置为 `true` .

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% include_example schema_merging scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example schema_merging java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">

{% include_example schema_merging python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">

{% include_example schema_merging r/RSparkSQLExample.R %}

</div>

</div>

### Hive metastore Parquet table conversion （Hive metastore Parquet table 转换）

当读取和写入 Hive metastore Parquet 表时, Spark SQL 将尝试使用自己的 Parquet support （Parquet 支持）, 而不是 Hive SerDe 来获得更好的性能. 此 behavior （行为）由 `spark.sql.hive.convertMetastoreParquet` 配置控制, 默认情况下 turned on （打开）.

#### Hive/Parquet Schema Reconciliation

从 table schema processing （表格模式处理）的角度来说, Hive 和 Parquet 之间有两个关键的区别.

1. Hive 不区分大小写, 而 Parquet 不是
1. Hive 认为所有 columns （列）都可以为空, 而 Parquet 中的可空性是 significant （重要）的.

由于这个原因, 当将 Hive metastore Parquet 表转换为 Spark SQL Parquet 表时, 我们必须调整 metastore schema 与 Parquet schema. reconciliation 规则是:

1. 在两个 schema 中具有 same name （相同名称）的 Fields （字段）必须具有 same data type （相同的数据类型）, 而不管 nullability （可空性）. reconciled field 应具有 Parquet 的数据类型, 以便 nullability （可空性）得到尊重.

1. reconciled schema （调和模式）正好包含 Hive metastore schema 中定义的那些字段.

   - 只出现在 Parquet schema 中的任何字段将被 dropped （删除）在 reconciled schema 中.
   - 仅在 Hive metastore schema 中出现的任何字段在 reconciled schema 中作为 nullable field （可空字段）添加.

#### Metadata Refreshing （元数据刷新）

Spark SQL 缓存 Parquet metadata 以获得更好的性能. 当启用 Hive metastore Parquet table conversion （转换）时, 这些 converted tables （转换表）的 metadata （元数据）也被 cached （缓存）. 如果这些表由 Hive 或其他外部工具更新, 则需要手动刷新以确保 consistent metadata （一致的元数据）.

<div class="codetabs">

<div data-lang="scala"  markdown="1">

{% highlight scala %}
// spark is an existing SparkSession
spark.catalog.refreshTable("my_table")
{% endhighlight %}

</div>

<div data-lang="java"  markdown="1">

{% highlight java %}
// spark is an existing SparkSession
spark.catalog().refreshTable("my_table");
{% endhighlight %}

</div>

<div data-lang="python"  markdown="1">

{% highlight python %}
# spark is an existing SparkSession
spark.catalog.refreshTable("my_table")
{% endhighlight %}

</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}
REFRESH TABLE my_table;
{% endhighlight %}

</div>

</div>

### Configuration （配置）

可以使用 `SparkSession` 上的 `setConf` 方法或使用 SQL 运行 `SET key = value` 命令来完成 Parquet 的配置.

<table class="table">
<tr><th>Property Name （参数名称）</th><th>Default（默认）</th><th>Meaning（含义）</th></tr>
<tr>
  <td><code>spark.sql.parquet.binaryAsString</code></td>
  <td>false</td>
  <td>
    一些其他 Parquet-producing systems （Parquet 生产系统）, 特别是 Impala, Hive 和旧版本的 Spark SQL , 在 writing out （写出） Parquet schema 时, 不区分 binary data （二进制数据）和 strings （字符串）. 该 flag 告诉 Spark SQL 将 binary data （二进制数据）解释为 string （字符串）以提供与这些系统的兼容性.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.int96AsTimestamp</code></td>
  <td>true</td>
  <td>
    一些 Parquet-producing systems , 特别是 Impala 和 Hive , 将 Timestamp 存入INT96 . 该 flag 告诉 Spark SQL 将 INT96 数据解析为 timestamp 以提供与这些系统的兼容性.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.cacheMetadata</code></td>
  <td>true</td>
  <td>
    打开 Parquet schema metadata 的缓存. 可以加快查询静态数据.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.compression.codec</code></td>
  <td>snappy</td>
  <td>
    在编写 Parquet 文件时设置 compression codec （压缩编解码器）的使用. 可接受的值包括: uncompressed, snappy, gzip, lzo .
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.filterPushdown</code></td>
  <td>true</td>
  <td>设置为 true 时启用 Parquet filter push-down optimization .</td>
</tr>
<tr>
  <td><code>spark.sql.hive.convertMetastoreParquet</code></td>
  <td>true</td>
  <td>
    当设置为 false 时, Spark SQL 将使用 Hive SerDe 作为 parquet tables , 而不是内置的支持.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.mergeSchema</code></td>
  <td>false</td>
  <td>
    <p>
      当为 true 时, Parquet data source （Parquet 数据源） merges （合并）从所有 data files （数据文件）收集的 schemas , 否则如果没有可用的 summary file , 则从 summary file 或 random data file 中挑选 schema .
    </p>
  </td>
</tr>
<tr>
  <td><code>spark.sql.optimizer.metadataOnly</code></td>
  <td>true</td>
  <td>
    <p>
      如果为 true , 则启用使用表的 metadata 的 metadata-only query optimization 来生成 partition columns （分区列）而不是 table scans （表扫描）. 当 scanned （扫描）的所有 columns （列）都是 partition columns （分区列）并且 query （查询）具有满足 distinct semantics （不同语义）的 aggregate operator （聚合运算符）时, 它将适用.
    </p>
  </td>
</tr>
</table>

## JSON Datasets （JSON 数据集）
<div class="codetabs">

<div data-lang="scala"  markdown="1">
Spark SQL 可以 automatically infer （自动推断）JSON dataset 的 schema, 并将其作为 `Dataset[Row]` 加载.
这个 conversion （转换）可以在 `Dataset[String]` 上使用 `SparkSession.read.json()` 来完成, 或 JSON 文件.

请注意, 以  _a json file_ 提供的文件不是典型的 JSON 文件. 每行必须包含一个 separate （单独的）,  self-contained valid （独立的有效的）JSON 对象. 有关更多信息, 请参阅 [JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/) .

对于 regular multi-line JSON file （常规的多行 JSON 文件）, 将 `multiLine` 选项设置为 `true` .

{% include_example json_dataset scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
Spark SQL 可以 automatically infer （自动推断）JSON dataset 的 schema, 并将其作为 `Dataset<Row>` 加载.
这个 conversion （转换）可以在 `Dataset<String>` 上使用 `SparkSession.read.json()` 来完成, 或 JSON 文件.

请注意, 以  _a json file_ 提供的文件不是典型的 JSON 文件. 每行必须包含一个 separate （单独的）,  self-contained valid （独立的有效的）JSON 对象. 有关更多信息, 请参阅 [JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/)

对于 regular multi-line JSON file （常规的多行 JSON 文件）, 将 `multiLine` 选项设置为 `true` .

{% include_example json_dataset java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
Spark SQL 可以 automatically infer （自动推断）JSON  dataset 的 schema , 并将其作为 DataFrame 加载.
可以使用 JSON 文件中的 `SparkSession.read.json` 进行此 conversion （转换）.

请注意, 以  _a json file_ 提供的文件不是典型的 JSON 文件. 每行必须包含一个 separate （单独的）,  self-contained valid （独立的有效的）JSON 对象. 有关更多信息, 请参阅 [JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/)

对于 regular multi-line JSON file （常规的多行 JSON 文件）, 将 `multiLine` 选项设置为 `true` .

{% include_example json_dataset python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">
Spark SQL 可以 automatically infer （自动推断）JSON dataset 的 schema , 并将其作为 DataFrame 加载. 使用 `read.json()` 函数, 它从 JSON 文件的目录中加载数据, 其中每一行文件都是一个 JSON 对象.

请注意, 以  _a json file_ 提供的文件不是典型的 JSON 文件. 每行必须包含一个 separate （单独的）,  self-contained valid （独立的有效的）JSON 对象. 有关更多信息, 请参阅 [JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/).

对于 regular multi-line JSON file （常规的多行 JSON 文件）, 将 `multiLine` 选项设置为 `true` .

{% include_example json_dataset r/RSparkSQLExample.R %}

</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}

CREATE TEMPORARY VIEW jsonTable
USING org.apache.spark.sql.json
OPTIONS (
  path "examples/src/main/resources/people.json"
)

SELECT * FROM jsonTable

{% endhighlight %}

</div>

</div>

## Hive 表

Spark SQL 还支持读取和写入存储在 [Apache Hive](http://hive.apache.org/) 中的数据。 
但是，由于 Hive 具有大量依赖关系，因此这些依赖关系不包含在默认 Spark 分发中。 
如果在类路径中找到 Hive 依赖项，Spark 将自动加载它们。 
请注意，这些 Hive 依赖关系也必须存在于所有工作节点上，因为它们将需要访问 Hive 序列化和反序列化库 (SerDes)，以访问存储在 Hive 中的数据。

通过将 `hive-site.xml`, `core-site.xml`（用于安全配置）和 `hdfs-site.xml` （用于 HDFS 配置）文件放在 `conf/` 中来完成配置。

当使用 Hive 时，必须用 Hive 支持实例化 `SparkSession`，包括连接到持续的 Hive 转移，支持 Hive serdes 和 Hive 用户定义的功能。 没有现有 Hive 部署的用户仍然可以启用 Hive 支持。 当 `hive-site.xml` 未配置时，上下文会自动在当前目录中创建 `metastore_db`，并创建由 `spark.sql.warehouse.dir` 配置的目录，该目录默认为Spark应用程序当前目录中的 `spark-warehouse` 目录 开始了 请注意，自从2.0.0以来，`hive-site.xml` 中的 `hive.metastore.warehouse.dir` 属性已被弃用。 而是使用 `spark.sql.warehouse.dir` 来指定仓库中数据库的默认位置。 您可能需要向启动 Spark 应用程序的用户授予写权限。å

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% include_example spark_hive scala/org/apache/spark/examples/sql/hive/SparkHiveExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example spark_hive java/org/apache/spark/examples/sql/hive/JavaSparkHiveExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example spark_hive python/sql/hive.py %}
</div>

<div data-lang="r"  markdown="1">

当使用 Hive 时，必须使用 Hive 支持实例化 `SparkSession`。这个增加了在 MetaStore 中查找表并使用 HiveQL 编写查询的支持。

{% include_example spark_hive r/RSparkSQLExample.R %}

</div>
</div>

### 指定 Hive 表的存储格式

创建 Hive 表时，需要定义如何 从/向 文件系统 read/write 数据，即 "输入格式" 和 "输出格式"。 
您还需要定义该表如何将数据反序列化为行，或将行序列化为数据，即 "serde"。 
以下选项可用于指定存储格式 ("serde", "input format", "output format")，例如，`CREATE TABLE src(id int) USING hive OPTIONS(fileFormat 'parquet')`。 
默认情况下，我们将以纯文本形式读取表格文件。 
请注意，Hive 存储处理程序在创建表时不受支持，您可以使用 Hive 端的存储处理程序创建一个表，并使用 Spark SQL 来读取它。

<table class="table">
  <tr><th>Property Name</th><th>Meaning</th></tr>
  <tr>
    <td><code>fileFormat</code></td>
    <td>
      fileFormat是一种存储格式规范的包，包括 "serde"，"input format" 和 "output format"。
      目前我们支持6个文件格式：'sequencefile'，'rcfile'，'orc'，'parquet'，'textfile'和'avro'。
    </td>
  </tr>

  <tr>
    <td><code>inputFormat, outputFormat</code></td>
    <td>
      这两个选项将相应的 "InputFormat" 和 "OutputFormat" 类的名称指定为字符串文字，例如: `org.apache.hadoop.hive.ql.io.orc.OrcInputFormat`。 这两个选项必须成对出现，如果您已经指定了 "fileFormat" 选项，则无法指定它们。
    </td>
  </tr>

  <tr>
    <td><code>serde</code></td>
    <td>
      此选项指定 serde 类的名称。 当指定 `fileFormat` 选项时，如果给定的 `fileFormat` 已经包含 serde 的信息，那么不要指定这个选项。 
      目前的 "sequencefile", "textfile" 和 "rcfile" 不包含 serde 信息，你可以使用这3个文件格式的这个选项。
    </td>
  </tr>

  <tr>
    <td><code>fieldDelim, escapeDelim, collectionDelim, mapkeyDelim, lineDelim</code></td>
    <td>
      这些选项只能与 "textfile" 文件格式一起使用。它们定义如何将分隔的文件读入行。
    </td>
  </tr>
</table>

使用 `OPTIONS` 定义的所有其他属性将被视为 Hive serde 属性。

### 与不同版本的 Hive Metastore 进行交互

Spark SQL 的 Hive 支持的最重要的部分之一是与 Hive metastore 进行交互，这使得 Spark SQL 能够访问 Hive 表的元数据。
从 Spark 1.4.0 开始，使用 Spark SQL 的单一二进制构建可以使用下面所述的配置来查询不同版本的 Hive 转移。 
请注意，独立于用于与转移点通信的 Hive 版本，内部 Spark SQL 将针对 Hive 1.2.1 进行编译，并使用这些类进行内部执行（serdes，UDF，UDAF等）。

以下选项可用于配置用于检索元数据的 Hive 版本：

<table class="table">
  <tr><th>属性名称</th><th>默认值</th><th>含义</th></tr>
  <tr>
    <td><code>spark.sql.hive.metastore.version</code></td>
    <td><code>1.2.1</code></td>
    <td>
      Hive metastore 版本。 可用选项为 <code>0.12.0</code> 至 <code>1.2.1</code>。
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.jars</code></td>
    <td><code>builtin</code></td>
    <td>
      当启用 <code>-Phive</code> 时，使用 Hive 1.2.1，它与 Spark 程序集捆绑在一起。选择此选项时，spark.sql.hive.metastore.version 必须为 <code>1.2.1</code> 或未定义。
      行家
      使用从Maven存储库下载的指定版本的Hive jar。 通常不建议在生产部署中使用此配置。 *****
      应用于实例化 HiveMetastoreClient 的 jar 的位置。该属性可以是三个选项之一:
      <ol>
        <li><code>builtin</code></li>
        当启用 <code>-Phive</code> 时，使用 Hive 1.2.1，它与 Spark 程序集捆绑在一起。选择此选项时，<code>spark.sql.hive.metastore.version</code> 必须为 <code>1.2.1</code> 或未定义。
        <li><code>maven</code></li>
        使用从 Maven 存储库下载的指定版本的 Hive jar。通常不建议在生产部署中使用此配置。
        <li>JVM 的标准格式的 classpath。 该类路径必须包含所有 Hive 及其依赖项，包括正确版本的 Hadoop。这些罐只需要存在于 driver 程序中，但如果您正在运行在 yarn 集群模式，那么您必须确保它们与应用程序一起打包。</li>
      </ol>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.sharedPrefixes</code></td>
    <td><code>com.mysql.jdbc,<br/>org.postgresql,<br/>com.microsoft.sqlserver,<br/>oracle.jdbc</code></td>
    <td>
      <p>
        使用逗号分隔的类前缀列表，应使用在 Spark SQL 和特定版本的 Hive 之间共享的类加载器来加载。
        一个共享类的示例就是用来访问 Hive metastore 的 JDBC driver。
        其它需要共享的类，是需要与已经共享的类进行交互的。
        例如，log4j 使用的自定义 appender。
      </p>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.barrierPrefixes</code></td>
    <td><code>(empty)</code></td>
    <td>
      <p>
        一个逗号分隔的类前缀列表，应该明确地为 Spark SQL 正在通信的 Hive 的每个版本重新加载。
        例如，在通常将被共享的前缀中声明的 Hive UDF （即： <code>org.apache.spark.*</code>）。
      </p>
    </td>
  </tr>
</table>


## JDBC 连接其它数据库

Spark SQL 还包括可以使用 JDBC 从其他数据库读取数据的数据源。此功能应优于使用 [JdbcRDD](api/scala/index.html#org.apache.spark.rdd.JdbcRDD)。 
这是因为结果作为 DataFrame 返回，并且可以轻松地在 Spark SQL 中处理或与其他数据源连接。 
JDBC 数据源也更容易从 Java 或 Python 使用，因为它不需要用户提供 ClassTag。（请注意，这不同于 Spark SQL JDBC 服务器，允许其他应用程序使用 Spark SQL 运行查询）。

要开始使用，您需要在 Spark 类路径中包含特定数据库的 JDBC driver 程序。
例如，要从 Spark Shell 连接到 postgres，您将运行以下命令:

{% highlight bash %}
bin/spark-shell --driver-class-path postgresql-9.4.1207.jar --jars postgresql-9.4.1207.jar
{% endhighlight %}

可以使用 Data Sources API 将来自远程数据库的表作为 DataFrame 或 Spark SQL 临时视图进行加载。
用户可以在数据源选项中指定 JDBC 连接属性。<code>用户</code> 和 <code>密码</code>通常作为登录数据源的连接属性提供。
除了连接属性外，Spark 还支持以下不区分大小写的选项:

<table class="table">
  <tr><th>属性名称</th><th>含义</th></tr>
  <tr>
    <td><code>url</code></td>
    <td>
      要连接的JDBC URL。 源特定的连接属性可以在URL中指定。 例如jdbc：<code>jdbc:postgresql://localhost/test?user=fred&password=secret</code>
    </td>
  </tr>

  <tr>
    <td><code>dbtable</code></td>
    <td>
      应该读取的 JDBC 表。请注意，可以使用在SQL查询的 <code>FROM</code> 子句中有效的任何内容。 
      例如，您可以使用括号中的子查询代替完整表。
    </td>
  </tr>

  <tr>
    <td><code>driver</code></td>
    <td>
      用于连接到此 URL 的 JDBC driver 程序的类名。
    </td>
  </tr>

  <tr>
    <td><code>partitionColumn, lowerBound, upperBound</code></td>
    <td>
      如果指定了这些选项，则必须指定这些选项。 另外，必须指定 <code>numPartitions</code>. 
      他们描述如何从多个工作人员并行阅读时分割表。<code>partitionColumn</code> 必须是有问题的表中的数字列。 
      请注意，<code>lowerBound</code> 和  <code>upperBound</code> 仅用于决定分区的大小，而不是用于过滤表中的行。 
      因此，表中的所有行将被分区并返回。此选项仅适用于阅读。
    </td>
  </tr>

  <tr>
    <td><code>numPartitions</code></td>
    <td>
      在表读写中可以用于并行度的最大分区数。这也确定并发JDBC连接的最大数量。 
      如果要写入的分区数超过此限制，则在写入之前通过调用 <code>coalesce(numPartitions)</code> 将其减少到此限制。
    </td>
  </tr>

  <tr>
    <td><code>fetchsize</code></td>
    <td>
      JDBC 提取大小，用于确定每次往返行程的行数。 
      这可以帮助JDBC driver 程序的性能降低，这些 driver 程序默认具有较低的提取大小（例如: 具有10行的 Oracle）。
      此选项仅适用于阅读。
    </td>
  </tr>

  <tr>
    <td><code>batchsize</code></td>
    <td>
      JDBC批量大小，用于确定每往返行数要插入的行数。 
      这可以帮助JDBC driver 程序的性能。
      此选项仅适用于写作。默认为 <code>1000</code>.
    </td>
  </tr>

  <tr>
    <td><code>isolationLevel</code></td>
    <td>
      事务隔离级别，适用于当前连接。
      它可以是 <code>NONE</code>, <code>READ_COMMITTED</code>, <code>READ_UNCOMMITTED</code>, <code>REPEATABLE_READ</code>, 或 <code>SERIALIZABLE</code> 之一，对应于 JDBC 连接对象定义的标准事务隔离级别，默认为 <code>READ_UNCOMMITTED</code>。 
      此选项仅适用于写作。请参考 <code>java.sql.Connection</code> 中的文档。
    </td>
  </tr>

  <tr>
    <td><code>truncate</code></td>
    <td>
      这是一个与 JDBC 相关的选项。
      启用 <code>SaveMode.Overwrite</code> 时，此选项会导致 Spark 截断现有表，而不是删除并重新创建。 
      这可以更有效，并且防止表元数据（例如，索引）被移除。 
      但是，在某些情况下，例如当新数据具有不同的模式时，它将无法工作。 它默认为 <code>false</code>。 
      此选项仅适用于写作。
   </td>
  </tr>

  <tr>
    <td><code>createTableOptions</code></td>
    <td>
      这是一个与JDBC相关的选项。
      如果指定，此选项允许在创建表时设置特定于数据库的表和分区选项（例如：<code>CREATE TABLE t (name string) ENGINE=InnoDB.</code> ）。此选项仅适用于写作。
   </td>
  </tr>

  <tr>
    <td><code>createTableColumnTypes</code></td>
    <td>
      使用数据库列数据类型而不是默认值，创建表时。
      数据类型信息应以与 CREATE TABLE 列语法相同的格式指定（例如：<code>"name CHAR(64), comments VARCHAR(1024)"</code>）。
      指定的类型应该是有效的 spark sql 数据类型。此选项仅适用于写作。
    </td>
  </tr>  
</table>

<div class="codetabs">

<div data-lang="scala"  markdown="1">
{% include_example jdbc_dataset scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
{% include_example jdbc_dataset java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
{% include_example jdbc_dataset python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">
{% include_example jdbc_dataset r/RSparkSQLExample.R %}
</div>

<div data-lang="sql"  markdown="1">

{% highlight sql %}

CREATE TEMPORARY VIEW jdbcTable
USING org.apache.spark.sql.jdbc
OPTIONS (
  url "jdbc:postgresql:dbserver",
  dbtable "schema.tablename",
  user 'username',
  password 'password'
)

INSERT INTO TABLE jdbcTable
SELECT * FROM resultTable
{% endhighlight %}

</div>
</div>

## 故障排除

* JDBC driver 程序类必须对客户端会话和所有执行程序上的原始类加载器可见。 这是因为 Java 的 DriverManager 类执行安全检查，导致它忽略原始类加载器不可见的所有 driver 程序，当打开连接时。一个方便的方法是修改所有工作节点上的compute_classpath.sh 以包含您的 driver 程序 JAR。
* 一些数据库，例如 H2，将所有名称转换为大写。 您需要使用大写字母来引用 Spark SQL 中的这些名称。


# 性能调优

对于某些工作负载，可以通过缓存内存中的数据或打开一些实验选项来提高性能。

## 在内存中缓存数据

Spark SQL 可以通过调用 `spark.catalog.cacheTable("tableName")` 或 `dataFrame.cache()` 来使用内存中的列格式来缓存表。 
然后，Spark SQL 将只扫描所需的列，并将自动调整压缩以最小化内存使用量和 GC 压力。
您可以调用 `spark.catalog.uncacheTable("tableName")` 从内存中删除该表。

内存缓存的配置可以使用 `SparkSession` 上的 `setConf` 方法或使用 SQL 运行 `SET key=value` 命令来完成。

<table class="table">
<tr><th>属性名称</th><th>默认</th><th>含义</th></tr>
<tr>
  <td><code>spark.sql.inMemoryColumnarStorage.compressed</code></td>
  <td>true</td>
  <td>
    当设置为 true 时，Spark SQL 将根据数据的统计信息为每个列自动选择一个压缩编解码器。
  </td>
</tr>
<tr>
  <td><code>spark.sql.inMemoryColumnarStorage.batchSize</code></td>
  <td>10000</td>
  <td>
    控制批量的柱状缓存的大小。更大的批量大小可以提高内存利用率和压缩率，但是在缓存数据时会冒出 OOM 风险。
  </td>
</tr>

</table>

## 其他配置选项

以下选项也可用于调整查询执行的性能。这些选项可能会在将来的版本中被废弃，因为更多的优化是自动执行的。

<table class="table">
  <tr><th>属性名称</th><th>默认值</th><th>含义</th></tr>
  <tr>
    <td><code>spark.sql.files.maxPartitionBytes</code></td>
    <td>134217728 (128 MB)</td>
    <td>
      在读取文件时，将单个分区打包的最大字节数。
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.files.openCostInBytes</code></td>
    <td>4194304 (4 MB)</td>
    <td>
      按照字节数来衡量的打开文件的估计费用可以在同一时间进行扫描。 
      将多个文件放入分区时使用。最好过度估计，那么具有小文件的分区将比具有较大文件的分区（首先计划的）更快。
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.broadcastTimeout</code></td>
    <td>300</td>
    <td>
    <p>
      广播连接中的广播等待时间超时（秒）
    </p>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.autoBroadcastJoinThreshold</code></td>
    <td>10485760 (10 MB)</td>
    <td>
      配置执行连接时将广播给所有工作节点的表的最大大小（以字节为单位）。 
      通过将此值设置为-1可以禁用广播。 请注意，目前的统计信息仅支持 Hive Metastore 表，其中已运行命令 <code>ANALYZE TABLE &lt;tableName&gt; COMPUTE STATISTICS noscan</code>。
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.shuffle.partitions</code></td>
    <td>200</td>
    <td>
      Configures the number of partitions to use when shuffling data for joins or aggregations.
    </td>
  </tr>
</table>

# 分布式 SQL 引擎

Spark SQL 也可以充当使用其 JDBC/ODBC 或命令行界面的分布式查询引擎。
在这种模式下，最终用户或应用程序可以直接与 Spark SQL 交互运行 SQL 查询，而不需要编写任何代码。

## 运行 Thrift JDBC/ODBC 服务器

这里实现的 Thrift JDBC/ODBC 服务器对应于 Hive 1.2 中的 [`HiveServer2`](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2)。
您可以使用 Spark 或 Hive 1.2.1 附带的直线脚本测试 JDBC 服务器。

要启动 JDBC/ODBC 服务器，请在 Spark 目录中运行以下命令:

    ./sbin/start-thriftserver.sh

此脚本接受所有 `bin/spark-submit` 命令行选项，以及 `--hiveconf` 选项来指定 Hive 属性。 
您可以运行 `./sbin/start-thriftserver.sh --help` 查看所有可用选项的完整列表。 
默认情况下，服务器监听 localhost:10000. 您可以通过环境变量覆盖此行为，即:

{% highlight bash %}
export HIVE_SERVER2_THRIFT_PORT=<listening-port>
export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
./sbin/start-thriftserver.sh \
  --master <master-uri> \
  ...
{% endhighlight %}

or system properties:

{% highlight bash %}
./sbin/start-thriftserver.sh \
  --hiveconf hive.server2.thrift.port=<listening-port> \
  --hiveconf hive.server2.thrift.bind.host=<listening-host> \
  --master <master-uri>
  ...
{% endhighlight %}

现在，您可以使用 beeline 来测试 Thrift JDBC/ODBC 服务器:

    ./bin/beeline

使用 beeline 方式连接到  JDBC/ODBC 服务器:

    beeline> !connect jdbc:hive2://localhost:10000

Beeline 将要求您输入用户名和密码。 
在非安全模式下，只需输入机器上的用户名和空白密码即可。 
对于安全模式，请按照 [beeline 文档](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients) 中的说明进行操作。

配置Hive是通过将 `hive-site.xml`, `core-site.xml` 和 `hdfs-site.xml` 文件放在 `conf/` 中完成的。

您也可以使用 Hive 附带的 beeline 脚本。

Thrift JDBC 服务器还支持通过 HTTP 传输发送 thrift RPC 消息。 
使用以下设置启用 HTTP 模式作为系统属性或在 `conf/` 中的 `hive-site.xml` 文件中启用:

    hive.server2.transport.mode - Set this to value: http
    hive.server2.thrift.http.port - HTTP port number to listen on; default is 10001
    hive.server2.http.endpoint - HTTP endpoint; default is cliservice

要测试，请使用 beeline 以 http 模式连接到 JDBC/ODBC 服务器:

    beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint>


## 运行 Spark SQL CLI

Spark SQL CLI 是在本地模式下运行 Hive 转移服务并执行从命令行输入的查询的方便工具。 
请注意，Spark SQL CLI 不能与 Thrift JDBC 服务器通信。

要启动 Spark SQL CLI，请在 Spark 目录中运行以下命令:

    ./bin/spark-sql

配置 Hive 是通过将 `hive-site.xml`, `core-site.xml` 和 `hdfs-site.xml` 文件放在 `conf/` 中完成的。 
您可以运行 `./bin/spark-sql --help` 获取所有可用选项的完整列表。

# 迁移指南

## 从 Spark SQL 2.1 升级到 2.2

  - Spark 2.1.1 介绍了一个新的配置 key: `spark.sql.hive.caseSensitiveInferenceMode`. 它的默认设置是 `NEVER_INFER`, 其行为与 2.1.0 保持一致. 但是，Spark 2.2.0 将此设置的默认值更改为 "INFER_AND_SAVE"，以恢复与底层文件 schema（模式）具有大小写混合的列名称的 Hive metastore 表的兼容性。使用 `INFER_AND_SAVE` 配置的 value, 在第一次访问 Spark 将对其尚未保存推测 schema（模式）的任何 Hive metastore 表执行 schema inference（模式推断）. 请注意，对于具有数千个 partitions（分区）的表，模式推断可能是非常耗时的操作。如果不兼容大小写混合的列名，您可以安全地将`spark.sql.hive.caseSensitiveInferenceMode` 设置为 `NEVER_INFER`，以避免模式推断的初始开销。请注意，使用新的默认`INFER_AND_SAVE` 设置，模式推理的结果被保存为 metastore key 以供将来使用。因此，初始模式推断仅发生在表的第一次访问。

## 从 Spark SQL 2.0 升级到 2.1

 - Datasource tables（数据源表）现在存储了 Hive metastore 中的 partition metadata（分区元数据）. 这意味着诸如 `ALTER TABLE PARTITION ... SET LOCATION` 这样的 Hive DDLs 现在使用 Datasource API 可用于创建 tables（表）.
    - 遗留的数据源表可以通过 `MSCK REPAIR TABLE` 命令迁移到这种格式。建议迁移遗留表利用 Hive DDL 的支持和提供的计划性能。
    - 要确定表是否已迁移，当在表上发出 `DESCRIBE FORMATTED` 命令时请查找 `PartitionProvider: Catalog` 属性.
 - Datasource tables（数据源表）的 `INSERT OVERWRITE TABLE ... PARTITION ...` 行为的更改。
    - 在以前的 Spark 版本中，`INSERT OVERWRITE` 覆盖了整个 Datasource table，即使给出一个指定的 partition. 现在只有匹配规范的 partition 被覆盖。
    - 请注意，这仍然与 Hive 表的行为不同，Hive 表仅覆盖与新插入数据重叠的分区。

## 从 Spark SQL 1.6 升级到 2.0

 - `SparkSession` 现在是 Spark 新的切入点, 它替代了老的 `SQLContext` 和 `HiveContext`。注意 : 为了向下兼容，老的       SQLContext 和 HiveContext 仍然保留。可以从 `SparkSession` 获取一个新的 `catalog` 接口 — 现有的访问数据库和表的 API，如 `listTables`，`createExternalTable`，`dropTempView`，`cacheTable` 都被移到该接口。

 - Dataset API 和 DataFrame API 进行了统一。在 Scala 中，`DataFrame` 变成了 `Dataset[Row]` 类型的一个别名，而 Java    API 使用者必须将 `DataFrame` 替换成 `Dataset<Row>`。Dataset 类既提供了强类型转换操作（如 `map`，`filter` 以及       `groupByKey`）也提供了非强类型转换操作（如 `select` 和 `groupBy`）。由于编译期的类型安全不是 Python 和 R 语言的一个特性，Dataset 的概念并不适用于这些语言的 API。相反，`DataFrame` 仍然是最基本的编程抽象, 就类似于这些语言中单节点 data frame 的概念。
   

 - Dataset 和 DataFrame API 中 unionAll 已经过时并且由 `union` 替代。
 - Dataset 和 DataFrame API 中 explode 已经过时，作为选择，可以结合 select 或 flatMap 使用 `functions.explode()` 。
 - Dataset 和 DataFrame API 中 `registerTempTable` 已经过时并且由 `createOrReplaceTempView` 替代。

 - 对 Hive tables `CREATE TABLE ... LOCATION` 行为的更改.
    - 从 Spark 2.0 开始，`CREATE TABLE ... LOCATION` 与 `CREATE EXTERNAL TABLE ... LOCATION` 是相同的，以防止意外丢弃用户提供的 locations（位置）中的现有数据。这意味着，在用户指定位置的 Spark SQL 中创建的 Hive 表始终是 Hive 外部表。删除外部表将不会删除数据。 用户不能指定 Hive managed tables（管理表）的位置.
      请注意，这与Hive行为不同。
    - 因此，这些表上的 "DROP TABLE" 语句不会删除数据。

## 从 Spark SQL 1.5 升级到 1.6

 - 从 Spark 1.6 开始，默认情况下服务器在多 session（会话）模式下运行。这意味着每个 JDBC/ODBC 连接拥有一份自己的 SQL 配置和临时函数注册。缓存表仍在并共享。如果您希望以旧的单会话模式运行 Thrift server，请设置选项 `spark.sql.hive.thriftServer.singleSession` 为` true`。您既可以将此选项添加到 `spark-defaults.conf`，或者通过 `--conf` 将它传递给 `start-thriftserver.sh`。

   {% highlight bash %}
   ./sbin/start-thriftserver.sh \
     --conf spark.sql.hive.thriftServer.singleSession=true \
     ...
   {% endhighlight %}
 
 - 从 1.6.1 开始，在 sparkR 中 withColumn 方法支持添加一个新列或更换 DataFrame 同名的现有列。

 - 从 Spark 1.6 开始，LongType 强制转换为 TimestampType 期望是秒，而不是微秒。这种更改是为了匹配 Hive 1.2 的行为，以便从 numeric（数值）类型进行更一致的类型转换到 TimestampType。更多详情请参阅 [SPARK-11724](https://issues.apache.org/jira/browse/SPARK-11724) 。

## 从 Spark SQL 1.4 升级到 1.5

 - 使用手动管理的内存优化执行，现在是默认启用的，以及代码生成表达式求值。这些功能既可以通过设置 `spark.sql.tungsten.enabled` 为 `false` 来禁止使用。
 - Parquet 的模式合并默认情况下不再启用。它可以通过设置 `spark.sql.parquet.mergeSchema` 到 `true` 以重新启用。
 - 字符串在 Python 列的 columns（列）现在支持使用点（`.`）来限定列或访问嵌套值。例如 `df['table.column.nestedField']`。但是，这意味着如果你的列名中包含任何圆点，你现在必须避免使用反引号（如 `table.`column.with.dots`.nested`）。
 - 在内存中的列存储分区修剪默认是开启的。它可以通过设置 `spark.sql.inMemoryColumnarStorage.partitionPruning` 为 `false` 来禁用。
 - 无限精度的小数列不再支持，而不是 Spark SQL 最大精度为 38 。当从 `BigDecimal` 对象推断模式时，现在使用（38，18）。在 DDL 没有指定精度时，则默认保留 `Decimal(10, 0)`。
 - 时间戳现在存储在 1 微秒的精度，而不是 1 纳秒的。
 - 在 sql 语句中，floating point（浮点数）现在解析为 decimal。HiveQL 解析保持不变。
 - SQL / DataFrame 函数的规范名称现在是小写（例如 sum  vs SUM）。
 - JSON 数据源不会自动加载由其他应用程序（未通过 Spark SQL 插入到数据集的文件）创建的新文件。对于 JSON 持久表（即表的元数据存储在 Hive Metastore），用户可以使用 `REFRESH TABLE` SQL 命令或 `HiveContext` 的 `refreshTable` 方法，把那些新文件列入到表中。对于代表一个 JSON dataset 的 DataFrame，用户需要重新创建 DataFrame，同时 DataFrame 中将包括新的文件。
 - PySpark 中 DataFrame 的 withColumn 方法支持添加新的列或替换现有的同名列。

## 从 Spark SQL 1.3 升级到 1.4

#### DataFrame data reader/writer interface

基于用户反馈，我们创建了一个新的更流畅的 API，用于读取 (`SQLContext.read`) 中的数据并写入数据 (`DataFrame.write`), 并且旧的 API 将过时（例如，`SQLContext.parquetFile`, `SQLContext.jsonFile`）.

针对 `SQLContext.read` (
  <a href="api/scala/index.html#org.apache.spark.sql.SQLContext@read:DataFrameReader">Scala</a>,
  <a href="api/java/org/apache/spark/sql/SQLContext.html#read()">Java</a>,
  <a href="api/python/pyspark.sql.html#pyspark.sql.SQLContext.read">Python</a>
) 和 `DataFrame.write` (
  <a href="api/scala/index.html#org.apache.spark.sql.DataFrame@write:DataFrameWriter">Scala</a>,
  <a href="api/java/org/apache/spark/sql/DataFrame.html#write()">Java</a>,
  <a href="api/python/pyspark.sql.html#pyspark.sql.DataFrame.write">Python</a>
) 的更多细节，请看 API 文档.

#### DataFrame.groupBy 保留 grouping columns（分组的列）

根据用户的反馈， 我们更改了 `DataFrame.groupBy().agg()` 的默认行为以保留 `DataFrame` 结果中的 grouping columns（分组列）. 为了在 1.3 中保持该行为，请设置 `spark.sql.retainGroupColumns` 为 `false`.

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% highlight scala %}

// In 1.3.x, in order for the grouping column "department" to show up,
// it must be included explicitly as part of the agg function call.
df.groupBy("department").agg($"department", max("age"), sum("expense"))

// In 1.4+, grouping column "department" is included automatically.
df.groupBy("department").agg(max("age"), sum("expense"))

// Revert to 1.3 behavior (not retaining grouping column) by:
sqlContext.setConf("spark.sql.retainGroupColumns", "false")

{% endhighlight %}
</div>

<div data-lang="java"  markdown="1">
{% highlight java %}

// In 1.3.x, in order for the grouping column "department" to show up,
// it must be included explicitly as part of the agg function call.
df.groupBy("department").agg(col("department"), max("age"), sum("expense"));

// In 1.4+, grouping column "department" is included automatically.
df.groupBy("department").agg(max("age"), sum("expense"));

// Revert to 1.3 behavior (not retaining grouping column) by:
sqlContext.setConf("spark.sql.retainGroupColumns", "false");

{% endhighlight %}
</div>

<div data-lang="python"  markdown="1">
{% highlight python %}

import pyspark.sql.functions as func

# In 1.3.x, in order for the grouping column "department" to show up,
# it must be included explicitly as part of the agg function call.
df.groupBy("department").agg(df["department"], func.max("age"), func.sum("expense"))

# In 1.4+, grouping column "department" is included automatically.
df.groupBy("department").agg(func.max("age"), func.sum("expense"))

# Revert to 1.3.x behavior (not retaining grouping column) by:
sqlContext.setConf("spark.sql.retainGroupColumns", "false")

{% endhighlight %}
</div>

</div>


#### DataFrame.withColumn 上的行为更改

之前 1.4 版本中，DataFrame.withColumn() 只支持添加列。该列将始终在 DateFrame 结果中被加入作为新的列，即使现有的列可能存在相同的名称。从 1.4 版本开始，DataFrame.withColumn() 支持添加与所有现有列的名称不同的列或替换现有的同名列。

请注意，这一变化仅适用于 Scala API，并不适用于 PySpark 和 SparkR。

## 从 Spark SQL 1.0-1.2 升级到 1.3

在 Spark 1.3 中，我们从 Spark SQL 中删除了 "Alpha" 的标签，作为一部分已经清理过的可用的 API 。从 Spark 1.3 版本以上，Spark SQL 将提供在 1.X 系列的其他版本的二进制兼容性。这种兼容性保证不包括被明确标记为不稳定的（即 DeveloperApi 类或 Experimental） API。

#### 重命名 DataFrame 的 SchemaRDD

升级到 Spark SQL 1.3 版本时，用户会发现最大的变化是，`SchemaRDD` 已更名为 `DataFrame`。这主要是因为 DataFrames 不再从 RDD 直接继承，而是由 RDDS 自己来实现这些功能。DataFrames 仍然可以通过调用 `.rdd` 方法转换为 RDDS 。

在 Scala 中，有一个从 `SchemaRDD` 到 `DataFrame` 类型别名，可以为一些情况提供源代码兼容性。它仍然建议用户更新他们的代码以使用 `DataFrame` 来代替。Java 和 Python 用户需要更新他们的代码。

#### Java 和 Scala APIs 的统一

此前 Spark 1.3 有单独的Java兼容类（`JavaSQLContext` 和 `JavaSchemaRDD`），借鉴于 Scala API。在 Spark 1.3 中，Java API 和 Scala API 已经统一。两种语言的用户可以使用 `SQLContext` 和 `DataFrame`。一般来说论文类尝试使用两种语言的共有类型（如 `Array` 替代了一些特定集合）。在某些情况下不通用的类型情况下，（例如，passing in closures 或 Maps）使用函数重载代替。

此外，该 Java 的特定类型的 API 已被删除。Scala 和 Java 的用户可以使用存在于 `org.apache.spark.sql.types` 类来描述编程模式。


#### 隔离隐式转换和删除 dsl 包（仅Scala）

许多 Spark 1.3 版本以前的代码示例都以 `import sqlContext._` 开始，这提供了从 sqlContext 范围的所有功能。在 Spark 1.3 中，我们移除了从 `RDD`s 到 `DateFrame` 再到 `SQLContext` 内部对象的隐式转换。用户现在应该写成 `import sqlContext.implicits._`.

此外，隐式转换现在只能使用方法 `toDF` 来增加由 `Product`（即 case classes or tuples）构成的 `RDD`，而不是自动应用。

当使用 DSL 内部的函数时（现在使用 `DataFrame` API 来替换）, 用户习惯导入 `org.apache.spark.sql.catalyst.dsl`. 相反，应该使用公共的 dataframe 函数 API: `import org.apache.spark.sql.functions._`.

#### 针对 DataType 删除在 org.apache.spark.sql 包中的一些类型别名（仅限于 Scala）

Spark 1.3 移除存在于基本 SQL 包的 `DataType` 类型别名。开发人员应改为导入类 `org.apache.spark.sql.types`。

#### UDF 注册迁移到 `sqlContext.udf` 中 (Java & Scala)

用于注册 UDF 的函数，不管是 DataFrame DSL 还是 SQL 中用到的，都被迁移到 `SQLContext` 中的 udf 对象中。

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% highlight scala %}

sqlContext.udf.register("strLen", (s: String) => s.length())

{% endhighlight %}
</div>

<div data-lang="java"  markdown="1">
{% highlight java %}

sqlContext.udf().register("strLen", (String s) -> s.length(), DataTypes.IntegerType);

{% endhighlight %}
</div>

</div>

Python UDF 注册保持不变。

#### Python DataTypes 不再是 Singletons（单例的）

在 Python 中使用 DataTypes 时，你需要先构造它们（如：`StringType()`），而不是引用一个单例对象。

## 与 Apache Hive 的兼容
Spark SQL 在设计时就考虑到了和 Hive metastore，SerDes 以及 UDF 之间的兼容性。目前 Hive SerDes 和 UDF 都是基于 Hive 1.2.1 版本，并且Spark SQL 可以连接到不同版本的Hive metastore（从 0.12.0 到 1.2.1，可以参考 [与不同版本的 Hive Metastore 交互]((#interacting-with-different-versions-of-hive-metastore))）

#### 在现有的 Hive Warehouses 中部署

Spark SQL Thrift JDBC server 采用了开箱即用的设计以兼容已有的 Hive 安装版本。你不需要修改现有的 Hive Metastore , 或者改变数据的位置和表的分区。

### 所支持的 Hive 特性

Spark SQL 支持绝大部分的 Hive 功能，如:

* Hive query（查询）语句, 包括:
  * `SELECT`
  * `GROUP BY`
  * `ORDER BY`
  * `CLUSTER BY`
  * `SORT BY`
* 所有 Hive 操作, 包括:
  * 关系运算符 (`=`, `⇔`, `==`, `<>`, `<`, `>`, `>=`, `<=`, 等等)
  * 算术运算符 (`+`, `-`, `*`, `/`, `%`, 等等)
  * 逻辑运算符 (`AND`, `&&`, `OR`, `||`, 等等)
  * 复杂类型的构造
  * 数学函数 (`sign`, `ln`, `cos`, 等等)
  * String 函数 (`instr`, `length`, `printf`, 等等)
* 用户定义函数 (UDF)
* 用户定义聚合函数 (UDAF)
* 用户定义 serialization formats (SerDes)
* 窗口函数
* Joins
  * `JOIN`
  * `{LEFT|RIGHT|FULL} OUTER JOIN`
  * `LEFT SEMI JOIN`
  * `CROSS JOIN`
* Unions
* Sub-queries（子查询）
  * `SELECT col FROM ( SELECT a + b AS col from t1) t2`
* Sampling
* Explain
* Partitioned tables including dynamic partition insertion
* View
* 所有的 Hive DDL 函数, 包括:
  * `CREATE TABLE`
  * `CREATE TABLE AS SELECT`
  * `ALTER TABLE`
* 大部分的 Hive Data types（数据类型）, 包括:
  * `TINYINT`
  * `SMALLINT`
  * `INT`
  * `BIGINT`
  * `BOOLEAN`
  * `FLOAT`
  * `DOUBLE`
  * `STRING`
  * `BINARY`
  * `TIMESTAMP`
  * `DATE`
  * `ARRAY<>`
  * `MAP<>`
  * `STRUCT<>`

### 为支持的 Hive 函数

以下是目前还不支持的 Hive 函数列表。在 Hive 部署中这些功能大部分都用不到。

**主要的 Hive 功能**

* Tables 使用 buckets 的 Tables: bucket 是 Hive table partition 中的 hash partitioning. Spark SQL 还不支持 buckets.


**Esoteric Hive 功能**

* `UNION` 类型
* Unique join
* Column 统计信息的收集: Spark SQL does not piggyback scans to collect column statistics at
  the moment and only supports populating the sizeInBytes field of the hive metastore.

**Hive Input/Output Formats**

* File format for CLI: For results showing back to the CLI, Spark SQL only supports TextOutputFormat.
* Hadoop archive

**Hive 优化**

有少数 Hive 优化还没有包含在 Spark 中。其中一些（比如 indexes 索引）由于 Spark SQL 的这种内存计算模型而显得不那么重要。另外一些在 Spark SQL 未来的版本中会持续跟踪。

* Block 级别的 bitmap indexes 和虚拟 columns (用于构建 indexes)
* 自动为 join 和 groupBy 计算 reducer 个数 : 目前在 Spark SQL 中, 你需要使用 "`SET spark.sql.shuffle.partitions=[num_tasks];`" 来控制 post-shuffle 的并行度.
* 仅 Meta-data 的 query: 对于只使用 metadata 就能回答的查询，Spark SQL 仍然会启动计算结果的任务.
* Skew data flag: Spark SQL 不遵循 Hive 中 skew 数据的标记.
* `STREAMTABLE` hint in join: Spark SQL 不遵循 `STREAMTABLE` hint.
* 对于查询结果合并多个小文件: 如果输出的结果包括多个小文件,
  Hive 可以可选的合并小文件到一些大文件中去，以避免溢出 HDFS metadata. Spark SQL 还不支持这样.

# 参考

## 数据类型

Spark SQL 和 DataFrames 支持下面的数据类型:

* Numeric types
    - `ByteType`: Represents 1-byte signed integer numbers.
    The range of numbers is from `-128` to `127`.
    - `ShortType`: Represents 2-byte signed integer numbers.
    The range of numbers is from `-32768` to `32767`.
    - `IntegerType`: Represents 4-byte signed integer numbers.
    The range of numbers is from `-2147483648` to `2147483647`.
    - `LongType`: Represents 8-byte signed integer numbers.
    The range of numbers is from `-9223372036854775808` to `9223372036854775807`.
    - `FloatType`: Represents 4-byte single-precision floating point numbers.
    - `DoubleType`: Represents 8-byte double-precision floating point numbers.
    - `DecimalType`: Represents arbitrary-precision signed decimal numbers. Backed internally by `java.math.BigDecimal`. A `BigDecimal` consists of an arbitrary precision integer unscaled value and a 32-bit integer scale.
* String type
    - `StringType`: Represents character string values.
* Binary type
    - `BinaryType`: Represents byte sequence values.
* Boolean type
    - `BooleanType`: Represents boolean values.
* Datetime type
    - `TimestampType`: Represents values comprising values of fields year, month, day,
    hour, minute, and second.
    - `DateType`: Represents values comprising values of fields year, month, day.
* Complex types
    - `ArrayType(elementType, containsNull)`: Represents values comprising a sequence of
    elements with the type of `elementType`. `containsNull` is used to indicate if
    elements in a `ArrayType` value can have `null` values.
    - `MapType(keyType, valueType, valueContainsNull)`:
    Represents values comprising a set of key-value pairs. The data type of keys are
    described by `keyType` and the data type of values are described by `valueType`.
    For a `MapType` value, keys are not allowed to have `null` values. `valueContainsNull`
    is used to indicate if values of a `MapType` value can have `null` values.
    - `StructType(fields)`: Represents values with the structure described by
    a sequence of `StructField`s (`fields`).
        * `StructField(name, dataType, nullable)`: Represents a field in a `StructType`.
        The name of a field is indicated by `name`. The data type of a field is indicated
        by `dataType`. `nullable` is used to indicate if values of this fields can have
        `null` values.

<div class="codetabs">
<div data-lang="scala"  markdown="1">

Spark SQL 的所有数据类型都在包 `org.apache.spark.sql.types` 中.
你可以用下示例示例来访问它们.

{% include_example data_types scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}

<table class="table">
<tr>
  <th style="width:20%">Data type（数据类型）</th>
  <th style="width:40%">Scala 中的 Value 类型</th>
  <th>访问或创建数据类型的 API</th></tr>
<tr>
  <td> <b>ByteType</b> </td>
  <td> Byte </td>
  <td>
  ByteType
  </td>
</tr>
<tr>
  <td> <b>ShortType</b> </td>
  <td> Short </td>
  <td>
  ShortType
  </td>
</tr>
<tr>
  <td> <b>IntegerType</b> </td>
  <td> Int </td>
  <td>
  IntegerType
  </td>
</tr>
<tr>
  <td> <b>LongType</b> </td>
  <td> Long </td>
  <td>
  LongType
  </td>
</tr>
<tr>
  <td> <b>FloatType</b> </td>
  <td> Float </td>
  <td>
  FloatType
  </td>
</tr>
<tr>
  <td> <b>DoubleType</b> </td>
  <td> Double </td>
  <td>
  DoubleType
  </td>
</tr>
<tr>
  <td> <b>DecimalType</b> </td>
  <td> java.math.BigDecimal </td>
  <td>
  DecimalType
  </td>
</tr>
<tr>
  <td> <b>StringType</b> </td>
  <td> String </td>
  <td>
  StringType
  </td>
</tr>
<tr>
  <td> <b>BinaryType</b> </td>
  <td> Array[Byte] </td>
  <td>
  BinaryType
  </td>
</tr>
<tr>
  <td> <b>BooleanType</b> </td>
  <td> Boolean </td>
  <td>
  BooleanType
  </td>
</tr>
<tr>
  <td> <b>TimestampType</b> </td>
  <td> java.sql.Timestamp </td>
  <td>
  TimestampType
  </td>
</tr>
<tr>
  <td> <b>DateType</b> </td>
  <td> java.sql.Date </td>
  <td>
  DateType
  </td>
</tr>
<tr>
  <td> <b>ArrayType</b> </td>
  <td> scala.collection.Seq </td>
  <td>
  ArrayType(<i>elementType</i>, [<i>containsNull</i>])<br />
  <b>Note（注意）:</b> <i>containsNull</i> 的默认值是 <i>true</i>.
  </td>
</tr>
<tr>
  <td> <b>MapType</b> </td>
  <td> scala.collection.Map </td>
  <td>
  MapType(<i>keyType</i>, <i>valueType</i>, [<i>valueContainsNull</i>])<br />
  <b>Note（注意）:</b> <i>valueContainsNull</i> 的默认值是 <i>true</i>.
  </td>
</tr>
<tr>
  <td> <b>StructType</b> </td>
  <td> org.apache.spark.sql.Row </td>
  <td>
  StructType(<i>fields</i>)<br />
  <b>Note（注意）:</b> <i>fields</i> 是 StructFields 的 Seq. 所有, 两个 fields 拥有相同的名称是不被允许的.
  </td>
</tr>
<tr>
  <td> <b>StructField</b> </td>
  <td> 该 field（字段）数据类型的 Scala 中的 value 类型
  (例如, 数据类型为 IntegerType 的 StructField 是 Int) </td>
  <td>
  StructField(<i>name</i>, <i>dataType</i>, [<i>nullable</i>])<br />
  <b>Note:</b>  <i>nullable</i> 的默认值是 <i>true</i>.
  </td>
</tr>
</table>

</div>

<div data-lang="java" markdown="1">

Spark SQL 的所有数据类型都在 `org.apache.spark.sql.types` 的包中.
要访问或者创建一个数据类型, 请使用 `org.apache.spark.sql.types.DataTypes` 中提供的 factory 方法.

<table class="table">
<tr>
  <th style="width:20%">Data type</th>
  <th style="width:40%">Value type in Java</th>
  <th>API to access or create a data type</th></tr>
<tr>
  <td> <b>ByteType</b> </td>
  <td> byte or Byte </td>
  <td>
  DataTypes.ByteType
  </td>
</tr>
<tr>
  <td> <b>ShortType</b> </td>
  <td> short or Short </td>
  <td>
  DataTypes.ShortType
  </td>
</tr>
<tr>
  <td> <b>IntegerType</b> </td>
  <td> int or Integer </td>
  <td>
  DataTypes.IntegerType
  </td>
</tr>
<tr>
  <td> <b>LongType</b> </td>
  <td> long or Long </td>
  <td>
  DataTypes.LongType
  </td>
</tr>
<tr>
  <td> <b>FloatType</b> </td>
  <td> float or Float </td>
  <td>
  DataTypes.FloatType
  </td>
</tr>
<tr>
  <td> <b>DoubleType</b> </td>
  <td> double or Double </td>
  <td>
  DataTypes.DoubleType
  </td>
</tr>
<tr>
  <td> <b>DecimalType</b> </td>
  <td> java.math.BigDecimal </td>
  <td>
  DataTypes.createDecimalType()<br />
  DataTypes.createDecimalType(<i>precision</i>, <i>scale</i>).
  </td>
</tr>
<tr>
  <td> <b>StringType</b> </td>
  <td> String </td>
  <td>
  DataTypes.StringType
  </td>
</tr>
<tr>
  <td> <b>BinaryType</b> </td>
  <td> byte[] </td>
  <td>
  DataTypes.BinaryType
  </td>
</tr>
<tr>
  <td> <b>BooleanType</b> </td>
  <td> boolean or Boolean </td>
  <td>
  DataTypes.BooleanType
  </td>
</tr>
<tr>
  <td> <b>TimestampType</b> </td>
  <td> java.sql.Timestamp </td>
  <td>
  DataTypes.TimestampType
  </td>
</tr>
<tr>
  <td> <b>DateType</b> </td>
  <td> java.sql.Date </td>
  <td>
  DataTypes.DateType
  </td>
</tr>
<tr>
  <td> <b>ArrayType</b> </td>
  <td> java.util.List </td>
  <td>
  DataTypes.createArrayType(<i>elementType</i>)<br />
  <b>Note:</b> The value of <i>containsNull</i> will be <i>true</i><br />
  DataTypes.createArrayType(<i>elementType</i>, <i>containsNull</i>).
  </td>
</tr>
<tr>
  <td> <b>MapType</b> </td>
  <td> java.util.Map </td>
  <td>
  DataTypes.createMapType(<i>keyType</i>, <i>valueType</i>)<br />
  <b>Note:</b> The value of <i>valueContainsNull</i> will be <i>true</i>.<br />
  DataTypes.createMapType(<i>keyType</i>, <i>valueType</i>, <i>valueContainsNull</i>)<br />
  </td>
</tr>
<tr>
  <td> <b>StructType</b> </td>
  <td> org.apache.spark.sql.Row </td>
  <td>
  DataTypes.createStructType(<i>fields</i>)<br />
  <b>Note:</b> <i>fields</i> is a List or an array of StructFields.
  Also, two fields with the same name are not allowed.
  </td>
</tr>
<tr>
  <td> <b>StructField</b> </td>
  <td> The value type in Java of the data type of this field
  (For example, int for a StructField with the data type IntegerType) </td>
  <td>
  DataTypes.createStructField(<i>name</i>, <i>dataType</i>, <i>nullable</i>)
  </td>
</tr>
</table>

</div>

<div data-lang="python"  markdown="1">

Spark SQL 的所有数据类型都在 `pyspark.sql.types` 的包中。你可以通过如下方式来访问它们.
{% highlight python %}
from pyspark.sql.types import *
{% endhighlight %}

<table class="table">
<tr>
  <th style="width:20%">Data type</th>
  <th style="width:40%">Value type in Python</th>
  <th>API to access or create a data type</th></tr>
<tr>
  <td> <b>ByteType</b> </td>
  <td>
  int or long <br />
  <b>Note:</b> Numbers will be converted to 1-byte signed integer numbers at runtime.
  Please make sure that numbers are within the range of -128 to 127.
  </td>
  <td>
  ByteType()
  </td>
</tr>
<tr>
  <td> <b>ShortType</b> </td>
  <td>
  int or long <br />
  <b>Note:</b> Numbers will be converted to 2-byte signed integer numbers at runtime.
  Please make sure that numbers are within the range of -32768 to 32767.
  </td>
  <td>
  ShortType()
  </td>
</tr>
<tr>
  <td> <b>IntegerType</b> </td>
  <td> int or long </td>
  <td>
  IntegerType()
  </td>
</tr>
<tr>
  <td> <b>LongType</b> </td>
  <td>
  long <br />
  <b>Note:</b> Numbers will be converted to 8-byte signed integer numbers at runtime.
  Please make sure that numbers are within the range of
  -9223372036854775808 to 9223372036854775807.
  Otherwise, please convert data to decimal.Decimal and use DecimalType.
  </td>
  <td>
  LongType()
  </td>
</tr>
<tr>
  <td> <b>FloatType</b> </td>
  <td>
  float <br />
  <b>Note:</b> Numbers will be converted to 4-byte single-precision floating
  point numbers at runtime.
  </td>
  <td>
  FloatType()
  </td>
</tr>
<tr>
  <td> <b>DoubleType</b> </td>
  <td> float </td>
  <td>
  DoubleType()
  </td>
</tr>
<tr>
  <td> <b>DecimalType</b> </td>
  <td> decimal.Decimal </td>
  <td>
  DecimalType()
  </td>
</tr>
<tr>
  <td> <b>StringType</b> </td>
  <td> string </td>
  <td>
  StringType()
  </td>
</tr>
<tr>
  <td> <b>BinaryType</b> </td>
  <td> bytearray </td>
  <td>
  BinaryType()
  </td>
</tr>
<tr>
  <td> <b>BooleanType</b> </td>
  <td> bool </td>
  <td>
  BooleanType()
  </td>
</tr>
<tr>
  <td> <b>TimestampType</b> </td>
  <td> datetime.datetime </td>
  <td>
  TimestampType()
  </td>
</tr>
<tr>
  <td> <b>DateType</b> </td>
  <td> datetime.date </td>
  <td>
  DateType()
  </td>
</tr>
<tr>
  <td> <b>ArrayType</b> </td>
  <td> list, tuple, or array </td>
  <td>
  ArrayType(<i>elementType</i>, [<i>containsNull</i>])<br />
  <b>Note:</b> The default value of <i>containsNull</i> is <i>True</i>.
  </td>
</tr>
<tr>
  <td> <b>MapType</b> </td>
  <td> dict </td>
  <td>
  MapType(<i>keyType</i>, <i>valueType</i>, [<i>valueContainsNull</i>])<br />
  <b>Note:</b> The default value of <i>valueContainsNull</i> is <i>True</i>.
  </td>
</tr>
<tr>
  <td> <b>StructType</b> </td>
  <td> list or tuple </td>
  <td>
  StructType(<i>fields</i>)<br />
  <b>Note:</b> <i>fields</i> is a Seq of StructFields. Also, two fields with the same
  name are not allowed.
  </td>
</tr>
<tr>
  <td> <b>StructField</b> </td>
  <td> The value type in Python of the data type of this field
  (For example, Int for a StructField with the data type IntegerType) </td>
  <td>
  StructField(<i>name</i>, <i>dataType</i>, [<i>nullable</i>])<br />
  <b>Note:</b> The default value of <i>nullable</i> is <i>True</i>.
  </td>
</tr>
</table>

</div>

<div data-lang="r"  markdown="1">

<table class="table">
<tr>
  <th style="width:20%">Data type</th>
  <th style="width:40%">Value type in R</th>
  <th>API to access or create a data type</th></tr>
<tr>
  <td> <b>ByteType</b> </td>
  <td>
  integer <br />
  <b>Note:</b> Numbers will be converted to 1-byte signed integer numbers at runtime.
  Please make sure that numbers are within the range of -128 to 127.
  </td>
  <td>
  "byte"
  </td>
</tr>
<tr>
  <td> <b>ShortType</b> </td>
  <td>
  integer <br />
  <b>Note:</b> Numbers will be converted to 2-byte signed integer numbers at runtime.
  Please make sure that numbers are within the range of -32768 to 32767.
  </td>
  <td>
  "short"
  </td>
</tr>
<tr>
  <td> <b>IntegerType</b> </td>
  <td> integer </td>
  <td>
  "integer"
  </td>
</tr>
<tr>
  <td> <b>LongType</b> </td>
  <td>
  integer <br />
  <b>Note:</b> Numbers will be converted to 8-byte signed integer numbers at runtime.
  Please make sure that numbers are within the range of
  -9223372036854775808 to 9223372036854775807.
  Otherwise, please convert data to decimal.Decimal and use DecimalType.
  </td>
  <td>
  "long"
  </td>
</tr>
<tr>
  <td> <b>FloatType</b> </td>
  <td>
  numeric <br />
  <b>Note:</b> Numbers will be converted to 4-byte single-precision floating
  point numbers at runtime.
  </td>
  <td>
  "float"
  </td>
</tr>
<tr>
  <td> <b>DoubleType</b> </td>
  <td> numeric </td>
  <td>
  "double"
  </td>
</tr>
<tr>
  <td> <b>DecimalType</b> </td>
  <td> Not supported </td>
  <td>
   Not supported
  </td>
</tr>
<tr>
  <td> <b>StringType</b> </td>
  <td> character </td>
  <td>
  "string"
  </td>
</tr>
<tr>
  <td> <b>BinaryType</b> </td>
  <td> raw </td>
  <td>
  "binary"
  </td>
</tr>
<tr>
  <td> <b>BooleanType</b> </td>
  <td> logical </td>
  <td>
  "bool"
  </td>
</tr>
<tr>
  <td> <b>TimestampType</b> </td>
  <td> POSIXct </td>
  <td>
  "timestamp"
  </td>
</tr>
<tr>
  <td> <b>DateType</b> </td>
  <td> Date </td>
  <td>
  "date"
  </td>
</tr>
<tr>
  <td> <b>ArrayType</b> </td>
  <td> vector or list </td>
  <td>
  list(type="array", elementType=<i>elementType</i>, containsNull=[<i>containsNull</i>])<br />
  <b>Note:</b> The default value of <i>containsNull</i> is <i>TRUE</i>.
  </td>
</tr>
<tr>
  <td> <b>MapType</b> </td>
  <td> environment </td>
  <td>
  list(type="map", keyType=<i>keyType</i>, valueType=<i>valueType</i>, valueContainsNull=[<i>valueContainsNull</i>])<br />
  <b>Note:</b> The default value of <i>valueContainsNull</i> is <i>TRUE</i>.
  </td>
</tr>
<tr>
  <td> <b>StructType</b> </td>
  <td> named list</td>
  <td>
  list(type="struct", fields=<i>fields</i>)<br />
  <b>Note:</b> <i>fields</i> is a Seq of StructFields. Also, two fields with the same
  name are not allowed.
  </td>
</tr>
<tr>
  <td> <b>StructField</b> </td>
  <td> The value type in R of the data type of this field
  (For example, integer for a StructField with the data type IntegerType) </td>
  <td>
  list(name=<i>name</i>, type=<i>dataType</i>, nullable=[<i>nullable</i>])<br />
  <b>Note:</b> The default value of <i>nullable</i> is <i>TRUE</i>.
  </td>
</tr>
</table>

</div>

</div>

## NaN Semantics

当处理一些不符合标准浮点数语义的 `float` 或 `double` 类型时，对于 Not-a-Number(NaN) 需要做一些特殊处理.
具体如下:
 - NaN = NaN 返回 true.
 - 在 aggregations（聚合）操作中，所有的 NaN values 将被分到同一个组中.
 - 在 join key 中 NaN 可以当做一个普通的值.
 - NaN 值在升序排序中排到最后，比任何其他数值都大.
