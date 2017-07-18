---
layout: global
title: Submitting Applications
---

在  script in Spark的 `bin` 目录中的`spark-submit` 脚本用与在集群上启动应用程序。它可以通过一个统一的接口使用所有 Spark 支持的 [cluster managers](cluster-overview.html#cluster-manager-types)，所以您不需要专门的为每个[cluster managers](cluster-overview.html#cluster-manager-types)配置您的应用程序。

# 打包应用依赖
如果您的代码依赖了其它的项目，为了分发代码到 Spark 集群中您将需要将它们和您的应用程序一起打包。为此，创建一个包含您的代码以及依赖的 assembly jar（或者 “uber” jar）。无论是
[sbt](https://github.com/sbt/sbt-assembly) 还是
[Maven](http://maven.apache.org/plugins/maven-shade-plugin/)
都有 assembly 插件。在创建 assembly jar 时，列出 Spark 和 Hadoop的依赖为`provided`。它们不需要被打包，因为在运行时它们已经被 Cluster Manager 提供了。如果您有一个 assembled jar 您就可以调用 `bin/spark-submit` 脚本（如下所示）来传递您的 jar。

对于 Python 来说，您可以使用 `spark-submit` 的 `--py-files` 参数来添加 `.py`, `.zip` 和 `.egg` 文件以与您的应用程序一起分发。如果您依赖了多个 Python 文件我们推荐将它们打包成一个 `.zip` 或者 `.egg` 文件。


# 用 spark-submit 启动应用

如果用户的应用程序被打包好了，它可以使用 `bin/spark-submit` 脚本来启动。这个脚本负责设置 Spark 和它的依赖的 classpath，并且可以支持 Spark 所支持的不同的 Cluster Manager 以及 deploy mode（部署模式）: 

{% highlight bash %}
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
{% endhighlight %}

一些常用的 options（选项）有 :

* `--class`: 您的应用程序的入口点（例如。 `org.apache.spark.examples.SparkPi`)
* `--master`: 集群的 [master URL](#master-urls)  (例如 `spark://23.195.26.187:7077`)
* `--deploy-mode`: 是在 worker 节点(`cluster`) 上还是在本地作为一个外部的客户端(`client`) 部署您的 driver(默认: `client`) <b> &#8224; </b>
* `--conf`: 按照 key=value 格式任意的 Spark 配置属性。对于包含空格的 value（值）使用引号包 “key=value” 起来。
* `application-jar`: 包括您的应用以及所有依赖的一个打包的 Jar 的路径。该 URL 在您的集群上必须是全局可见的，例如，一个 `hdfs://` path 或者一个 `file://` 在所有节点是可见的。
* `application-arguments`: 传递到您的 main class 的 main 方法的参数，如果有的话。

<b>&#8224;</b> 常见的部署策略是从一台 gateway 机器物理位置与您 worker 在一起的机器（比如，在 standalone EC2 集群中的 Master 节点上）来提交您的应用。在这种设置中， `client` 模式是合适的。在 `client` 模式中，driver 直接运行在一个充当集群 client 的 `spark-submit` 进程内。应用程序的输入和输出直接连到控制台。因此，这个模式特别适合那些设计 REPL（例如，Spark shell）的应用程序。

另外，如果您从一台远离 worker 机器的机器（例如，本地的笔记本电脑上）提交应用程序，通常使用 `cluster` 模式来降低 driver 和 executor 之间的延迟。目前，Standalone 模式不支持 Cluster 模式的 Python 应用。

对于 Python 应用，在 `<application-jar>` 的位置简单的传递一个 `.py` 文件而不是一个 JAR，并且可以用 `--py-files` 添加 Python `.zip`，`.egg` 或者 `.py` 文件到 search path（搜索路径）。

这里有一些选项可用于特定的
[cluster manager](cluster-overview.html#cluster-manager-types) 中。例如， [Spark standalone cluster](spark-standalone.html) 用 `cluster` 部署模式,
您也可以指定 `--supervise` 来确保 driver 在 non-zero exit code 失败时可以自动重启。为了列出所有  `spark-submit`,
可用的选项，用 `--help`. 来运行它。这里是一些常见选项的例子 :

{% highlight bash %}
# Run application locally on 8 cores
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master local[8] \
  /path/to/examples.jar \
  100

# Run on a Spark standalone cluster in client deploy mode
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# Run on a Spark standalone cluster in cluster deploy mode with supervise
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# Run on a YARN cluster
export HADOOP_CONF_DIR=XXX
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \  # can be client for client mode
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar \
  1000

# Run a Python application on a Spark standalone cluster
./bin/spark-submit \
  --master spark://207.184.161.138:7077 \
  examples/src/main/python/pi.py \
  1000

# Run on a Mesos cluster in cluster deploy mode with supervise
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  http://path/to/examples.jar \
  1000

{% endhighlight %}

# Master URLs

传递给 Spark 的 master URL 可以使用下列格式中的一种 : 

<table class="table">
<tr><th>Master URL</th><th>Meaning</th></tr>
<tr><td> <code>local</code> </td><td> 使用一个线程本地运行 Spark（即，没有并行性）。 </td></tr>
<tr><td> <code>local[K]</code> </td><td> 使用 K 个 worker 线程本地运行 Spark（理想情况下，设置这个值的数量为您机器的 core 数量）。 </td></tr>
<tr><td> <code>local[K,F]</code> </td><td> 使用 K 个 worker 线程本地运行 Spark并允许最多失败 F次  (查阅 <a href="configuration.html#scheduling">spark.task.maxFailures</a> 以获取对该变量的解释) </td></tr>
<tr><td> <code>local[*]</code> </td><td> 使用更多的 worker 线程作为逻辑的 core 在您的机器上来本地的运行 Spark。</td></tr>
<tr><td> <code>local[*,F]</code> </td><td> 使用更多的 worker 线程作为逻辑的 core 在您的机器上来本地的运行 Spark并允许最多失败 F次。</td></tr>
<tr><td> <code>spark://HOST:PORT</code> </td><td> 连接至给定的 <a href="spark-standalone.html">Spark standalone
        cluster</a> master. master。该 port（端口）必须有一个作为您的 master 配置来使用，默认是 7077。
</td></tr>
<tr><td> <code>spark://HOST1:PORT1,HOST2:PORT2</code> </td><td> 连接至给定的 <a href="spark-standalone.html#standby-masters-with-zookeeper">Spark standalone
        cluster with standby masters with Zookeeper</a>. 该列表必须包含由zookeeper设置的高可用集群中的所有master主机。该 port（端口）必须有一个作为您的 master 配置来使用，默认是 7077。
</td></tr>
<tr><td> <code>mesos://HOST:PORT</code> </td><td> 连接至给定的 <a href="running-on-mesos.html">Mesos</a> 集群.
        该 port（端口）必须有一个作为您的配置来使用，默认是 5050。或者，对于使用了 ZooKeeper 的 Mesos cluster 来说，使用 <code>mesos://zk://...</code>.
        。使用 <code>--deploy-mode cluster</code>, 来提交，该 HOST:PORT 应该被配置以连接到 <a href="running-on-mesos.html#cluster-mode">MesosClusterDispatcher</a>.
</td></tr>
<tr><td> <code>yarn</code> </td><td> 连接至一个 <a href="running-on-yarn.html"> YARN </a> cluster in
        <code>client</code> or <code>cluster</code> mode 取决于 <code>--deploy-mode</code>.
        的值在 client 或者 cluster 模式中。该 cluster 的位置将根据 <code>HADOOP_CONF_DIR</code>  或者 <code>YARN_CONF_DIR</code> 变量来找到。
</td></tr>
</table>


# 从文件中加载配置

The `spark-submit` script can load default [Spark configuration values](configuration.html) from a
properties file and pass them on to your application. By default it will read options
from `conf/spark-defaults.conf` in the Spark directory. For more detail, see the section on
[loading default configurations](configuration.html#loading-default-configurations).

Loading default Spark configurations this way can obviate the need for certain flags to
`spark-submit`. For instance, if the `spark.master` property is set, you can safely omit the
`--master` flag from `spark-submit`. In general, configuration values explicitly set on a
`SparkConf` take the highest precedence, then flags passed to `spark-submit`, then values in the
defaults file.

If you are ever unclear where configuration options are coming from, you can print out fine-grained
debugging information by running `spark-submit` with the `--verbose` option.

# Advanced Dependency Management
When using `spark-submit`, the application jar along with any jars included with the `--jars` option
will be automatically transferred to the cluster. URLs supplied after `--jars` must be separated by commas. That list is included on the driver and executor classpaths. Directory expansion does not work with `--jars`.

Spark uses the following URL scheme to allow different strategies for disseminating jars:

- **file:** - Absolute paths and `file:/` URIs are served by the driver's HTTP file server, and
  every executor pulls the file from the driver HTTP server.
- **hdfs:**, **http:**, **https:**, **ftp:** - these pull down files and JARs from the URI as expected
- **local:** - a URI starting with local:/ is expected to exist as a local file on each worker node.  This
  means that no network IO will be incurred, and works well for large files/JARs that are pushed to each worker,
  or shared via NFS, GlusterFS, etc.

Note that JARs and files are copied to the working directory for each SparkContext on the executor nodes.
This can use up a significant amount of space over time and will need to be cleaned up. With YARN, cleanup
is handled automatically, and with Spark standalone, automatic cleanup can be configured with the
`spark.worker.cleanup.appDataTtl` property.

Users may also include any other dependencies by supplying a comma-delimited list of Maven coordinates
with `--packages`. All transitive dependencies will be handled when using this command. Additional
repositories (or resolvers in SBT) can be added in a comma-delimited fashion with the flag `--repositories`.
(Note that credentials for password-protected repositories can be supplied in some cases in the repository URI,
such as in `https://user:password@host/...`. Be careful when supplying credentials this way.)
These commands can be used with `pyspark`, `spark-shell`, and `spark-submit` to include Spark Packages.

For Python, the equivalent `--py-files` option can be used to distribute `.egg`, `.zip` and `.py` libraries
to executors.

# More Information

Once you have deployed your application, the [cluster mode overview](cluster-overview.html) describes
the components involved in distributed execution, and how to monitor and debug applications.
