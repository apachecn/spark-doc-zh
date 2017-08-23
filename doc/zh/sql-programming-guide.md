---
layout: global
displayTitle: Spark SQL, DataFrames and Datasets Guide
title: Spark SQL and DataFrames
---

* This will become a table of contents (this text will be scraped).
{:toc}

# Overview

Spark SQL 是 Spark 处理结构化数据的一个模块。与基础的 Spark RDD API 不同，Spark SQL 提供了查询结构化数据及计算结果等信息的接口。在内部，Spark SQL 使用这个额外的信息去执行额外的优化。有几种方式可以跟 Spark SQL 进行交互，包括 SQL 和 Dataset API。当使用相同执行引擎进行计算时，无论使用哪种 API / 语言都可以快速的计算。这种统一意味着开发人员能够在基于提供最自然的方式来表达一个给定的 transformation API 之间实现轻松的来回切换不同的 。

该页面所有例子使用的示例数据都包含在 Spark 的发布中，并且可以使用 `spark-shell`, `pyspark` shell, 或者 `sparkR` shell来运行.


## SQL

Spark SQL 的功能之一是执行 SQL 查询。Spark SQL 也能够被用于从已存在的 Hive 环境中读取数据。更多关于如何配置这个特性的信息，请参考 [Hive 表](#hive-tables) 这部分. 当以另外的编程语言运行SQL  时，查询结果将以 [Dataset/DataFrame](#datasets-and-dataframes)的形式返回。您也可以使用 [命令行](#running-the-spark-sql-cli)或者通过 [JDBC/ODBC](#running-the-thrift-jdbcodbc-server)与 SQL 接口交互。

## Datasets and DataFrames

一个 Dataset 是一个分布式的数据集合
Dataset 是在 Spark 1.6 中被添加的新接口，它提供了 RDD 的优点（强类型化，能够使用强大的 lambda 函数）与Spark SQL执行引擎的优点。一个 Dataset 可以从 JVM 对象来 [构造](#creating-datasets) 并且使用转换功能（map，flatMap，filter，等等）。
Dataset API 在[Scala][scala-datasets] 和
[Java][java-datasets]是可用的。Python 不支持 Dataset API。但是由于 Python 的动态特性，许多 Dataset API 的优点已经可用了 (也就是说，你可能通过 name 天生的`row.columnName`属性访问一行中的字段)。这种情况和 R 相似。

一个 DataFrame 是一个 *Dataset* 组成的指定列。它的概念与一个在关系型数据库或者在 R/Python 中的表是相等的， 但是有很多优化. DataFrames 可以从大量的 [sources](#data-sources) 中构造出来，比如: 结构化的文本文件, Hive中的表, 外部数据库, 或者已经存在的 RDDs。
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

注意第一次调用时, `sparkR.session()` 初始化一个全局的 `SparkSession` 单实例, 并且总是返回一个引用此实例，可以连续的调用. 通过这种方式, 用户仅需要创建一次 `SparkSession` , 然后像 `read.df` SparkR函数就能够立即获取全局的实例,用户不需要再 `SparkSession` 之间进行实例的传递.
</div>
</div>

Spark 2.0 中的`SparkSession` 为 Hive 特性提供了内嵌的支持，包括使用 HiveQL 编写查询的能力，访问 Hive UDF,以及从 Hive 表中读取数据的能力。为了使用这些特性，你不需要去有一个已存在的 Hive 设置。

## 创建 DataFrames

<div class="codetabs">
<div data-lang="scala"  markdown="1">
在一个 `SparkSession`中, 应用程序可以从一个 [已经存在的 `RDD`](#interoperating-with-rdds),
从hive表, 或者从 [Spark数据源](#data-sources)中创建一个DataFrames.

举个例子, 下面就是基于一个JSON文件创建一个DataFrame:

{% include_example create_df scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java"  markdown="1">
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

<div data-lang="r"  markdown="1">
在一个 `SparkSession`中, 应用程序可以从一个本地的R frame 数据,
从hive表, 或者从[Spark数据源](#data-sources).

举个例子, 下面就是基于一个JSON文件创建一个DataFrame:

{% include_example create_df r/RSparkSQLExample.R %}

</div>
</div>


## 无类型的Dataset操作 (aka DataFrame 操作)

DataFrames 提供了一个特定的语法用在 [Scala](api/scala/index.html#org.apache.spark.sql.Dataset), [Java](api/java/index.html?org/apache/spark/sql/Dataset.html), [Python](api/python/pyspark.sql.html#pyspark.sql.DataFrame) and [R](api/R/SparkDataFrame.html)中机构化数据的操作.

正如上面提到的一样, Spark 2.0中, DataFrames在Scala 和 Java API中，仅仅是多个 `Row`s的Dataset. 这些操作也参考了与强类型的Scala/Java Datasets中的"类型转换" 对应的"无类型转换" 。

这里包括一些使用 Dataset 进行结构化数据处理的示例 :

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example untyped_ops scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}

能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/scala/index.html#org.apache.spark.sql.Dataset).

除了简单的列引用和表达式之外，DataFrame 也有丰富的函数库，包括 string 操作，date 算术，常见的 math 操作以及更多。可用的完整列表请参考  [DataFrame 函数指南](api/scala/index.html#org.apache.spark.sql.functions$).
</div>

<div data-lang="java"  markdown="1">

{% include_example untyped_ops java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}

为了能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/java/org/apache/spark/sql/Dataset.html).

除了简单的列引用和表达式之外，DataFrame 也有丰富的函数库，包括 string 操作，date 算术，常见的 math 操作以及更多。可用的完整列表请参考  [DataFrame 函数指南](api/java/org/apache/spark/sql/functions.html).
</div>

<div data-lang="python"  markdown="1">
在Python中，可以通过(`df.age`) 或者(`df['age']`)来获取DataFrame的列. 虽然前者便于交互式操作, 但是还是建议用户使用后者, 这样不会破坏列名，也能引用DataFrame的类.

{% include_example untyped_ops python/sql/basic.py %}
为了能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/python/pyspark.sql.html#pyspark.sql.DataFrame).

除了简单的列引用和表达式之外，DataFrame 也有丰富的函数库，包括 string 操作，date 算术，常见的 math 操作以及更多。可用的完整列表请参考  [DataFrame 函数指南](api/python/pyspark.sql.html#module-pyspark.sql.functions).

</div>

<div data-lang="r"  markdown="1">

{% include_example untyped_ops r/RSparkSQLExample.R %}

为了能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](api/R/index.html).

除了简单的列引用和表达式之外，DataFrame 也有丰富的函数库，包括 string 操作，date 算术，常见的 math 操作以及更多。可用的完整列表请参考  [DataFrame 函数指南](api/R/SparkDataFrame.html).

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

Spark SQL中的临时视图是session级别的，也就是会随着session的消失而消失. 如果你想让一个临时视图在所有session中相互传递并且可用，直到Spark 应用退出, 你可以建立一个全局的临时视图.全局的临时视图存在于系统数据库 `global_temp`中, 我们必须加上库名去引用它, 比如. `SELECT * FROM global_temp.view1`.

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

Dataset 与 RDD 相似，然而，并不是使用 Java 序列化或者 Kryo [编码器](api/scala/index.html#org.apache.spark.sql.Encoder) 来序列化用于处理或者通过网络进行传输的对象. 虽然编码器和标准的序列化都负责将一个对象序列化成字节，编码器是动态生成的代码，并且使用了一种允许 Spark 去执行许多像 filtering，sorting 以及 hashing 这样的操作，不需要将字节反序列化成对象的格式。

<div class="codetabs">
<div data-lang="scala"  markdown="1">
{% include_example create_ds scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java" markdown="1">
{% include_example create_ds java/org/apache/spark/examples/sql/JavaSparkSQLExample.java %}
</div>
</div>

## RDD的互操作性

Spark SQL 支持两种不同的方法用于转换已存在的 RDD 成为 Dataset。第一种方法是使用反射去推断一个包含指定的对象类型的 RDD 的 Schema。在你的 Spark 应用程序中当你已知 Schema 时这个基于方法的反射可以让你的代码更简洁。

第二种用于创建 Dataset 的方法是通过一个允许你构造一个 Schema 然后把它应用到一个已存在的 RDD 的编程接口。然而这种方法更繁琐，当列和它们的类型知道运行时都是未知时它允许你去构造 Dataset。

### 使用反射推断Schema
<div class="codetabs">

<div data-lang="scala"  markdown="1">

Spark SQL 的 Scala 接口支持自动转换一个包含 case classes 的 RDD 为 DataFrame。Case class 定义了表的 Schema。Case class 的参数名使用反射读取并且成为了列名。Case class 也可以是嵌套的或者包含像 `Seq` 或者 `Array` 这样的复杂类型。这个 RDD 能够被隐式转换成一个 DataFrame 然后被注册为一个表。表可以用于后续的 SQL 语句。

{% include_example schema_inferring scala/org/apache/spark/examples/sql/SparkSQLExample.scala %}
</div>

<div data-lang="java"  markdown="1">

Spark SQL 支持一个[JavaBeans]的RDD(http://stackoverflow.com/questions/3295496/what-is-a-javabean-exactly)自动转换为一个DataFrame.
`BeanInfo`利用反射定义表的schema. 目前Spark SQL不支持含有`Map`的JavaBeans. 但是支持嵌套`List`或者 `Array`JavaBeans . 
你可以通过创建一个有getters和setters的序列化的类来创建一个JavaBean。

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

当 case class 不能够在执行之前被定义（例如，records 记录的结构在一个 string 字符串中被编码了，或者一个 text 文本 dataset 将被解析并且不同的用户投影的字段是不一样的）。一个 `DataFrame` 可以使用下面的三步以编程的方式来创建。

1. 从原始的 RDD 创建 RDD 的 `Row`（行）;
2. Step 1 被创建后，创建 Schema 表示一个 `StructType` 匹配 RDD 中的 `Row`（行）的结构.
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

当一个字典不能被提前定义 (例如,记录的结构是在一个字符串中, 抑或一个文本中解析，被不同的用户所属),
一个 `DataFrame` 可以通过以下3步来创建.

1. RDD从原始的RDD穿件一个RDD的toples或者一个列表;
2. Step 1 被创建后，创建 Schema 表示一个 `StructType` 匹配 RDD 中的结构.
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

# Data Sources

Spark SQL supports operating on a variety of data sources through the DataFrame interface.
A DataFrame can be operated on using relational transformations and can also be used to create a temporary view.
Registering a DataFrame as a temporary view allows you to run SQL queries over its data. This section
describes the general methods for loading and saving data using the Spark Data Sources and then
goes into specific options that are available for the built-in data sources.

## Generic Load/Save Functions

In the simplest form, the default data source (`parquet` unless otherwise configured by
`spark.sql.sources.default`) will be used for all operations.

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

### Manually Specifying Options

You can also manually specify the data source that will be used along with any extra options
that you would like to pass to the data source. Data sources are specified by their fully qualified
name (i.e., `org.apache.spark.sql.parquet`), but for built-in sources you can also use their short
names (`json`, `parquet`, `jdbc`, `orc`, `libsvm`, `csv`, `text`). DataFrames loaded from any data
source type can be converted into other types using this syntax.

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

### Run SQL on files directly

Instead of using read API to load a file into DataFrame and query it, you can also query that
file directly with SQL.

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

### Save Modes

Save operations can optionally take a `SaveMode`, that specifies how to handle existing data if
present. It is important to realize that these save modes do not utilize any locking and are not
atomic. Additionally, when performing an `Overwrite`, the data will be deleted before writing out the
new data.

<table class="table">
<tr><th>Scala/Java</th><th>Any Language</th><th>Meaning</th></tr>
<tr>
  <td><code>SaveMode.ErrorIfExists</code> (default)</td>
  <td><code>"error"</code> (default)</td>
  <td>
    When saving a DataFrame to a data source, if data already exists,
    an exception is expected to be thrown.
  </td>
</tr>
<tr>
  <td><code>SaveMode.Append</code></td>
  <td><code>"append"</code></td>
  <td>
    When saving a DataFrame to a data source, if data/table already exists,
    contents of the DataFrame are expected to be appended to existing data.
  </td>
</tr>
<tr>
  <td><code>SaveMode.Overwrite</code></td>
  <td><code>"overwrite"</code></td>
  <td>
    Overwrite mode means that when saving a DataFrame to a data source,
    if data/table already exists, existing data is expected to be overwritten by the contents of
    the DataFrame.
  </td>
</tr>
<tr>
  <td><code>SaveMode.Ignore</code></td>
  <td><code>"ignore"</code></td>
  <td>
    Ignore mode means that when saving a DataFrame to a data source, if data already exists,
    the save operation is expected to not save the contents of the DataFrame and to not
    change the existing data. This is similar to a <code>CREATE TABLE IF NOT EXISTS</code> in SQL.
  </td>
</tr>
</table>

### Saving to Persistent Tables

`DataFrames` can also be saved as persistent tables into Hive metastore using the `saveAsTable`
command. Notice that an existing Hive deployment is not necessary to use this feature. Spark will create a
default local Hive metastore (using Derby) for you. Unlike the `createOrReplaceTempView` command,
`saveAsTable` will materialize the contents of the DataFrame and create a pointer to the data in the
Hive metastore. Persistent tables will still exist even after your Spark program has restarted, as
long as you maintain your connection to the same metastore. A DataFrame for a persistent table can
be created by calling the `table` method on a `SparkSession` with the name of the table.

For file-based data source, e.g. text, parquet, json, etc. you can specify a custom table path via the
`path` option, e.g. `df.write.option("path", "/some/path").saveAsTable("t")`. When the table is dropped,
the custom table path will not be removed and the table data is still there. If no custom table path is
specified, Spark will write data to a default table path under the warehouse directory. When the table is
dropped, the default table path will be removed too.

Starting from Spark 2.1, persistent datasource tables have per-partition metadata stored in the Hive metastore. This brings several benefits:

- Since the metastore can return only necessary partitions for a query, discovering all the partitions on the first query to the table is no longer needed.
- Hive DDLs such as `ALTER TABLE PARTITION ... SET LOCATION` are now available for tables created with the Datasource API.

Note that partition information is not gathered by default when creating external datasource tables (those with a `path` option). To sync the partition information in the metastore, you can invoke `MSCK REPAIR TABLE`.

### Bucketing, Sorting and Partitioning

For file-based data source, it is also possible to bucket and sort or partition the output. 
Bucketing and sorting are applicable only to persistent tables:

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

while partitioning can be used with both `save` and `saveAsTable` when using the Dataset APIs.


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

It is possible to use both partitioning and bucketing for a single table:

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

`partitionBy` creates a directory structure as described in the [Partition Discovery](#partition-discovery) section.
Thus, it has limited applicability to columns with high cardinality. In contrast 
 `bucketBy` distributes
data across a fixed number of buckets and can be used when a number of unique values is unbounded.

## Parquet Files

[Parquet](http://parquet.io) is a columnar format that is supported by many other data processing systems.
Spark SQL provides support for both reading and writing Parquet files that automatically preserves the schema
of the original data. When writing Parquet files, all columns are automatically converted to be nullable for
compatibility reasons.

### Loading Data Programmatically

Using the data from the above example:

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

### Partition Discovery

Table partitioning is a common optimization approach used in systems like Hive. In a partitioned
table, data are usually stored in different directories, with partitioning column values encoded in
the path of each partition directory. The Parquet data source is now able to discover and infer
partitioning information automatically. For example, we can store all our previously used
population data into a partitioned table using the following directory structure, with two extra
columns, `gender` and `country` as partitioning columns:

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

By passing `path/to/table` to either `SparkSession.read.parquet` or `SparkSession.read.load`, Spark SQL
will automatically extract the partitioning information from the paths.
Now the schema of the returned DataFrame becomes:

{% highlight text %}

root
|-- name: string (nullable = true)
|-- age: long (nullable = true)
|-- gender: string (nullable = true)
|-- country: string (nullable = true)

{% endhighlight %}

Notice that the data types of the partitioning columns are automatically inferred. Currently,
numeric data types and string type are supported. Sometimes users may not want to automatically
infer the data types of the partitioning columns. For these use cases, the automatic type inference
can be configured by `spark.sql.sources.partitionColumnTypeInference.enabled`, which is default to
`true`. When type inference is disabled, string type will be used for the partitioning columns.

Starting from Spark 1.6.0, partition discovery only finds partitions under the given paths
by default. For the above example, if users pass `path/to/table/gender=male` to either
`SparkSession.read.parquet` or `SparkSession.read.load`, `gender` will not be considered as a
partitioning column. If users need to specify the base path that partition discovery
should start with, they can set `basePath` in the data source options. For example,
when `path/to/table/gender=male` is the path of the data and
users set `basePath` to `path/to/table/`, `gender` will be a partitioning column.

### Schema Merging

Like ProtocolBuffer, Avro, and Thrift, Parquet also supports schema evolution. Users can start with
a simple schema, and gradually add more columns to the schema as needed. In this way, users may end
up with multiple Parquet files with different but mutually compatible schemas. The Parquet data
source is now able to automatically detect this case and merge schemas of all these files.

Since schema merging is a relatively expensive operation, and is not a necessity in most cases, we
turned it off by default starting from 1.5.0. You may enable it by

1. setting data source option `mergeSchema` to `true` when reading Parquet files (as shown in the
   examples below), or
2. setting the global SQL option `spark.sql.parquet.mergeSchema` to `true`.

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

### Hive metastore Parquet table conversion

When reading from and writing to Hive metastore Parquet tables, Spark SQL will try to use its own
Parquet support instead of Hive SerDe for better performance. This behavior is controlled by the
`spark.sql.hive.convertMetastoreParquet` configuration, and is turned on by default.

#### Hive/Parquet Schema Reconciliation

There are two key differences between Hive and Parquet from the perspective of table schema
processing.

1. Hive is case insensitive, while Parquet is not
1. Hive considers all columns nullable, while nullability in Parquet is significant

Due to this reason, we must reconcile Hive metastore schema with Parquet schema when converting a
Hive metastore Parquet table to a Spark SQL Parquet table. The reconciliation rules are:

1. Fields that have the same name in both schema must have the same data type regardless of
   nullability. The reconciled field should have the data type of the Parquet side, so that
   nullability is respected.

1. The reconciled schema contains exactly those fields defined in Hive metastore schema.

   - Any fields that only appear in the Parquet schema are dropped in the reconciled schema.
   - Any fields that only appear in the Hive metastore schema are added as nullable field in the
     reconciled schema.

#### Metadata Refreshing

Spark SQL caches Parquet metadata for better performance. When Hive metastore Parquet table
conversion is enabled, metadata of those converted tables are also cached. If these tables are
updated by Hive or other external tools, you need to refresh them manually to ensure consistent
metadata.

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

### Configuration

Configuration of Parquet can be done using the `setConf` method on `SparkSession` or by running
`SET key=value` commands using SQL.

<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.sql.parquet.binaryAsString</code></td>
  <td>false</td>
  <td>
    Some other Parquet-producing systems, in particular Impala, Hive, and older versions of Spark SQL, do
    not differentiate between binary data and strings when writing out the Parquet schema. This
    flag tells Spark SQL to interpret binary data as a string to provide compatibility with these systems.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.int96AsTimestamp</code></td>
  <td>true</td>
  <td>
    Some Parquet-producing systems, in particular Impala and Hive, store Timestamp into INT96. This
    flag tells Spark SQL to interpret INT96 data as a timestamp to provide compatibility with these systems.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.cacheMetadata</code></td>
  <td>true</td>
  <td>
    Turns on caching of Parquet schema metadata. Can speed up querying of static data.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.compression.codec</code></td>
  <td>snappy</td>
  <td>
    Sets the compression codec use when writing Parquet files. Acceptable values include:
    uncompressed, snappy, gzip, lzo.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.filterPushdown</code></td>
  <td>true</td>
  <td>Enables Parquet filter push-down optimization when set to true.</td>
</tr>
<tr>
  <td><code>spark.sql.hive.convertMetastoreParquet</code></td>
  <td>true</td>
  <td>
    When set to false, Spark SQL will use the Hive SerDe for parquet tables instead of the built in
    support.
  </td>
</tr>
<tr>
  <td><code>spark.sql.parquet.mergeSchema</code></td>
  <td>false</td>
  <td>
    <p>
      When true, the Parquet data source merges schemas collected from all data files, otherwise the
      schema is picked from the summary file or a random data file if no summary file is available.
    </p>
  </td>
</tr>
<tr>
  <td><code>spark.sql.optimizer.metadataOnly</code></td>
  <td>true</td>
  <td>
    <p>
      When true, enable the metadata-only query optimization that use the table's metadata to
      produce the partition columns instead of table scans. It applies when all the columns scanned
      are partition columns and the query has an aggregate operator that satisfies distinct
      semantics.
    </p>
  </td>
</tr>
</table>

## JSON Datasets
<div class="codetabs">

<div data-lang="scala"  markdown="1">
Spark SQL can automatically infer the schema of a JSON dataset and load it as a `Dataset[Row]`.
This conversion can be done using `SparkSession.read.json()` on either a `Dataset[String]`,
or a JSON file.

Note that the file that is offered as _a json file_ is not a typical JSON file. Each
line must contain a separate, self-contained valid JSON object. For more information, please see
[JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/).

For a regular multi-line JSON file, set the `multiLine` option to `true`.

{% include_example json_dataset scala/org/apache/spark/examples/sql/SQLDataSourceExample.scala %}
</div>

<div data-lang="java"  markdown="1">
Spark SQL can automatically infer the schema of a JSON dataset and load it as a `Dataset<Row>`.
This conversion can be done using `SparkSession.read().json()` on either a `Dataset<String>`,
or a JSON file.

Note that the file that is offered as _a json file_ is not a typical JSON file. Each
line must contain a separate, self-contained valid JSON object. For more information, please see
[JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/).

For a regular multi-line JSON file, set the `multiLine` option to `true`.

{% include_example json_dataset java/org/apache/spark/examples/sql/JavaSQLDataSourceExample.java %}
</div>

<div data-lang="python"  markdown="1">
Spark SQL can automatically infer the schema of a JSON dataset and load it as a DataFrame.
This conversion can be done using `SparkSession.read.json` on a JSON file.

Note that the file that is offered as _a json file_ is not a typical JSON file. Each
line must contain a separate, self-contained valid JSON object. For more information, please see
[JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/).

For a regular multi-line JSON file, set the `multiLine` parameter to `True`.

{% include_example json_dataset python/sql/datasource.py %}
</div>

<div data-lang="r"  markdown="1">
Spark SQL can automatically infer the schema of a JSON dataset and load it as a DataFrame. using
the `read.json()` function, which loads data from a directory of JSON files where each line of the
files is a JSON object.

Note that the file that is offered as _a json file_ is not a typical JSON file. Each
line must contain a separate, self-contained valid JSON object. For more information, please see
[JSON Lines text format, also called newline-delimited JSON](http://jsonlines.org/).

For a regular multi-line JSON file, set a named parameter `multiLine` to `TRUE`.

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

Spark SQL also supports reading and writing data stored in [Apache Hive](http://hive.apache.org/).
However, since Hive has a large number of dependencies, these dependencies are not included in the
default Spark distribution. If Hive dependencies can be found on the classpath, Spark will load them
automatically. Note that these Hive dependencies must also be present on all of the worker nodes, as
they will need access to the Hive serialization and deserialization libraries (SerDes) in order to
access data stored in Hive.

Configuration of Hive is done by placing your `hive-site.xml`, `core-site.xml` (for security configuration),
and `hdfs-site.xml` (for HDFS configuration) file in `conf/`.

When working with Hive, one must instantiate `SparkSession` with Hive support, including
connectivity to a persistent Hive metastore, support for Hive serdes, and Hive user-defined functions.
Users who do not have an existing Hive deployment can still enable Hive support. When not configured
by the `hive-site.xml`, the context automatically creates `metastore_db` in the current directory and
creates a directory configured by `spark.sql.warehouse.dir`, which defaults to the directory
`spark-warehouse` in the current directory that the Spark application is started. Note that
the `hive.metastore.warehouse.dir` property in `hive-site.xml` is deprecated since Spark 2.0.0.
Instead, use `spark.sql.warehouse.dir` to specify the default location of database in warehouse.
You may need to grant write privilege to the user who starts the Spark application.

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

When working with Hive one must instantiate `SparkSession` with Hive support. This
adds support for finding tables in the MetaStore and writing queries using HiveQL.

{% include_example spark_hive r/RSparkSQLExample.R %}

</div>
</div>

### Specifying storage format for Hive tables

When you create a Hive table, you need to define how this table should read/write data from/to file system,
i.e. the "input format" and "output format". You also need to define how this table should deserialize the data
to rows, or serialize rows to data, i.e. the "serde". The following options can be used to specify the storage
format("serde", "input format", "output format"), e.g. `CREATE TABLE src(id int) USING hive OPTIONS(fileFormat 'parquet')`.
By default, we will read the table files as plain text. Note that, Hive storage handler is not supported yet when
creating table, you can create a table using storage handler at Hive side, and use Spark SQL to read it.

<table class="table">
  <tr><th>Property Name</th><th>Meaning</th></tr>
  <tr>
    <td><code>fileFormat</code></td>
    <td>
      A fileFormat is kind of a package of storage format specifications, including "serde", "input format" and
      "output format". Currently we support 6 fileFormats: 'sequencefile', 'rcfile', 'orc', 'parquet', 'textfile' and 'avro'.
    </td>
  </tr>

  <tr>
    <td><code>inputFormat, outputFormat</code></td>
    <td>
      These 2 options specify the name of a corresponding `InputFormat` and `OutputFormat` class as a string literal,
      e.g. `org.apache.hadoop.hive.ql.io.orc.OrcInputFormat`. These 2 options must be appeared in pair, and you can not
      specify them if you already specified the `fileFormat` option.
    </td>
  </tr>

  <tr>
    <td><code>serde</code></td>
    <td>
      This option specifies the name of a serde class. When the `fileFormat` option is specified, do not specify this option
      if the given `fileFormat` already include the information of serde. Currently "sequencefile", "textfile" and "rcfile"
      don't include the serde information and you can use this option with these 3 fileFormats.
    </td>
  </tr>

  <tr>
    <td><code>fieldDelim, escapeDelim, collectionDelim, mapkeyDelim, lineDelim</code></td>
    <td>
      These options can only be used with "textfile" fileFormat. They define how to read delimited files into rows.
    </td>
  </tr>
</table>

All other properties defined with `OPTIONS` will be regarded as Hive serde properties.

### Interacting with Different Versions of Hive Metastore

One of the most important pieces of Spark SQL's Hive support is interaction with Hive metastore,
which enables Spark SQL to access metadata of Hive tables. Starting from Spark 1.4.0, a single binary
build of Spark SQL can be used to query different versions of Hive metastores, using the configuration described below.
Note that independent of the version of Hive that is being used to talk to the metastore, internally Spark SQL
will compile against Hive 1.2.1 and use those classes for internal execution (serdes, UDFs, UDAFs, etc).

The following options can be used to configure the version of Hive that is used to retrieve metadata:

<table class="table">
  <tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
  <tr>
    <td><code>spark.sql.hive.metastore.version</code></td>
    <td><code>1.2.1</code></td>
    <td>
      Version of the Hive metastore. Available
      options are <code>0.12.0</code> through <code>1.2.1</code>.
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.jars</code></td>
    <td><code>builtin</code></td>
    <td>
      Location of the jars that should be used to instantiate the HiveMetastoreClient. This
      property can be one of three options:
      <ol>
        <li><code>builtin</code></li>
        Use Hive 1.2.1, which is bundled with the Spark assembly when <code>-Phive</code> is
        enabled. When this option is chosen, <code>spark.sql.hive.metastore.version</code> must be
        either <code>1.2.1</code> or not defined.
        <li><code>maven</code></li>
        Use Hive jars of specified version downloaded from Maven repositories. This configuration
        is not generally recommended for production deployments.
        <li>A classpath in the standard format for the JVM. This classpath must include all of Hive
        and its dependencies, including the correct version of Hadoop. These jars only need to be
        present on the driver, but if you are running in yarn cluster mode then you must ensure
        they are packaged with your application.</li>
      </ol>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.sharedPrefixes</code></td>
    <td><code>com.mysql.jdbc,<br/>org.postgresql,<br/>com.microsoft.sqlserver,<br/>oracle.jdbc</code></td>
    <td>
      <p>
        A comma separated list of class prefixes that should be loaded using the classloader that is
        shared between Spark SQL and a specific version of Hive. An example of classes that should
        be shared is JDBC drivers that are needed to talk to the metastore. Other classes that need
        to be shared are those that interact with classes that are already shared. For example,
        custom appenders that are used by log4j.
      </p>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.hive.metastore.barrierPrefixes</code></td>
    <td><code>(empty)</code></td>
    <td>
      <p>
        A comma separated list of class prefixes that should explicitly be reloaded for each version
        of Hive that Spark SQL is communicating with. For example, Hive UDFs that are declared in a
        prefix that typically would be shared (i.e. <code>org.apache.spark.*</code>).
      </p>
    </td>
  </tr>
</table>


## JDBC To Other Databases

Spark SQL also includes a data source that can read data from other databases using JDBC. This
functionality should be preferred over using [JdbcRDD](api/scala/index.html#org.apache.spark.rdd.JdbcRDD).
This is because the results are returned
as a DataFrame and they can easily be processed in Spark SQL or joined with other data sources.
The JDBC data source is also easier to use from Java or Python as it does not require the user to
provide a ClassTag.
(Note that this is different than the Spark SQL JDBC server, which allows other applications to
run queries using Spark SQL).

To get started you will need to include the JDBC driver for you particular database on the
spark classpath. For example, to connect to postgres from the Spark Shell you would run the
following command:

{% highlight bash %}
bin/spark-shell --driver-class-path postgresql-9.4.1207.jar --jars postgresql-9.4.1207.jar
{% endhighlight %}

Tables from the remote database can be loaded as a DataFrame or Spark SQL temporary view using
the Data Sources API. Users can specify the JDBC connection properties in the data source options.
<code>user</code> and <code>password</code> are normally provided as connection properties for
logging into the data sources. In addition to the connection properties, Spark also supports
the following case-insensitive options:

<table class="table">
  <tr><th>Property Name</th><th>Meaning</th></tr>
  <tr>
    <td><code>url</code></td>
    <td>
      The JDBC URL to connect to. The source-specific connection properties may be specified in the URL. e.g., <code>jdbc:postgresql://localhost/test?user=fred&password=secret</code>
    </td>
  </tr>

  <tr>
    <td><code>dbtable</code></td>
    <td>
      The JDBC table that should be read. Note that anything that is valid in a <code>FROM</code> clause of
      a SQL query can be used. For example, instead of a full table you could also use a
      subquery in parentheses.
    </td>
  </tr>

  <tr>
    <td><code>driver</code></td>
    <td>
      The class name of the JDBC driver to use to connect to this URL.
    </td>
  </tr>

  <tr>
    <td><code>partitionColumn, lowerBound, upperBound</code></td>
    <td>
      These options must all be specified if any of them is specified. In addition,
      <code>numPartitions</code> must be specified. They describe how to partition the table when
      reading in parallel from multiple workers.
      <code>partitionColumn</code> must be a numeric column from the table in question. Notice
      that <code>lowerBound</code> and <code>upperBound</code> are just used to decide the
      partition stride, not for filtering the rows in table. So all rows in the table will be
      partitioned and returned. This option applies only to reading.
    </td>
  </tr>

  <tr>
     <td><code>numPartitions</code></td>
     <td>
       The maximum number of partitions that can be used for parallelism in table reading and
       writing. This also determines the maximum number of concurrent JDBC connections.
       If the number of partitions to write exceeds this limit, we decrease it to this limit by
       calling <code>coalesce(numPartitions)</code> before writing.
     </td>
  </tr>

  <tr>
    <td><code>fetchsize</code></td>
    <td>
      The JDBC fetch size, which determines how many rows to fetch per round trip. This can help performance on JDBC drivers which default to low fetch size (eg. Oracle with 10 rows). This option applies only to reading.
    </td>
  </tr>

  <tr>
     <td><code>batchsize</code></td>
     <td>
       The JDBC batch size, which determines how many rows to insert per round trip. This can help performance on JDBC drivers. This option applies only to writing. It defaults to <code>1000</code>.
     </td>
  </tr>

  <tr>
     <td><code>isolationLevel</code></td>
     <td>
       The transaction isolation level, which applies to current connection. It can be one of <code>NONE</code>, <code>READ_COMMITTED</code>, <code>READ_UNCOMMITTED</code>, <code>REPEATABLE_READ</code>, or <code>SERIALIZABLE</code>, corresponding to standard transaction isolation levels defined by JDBC's Connection object, with default of <code>READ_UNCOMMITTED</code>. This option applies only to writing. Please refer the documentation in <code>java.sql.Connection</code>.
     </td>
   </tr>

  <tr>
    <td><code>truncate</code></td>
    <td>
     This is a JDBC writer related option. When <code>SaveMode.Overwrite</code> is enabled, this option causes Spark to truncate an existing table instead of dropping and recreating it. This can be more efficient, and prevents the table metadata (e.g., indices) from being removed. However, it will not work in some cases, such as when the new data has a different schema. It defaults to <code>false</code>. This option applies only to writing.
   </td>
  </tr>

  <tr>
    <td><code>createTableOptions</code></td>
    <td>
     This is a JDBC writer related option. If specified, this option allows setting of database-specific table and partition options when creating a table (e.g., <code>CREATE TABLE t (name string) ENGINE=InnoDB.</code>). This option applies only to writing.
   </td>
  </tr>

  <tr>
    <td><code>createTableColumnTypes</code></td>
    <td>
     The database column data types to use instead of the defaults, when creating the table. Data type information should be specified in the same format as CREATE TABLE columns syntax (e.g: <code>"name CHAR(64), comments VARCHAR(1024)")</code>. The specified types should be valid spark sql data types. This option applies only to writing.
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

## Troubleshooting

 * The JDBC driver class must be visible to the primordial class loader on the client session and on all executors. This is because Java's DriverManager class does a security check that results in it ignoring all drivers not visible to the primordial class loader when one goes to open a connection. One convenient way to do this is to modify compute_classpath.sh on all worker nodes to include your driver JARs.
 * Some databases, such as H2, convert all names to upper case. You'll need to use upper case to refer to those names in Spark SQL.


# Performance Tuning

For some workloads it is possible to improve performance by either caching data in memory, or by
turning on some experimental options.

## Caching Data In Memory

Spark SQL can cache tables using an in-memory columnar format by calling `spark.catalog.cacheTable("tableName")` or `dataFrame.cache()`.
Then Spark SQL will scan only required columns and will automatically tune compression to minimize
memory usage and GC pressure. You can call `spark.catalog.uncacheTable("tableName")` to remove the table from memory.

Configuration of in-memory caching can be done using the `setConf` method on `SparkSession` or by running
`SET key=value` commands using SQL.

<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.sql.inMemoryColumnarStorage.compressed</code></td>
  <td>true</td>
  <td>
    When set to true Spark SQL will automatically select a compression codec for each column based
    on statistics of the data.
  </td>
</tr>
<tr>
  <td><code>spark.sql.inMemoryColumnarStorage.batchSize</code></td>
  <td>10000</td>
  <td>
    Controls the size of batches for columnar caching. Larger batch sizes can improve memory utilization
    and compression, but risk OOMs when caching data.
  </td>
</tr>

</table>

## Other Configuration Options

The following options can also be used to tune the performance of query execution. It is possible
that these options will be deprecated in future release as more optimizations are performed automatically.

<table class="table">
  <tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
  <tr>
    <td><code>spark.sql.files.maxPartitionBytes</code></td>
    <td>134217728 (128 MB)</td>
    <td>
      The maximum number of bytes to pack into a single partition when reading files.
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.files.openCostInBytes</code></td>
    <td>4194304 (4 MB)</td>
    <td>
      The estimated cost to open a file, measured by the number of bytes could be scanned in the same
      time. This is used when putting multiple files into a partition. It is better to over estimated,
      then the partitions with small files will be faster than partitions with bigger files (which is
      scheduled first).
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.broadcastTimeout</code></td>
    <td>300</td>
    <td>
    <p>
      Timeout in seconds for the broadcast wait time in broadcast joins
    </p>
    </td>
  </tr>
  <tr>
    <td><code>spark.sql.autoBroadcastJoinThreshold</code></td>
    <td>10485760 (10 MB)</td>
    <td>
      Configures the maximum size in bytes for a table that will be broadcast to all worker nodes when
      performing a join. By setting this value to -1 broadcasting can be disabled. Note that currently
      statistics are only supported for Hive Metastore tables where the command
      <code>ANALYZE TABLE &lt;tableName&gt; COMPUTE STATISTICS noscan</code> has been run.
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

# Distributed SQL Engine

Spark SQL can also act as a distributed query engine using its JDBC/ODBC or command-line interface.
In this mode, end-users or applications can interact with Spark SQL directly to run SQL queries,
without the need to write any code.

## Running the Thrift JDBC/ODBC server

The Thrift JDBC/ODBC server implemented here corresponds to the [`HiveServer2`](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2)
in Hive 1.2.1 You can test the JDBC server with the beeline script that comes with either Spark or Hive 1.2.1.

To start the JDBC/ODBC server, run the following in the Spark directory:

    ./sbin/start-thriftserver.sh

This script accepts all `bin/spark-submit` command line options, plus a `--hiveconf` option to
specify Hive properties. You may run `./sbin/start-thriftserver.sh --help` for a complete list of
all available options. By default, the server listens on localhost:10000. You may override this
behaviour via either environment variables, i.e.:

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

Now you can use beeline to test the Thrift JDBC/ODBC server:

    ./bin/beeline

Connect to the JDBC/ODBC server in beeline with:

    beeline> !connect jdbc:hive2://localhost:10000

Beeline will ask you for a username and password. In non-secure mode, simply enter the username on
your machine and a blank password. For secure mode, please follow the instructions given in the
[beeline documentation](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients).

Configuration of Hive is done by placing your `hive-site.xml`, `core-site.xml` and `hdfs-site.xml` files in `conf/`.

You may also use the beeline script that comes with Hive.

Thrift JDBC server also supports sending thrift RPC messages over HTTP transport.
Use the following setting to enable HTTP mode as system property or in `hive-site.xml` file in `conf/`:

    hive.server2.transport.mode - Set this to value: http
    hive.server2.thrift.http.port - HTTP port number to listen on; default is 10001
    hive.server2.http.endpoint - HTTP endpoint; default is cliservice

To test, use beeline to connect to the JDBC/ODBC server in http mode with:

    beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint>


## Running the Spark SQL CLI

The Spark SQL CLI is a convenient tool to run the Hive metastore service in local mode and execute
queries input from the command line. Note that the Spark SQL CLI cannot talk to the Thrift JDBC server.

To start the Spark SQL CLI, run the following in the Spark directory:

    ./bin/spark-sql

Configuration of Hive is done by placing your `hive-site.xml`, `core-site.xml` and `hdfs-site.xml` files in `conf/`.
You may run `./bin/spark-sql --help` for a complete list of all available
options.

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
