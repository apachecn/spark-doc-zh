---
layout: global
title: 快速入门
description: Spark SPARK_VERSION_SHORT 快速入门教程
---

* This will become a table of contents (this text will be scraped).
{:toc}

本教程提供了如何使用 Spark 的快速入门介绍。首先通过运行 Spark 交互式的 shell（在 Python 或 Scala 中）来介绍 API, 然后展示如何使用 Java , Scala 和 Python 来编写应用程序。

为了继续阅读本指南, 首先从 [Spark 官网](http://spark.apache.org/downloads.html) 下载 Spark 的发行包。因为我们将不使用 HDFS, 所以你可以下载一个任何 Hadoop 版本的软件包。

请注意, 在 Spark 2.0 之前, Spark 的主要编程接口是弹性分布式数据集（RDD）。 在 Spark 2.0 之后, RDD 被 Dataset 替换, 它是像RDD 一样的 strongly-typed（强类型）, 但是在引擎盖下更加优化。 RDD 接口仍然受支持, 您可以在 [RDD 编程指南](rdd-programming-guide.html) 中获得更完整的参考。 但是, 我们强烈建议您切换到使用 Dataset（数据集）, 其性能要更优于 RDD。 请参阅 [SQL 编程指南](sql-programming-guide.html) 获取更多有关 Dataset 的信息。

# 使用 Spark Shell 进行交互式分析

## 基础

Spark shell 提供了一种来学习该 API 比较简单的方式, 以及一个强大的来分析数据交互的工具。在 Scala（运行于 Java 虚拟机之上, 并能很好的调用已存在的 Java 类库）或者 Python 中它是可用的。通过在 Spark 目录中运行以下的命令来启动它:

<div class="codetabs">
<div data-lang="scala" markdown="1">

    ./bin/spark-shell

Spark 的主要抽象是一个称为 Dataset 的分布式的 item 集合。Datasets 可以从 Hadoop 的 InputFormats（例如 HDFS文件）或者通过其它的 Datasets 转换来创建。让我们从 Spark 源目录中的 README 文件来创建一个新的 Dataset: 

{% highlight scala %}
scala> val textFile = spark.read.textFile("README.md")
textFile: org.apache.spark.sql.Dataset[String] = [value: string]
{% endhighlight %}

您可以直接从 Dataset 中获取 values（值）, 通过调用一些 actions（动作）, 或者 transform（转换）Dataset 以获得一个新的。更多细节, 请参阅 _[API doc](api/scala/index.html#org.apache.spark.sql.Dataset)_。

{% highlight scala %}
scala> textFile.count() // Number of items in this Dataset
res0: Long = 126 // May be different from yours as README.md will change over time, similar to other outputs

scala> textFile.first() // First item in this Dataset
res1: String = # Apache Spark
{% endhighlight %}

现在让我们 transform 这个 Dataset 以获得一个新的。我们调用 `filter` 以返回一个新的 Dataset, 它是文件中的 items 的一个子集。

{% highlight scala %}
scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
linesWithSpark: org.apache.spark.sql.Dataset[String] = [value: string]
{% endhighlight %}

我们可以链式操作 transformation（转换）和 action（动作）:

{% highlight scala %}
scala> textFile.filter(line => line.contains("Spark")).count() // How many lines contain "Spark"?
res3: Long = 15
{% endhighlight %}

</div>
<div data-lang="python" markdown="1">

    ./bin/pyspark

Spark's primary abstraction is a distributed collection of items called a Dataset. Datasets can be created from Hadoop InputFormats (such as HDFS files) or by transforming other Datasets. Due to Python's dynamic nature, we don't need the Dataset to be strongly-typed in Python. As a result, all Datasets in Python are Dataset[Row], and we call it `DataFrame` to be consistent with the data frame concept in Pandas and R. Let's make a new DataFrame from the text of the README file in the Spark source directory:

{% highlight python %}
>>> textFile = spark.read.text("README.md")
{% endhighlight %}

You can get values from DataFrame directly, by calling some actions, or transform the DataFrame to get a new one. For more details, please read the _[API doc](api/python/index.html#pyspark.sql.DataFrame)_.

{% highlight python %}
>>> textFile.count()  # Number of rows in this DataFrame
126

>>> textFile.first()  # First row in this DataFrame
Row(value=u'# Apache Spark')
{% endhighlight %}

Now let's transform this DataFrame to a new one. We call `filter` to return a new DataFrame with a subset of the lines in the file.

{% highlight python %}
>>> linesWithSpark = textFile.filter(textFile.value.contains("Spark"))
{% endhighlight %}

We can chain together transformations and actions:

{% highlight python %}
>>> textFile.filter(textFile.value.contains("Spark")).count()  # How many lines contain "Spark"?
15
{% endhighlight %}

</div>
</div>


## Dataset 上的更多操作
Dataset actions（操作）和 transformations（转换）可以用于更复杂的计算。例如, 统计出现次数最多的单词 : 

<div class="codetabs">
<div data-lang="scala" markdown="1">

{% highlight scala %}
scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
res4: Long = 15
{% endhighlight %}

第一个 map 操作创建一个新的 Dataset, 将一行数据 map 为一个整型值。在 Dataset 上调用 `reduce` 来找到最大的行计数。参数 `map` 与 `reduce` 是 Scala 函数（closures）, 并且可以使用 Scala/Java 库的任何语言特性。例如, 我们可以很容易地调用函数声明, 我们将定义一个 max 函数来使代码更易于理解 : 

{% highlight scala %}
scala> import java.lang.Math
import java.lang.Math

scala> textFile.map(line => line.split(" ").size).reduce((a, b) => Math.max(a, b))
res5: Int = 15
{% endhighlight %}

一种常见的数据流模式是被 Hadoop 所推广的 MapReduce。Spark 可以很容易实现 MapReduce: 

{% highlight scala %}
scala> val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]
{% endhighlight %}

在这里, 我们调用了 `flatMap` 以 transform 一个 lines 的 Dataset 为一个 words 的 Dataset, 然后结合 `groupByKey` 和 `count` 来计算文件中每个单词的 counts 作为一个 (String, Long) 的 Dataset pairs。要在 shell 中收集 word counts, 我们可以调用 `collect`:

{% highlight scala %}
scala> wordCounts.collect()
res6: Array[(String, Int)] = Array((means,1), (under,2), (this,3), (Because,1), (Python,2), (agree,1), (cluster.,1), ...)
{% endhighlight %}

</div>
<div data-lang="python" markdown="1">

{% highlight python %}
>>> from pyspark.sql.functions import *
>>> textFile.select(size(split(textFile.value, "\s+")).name("numWords")).agg(max(col("numWords"))).collect()
[Row(max(numWords)=15)]
{% endhighlight %}

This first maps a line to an integer value and aliases it as "numWords", creating a new DataFrame. `agg` is called on that DataFrame to find the largest word count. The arguments to `select` and `agg` are both _[Column](api/python/index.html#pyspark.sql.Column)_, we can use `df.colName` to get a column from a DataFrame. We can also import pyspark.sql.functions, which provides a lot of convenient functions to build a new Column from an old one.

One common data flow pattern is MapReduce, as popularized by Hadoop. Spark can implement MapReduce flows easily:

{% highlight python %}
>>> wordCounts = textFile.select(explode(split(textFile.value, "\s+")).as("word")).groupBy("word").count()
{% endhighlight %}

Here, we use the `explode` function in `select`, to transfrom a Dataset of lines to a Dataset of words, and then combine `groupBy` and `count` to compute the per-word counts in the file as a DataFrame of 2 columns: "word" and "count". To collect the word counts in our shell, we can call `collect`:

{% highlight python %}
>>> wordCounts.collect()
[Row(word=u'online', count=1), Row(word=u'graphs', count=1), ...]
{% endhighlight %}

</div>
</div>

## 缓存
Spark 还支持 Pulling（拉取）数据集到一个群集范围的内存缓存中。例如当查询一个小的 "hot" 数据集或运行一个像 PageRANK 这样的迭代算法时, 在数据被重复访问时是非常高效的。举一个简单的例子, 让我们标记我们的 `linesWithSpark` 数据集到缓存中:


<div class="codetabs">
<div data-lang="scala" markdown="1">

{% highlight scala %}
scala> linesWithSpark.cache()
res7: linesWithSpark.type = [value: string]

scala> linesWithSpark.count()
res8: Long = 15

scala> linesWithSpark.count()
res9: Long = 15
{% endhighlight %}

使用 Spark 来探索和缓存一个 100 行的文本文件看起来比较愚蠢。有趣的是, 即使在他们跨越几十或者几百个节点时, 这些相同的函数也可以用于非常大的数据集。您也可以像 [编程指南](rdd-programming-guide.html#using-the-shell). 中描述的一样通过连接 `bin/spark-shell` 到集群中, 使用交互式的方式来做这件事情。


</div>
<div data-lang="python" markdown="1">

{% highlight python %}
>>> linesWithSpark.cache()

>>> linesWithSpark.count()
15

>>> linesWithSpark.count()
15
{% endhighlight %}

It may seem silly to use Spark to explore and cache a 100-line text file. The interesting part is
that these same functions can be used on very large data sets, even when they are striped across
tens or hundreds of nodes. You can also do this interactively by connecting `bin/pyspark` to
a cluster, as described in the [RDD programming guide](rdd-programming-guide.html#using-the-shell).

</div>
</div>

# 独立的应用
假设我们希望使用 Spark API 来创建一个独立的应用程序。我们在 Scala（SBT）, Java（Maven）和 Python 中练习一个简单应用程序。

<div class="codetabs">
<div data-lang="scala" markdown="1">

我们将在 Scala 中创建一个非常简单的 Spark 应用程序 - 很简单的, 事实上, 它名为 `SimpleApp.scala`:

{% highlight scala %}
/* SimpleApp.scala */
import org.apache.spark.sql.SparkSession

object SimpleApp {
  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
    val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
    val logData = spark.read.textFile(logFile).cache()
    val numAs = logData.filter(line => line.contains("a")).count()
    val numBs = logData.filter(line => line.contains("b")).count()
    println(s"Lines with a: $numAs, Lines with b: $numBs")
    spark.stop()
  }
}
{% endhighlight %}

注意, 这个应用程序我们应该定义一个 `main()` 方法而不是去扩展 `scala.App`。使用 `scala.App` 的子类可能不会正常运行。

该程序仅仅统计了 Spark README 文件中每一行包含 'a' 的数量和包含 'b' 的数量。注意, 您需要将 YOUR_SPARK_HOME 替换为您 Spark 安装的位置。不像先前使用 spark shell 操作的示例, 它们初始化了它们自己的 SparkContext, 我们初始化了一个 SparkContext 作为应用程序的一部分。


我们调用 `SparkSession.builder` 以构造一个 [[SparkSession]], 然后设置 application name（应用名称）, 最终调用 `getOrCreate` 以获得 [[SparkSession]] 实例。

我们的应用依赖了 Spark API, 所以我们将包含一个名为 `build.sbt` 的 sbt 配置文件, 它描述了 Spark 的依赖。该文件也会添加一个 Spark 依赖的 repository:

{% highlight scala %}
name := "Simple Project"

version := "1.0"

scalaVersion := "{{site.SCALA_VERSION}}"

libraryDependencies += "org.apache.spark" %% "spark-sql" % "{{site.SPARK_VERSION}}"
{% endhighlight %}

为了让 sbt 正常的运行, 我们需要根据经典的目录结构来布局 `SimpleApp.scala` 和 `build.sbt` 文件。在成功后, 我们可以创建一个包含应用程序代码的 JAR 包, 然后使用 `spark-submit` 脚本来运行我们的程序。


{% highlight bash %}
# Your directory layout should look like this
$ find .
.
./build.sbt
./src
./src/main
./src/main/scala
./src/main/scala/SimpleApp.scala

# Package a jar containing your application
$ sbt package
...
[info] Packaging {..}/{..}/target/scala-{{site.SCALA_BINARY_VERSION}}/simple-project_{{site.SCALA_BINARY_VERSION}}-1.0.jar

# Use spark-submit to run your application
$ YOUR_SPARK_HOME/bin/spark-submit \
  --class "SimpleApp" \
  --master local[4] \
  target/scala-{{site.SCALA_BINARY_VERSION}}/simple-project_{{site.SCALA_BINARY_VERSION}}-1.0.jar
...
Lines with a: 46, Lines with b: 23
{% endhighlight %}

</div>
<div data-lang="java" markdown="1">
This example will use Maven to compile an application JAR, but any similar build system will work.

We'll create a very simple Spark application, `SimpleApp.java`:

{% highlight java %}
/* SimpleApp.java */
import org.apache.spark.sql.SparkSession;

public class SimpleApp {
  public static void main(String[] args) {
    String logFile = "YOUR_SPARK_HOME/README.md"; // Should be some file on your system
    SparkSession spark = SparkSession.builder().appName("Simple Application").getOrCreate();
    Dataset<String> logData = spark.read.textFile(logFile).cache();

    long numAs = logData.filter(s -> s.contains("a")).count();
    long numBs = logData.filter(s -> s.contains("b")).count();

    System.out.println("Lines with a: " + numAs + ", lines with b: " + numBs);

    spark.stop();
  }
}
{% endhighlight %}

This program just counts the number of lines containing 'a' and the number containing 'b' in the
Spark README. Note that you'll need to replace YOUR_SPARK_HOME with the location where Spark is
installed. Unlike the earlier examples with the Spark shell, which initializes its own SparkSession,
we initialize a SparkSession as part of the program.

To build the program, we also write a Maven `pom.xml` file that lists Spark as a dependency.
Note that Spark artifacts are tagged with a Scala version.

{% highlight xml %}
<project>
  <groupId>edu.berkeley</groupId>
  <artifactId>simple-project</artifactId>
  <modelVersion>4.0.0</modelVersion>
  <name>Simple Project</name>
  <packaging>jar</packaging>
  <version>1.0</version>
  <dependencies>
    <dependency> <!-- Spark dependency -->
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-sql_{{site.SCALA_BINARY_VERSION}}</artifactId>
      <version>{{site.SPARK_VERSION}}</version>
    </dependency>
  </dependencies>
</project>
{% endhighlight %}

We lay out these files according to the canonical Maven directory structure:
{% highlight bash %}
$ find .
./pom.xml
./src
./src/main
./src/main/java
./src/main/java/SimpleApp.java
{% endhighlight %}

Now, we can package the application using Maven and execute it with `./bin/spark-submit`.

{% highlight bash %}
# Package a JAR containing your application
$ mvn package
...
[INFO] Building jar: {..}/{..}/target/simple-project-1.0.jar

# Use spark-submit to run your application
$ YOUR_SPARK_HOME/bin/spark-submit \
  --class "SimpleApp" \
  --master local[4] \
  target/simple-project-1.0.jar
...
Lines with a: 46, Lines with b: 23
{% endhighlight %}

</div>
<div data-lang="python" markdown="1">

Now we will show how to write an application using the Python API (PySpark).

As an example, we'll create a simple Spark application, `SimpleApp.py`:

{% highlight python %}
"""SimpleApp.py"""
from pyspark.sql import SparkSession

logFile = "YOUR_SPARK_HOME/README.md"  # Should be some file on your system
spark = SparkSession.builder().appName(appName).master(master).getOrCreate()
logData = spark.read.text(logFile).cache()

numAs = logData.filter(logData.value.contains('a')).count()
numBs = logData.filter(logData.value.contains('b')).count()

print("Lines with a: %i, lines with b: %i" % (numAs, numBs))

spark.stop()
{% endhighlight %}


This program just counts the number of lines containing 'a' and the number containing 'b' in a
text file.
Note that you'll need to replace YOUR_SPARK_HOME with the location where Spark is installed.
As with the Scala and Java examples, we use a SparkSession to create Datasets.
For applications that use custom classes or third-party libraries, we can also add code
dependencies to `spark-submit` through its `--py-files` argument by packaging them into a
.zip file (see `spark-submit --help` for details).
`SimpleApp` is simple enough that we do not need to specify any code dependencies.

We can run this application using the `bin/spark-submit` script:

{% highlight bash %}
# Use spark-submit to run your application
$ YOUR_SPARK_HOME/bin/spark-submit \
  --master local[4] \
  SimpleApp.py
...
Lines with a: 46, Lines with b: 23
{% endhighlight %}

</div>
</div>

# 快速跳转
恭喜您成功的运行了您的第一个 Spark 应用程序！

* 更多 API 的深入概述, 从 [RDD programming guide](rdd-programming-guide.html) 和 [SQL programming guide](sql-programming-guide.html) 这里开始, 或者看看 "编程指南" 菜单中的其它组件。
* 为了在集群上运行应用程序, 请前往 [deployment overview](cluster-overview.html).
* 最后, 在 Spark 的 `examples` 目录中包含了一些
([Scala]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/scala/org/apache/spark/examples),
 [Java]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/java/org/apache/spark/examples),
 [Python]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/python),
 [R]({{site.SPARK_GITHUB_URL}}/tree/master/examples/src/main/r)) 示例。您可以按照如下方式来运行它们:

{% highlight bash %}
# 针对 Scala 和 Java, 使用 run-example:
./bin/run-example SparkPi

# 针对 Python 示例, 直接使用 spark-submit:
./bin/spark-submit examples/src/main/python/pi.py

# 针对 R 示例, 直接使用 spark-submit:

./bin/spark-submit examples/src/main/r/dataframe.R
{% endhighlight %}
