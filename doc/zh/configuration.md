---
layout: global
displayTitle: Spark 配置
title: 配置
---
* This will become a table of contents (this text will be scraped).
{:toc}

Spark 提供了三个位置来配置系统:

* [Spark 属性](#spark-properties) 控制着大多数应用参数, 并且可以通过使用一个 [SparkConf](api/scala/index.html#org.apache.spark.SparkConf) 对象来设置, 或者通过 Java 系统属性来设置. 
* [环境变量](#environment-variables) 可用于在每个节点上通过 `conf/spark-env.sh` 脚本来设置每台机器设置, 例如 IP 地址. 
* [Logging](#configuring-logging) 可以通过 `log4j.properties` 来设置. 

# Spark 属性

Spark 属性控制大多数应用程序设置, 并为每个应用程序单独配置.  这些属性可以直接在 [SparkConf](api/scala/index.html#org.apache.spark.SparkConf) 上设置并传递给您的 `SparkContext` .  `SparkConf` 可以让你配置一些常见的属性（例如 master URL 和应用程序名称）, 以及通过 `set()` 方法来配置任意 key-value pairs （键值对）.  例如, 我们可以使用两个线程初始化一个应用程序, 如下所示：

请注意, 我们运行 local[2] , 意思是两个线程 - 代表 "最小" 并行性, 这可以帮助检测在只存在于分布式环境中运行时的错误. 

{% highlight scala %}
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
{% endhighlight %}

注意, 本地模式下, 我们可以使用多个线程, 而且在像 Spark Streaming 这样的场景下, 我们可能需要多个线程来防止任一类型的类似 starvation issues （线程饿死） 这样的问题. 
配置时间段的属性应该写明时间单位, 如下格式都是可接受的:  

    25ms (milliseconds)
    5s (seconds)
    10m or 10min (minutes)
    3h (hours)
    5d (days)
    1y (years)


指定 byte size （字节大小）的属性应该写明单位. 
如下格式都是可接受的：

    1b (bytes)
    1k or 1kb (kibibytes = 1024 bytes)
    1m or 1mb (mebibytes = 1024 kibibytes)
    1g or 1gb (gibibytes = 1024 mebibytes)
    1t or 1tb (tebibytes = 1024 gibibytes)
    1p or 1pb (pebibytes = 1024 tebibytes)

## 动态加载 Spark 属性

在某些场景下, 你可能想避免将属性值写死在 SparkConf 中. 例如, 你可能希望在同一个应用上使用不同的 master 或不同的内存总量.  Spark 允许你简单地创建一个空的 conf : 

{% highlight scala %}
val sc = new SparkContext(new SparkConf())
{% endhighlight %}

然后在运行时设置这些属性 : 
{% highlight bash %}
./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
{% endhighlight %}

Spark shell 和 [`spark-submit`](submitting-applications.html) 工具支持两种动态加载配置的方法. 第一种, 通过命令行选项, 如 : 上面提到的 `--master` .  `spark-submit` 可以使用 `--conf` flag 来接受任何 Spark 属性标志, 但对于启动 Spark 应用程序的属性使用 special flags （特殊标志）.  运行 `./bin/spark-submit --help` 可以展示这些选项的完整列表. 

`bin/spark-submit` 也支持从 `conf/spark-defaults.conf` 中读取配置选项, 其中每行由一个 key （键）和一个由 whitespace （空格）分隔的 value （值）组成, 如下:

    spark.master            spark://5.6.7.8:7077
    spark.executor.memory   4g
    spark.eventLog.enabled  true
    spark.serializer        org.apache.spark.serializer.KryoSerializer

指定为 flags （标志）或属性文件中的任何值都将传递给应用程序并与通过 SparkConf 指定的那些值 merge （合并）.  属性直接在 SparkConf 上设置采取最高优先级, 然后 flags （标志）传递给 `spark-submit` 或 `spark-shell` , 然后选项在 `spark-defaults.conf` 文件中.  自从 Spark 版本的早些时候, 一些 configuration keys （配置键）已被重命名 ; 在这种情况下, 旧的 key names （键名）仍然被接受, 但要比较新的 key 优先级都要低一些. 

## 查看 Spark 属性

在应用程序的 web UI `http://<driver>:4040` 中,  "Environment" tab （“环境”选项卡）中列出了 Spark 的属性. 这是一个检查您是否正确设置了您的属性的一个非常有用的地方. 注意, 只有显示地通过 `spark-defaults.conf` ,  `SparkConf` 或者命令行设置的值将会出现. 对于所有其他配置属性, 您可以认为使用的都是默认值. 

## 可用属性

大多数控制 internal settings （内部设置） 的属性具有合理的默认值. 一些常见的选项是：

### 应用程序属性

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.app.name</code></td>
  <td>(none)</td>
  <td>
    Spark 应用的名字。会在 SparkUI 和日志中出现。
  </td>
</tr>
<tr>
  <td><code>spark.driver.cores</code></td>
  <td>1</td>
  <td>
    在 cluster 模式下，用几个 core 运行 driver 进程。
  </td>
</tr>
<tr>
  <td><code>spark.driver.maxResultSize</code></td>
  <td>1g</td>
  <td>
    Spark action 算子返回的结果集的最大数量。至少要 1M，可以设为 0 表示无限制。如果结果超过这一大小，Spark job 会直接中断退出。但是，设得过高有可能导致 driver 出现 out-of-memory 异常（取决于 spark.driver.memory 设置，以及驱动器 JVM 的内存限制）。设一个合理的值，以避免 driver 出现 out-of-memory 异常。
  </td>
</tr>
<tr>
  <td><code>spark.driver.memory</code></td>
  <td>1g</td>
  <td>
    driver进程可以使用的内存总量（如 : <code>1g</code>, <code>2g</code>）。注意，在 client 模式下，这个配置不能在 SparkConf 中直接设置，应为在那个时候 driver 进程的 JVM 已经启动了。因此需要在命令行里用 --driver-memory 选项 或者在默认属性配置文件里设置。
  </td>
</tr>
<tr>
  <td><code>spark.executor.memory</code></td>
  <td>1g</td>
  <td>
    每个 executor 进程使用的内存总量（如，<code>2g</code>, <code>8g</code>）。
    Amount of memory to use per executor process (e.g. <code>2g</code>, <code>8g</code>).
  </td>
</tr>
<tr>
  <td><code>spark.extraListeners</code></td>
  <td>(none)</td>
  <td>
    逗号分隔的实现 <code>SparkListener</code> 接口的类名列表；初始化 SparkContext 时，这些类的实例会被创建出来，并且注册到 Spark 的监听器上。如果这些类有一个接受 SparkConf 作为唯一参数的构造函数，那么这个构造函数会被调用；否则，就调用无参构造函数。如果没有合适的构造函数，SparkContext 创建的时候会抛异常。
  </td>
</tr>
<tr>
  <td><code>spark.local.dir</code></td>
  <td>/tmp</td>
  <td>
    Spark 的”草稿“目录，包括 map 输出的临时文件以及 RDD 存在磁盘上的数据。这个目录最好在本地文件系统中。这个配置可以接受一个以逗号分隔的多个挂载到不同磁盘上的目录列表。注意 : Spark-1.0 及以后版本中，这个属性会被 cluster manager 设置的环境变量覆盖 : SPARK_LOCAL_DIRS（Standalone，Mesos）或者 LOCAL_DIRS（YARN）。
  </td>
</tr>
<tr>
  <td><code>spark.logConf</code></td>
  <td>false</td>
  <td>
    SparkContext 启动时是否把生效的 SparkConf 属性以 INFO 日志打印到日志里。
  </td>
</tr>
<tr>
  <td><code>spark.master</code></td>
  <td>(none)</td>
  <td>
    要连接的 cluster manager。参考 <a href="submitting-applications.html#master-urls">Cluster Manager</a> 类型。
  </td>
</tr>
<tr>
  <td><code>spark.submit.deployMode</code></td>
  <td>(none)</td>
  <td>
    Spark driver 程序的部署模式，可以是 "client" 或 "cluster"，意味着部署 dirver 程序本地（"client"）或者远程（"cluster"）在 Spark 集群的其中一个节点上。
  </td>
</tr>
<tr>
  <td><code>spark.log.callerContext</code></td>
  <td>(none)</td>
  <td>
    Application information that will be written into Yarn RM log/HDFS audit log when running on Yarn/HDFS.
    Its length depends on the Hadoop configuration <code>hadoop.caller.context.max.size</code>. It should be concise,
    and typically can have up to 50 characters.
  </td>
</tr>
<tr>
  <td><code>spark.driver.supervise</code></td>
  <td>false</td>
  <td>
    If true, restarts the driver automatically if it fails with a non-zero exit status.
    Only has effect in Spark standalone mode or Mesos cluster deploy mode.
  </td>
</tr>
</table>

Apart from these, the following properties are also available, and may be useful in some situations:

### 运行环境

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.driver.extraClassPath</code></td>
  <td>(none)</td>
  <td>
    额外的classpath条目需预先添加到驱动程序 classpath中。
注意 : 在客户端模式下，这一套配置不能通过 SparkConf 直接在应用在应用程序中，因为 JVM 驱动已经启用了。相反，请在配置文件中通过设置 --driver-class-path 选项或者选择默认属性。
  </td>
</tr>
<tr>
  <td><code>spark.driver.extraJavaOptions</code></td>
  <td>(none)</td>
  <td>
    一些额外的 JVM 属性传递给驱动。例如，GC 设置或其他日志方面设置。注意，设置最大堆大小（-Xmx）是不合法的。最大堆大小设置可以通过在集群模式下设置 <code>spark.driver.memory</code> 选项，并且可以通过<code>--driver-memory</code> 在客户端模式设置。<br /><em>注意:</em> 在客户端模式下，这一套配置不能通过 <code>SparkConf</code> 直接应用在应用程序中，因为 JVM 驱动已经启用了。相反，请在配置文件中通过设置 <code>--driver-java-options</code> 选项或者选择默认属性。
  </td>
</tr>
<tr>
  <td><code>spark.driver.extraLibraryPath</code></td>
  <td>(none)</td>
  <td>
    当启动 JVM 驱动程序时设置一个额外的库路径。<br /><em>注意:</em> 在客户端模式下，这一套配置不能通过 <code>SparkConf</code> 直接在应用在应用程序中，因为 JVM 驱动已经启用了。相反，请在配置文件中通过设置 <code>--driver-library-path</code> 选项或者选择默认属性。
  </td>
</tr>
<tr>
  <td><code>spark.driver.userClassPathFirst</code></td>
  <td>false</td>
  <td>
    （实验）在驱动程序加载类库时，用户添加的 Jar 包是否优先于 Spark 自身的 Jar 包。这个特性可以用来缓解冲突引发的依赖性和用户依赖。目前只是实验功能。这是仅用于集群模式。
  </td>
</tr>
<tr>
  <td><code>spark.executor.extraClassPath</code></td>
  <td>(none)</td>
  <td>
    额外的类路径要预先考虑到 executor 的 classpath。这主要是为与旧版本的 Spark 向后兼容。用户通常不应该需要设置这个选项。
  </td>
</tr>
<tr>
  <td><code>spark.executor.extraJavaOptions</code></td>
  <td>(none)</td>
  <td>
    一些额外的 JVM 属性传递给 executor。例如，GC 设置或其他日志方面设置。注意，设置最大堆大小（-Xmx）是不合法的。Spark 应该使用 SparkConf 对象或 Spark 脚本中使用的 spark-defaults.conf 文件中设置。最大堆大小设置可以在 spark.executor.memory 进行设置。
  </td>
</tr>
<tr>
  <td><code>spark.executor.extraLibraryPath</code></td>
  <td>(none)</td>
  <td>
     当启动 JVM 的可执行程序时设置额外的类库路径。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.maxRetainedFiles</code></td>
  <td>(none)</td>
  <td>
    最新回滚的日志文件将被系统保留。旧的日志文件将被删除。默认情况下禁用。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.enableCompression</code></td>
  <td>false</td>
  <td>
    Enable executor log compression. If it is enabled, the rolled executor logs will be compressed.
    Disabled by default.
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.maxSize</code></td>
  <td>(none)</td>
  <td>
    设置最大文件的大小,以字节为单位日志将被回滚。默认禁用。见 <code>spark.executor.logs.rolling.maxRetainedFiles</code> 旧日志的自动清洗。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.strategy</code></td>
  <td>(none)</td>
  <td>
    设置 executor 日志的回滚策略。它可以被设置为 “时间”（基于时间的回滚）或 “大小”（基于大小的回滚）。对于 “时间”，使用 <code>spark.executor.logs.rolling.time.interval</code> 设置回滚间隔。用 <code>spark.executor.logs.rolling.maxSize</code> 设置最大文件大小回滚。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.time.interval</code></td>
  <td>daily</td>
  <td>
    设定的时间间隔，executor 日志将回滚。默认情况下是禁用的。有效值是<code>每天</code>，<code>每小时</code>，<code>每分钟</code>或任何时间间隔在几秒钟内。见 <code>spark.executor.logs.rolling.maxRetainedFiles</code> 旧日志的自动清洗。
  </td>
</tr>
<tr>
  <td><code>spark.executor.userClassPathFirst</code></td>
  <td>false</td>
  <td>
    （实验）与 <code>spark.driver.userClassPathFirst</code> 相同的功能，但适用于执行程序的实例。
  </td>
</tr>
<tr>
  <td><code>spark.executorEnv.[EnvironmentVariableName]</code></td>
  <td>(none)</td>
  <td>
    通过添加指定的环境变量 <code>EnvironmentVariableName</code> 给 executor 进程。用户可以设置多个环境变量。
  </td>
</tr>
<tr>
  <td><code>spark.redaction.regex</code></td>
  <td>(?i)secret|password</td>
  <td>
    Regex to decide which Spark configuration properties and environment variables in driver and
    executor environments contain sensitive information. When this regex matches a property key or
    value, the value is redacted from the environment UI and various logs like YARN and event logs.
  </td>
</tr>
<tr>
  <td><code>spark.python.profile</code></td>
  <td>false</td>
  <td>
    启用在 python 中的 profile。结果将由 <code>sc.show_profiles()</code> 显示,或者它将会在驱动程序退出后显示。它还可以通过 <code>sc.dump_profiles(path)</code> dump 到磁盘。如果一些 profile 文件的结果已经显示，那么它们将不会再驱动程序退出后再次显示。默认情况下，<code>pyspark.profiler.BasicProfiler</code> 将被使用，但这可以通过传递一个 profile 类作为一个参数到 <code>SparkContext</code> 中进行覆盖。
  </td>
</tr>
<tr>
  <td><code>spark.python.profile.dump</code></td>
  <td>(none)</td>
  <td>
    这个目录是在驱动程序退出后，proflie 文件 dump 到磁盘中的文件目录。结果将为每一个 RDD dump 为分片文件。它们可以通过 ptats.Stats() 加载。如果指定，profile 结果将不会自动显示。
  </td>
</tr>
<tr>
  <td><code>spark.python.worker.memory</code></td>
  <td>512m</td>
  <td>
    在聚合期间，每个python工作进程使用的内存量，与JVM内存条（例如：<code>512m</code>, <code>2g</code>）格式相同。如果在聚合过程中使用的内存高于此数量，则会将数据溢出到磁盘中。
  </td>
</tr>
<tr>
  <td><code>spark.python.worker.reuse</code></td>
  <td>true</td>
  <td>
    重用 python worker。如果为 true，它将使用固定数量的 worker 数量。不需要为每一个任务分配 python 进程。如果是大型的这将是非常有用。
  </td>
</tr>
<tr>
  <td><code>spark.files</code></td>
  <td></td>
  <td>
    Comma-separated list of files to be placed in the working directory of each executor.
  </td>
</tr>
<tr>
  <td><code>spark.submit.pyFiles</code></td>
  <td></td>
  <td>
    Comma-separated list of .zip, .egg, or .py files to place on the PYTHONPATH for Python apps.
  </td>
</tr>
<tr>
  <td><code>spark.jars</code></td>
  <td></td>
  <td>
    Comma-separated list of local jars to include on the driver and executor classpaths.
  </td>
</tr>
<tr>
  <td><code>spark.jars.packages</code></td>
  <td></td>
  <td>
    Comma-separated list of Maven coordinates of jars to include on the driver and executor
    classpaths. The coordinates should be groupId:artifactId:version. If <code>spark.jars.ivySettings</code>
    is given artifacts will be resolved according to the configuration in the file, otherwise artifacts
    will be searched for in the local maven repo, then maven central and finally any additional remote
    repositories given by the command-line option <code>--repositories</code>. For more details, see
    <a href="submitting-applications.html#advanced-dependency-management">Advanced Dependency Management</a>.
  </td>
</tr>
<tr>
  <td><code>spark.jars.excludes</code></td>
  <td></td>
  <td>
    Comma-separated list of groupId:artifactId, to exclude while resolving the dependencies
    provided in <code>spark.jars.packages</code> to avoid dependency conflicts.
  </td>
</tr>
<tr>
  <td><code>spark.jars.ivy</code></td>
  <td></td>
  <td>
    Path to specify the Ivy user directory, used for the local Ivy cache and package files from
    <code>spark.jars.packages</code>. This will override the Ivy property <code>ivy.default.ivy.user.dir</code>
    which defaults to ~/.ivy2.
  </td>
</tr>
<tr>
  <td><code>spark.jars.ivySettings</code></td>
  <td></td>
  <td>
    Path to an Ivy settings file to customize resolution of jars specified using <code>spark.jars.packages</code>
    instead of the built-in defaults, such as maven central. Additional repositories given by the command-line
    option <code>--repositories</code> will also be included. Useful for allowing Spark to resolve artifacts from behind
    a firewall e.g. via an in-house artifact server like Artifactory. Details on the settings file format can be
    found at http://ant.apache.org/ivy/history/latest-milestone/settings.html
  </td>
</tr>
<tr>
  <td><code>spark.pyspark.driver.python</code></td>
  <td></td>
  <td>
    Python binary executable to use for PySpark in driver.
    (default is <code>spark.pyspark.python</code>)
  </td>
</tr>
<tr>
  <td><code>spark.pyspark.python</code></td>
  <td></td>
  <td>
    Python binary executable to use for PySpark in both driver and executors.
  </td>
</tr>
</table>

### Shuffle Behavior （Shuffle 行为）

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.reducer.maxSizeInFlight</code></td>
  <td>48m</td>
  <td>
    从每个 Reduce 任务中并行的 fetch 数据的最大大小。因为每个输出都要求我们创建一个缓冲区，这代表要为每一个 Reduce 任务分配一个固定大小的内存。除非内存足够大否则尽量设置小一点。
  </td>
</tr>
<tr>
  <td><code>spark.reducer.maxReqsInFlight</code></td>
  <td>Int.MaxValue</td>
  <td>
    在集群节点上，这个配置限制了远程 fetch 数据块的连接数目。当集群中的主机数量的增加时候，这可能导致大量的到一个或多个节点的主动连接，导致负载过多而失败。通过限制获取请求的数量，可以缓解这种情况。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.compress</code></td>
  <td>true</td>
  <td>
    是否要对 map 输出的文件进行压缩。默认为 true，使用 <code>spark.io.compression.codec</code>。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.file.buffer</code></td>
  <td>32k</td>
  <td>
    每个 shuffle 文件输出流的内存大小。这些缓冲区的数量减少了磁盘寻道和系统调用创建的 shuffle 文件。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.maxRetries</code></td>
  <td>3</td>
  <td>
    （仅适用于 Netty）如果设置了非 0 值，与 IO 异常相关失败的 fetch 将自动重试。在遇到长时间的 GC 问题或者瞬态网络连接问题时候，这种重试有助于大量 shuffle 的稳定性。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.numConnectionsPerPeer</code></td>
  <td>1</td>
  <td>
    （仅Netty）重新使用主机之间的连接，以减少大型集群的连接建立。 对于具有许多硬盘和少量主机的群集，这可能导致并发性不足以使所有磁盘饱和，因此用户可考虑增加此值。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.preferDirectBufs</code></td>
  <td>true</td>
  <td>
    （仅适用于 Netty）堆缓冲区用于减少在 shuffle 和缓存块传输中的垃圾回收。对于严格限制的堆内存环境中，用户可能希望把这个设置关闭，以强制Netty的所有分配都在堆上。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.retryWait</code></td>
  <td>5s</td>
  <td>
    （仅适用于 Netty）fetch 重试的等待时长。默认 15s。计算公式是 <code>maxRetries * retryWait</code>。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.service.enabled</code></td>
  <td>false</td>
  <td>
    启用外部随机播放服务。 此服务保留由执行者编写的随机播放文件，以便可以安全地删除执行程序。 如果<code>spark.dynamicAllocation.enabled</code> 为 "true"，则必须启用此功能。 必须设置外部随机播放服务才能启用它。 有关详细信息，请参阅 <a href="job-scheduling.html#configuration-and-setup">动态分配配置和设置文档</a>。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.service.port</code></td>
  <td>7337</td>
  <td>
    外部 shuffle 的运行端口。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.service.index.cache.entries</code></td>
  <td>1024</td>
  <td>
    Max number of entries to keep in the index cache of the shuffle service.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.sort.bypassMergeThreshold</code></td>
  <td>200</td>
  <td>
    (Advanced) In the sort-based shuffle manager, avoid merge-sorting data if there is no
    map-side aggregation and there are at most this many reduce partitions.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.spill.compress</code></td>
  <td>true</td>
  <td>
    Whether to compress data spilled during shuffles. Compression will use
    <code>spark.io.compression.codec</code>.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.accurateBlockThreshold</code></td>
  <td>100 * 1024 * 1024</td>
  <td>
    When we compress the size of shuffle blocks in HighlyCompressedMapStatus, we will record the
    size accurately if it's above this config. This helps to prevent OOM by avoiding
    underestimating shuffle block size when fetch shuffle blocks.
  </td>
</tr>
<tr>
  <td><code>spark.io.encryption.enabled</code></td>
  <td>false</td>
  <td>
    Enable IO encryption. Currently supported by all modes except Mesos. It's recommended that RPC encryption
    be enabled when using this feature.
  </td>
</tr>
<tr>
  <td><code>spark.io.encryption.keySizeBits</code></td>
  <td>128</td>
  <td>
    IO encryption key size in bits. Supported values are 128, 192 and 256.
  </td>
</tr>
<tr>
  <td><code>spark.io.encryption.keygen.algorithm</code></td>
  <td>HmacSHA1</td>
  <td>
    The algorithm to use when generating the IO encryption key. The supported algorithms are
    described in the KeyGenerator section of the Java Cryptography Architecture Standard Algorithm
    Name Documentation.
  </td>
</tr>
</table>

### Spark UI

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.eventLog.compress</code></td>
  <td>false</td>
  <td>
    是否压缩记录的事件，如果 <code>spark.eventLog.enabled</code> 为true。压缩将使用<code>spark.io.compression.codec</code>。
  </td>
</tr>
<tr>
  <td><code>spark.eventLog.dir</code></td>
  <td>file:///tmp/spark-events</td>
  <td>
    Spark 事件日志的文件路径。如果 <code>spark.eventLog.enabled</code> 为 true。在这个基本目录下，Spark 为每个应用程序创建一个二级目录，日志事件特定于应用程序的目录。用户可能希望设置一个统一的文件目录像一个 HDFS 目录那样，所以历史文件可以从历史文件服务器中读取。
  </td>
</tr>
<tr>
  <td><code>spark.eventLog.enabled</code></td>
  <td>false</td>
  <td>
    是否对 Spark 事件记录日志。在应用程序启动后有助于重建 Web UI。
  </td>
</tr>
<tr>
  <td><code>spark.ui.enabled</code></td>
  <td>true</td>
  <td>
    Whether to run the web UI for the Spark application.
  </td>
</tr>
<tr>
  <td><code>spark.ui.killEnabled</code></td>
  <td>true</td>
  <td>
    允许从 Web UI 中结束相应的工作进程。
  </td>
</tr>
<tr>
  <td><code>spark.ui.port</code></td>
  <td>4040</td>
  <td>
    应用 UI 的端口，用于显示内存和工作负载数据。
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedJobs</code></td>
  <td>1000</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 Job 可以留存。
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedStages</code></td>
  <td>1000</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 Stage 可以留存。
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedTasks</code></td>
  <td>100000</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 Task 可以留存。
  </td>
</tr>
<tr>
  <td><code>spark.ui.reverseProxy</code></td>
  <td>false</td>
  <td>
    Enable running Spark Master as reverse proxy for worker and application UIs. In this mode, Spark master will reverse proxy the worker and application UIs to enable access without requiring direct access to their hosts. Use it with caution, as worker and application UI will not be accessible directly, you will only be able to access them through spark master/proxy public URL. This setting affects all the workers and application UIs running in the cluster and must be set on all the workers, drivers and masters.
  </td>
</tr>
<tr>
  <td><code>spark.ui.reverseProxyUrl</code></td>
  <td></td>
  <td>
    This is the URL where your proxy is running. This URL is for proxy which is running in front of Spark Master. This is useful when running proxy for authentication e.g. OAuth proxy. Make sure this is a complete URL including scheme (http/https) and port to reach your proxy.
  </td>
</tr>
<tr>
  <td><code>spark.ui.showConsoleProgress</code></td>
  <td>true</td>
  <td>
    Show the progress bar in the console. The progress bar shows the progress of stages
    that run for longer than 500ms. If multiple stages run at the same time, multiple
    progress bars will be displayed on the same line.
  </td>
</tr>
<tr>
  <td><code>spark.worker.ui.retainedExecutors</code></td>
  <td>1000</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 execution 已经完成。
  </td>
</tr>
<tr>
  <td><code>spark.worker.ui.retainedDrivers</code></td>
  <td>1000</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 driver 已经完成。
  </td>
</tr>
<tr>
  <td><code>spark.sql.ui.retainedExecutions</code></td>
  <td>1000</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 execution 已经完成。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.ui.retainedBatches</code></td>
  <td>1000</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 batch 已经完成。
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedDeadExecutors</code></td>
  <td>100</td>
  <td>
    在垃圾回收前，Spark UI 和 API 有多少 dead executors。
  </td>
</tr>
</table>

### Compression and Serialization （压缩和序列化）

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.broadcast.compress</code></td>
  <td>true</td>
  <td>
    是否在发送之前压缩广播变量。一般是个好主意压缩将使用 <code>spark.io.compression.codec</code>。
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.codec</code></td>
  <td>lz4</td>
  <td>
    内部数据使用的压缩编解码器，如 RDD 分区，广播变量和混洗输出。 默认情况下，Spark 提供三种编解码器 : <code>lz4</code>, <code>lzf</code>, 和 <code>snappy</code>。您还可以使用完全限定类名来指定编码解码器，
    例如 : 
    <code>org.apache.spark.io.LZ4CompressionCodec</code>，
    <code>org.apache.spark.io.LZFCompressionCodec</code>，
    和 <code>org.apache.spark.io.SnappyCompressionCodec</code>。
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.lz4.blockSize</code></td>
  <td>32k</td>
  <td>
    在采用 LZ4 压缩编解码器的情况下，LZ4 压缩使用的块大小。减少块大小还将降低采用 LZ4 时的混洗内存使用。
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.snappy.blockSize</code></td>
  <td>32k</td>
  <td>
    在采用 Snappy 压缩编解码器的情况下，Snappy 压缩使用的块大小。减少块大小还将降低采用 Snappy 时的混洗内存使用。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.classesToRegister</code></td>
  <td>(none)</td>
  <td>
    如果你采用 Kryo 序列化，给一个以逗号分隔的自定义类名列以注册 Kryo。有关详细信息，请参阅<a href="tuning.html#data-serialization">调优指南</a>。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.referenceTracking</code></td>
  <td>true</td>
  <td>
    当使用 Kryo 序列化数据时，是否跟踪对同一对象的引用，如果对象图具有循环，并且如果它们包含同一对象的多个副本对效率有用，则这是必需的。 如果您知道这不是这样，可以禁用此功能来提高性能。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.registrationRequired</code></td>
  <td>false</td>
  <td>
    是否需要注册 Kryo。 如果设置为 'true'，如果未注册的类被序列化，Kryo 将抛出异常。如果设置为 false（默认值），Kryo 将与每个对象一起写入未注册的类名。 编写类名可能会导致显著的性能开销，因此启用此选项可以严格强制用户没有从注册中省略类。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.registrator</code></td>
  <td>(none)</td>
  <td>
    如果你采用 Kryo 序列化，则给一个逗号分隔的类列表，以使用 Kryo 注册你的自定义类。 如果你需要以自定义方式注册你的类，则此属性很有用，例如以指定自定义字段序列化程序。 否则，使用 spark.kryo.classesToRegisteris 更简单。 它应该设置为 <a href="api/scala/index.html#org.apache.spark.serializer.KryoRegistrator">
    <code>KryoRegistrator</code></a> 的子类。 详见 : <a href="tuning.html#data-serialization">调整指南</a>。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.unsafe</code></td>
  <td>false</td>
  <td>
    Whether to use unsafe based Kryo serializer. Can be
    substantially faster by using Unsafe Based IO.
  </td>
</tr>
<tr>
  <td><code>spark.kryoserializer.buffer.max</code></td>
  <td>64m</td>
  <td>
    Kryo 序列化缓冲区的最大允许大小。 这必须大于你需要序列化的任何对象。 如果你在 Kryo 中得到一个 “buffer limit exceeded” 异常，你就需要增加这个值。
  </td>
</tr>
<tr>
  <td><code>spark.kryoserializer.buffer</code></td>
  <td>64k</td>
  <td>
    Kryo 序列化缓冲区的初始大小。 注意，每个 worker上 <i>每个 core</i> 会有一个缓冲区。 如果需要，此缓冲区将增长到 <code>spark.kryoserializer.buffer.max</code>。
  </td>
</tr>
<tr>
  <td><code>spark.rdd.compress</code></td>
  <td>false</td>
  <td>
    是否压缩序列化RDD分区（例如，在 Java 和 Scala 中为 <code>StorageLevel.MEMORY_ONLY_SER</code> 或在 Python 中为 <code>StorageLevel.MEMORY_ONLY</code>）。 可以节省大量空间，花费一些额外的CPU时间。 压缩将使用 <code>spark.io.compression.codec</code>。
  </td>
</tr>
<tr>
  <td><code>spark.serializer</code></td>
  <td>
    org.apache.spark.serializer.<br />JavaSerializer
  </td>
  <td>
    用于序列化将通过网络发送或需要以序列化形式缓存的对象的类。 Java 序列化的默认值与任何Serializable Java对象一起使用，但速度相当慢，所以我们建议您在需要速度时使用  <a href="tuning.html">使用
    <code>org.apache.spark.serializer.KryoSerializer</code> 并配置 Kryo 序列化 </a>。 可以是 <a href="api/scala/index.html#org.apache.spark.serializer.Serializer">
    <code>org.apache.spark.Serializer</code></a> 的任何子类。
  </td>
</tr>
<tr>
  <td><code>spark.serializer.objectStreamReset</code></td>
  <td>100</td>
  <td>
    当正使用 org.apache.spark.serializer.JavaSerializer 序列化时, 序列化器缓存对象虽然可以防止写入冗余数据，但是却停止这些缓存对象的垃圾回收。通过调用 'reset' 你从序列化程序中清除该信息，并允许收集旧的对象。 要禁用此周期性重置，请将其设置为 -1。 默认情况下，序列化器会每过 100 个对象被重置一次。
  </td>
</tr>
</table>

### Memory Management （内存管理）

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.memory.fraction</code></td>
  <td>0.6</td>
  <td>
    用于执行和存储的（堆空间 - 300MB）的分数。这个值越低，溢出和缓存数据逐出越频繁。 此配置的目的是在稀疏、异常大的记录的情况下为内部元数据，用户数据结构和不精确的大小估计预留内存。推荐使用默认值。 有关更多详细信息，包括关于在增加此值时正确调整 JVM 垃圾回收的重要信息，请参阅 <a href="tuning.html#memory-management-overview">this description</a>。
  </td>
</tr>
<tr>
  <td><code>spark.memory.storageFraction</code></td>
  <td>0.5</td>
  <td>
    不会被逐出内存的总量，表示为 <code>s​park.memory.fraction</code> 留出的区域大小的一小部分。 这个越高，工作内存可能越少，执行和任务可能更频繁地溢出到磁盘。 推荐使用默认值。有关更多详细信息，请参阅 <a href="tuning.html#memory-management-overview">this description</a>。
  </td>
</tr>
<tr>
  <td><code>spark.memory.offHeap.enabled</code></td>
  <td>false</td>
  <td>
    如果为 true，Spark 会尝试对某些操作使用堆外内存。 如果启用了堆外内存使用，则 <code>spark.memory.offHeap.size</code> 必须为正值。
  </td>
</tr>
<tr>
  <td><code>spark.memory.offHeap.size</code></td>
  <td>0</td>
  <td>
    可用于堆外分配的绝对内存量（以字节为单位）。 此设置对堆内存使用没有影响，因此如果您的执行器的总内存消耗必须满足一些硬限制，那么请确保相应地缩减JVM堆大小。 当 <code>spark.memory.offHeap.enabled=true</code> 时，必须将此值设置为正值。
  </td>
</tr>
<tr>
  <td><code>spark.memory.useLegacyMode</code></td>
  <td>false</td>
  <td>
    是否启用 Spark 1.5 及以前版本中使用的传统内存管理模式。 传统模式将堆空间严格划分为固定大小的区域，如果未调整应用程序，可能导致过多溢出。 必须启用本参数，以下选项才可用 : 
    <code>spark.shuffle.memoryFraction</code><br>
    <code>spark.storage.memoryFraction</code><br>
    <code>spark.storage.unrollFraction</code>
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.memoryFraction</code></td>
  <td>0.2</td>
  <td>
    （过时）只有在启用 <code>spark.memory.useLegacyMode</code> 时，此属性才是可用的。 混洗期间用于聚合和 cogroups 的 Java 堆的分数。 在任何给定时间，用于混洗的所有内存映射的集合大小不会超过这个上限，超过该限制的内容将开始溢出到磁盘。 如果溢出频繁，请考虑增加此值，但这以 <code>spark.storage.memoryFraction</code> 为代价。
  </td>
</tr>
<tr>
  <td><code>spark.storage.memoryFraction</code></td>
  <td>0.6</td>
  <td>
    （过时）只有在启用 <code>spark.memory.useLegacyMode</code> 时，此属性才是可用的。 Java 堆的分数，用于 Spark 的内存缓存。 这个值不应该大于 JVM 中老生代（old generation) 对象所占用的内存，默认情况下，它提供 0.6 的堆，但是如果配置你所用的老生代对象大小，你可以增加它。
  </td>
</tr>
<tr>
  <td><code>spark.storage.unrollFraction</code></td>
  <td>0.2</td>
  <td>
    （过时）只有在启用 <code>spark.memory.useLegacyMode</code> 时，此属性才是可用的。 <code>spark.storage.memoryFraction</code> 用于在内存中展开块的分数。 当没有足够的空闲存储空间来完全展开新块时，通过删除现有块来动态分配。
  </td>
</tr>
<tr>
  <td><code>spark.storage.replication.proactive</code></td>
  <td>false</td>
  <td>
    Enables proactive block replication for RDD blocks. Cached RDD block replicas lost due to
    executor failures are replenished if there are any existing available replicas. This tries
    to get the replication level of the block to the initial number.
  </td>
</tr>
</table>

### Execution Behavior （执行行为）

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认行为）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.broadcast.blockSize</code></td>
  <td>4m</td>
  <td>
    <code>TorrentBroadcastFactory</code> 的一个块的每个分片大小。 过大的值会降低广播期间的并行性（更慢了）; 但是，如果它过小，<code>BlockManager</code> 可能会受到性能影响。
  </td>
</tr>
<tr>
  <td><code>spark.executor.cores</code></td>
  <td>
    在 YARN 模式下默认为 1，standlone 和 Mesos 粗粒度模型中的 worker 节点的所有可用的 core。
  </td>
  <td>
    在每个 executor（执行器）上使用的 core 数。 在 standlone 和 Mesos 的粗粒度模式下，设置此参数允许应用在相同的 worker 上运行多个 executor（执行器），只要该 worker 上有足够的 core。 否则，每个 application（应用）在单个 worker 上只会启动一个 executor（执行器）。
  </td>
</tr>
<tr>
  <td><code>spark.default.parallelism</code></td>
  <td>
    对于分布式混洗（shuffle）操作，如 <code>reduceByKey</code> 和 <code>join</code>，父 RDD 中分区的最大数量。 对于没有父 RDD 的 <code>parallelize</code> 操作，它取决于集群管理器 :
    <ul>
      <li>本地模式 : 本地机器上的 core 数</li>
      <li>Mesos 细粒度模式 : 8</li>
      <li>其他 : 所有执行器节点上的 core 总数或者 2，以较大者为准</li>
    </ul>
  </td>
  <td>
    如果用户没有指定参数值，则这个属性是 <code>join</code>, <code>reduceByKey</code>, 和 <code>parallelize</code> 等转换返回的 RDD 中的默认分区数。
  </td>
</tr>
<tr>
    <td><code>spark.executor.heartbeatInterval</code></td>
    <td>10s</td>
    <td>每个执行器的心跳与驱动程序之间的间隔。 心跳让驱动程序知道执行器仍然存活，并用正在进行的任务的指标更新它</td>
</tr>
<tr>
  <td><code>spark.files.fetchTimeout</code></td>
  <td>60s</td>
  <td>
    获取文件的通讯超时，所获取的文件是从驱动程序通过 SparkContext.addFile() 添加的。
  </td>
</tr>
<tr>
  <td><code>spark.files.useFetchCache</code></td>
  <td>true</td>
  <td>
    如果设置为 true（默认），文件提取将使用由属于同一应用程序的执行器共享的本地缓存，这可以提高在同一主机上运行许多执行器时的任务启动性能。 如果设置为 false，这些缓存优化将被禁用，所有执行器将获取它们自己的文件副本。 如果使用驻留在 NFS 文件系统上的 Spark 本地目录，可以禁用此优化（有关详细信息，请参阅 <a href="https://issues.apache.org/jira/browse/SPARK-6313">SPARK-6313</a> ）。
  </td>
</tr>
<tr>
  <td><code>spark.files.overwrite</code></td>
  <td>false</td>
  <td>
    当目标文件存在且其内容与源不匹配的情况下，是否覆盖通过 SparkContext.addFile() 添加的文件。
  </td>
</tr>
<tr>
  <td><code>spark.files.maxPartitionBytes</code></td>
  <td>134217728 (128 MB)</td>
  <td>
    The maximum number of bytes to pack into a single partition when reading files.
  </td>
</tr>
<tr>
  <td><code>spark.files.openCostInBytes</code></td>
  <td>4194304 (4 MB)</td>
  <td>
    The estimated cost to open a file, measured by the number of bytes could be scanned in the same
    time. This is used when putting multiple files into a partition. It is better to over estimate,
    then the partitions with small files will be faster than partitions with bigger files.
  </td>
</tr>
<tr>
    <td><code>spark.hadoop.cloneConf</code></td>
    <td>false</td>
    <td>
      如果设置为true，则为每个任务克隆一个新的Hadoop <code>Configuration</code> 对象。 应该启用此选项以解决 <code>Configuration</code> 线程安全问题（有关详细信息，请参阅 <a href="https://issues.apache.org/jira/browse/SPARK-2546">SPARK-2546</a>）。 默认情况下，这是禁用的，以避免不受这些问题影响的作业的意外性能回归。
    </td>
</tr>
<tr>
    <td><code>spark.hadoop.validateOutputSpecs</code></td>
    <td>true</td>
    <td>
      如果设置为 true，则验证 saveAsHadoopFile 和其他变体中使用的输出规范（例如，检查输出目录是否已存在）。 可以禁用此选项以静默由于预先存在的输出目录而导致的异常。 我们建议用户不要禁用此功能，除非需要实现与以前版本的 Spark 的兼容性。 可以简单地使用 Hadoop 的 FileSystem API 手动删除输出目录。 对于通过 Spark Streaming 的StreamingContext 生成的作业会忽略此设置，因为在检查点恢复期间可能需要将数据重写到预先存在的输出目录。
    </td>
</tr>
<tr>
  <td><code>spark.storage.memoryMapThreshold</code></td>
  <td>2m</td>
  <td>
    当从磁盘读取块时，Spark 内存映射的块大小。 这会阻止 Spark 从内存映射过小的块。 通常，存储器映射对于接近或小于操作系统的页大小的块具有高开销。
  </td>
</tr>
<tr>
  <td><code>spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version</code></td>
  <td>1</td>
  <td>
    The file output committer algorithm version, valid algorithm version number: 1 or 2.
    Version 2 may have better performance, but version 1 may handle failures better in certain situations,
    as per <a href="https://issues.apache.org/jira/browse/MAPREDUCE-4815">MAPREDUCE-4815</a>.
  </td>
</tr>
</table>

### Networking （网络）

<table class="table"><tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.rpc.message.maxSize</code></td>
  <td>128</td>
  <td>
    在 “control plane” 通信中允许的最大消息大小（以 MB 为单位）; 一般只适用于在 executors 和 driver 之间发送的映射输出大小信息。 如果您正在运行带有数千个 map 和 reduce 任务的作业，并查看有关 RPC 消息大小的消息，请增加此值。
  </td>
</tr>
<tr>
  <td><code>spark.blockManager.port</code></td>
  <td>(random)</td>
  <td>
    所有块管理器监听的端口。 这些都存在于 driver 和 executors 上。
  </td>
</tr>
<tr>
  <td><code>spark.driver.blockManager.port</code></td>
  <td>(value of spark.blockManager.port)</td>
  <td>
    Driver-specific port for the block manager to listen on, for cases where it cannot use the same
    configuration as executors.
  </td>
</tr>
<tr>
  <td><code>spark.driver.bindAddress</code></td>
  <td>(value of spark.driver.host)</td>
  <td>
    Hostname or IP address where to bind listening sockets. This config overrides the SPARK_LOCAL_IP
    environment variable (see below).

    <br />It also allows a different address from the local one to be advertised to executors or external systems.
    This is useful, for example, when running containers with bridged networking. For this to properly work,
    the different ports used by the driver (RPC, block manager and UI) need to be forwarded from the
    container's host.
  </td>
</tr>
<tr>
  <td><code>spark.driver.host</code></td>
  <td>(local hostname)</td>
  <td>
    要监听的 driver 的主机名或 IP 地址。 这用于与 executors 和 standalone Master 进行通信。
  </td>
</tr>
<tr>
  <td><code>spark.driver.port</code></td>
  <td>(random)</td>
  <td>
    要监听的 driver 的端口。这用于与 executors 和 standalone Master 进行通信。
  </td>
</tr>
<tr>
  <td><code>spark.network.timeout</code></td>
  <td>120s</td>
  <td>
    所有网络交互的默认超时。 如果未配置此项，将使用此配置替换 
    <code>spark.core.connection.ack.wait.timeout</code>,
    <code>spark.storage.blockManagerSlaveTimeoutMs</code>,
    <code>spark.shuffle.io.connectionTimeout</code>, <code>spark.rpc.askTimeout</code> or
    <code>spark.rpc.lookupTimeout</code>。
  </td>
</tr>
<tr>
  <td><code>spark.port.maxRetries</code></td>
  <td>16</td>
  <td>
    在绑定端口放弃之前的最大重试次数。 当端口被赋予特定值（非 0）时，每次后续重试将在重试之前将先前尝试中使用的端口增加 1。 这本质上允许它尝试从指定的开始端口到端口 + maxRetries 的一系列端口。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.numRetries</code></td>
  <td>3</td>
  <td>
    在 RPC 任务放弃之前重试的次数。 RPC 任务将在此数字的大多数时间运行。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.retry.wait</code></td>
  <td>3s</td>
  <td>
    RPC 请求操作在重试之前等待的持续时间。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.askTimeout</code></td>
  <td><code>spark.network.timeout</code></td>
  <td>
    RPC 请求操作在超时前等待的持续时间。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.lookupTimeout</code></td>
  <td>120s</td>
  <td>
    RPC 远程端点查找操作在超时之前等待的持续时间。
  </td>
</tr>
</table>

### Scheduling （调度）

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.cores.max</code></td>
  <td>(not set)</td>
  <td>
    当以 “coarse-grained（粗粒度）” 共享模式在 <a href="spark-standalone.html">standalone deploy cluster</a> 或 <a href="running-on-mesos.html#mesos-run-modes">Mesos cluster in "coarse-grained"
    sharing mode</a> 上运行时, 从集群（而不是每台计算机）请求应用程序的最大 CPU 内核数量.  如果未设置, 默认值将是 Spar k的 standalone deploy 管理器上的 <code>spark.deploy.defaultCores</code> , 或者 Mesos上的无限（所有可用核心）. 
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait</code></td>
  <td>3s</td>
  <td>
    等待启动本地数据任务多长时间, 然后在较少本地节点上放弃并启动它.  相同的等待将用于跨越多个地点级别（process-local, node-local, rack-local 等所有）.  也可以通过设置 <code>spark.locality.wait.node</code> 等来自定义每个级别的等待时间. 如果任务很长并且局部性较差, 则应该增加此设置, 但是默认值通常很好. 
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.node</code></td>
  <td>spark.locality.wait</td>
  <td>
    自定义 node locality 等待时间.  例如, 您可以将其设置为 0 以跳过 node locality, 并立即搜索机架位置（如果群集具有机架信息）. 
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.process</code></td>
  <td>spark.locality.wait</td>
  <td>
    自定义 process locality 等待时间. 这会影响尝试访问特定执行程序进程中的缓存数据的任务. 
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.rack</code></td>
  <td>spark.locality.wait</td>
  <td>
    自定义 rack locality 等待时间. 
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.maxRegisteredResourcesWaitingTime</code></td>
  <td>30s</td>
  <td>
    在调度开始之前等待资源注册的最大时间量. 
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.minRegisteredResourcesRatio</code></td>
  <td>0.8 for YARN mode; 0.0 for standalone mode and Mesos coarse-grained mode</td>
  <td>
    注册资源（注册资源/总预期资源）的最小比率（资源是 yarn 模式下的执行程序, standalone 模式下的 CPU 核心和 Mesos coarsed-grained 模式 'spark.cores.max' 值是 Mesos  coarse-grained 模式下的总体预期资源]）在调度开始之前等待.  指定为 0.0 和 1.0 之间的双精度.  无论是否已达到资源的最小比率, 在调度开始之前将等待的最大时间量由配置<code>spark.scheduler.maxRegisteredResourcesWaitingTime</code> 控制. 
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.mode</code></td>
  <td>FIFO</td>
  <td>
    作业之间的 <a href="job-scheduling.html#scheduling-within-an-application">scheduling mode （调度模式）</a> 提交到同一个 SparkContext.  可以设置为 <code>FAIR</code> 使用公平共享, 而不是一个接一个排队作业.  对多用户服务有用. 
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.revive.interval</code></td>
  <td>1s</td>
  <td>
    调度程序复活工作资源去运行任务的间隔长度. 
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.enabled</code></td>
  <td>
    false
  </td>
  <td>
    If set to "true", prevent Spark from scheduling tasks on executors that have been blacklisted
    due to too many task failures. The blacklisting algorithm can be further controlled by the
    other "spark.blacklist" configuration options.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.timeout</code></td>
  <td>1h</td>
  <td>
    (Experimental) How long a node or executor is blacklisted for the entire application, before it
    is unconditionally removed from the blacklist to attempt running new tasks.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.task.maxTaskAttemptsPerExecutor</code></td>
  <td>1</td>
  <td>
    (Experimental) For a given task, how many times it can be retried on one executor before the
    executor is blacklisted for that task.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.task.maxTaskAttemptsPerNode</code></td>
  <td>2</td>
  <td>
    (Experimental) For a given task, how many times it can be retried on one node, before the entire
    node is blacklisted for that task.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.stage.maxFailedTasksPerExecutor</code></td>
  <td>2</td>
  <td>
    (Experimental) How many different tasks must fail on one executor, within one stage, before the
    executor is blacklisted for that stage.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.stage.maxFailedExecutorsPerNode</code></td>
  <td>2</td>
  <td>
    (Experimental) How many different executors are marked as blacklisted for a given stage, before
    the entire node is marked as failed for the stage.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.application.maxFailedTasksPerExecutor</code></td>
  <td>2</td>
  <td>
    (Experimental) How many different tasks must fail on one executor, in successful task sets,
    before the executor is blacklisted for the entire application.  Blacklisted executors will
    be automatically added back to the pool of available resources after the timeout specified by
    <code>spark.blacklist.timeout</code>.  Note that with dynamic allocation, though, the executors
    may get marked as idle and be reclaimed by the cluster manager.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.application.maxFailedExecutorsPerNode</code></td>
  <td>2</td>
  <td>
    (Experimental) How many different executors must be blacklisted for the entire application,
    before the node is blacklisted for the entire application.  Blacklisted nodes will
    be automatically added back to the pool of available resources after the timeout specified by
    <code>spark.blacklist.timeout</code>.  Note that with dynamic allocation, though, the executors
    on the node may get marked as idle and be reclaimed by the cluster manager.
  </td>
</tr>
<tr>
  <td><code>spark.blacklist.killBlacklistedExecutors</code></td>
  <td>false</td>
  <td>
    (Experimental) If set to "true", allow Spark to automatically kill, and attempt to re-create,
    executors when they are blacklisted.  Note that, when an entire node is added to the blacklist,
    all of the executors on that node will be killed.
  </td>
</tr>
<tr>
  <td><code>spark.speculation</code></td>
  <td>false</td>
  <td>
    如果设置为 "true" , 则执行任务的推测执行.  这意味着如果一个或多个任务在一个阶段中运行缓慢, 则将重新启动它们. 
  </td>
</tr>
<tr>
  <td><code>spark.speculation.interval</code></td>
  <td>100ms</td>
  <td>
   Spark 检查要推测的任务的时间间隔. 
  </td>
</tr>
<tr>
  <td><code>spark.speculation.multiplier</code></td>
  <td>1.5</td>
  <td>
    一个任务的速度可以比推测的平均值慢多少倍. 
  </td>
</tr>
<tr>
  <td><code>spark.speculation.quantile</code></td>
  <td>0.75</td>
  <td>
    对特定阶段启用推测之前必须完成的任务的分数. 
  </td>
</tr>
<tr>
  <td><code>spark.task.cpus</code></td>
  <td>1</td>
  <td>
    要为每个任务分配的核心数. 
  </td>
</tr>
<tr>
  <td><code>spark.task.maxFailures</code></td>
  <td>4</td>
  <td>
    放弃作业之前任何特定任务的失败次数.  分散在不同任务中的故障总数不会导致作业失败; 一个特定的任务允许失败这个次数.  应大于或等于 1. 允许重试次数=此值 - 1. 
  </td>
</tr>
<tr>
  <td><code>spark.task.reaper.enabled</code></td>
  <td>false</td>
  <td>
    Enables monitoring of killed / interrupted tasks. When set to true, any task which is killed
    will be monitored by the executor until that task actually finishes executing. See the other
    <code>spark.task.reaper.*</code> configurations for details on how to control the exact behavior
    of this monitoring. When set to false (the default), task killing will use an older code
    path which lacks such monitoring.
  </td>
</tr>
<tr>
  <td><code>spark.task.reaper.pollingInterval</code></td>
  <td>10s</td>
  <td>
    When <code>spark.task.reaper.enabled = true</code>, this setting controls the frequency at which
    executors will poll the status of killed tasks. If a killed task is still running when polled
    then a warning will be logged and, by default, a thread-dump of the task will be logged
    (this thread dump can be disabled via the <code>spark.task.reaper.threadDump</code> setting,
    which is documented below).
  </td>
</tr>
<tr>
  <td><code>spark.task.reaper.threadDump</code></td>
  <td>true</td>
  <td>
    When <code>spark.task.reaper.enabled = true</code>, this setting controls whether task thread
    dumps are logged during periodic polling of killed tasks. Set this to false to disable
    collection of thread dumps.
  </td>
</tr>
<tr>
  <td><code>spark.task.reaper.killTimeout</code></td>
  <td>-1</td>
  <td>
    When <code>spark.task.reaper.enabled = true</code>, this setting specifies a timeout after
    which the executor JVM will kill itself if a killed task has not stopped running. The default
    value, -1, disables this mechanism and prevents the executor from self-destructing. The purpose
    of this setting is to act as a safety-net to prevent runaway uncancellable tasks from rendering
    an executor unusable.
  </td>
</tr>
<tr>
  <td><code>spark.stage.maxConsecutiveAttempts</code></td>
  <td>4</td>
  <td>
    Number of consecutive stage attempts allowed before a stage is aborted.
  </td>
</tr>
</table>

### Dynamic Allocation （动态分配）

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.dynamicAllocation.enabled</code></td>
  <td>false</td>
  <td>
    是否使用动态资源分配, 它根据工作负载调整为此应用程序注册的执行程序数量.  有关更多详细信息, 请参阅 <a href="job-scheduling.html#dynamic-resource-allocation">here</a> 的说明. 
    <br><br>
    这需要设置 <code>spark.shuffle.service.enabled</code> .  以下配置也相关 : <code>spark.dynamicAllocation.minExecutors</code>, <code>spark.dynamicAllocation.maxExecutors</code> 和<code>spark.dynamicAllocation.initialExecutors</code> . 
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.executorIdleTimeout</code></td>
  <td>60s</td>
  <td>
    如果启用动态分配, 并且执行程序已空闲超过此持续时间, 则将删除执行程序.  有关更多详细信息, 请参阅此<a href="job-scheduling.html#resource-allocation-policy">description</a>.
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.cachedExecutorIdleTimeout</code></td>
  <td>infinity</td>
  <td>
    如果启用动态分配, 并且已缓存数据块的执行程序已空闲超过此持续时间, 则将删除执行程序.  有关详细信息, 请参阅此 <a href="job-scheduling.html#resource-allocation-policy">description</a> . 
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.initialExecutors</code></td>
  <td><code>spark.dynamicAllocation.minExecutors</code></td>
  <td>
    启用动态分配时要运行的执行程序的初始数. 
    <br /><br />
    如果 `--num-executors`（或 `spark.executor.instances` ）被设置并大于此值, 它将被用作初始执行器数. 
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.maxExecutors</code></td>
  <td>infinity</td>
  <td>
    启用动态分配的执行程序数量的上限. 
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.minExecutors</code></td>
  <td>0</td>
  <td>
    启用动态分配的执行程序数量的下限. 
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.schedulerBacklogTimeout</code></td>
  <td>1s</td>
  <td>
    如果启用动态分配, 并且有超过此持续时间的挂起任务积压, 则将请求新的执行者.  有关更多详细信息, 请参阅此 <a href="job-scheduling.html#resource-allocation-policy">description</a> . 
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.sustainedSchedulerBacklogTimeout</code></td>
  <td><code>schedulerBacklogTimeout</code></td>
  <td>
    与 <code>spark.dynamicAllocation.schedulerBacklogTimeout</code> 相同, 但仅用于后续执行者请求.  有关更多详细信息, 请参阅此 <a href="job-scheduling.html#resource-allocation-policy">description</a> .
  </td>
</tr>
</table>

### Security （安全）

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.acls.enable</code></td>
  <td>false</td>
  <td>
    是否开启 Spark acls. 如果开启了, 它检查用户是否有权限去查看或修改 job.  Note this requires the user to be known, so if the user comes across as null no checks are done. UI 利用使用过滤器验证和设置用户. 
  </td>
</tr>
<tr>
  <td><code>spark.admin.acls</code></td>
  <td>Empty</td>
  <td>
    逗号分隔的用户或者管理员列表, 列表中的用户或管理员有查看和修改所有 Spark job 的权限. 如果你运行在一个共享集群, 有一组管理员或开发者帮助 debug, 这个选项有用. 
  </td>
</tr>
<tr>
  <td><code>spark.admin.acls.groups</code></td>
  <td>Empty</td>
  <td>
    具有查看和修改对所有Spark作业的访问权限的组的逗号分隔列表. 如果您有一组帮助维护和调试的 administrators 或 developers 可以使用此功能基础设施.  在列表中输入 "*" 表示任何组中的任何用户都可以使用 admin 的特权.  用户组是从 groups mapping provider 的实例获得的. 由 <code>spark.user.groups.mapping</code> 指定.  检查 entry <code> spark.user.groups.mapping</code> 了解更多详细信息. 
  </td>
</tr>
<tr>
  <td><code>spark.user.groups.mapping</code></td>
  <td><code>org.apache.spark.security.ShellBasedGroupsMappingProvider</code></td>
  <td>
    用户的组列表由特征定义的 group mapping service 决定可以通过此属性配置的org.apache.spark.security.GroupMappingServiceProvider. 提供了基于 unix shell 的默认实现 <code>org.apache.spark.security.ShellBasedGroupsMappingProvider</code> 可以指定它来解析用户的组列表. 
     <em>注意:</em> 此实现仅支持基于 Unix/Linux 的环境.  Windows 环境是
     目前 <b>不</b> 支持.  但是, 通过实现可以支持新的 platform/protocol （平台/协议） trait <code>org.apache.spark.security.GroupMappingServiceProvider</code> . 
  </td>
</tr>
<tr>
  <td><code>spark.authenticate</code></td>
  <td>false</td>
  <td>
    是否 Spark 验证其内部连接. 如果不是运行在 YARN 上, 请看 <code>spark.authenticate.secret</code> . 
  </td>
</tr>
<tr>
  <td><code>spark.authenticate.secret</code></td>
  <td>None</td>
  <td>
    设置密钥用于 spark 组件之间进行身份验证.  这需要设置 不启用运行在 yarn 和身份验证. 
  </td>
</tr>
<tr>
  <td><code>spark.network.crypto.enabled</code></td>
  <td>false</td>
  <td>
    Enable encryption using the commons-crypto library for RPC and block transfer service.
    Requires <code>spark.authenticate</code> to be enabled.
  </td>
</tr>
<tr>
  <td><code>spark.network.crypto.keyLength</code></td>
  <td>128</td>
  <td>
    The length in bits of the encryption key to generate. Valid values are 128, 192 and 256.
  </td>
</tr>
<tr>
  <td><code>spark.network.crypto.keyFactoryAlgorithm</code></td>
  <td>PBKDF2WithHmacSHA1</td>
  <td>
    The key factory algorithm to use when generating encryption keys. Should be one of the
    algorithms supported by the javax.crypto.SecretKeyFactory class in the JRE being used.
  </td>
</tr>
<tr>
  <td><code>spark.network.crypto.saslFallback</code></td>
  <td>true</td>
  <td>
    Whether to fall back to SASL authentication if authentication fails using Spark's internal
    mechanism. This is useful when the application is connecting to old shuffle services that
    do not support the internal Spark authentication protocol. On the server side, this can be
    used to block older clients from authenticating against a new shuffle service.
  </td>
</tr>
<tr>
  <td><code>spark.network.crypto.config.*</code></td>
  <td>None</td>
  <td>
    Configuration values for the commons-crypto library, such as which cipher implementations to
    use. The config name should be the name of commons-crypto configuration without the
    "commons.crypto" prefix.
  </td>
</tr>
<tr>
  <td><code>spark.authenticate.enableSaslEncryption</code></td>
  <td>false</td>
  <td>
    身份验证时启用加密通信.  这是 block transfer service （块传输服务）和支持 RPC 的端点. 
  </td>
</tr>
<tr>
  <td><code>spark.network.sasl.serverAlwaysEncrypt</code></td>
  <td>false</td>
  <td>
    禁用未加密的连接服务, 支持 SASL 验证.  这是目前支持的外部转移服务. 
  </td>
</tr>
<tr>
  <td><code>spark.core.connection.ack.wait.timeout</code></td>
  <td><code>spark.network.timeout</code></td>
  <td>
    连接在 timing out （超时）和 giving up （放弃）之前等待 ack occur 的时间. 为了避免长时间 pause （暂停）, 如 GC, 导致的不希望的超时, 你可以设置较大的值. 
  </td>
</tr>
<tr>
  <td><code>spark.modify.acls</code></td>
  <td>Empty</td>
  <td>
    逗号分隔的用户列表, 列表中的用户有查看 Spark web UI 的权限. 默认情况下, 只有启动 Spark job 的用户有修改（比如杀死它）权限. 在列表中加入 "*" 意味着任何用户可以访问以修改它. 
  </td>
</tr>
<tr>
  <td><code>spark.modify.acls.groups</code></td>
  <td>Empty</td>
  <td>
    具有对 Spark job 的修改访问权限的组的逗号分隔列表.  如果你可以使用这个有一组来自同一个 team 的 administrators 或 developers 可以访问控制工作. 在列表中放置 "*" 表示任何组中的任何用户都有权修改 Spark job . 用户组是从 <code>spark.user.groups.mapping</code> 指定的 groups mapping 提供者的实例获得的.  查看 entry <code>spark.user.groups.mapping</code> 来了解更多细节. 
  </td>
</tr>
<tr>
  <td><code>spark.ui.filters</code></td>
  <td>None</td>
  <td>
    应用到 Spark web UI 的用于 filter class （过滤类）名的逗号分隔的列表. 过滤器必须是标准的 <a href="http://docs.oracle.com/javaee/6/api/javax/servlet/Filter.html">
    javax servlet Filter</a> .  每个过滤器的参数也可以通过设置一个 java 系统属性来指定 spark .
    java 系统属性: <br />
    <code>spark.&lt;class name of filter&gt;.params='param1=value1,param2=value2'</code><br />
    例如: <br />
    <code>-Dspark.ui.filters=com.test.filter1</code> <br />
    <code>-Dspark.com.test.filter1.params='param1=foo,param2=testing'</code>
  </td>
</tr>
<tr>
  <td><code>spark.ui.view.acls</code></td>
  <td>Empty</td>
  <td>
    逗号分隔的可以访问 Spark web ui 的用户列表.  默认情况下只有启动 Spark job 的用户具有 view 访问权限.  在列表中放入 "*" 表示任何用户都可以具有访问此 Spark job 的 view . 
  </td>
</tr>
<tr>
  <td><code>spark.ui.view.acls.groups</code></td>
  <td>Empty</td>
  <td>
    逗号分隔的列表, 可以查看访问 Spark web ui 的组, 以查看 Spark Job 细节.  如果您有一组 administrators 或 developers 或可以使用的用户, 则可以使用此功能 monitor （监控）提交的 Spark job .  在列表中添加 "*" 表示任何组中的任何用户都可以查看 Spark web ui 上的 Spark 工作详细信息.  用户组是从 由<code> spark.user.groups.mapping</code> 指定的 groups mapping provider （组映射提供程序）实例获得的. 查看 entry <code>spark.user.groups.mapping</code> 来了解更多细节. 
  </td>
</tr>
</table>

### TLS / SSL

<table class="table">
    <tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
    <tr>
        <td><code>spark.ssl.enabled</code></td>
        <td>false</td>
        <td>
            Whether to enable SSL connections on all supported protocols.
            <br />When <code>spark.ssl.enabled</code> is configured, <code>spark.ssl.protocol</code>
            is required.
            <br />All the SSL settings like <code>spark.ssl.xxx</code> where <code>xxx</code> is a
            particular configuration property, denote the global configuration for all the supported
            protocols. In order to override the global configuration for the particular protocol,
            the properties must be overwritten in the protocol-specific namespace.
            <br />Use <code>spark.ssl.YYY.XXX</code> settings to overwrite the global configuration for
            particular protocol denoted by <code>YYY</code>. Example values for <code>YYY</code>
            include <code>fs</code>, <code>ui</code>, <code>standalone</code>, and
            <code>historyServer</code>.  See <a href="security.html#ssl-configuration">SSL
            Configuration</a> for details on hierarchical SSL configuration for services.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.[namespace].port</code></td>
        <td>None</td>
        <td>
            The port where the SSL service will listen on.
            <br />The port must be defined within a namespace configuration; see
            <a href="security.html#ssl-configuration">SSL Configuration</a> for the available
            namespaces.
            <br />When not set, the SSL port will be derived from the non-SSL port for the
            same service. A value of "0" will make the service bind to an ephemeral port.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.enabledAlgorithms</code></td>
        <td>Empty</td>
        <td>
            A comma separated list of ciphers. The specified ciphers must be supported by JVM.
            The reference list of protocols one can find on
            <a href="https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https">this</a>
            page.
            Note: If not set, it will use the default cipher suites of JVM.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyPassword</code></td>
        <td>None</td>
        <td>
            A password to the private key in key-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStore</code></td>
        <td>None</td>
        <td>
            A path to a key-store file. The path can be absolute or relative to the directory where
            the component is started in.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStorePassword</code></td>
        <td>None</td>
        <td>
            A password to the key-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStoreType</code></td>
        <td>JKS</td>
        <td>
            The type of the key-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.protocol</code></td>
        <td>None</td>
        <td>
            A protocol name. The protocol must be supported by JVM. The reference list of protocols
            one can find on <a href="https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https">this</a>
            page.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.needClientAuth</code></td>
        <td>false</td>
        <td>
            Set true if SSL needs client authentication.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStore</code></td>
        <td>None</td>
        <td>
            A path to a trust-store file. The path can be absolute or relative to the directory
            where the component is started in.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStorePassword</code></td>
        <td>None</td>
        <td>
            A password to the trust-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStoreType</code></td>
        <td>JKS</td>
        <td>
            The type of the trust-store.
        </td>
    </tr>
</table>


### Spark SQL

运行 <code>SET -v</code> 命令将显示 SQL 配置的整个列表.

<div class="codetabs">
<div data-lang="scala"  markdown="1">

{% highlight scala %}
// spark is an existing SparkSession
spark.sql("SET -v").show(numRows = 200, truncate = false)
{% endhighlight %}

</div>

<div data-lang="java"  markdown="1">

{% highlight java %}
// spark is an existing SparkSession
spark.sql("SET -v").show(200, false);
{% endhighlight %}
</div>

<div data-lang="python"  markdown="1">

{% highlight python %}
# spark is an existing SparkSession
spark.sql("SET -v").show(n=200, truncate=False)
{% endhighlight %}

</div>

<div data-lang="r"  markdown="1">

{% highlight r %}
sparkR.session()
properties <- sql("SET -v")
showDF(properties, numRows = 200, truncate = FALSE)
{% endhighlight %}

</div>
</div>


### Spark Streaming

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.streaming.backpressure.enabled</code></td>
  <td>false</td>
  <td>
    开启或关闭 Spark Streaming 内部的 backpressure mecheanism（自 1.5 开始）. 基于当前批次调度延迟和处理时间, 这使得 Spark Streaming 能够控制数据的接收率, 因此, 系统接收数据的速度会和系统处理的速度一样快. 从内部来说, 这动态地设置了 receivers 的最大接收率. 这个速率上限通过 <code>spark.streaming.receiver.maxRate</code> 和 <code>spark.streaming.kafka.maxRatePerPartition</code> 两个参数设定（如下）. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.backpressure.initialRate</code></td>
  <td>not set</td>
  <td>
    当 backpressure mecheanism 开启时, 每个 receiver 接受数据的初始最大值. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.blockInterval</code></td>
  <td>200ms</td>
  <td>
    在这个时间间隔（ms）内, 通过 Spark Streaming receivers 接收的数据在保存到 Spark 之前, chunk 为数据块. 推荐的最小值为 50ms. 具体细节见 Spark Streaming 指南的 <a href="streaming-programming-guide.html#level-of-parallelism-in-data-receiving">performance
     tuning</a> 一节. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.maxRate</code></td>
  <td>not set</td>
  <td>
    每秒钟每个 receiver 将接收的数据的最大速率（每秒钟的记录数目）. 有效的情况下, 每个流每秒将最多消耗这个数目的记录. 设置这个配置为 0 或者 -1 将会不作限制. 细节参见 Spark Streaming 编程指南的 <a href="streaming-programming-guide.html#deploying-applications">deployment guide</a> 一节. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.writeAheadLog.enable</code></td>
  <td>false</td>
  <td>
    为 receiver 启用 write ahead logs. 所有通过接收器接收输入的数据将被保存到 write ahead logs, 以便它在驱动程序故障后进行恢复. 见星火流编程指南部署指南了解更多详情. 细节参见 Spark Streaming 编程指南的 <a href="streaming-programming-guide.html#deploying-applications">deployment guide</a> 一节. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.unpersist</code></td>
  <td>true</td>
  <td>
    强制通过 Spark Streaming 生成并持久化的 RDD 自动从 Spark 内存中非持久化. 通过 Spark Streaming 接收的原始输入数据也将清除. 设置这个属性为 false 允许流应用程序访问原始数据和持久化 RDD, 因为它们没有被自动清除. 但是它会造成更高的内存花费.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.stopGracefullyOnShutdown</code></td>
  <td>false</td>
  <td>
    如果为 <code>true</code> , Spark 将 gracefully （缓慢地）关闭在 JVM 运行的 StreamingContext , 而非立即执行. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.kafka.maxRatePerPartition</code></td>
  <td>not set</td>
  <td>
    在使用新的 Kafka direct stream API 时, 从每个 kafka 分区读到的最大速率（每秒的记录数目）. 详见 <a href="streaming-kafka-integration.html">Kafka Integration guide</a> . 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.kafka.maxRetries</code></td>
  <td>1</td>
  <td>
    driver 连续重试的最大次数, 以此找到每个分区 leader 的最近的（latest）的偏移量（默认为 1 意味着 driver 将尝试最多两次）. 仅应用于新的 kafka direct stream API. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.ui.retainedBatches</code></td>
  <td>1000</td>
  <td>
    在垃圾回收之前, Spark Streaming UI 和状态API 所能记得的 批处理（batches）数量. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.driver.writeAheadLog.closeFileAfterWrite</code></td>
  <td>false</td>
  <td>
   在写入一条 driver 中的 write ahead log 记录 之后, 是否关闭文件. 如果你想为 driver 中的元数据 WAL 使用 S3（或者任何文件系统而不支持 flushing）, 设定为 true. 
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.writeAheadLog.closeFileAfterWrite</code></td>
  <td>false</td>
  <td>
    在写入一条 reveivers 中的 write ahead log 记录 之后, 是否关闭文件. 如果你想为 reveivers 中的元数据 WAL 使用 S3（或者任何文件系统而不支持 flushing）, 设定为 true. 
  </td>
</tr>
</table>

### SparkR

<table class="table">
<tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
<tr>
  <td><code>spark.r.numRBackendThreads</code></td>
  <td>2</td>
  <td>
    使用 RBackend 处理来自 SparkR 包中的 RPC 调用的线程数.
  </td>
</tr>
<tr>
  <td><code>spark.r.command</code></td>
  <td>Rscript</td>
  <td>
    在 driver 和 worker 两种集群模式下可执行的 R 脚本.
  </td>
</tr>
<tr>
  <td><code>spark.r.driver.command</code></td>
  <td>spark.r.command</td>
  <td>
    在 driver 的 client 模式下可执行的 R 脚本. 在集群模式下被忽略.
  </td>
</tr>
<tr>
  <td><code>spark.r.shell.command</code></td>
  <td>R</td>
  <td>
    Executable for executing sparkR shell in client modes for driver. Ignored in cluster modes. It is the same as environment variable <code>SPARKR_DRIVER_R</code>, but take precedence over it.
    <code>spark.r.shell.command</code> is used for sparkR shell while <code>spark.r.driver.command</code> is used for running R script.
  </td>
</tr>
<tr>
  <td><code>spark.r.backendConnectionTimeout</code></td>
  <td>6000</td>
  <td>
    Connection timeout set by R process on its connection to RBackend in seconds.
  </td>
</tr>
<tr>
  <td><code>spark.r.heartBeatInterval</code></td>
  <td>100</td>
  <td>
    Interval for heartbeats sent from SparkR backend to R process to prevent connection timeout.
  </td>
</tr>

</table>

### GraphX

<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.graphx.pregel.checkpointInterval</code></td>
  <td>-1</td>
  <td>
    Checkpoint interval for graph and message in Pregel. It used to avoid stackOverflowError due to long lineage chains
  after lots of iterations. The checkpoint is disabled by default.
  </td>
</tr>
</table>

### Deploy （部署）

<table class="table">
  <tr><th>Property Name （属性名称）</th><th>Default （默认值）</th><th>Meaning （含义）</th></tr>
  <tr>
    <td><code>spark.deploy.recoveryMode</code></td>
    <td>NONE</td>
    <td>集群模式下, Spark jobs 执行失败或者重启时, 恢复提交 Spark jobs 的恢复模式设定.</td>
  </tr>
  <tr>
    <td><code>spark.deploy.zookeeper.url</code></td>
    <td>None</td>
    <td>当 `spark.deploy.recoveryMode` 被设定为 ZOOKEEPER , 这一配置被用来连接 zookeeper URL.</td>
  </tr>
  <tr>
    <td><code>spark.deploy.zookeeper.dir</code></td>
    <td>None</td>
    <td>当 `spark.deploy.recoveryMode` 被设定为 ZOOKEEPER, 这一配置被用来设定 zookeeper 目录为 store recovery state.</td>
  </tr>
</table>


### Cluster Managers （集群管理器）

Spark 中的每个集群管理器都有额外的配置选项, 这些配置可以在每个模式的页面中找到:

#### [YARN](running-on-yarn.html#configuration)

#### [Mesos](running-on-mesos.html#configuration)

#### [Standalone Mode](spark-standalone.html#cluster-launch-scripts)

# Environment Variables （环境变量）

通过环境变量配置特定的 Spark 设置. 环境变量从 Spark 安装目录下的 `conf/spark-env.sh` 脚本读取（或者是 window 环境下的 `conf/spark-env.cmd` ）. 在 Standalone 和 Mesos 模式下, 这个文件可以指定机器的特定信息, 比如 hostnames . 它也可以为正在运行的 Spark Application 或者提交脚本提供 sourced （来源）.  
注意, 当 Spark 被安装, 默认情况下 `conf/spark-env.sh` 是不存在的. 但是, 你可以通过拷贝 `conf/spark-env.sh.template` 来创建它. 确保你的拷贝文件时可执行的. 
`spark-env.sh` : 中有有以下变量可以被设置 :


<table class="table">
  <tr><th style="width:21%">Environment Variable （环境变量）</th><th>Meaning （含义）</th></tr>
  <tr>
    <td><code>JAVA_HOME</code></td>
    <td>Java 的安装路径（如果不在你的默认 <code>PATH</code> 下）.</td>
  </tr>
  <tr>
    <td><code>PYSPARK_PYTHON</code></td>
    <td>在 driver 和 worker 中 PySpark 用到的 Python 二进制可执行文件（如何有默认为 <code>python2.7</code>, 否则为 <code>python</code> ）. 如果设置了属性 <code>spark.pyspark.python</code>, 则会优先考虑.</td>
  </tr>
  <tr>
    <td><code>PYSPARK_DRIVER_PYTHON</code></td>
    <td>只在 driver 中 PySpark 用到的 Python 二进制可执行文件（默认为 <code>PYSPARK_PYTHON</code> ）. 如果设置了属性 <code>spark.pyspark.driver.python</code> ,则优先考虑.</td>
  </tr>
  <tr>
    <td><code>SPARKR_DRIVER_R</code></td>
    <td>SparkR shell 用到的 R 二进制可执行文件（默认为 <code>R</code> ）. 如果设置了属性 <code>spark.r.shell.command</code> 则会优先考虑.</td>
  </tr>
  <tr>
    <td><code>SPARK_LOCAL_IP</code></td>
    <td>机器绑定的 IP 地址.</td>
  </tr>
  <tr>
    <td><code>SPARK_PUBLIC_DNS</code></td>
    <td>你的 Spark 程序通知其他机器的 Hostname.</td>
  </tr>
</table>

除了以上参数, [standalone cluster scripts](spark-standalone.html#cluster-launch-scripts) 也可以设置其他选项, 比如每个机器使用的 CPU 核数和最大内存. 

因为 `spark-env.sh` 是 shell 脚本, 一些可以通过程序的方式来设置, 比如你可以通过特定的网络接口来计算 `SPARK_LOCAL_IP` . 

注意 : 当以 `cluster` mode （集群模式）运行 Spark on YARN 时 , 环境变量需要通过在您的 `conf/spark-defaults.conf` 文件中 `spark.yarn.appMasterEnv.[EnvironmentVariableName]` 来设定.
`cluster` mode （集群模式）下, `spark-env.sh` 中设定的环境变量将不会在 YARN Application Master 过程中反应出来. 详见 [YARN-related Spark Properties](running-on-yarn.html#spark-properties). 

# Configuring Logging （配置 Logging）

Spark 用 [log4j](http://logging.apache.org/log4j/) 生成日志, 你可以通过在 `conf` 目录下添加 `log4j.properties` 文件来配置.一种方法是拷贝 `log4j.properties.template` 文件.

# Overriding configuration directory （覆盖配置目录）

如果你想指定不同的配置目录, 而不是默认的 "SPARK_HOME/conf" , 你可以设置 SPARK_CONF_DIR. Spark 将从这一目录下读取文件（ spark-defaults.conf, spark-env.sh, log4j.properties 等）

# Inheriting Hadoop Cluster Configuration （继承 Hadoop 集群配置）

如果你想用 Spark 来读写 HDFS, 在 Spark 的 classpath 就需要包括两个 Hadoop 配置文件:

* `hdfs-site.xml`, 为 HDFS client 提供 default behaviors （默认的行为）.
* `core-site.xml`, 设定默认的文件系统名称.

这些配置文件的位置因 Hadoop 版本而异, 但是一个常见的位置在 `/etc/hadoop/conf` 内.  一些工具创建配置 on-the-fly, 但提供了一种机制来下载它们的副本. 

为了使这些文件对 Spark 可见, 需要设定 `$SPARK_HOME/spark-env.sh` 中的 `HADOOP_CONF_DIR` 到一个包含配置文件的位置.
