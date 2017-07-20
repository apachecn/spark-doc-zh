---
layout: global
title: 构建 Spark
redirect_from: "building-with-maven.html"
---

* This will become a table of contents (this text will be scraped).
{:toc}

# 构建 Apache Spark

## Apache Maven

基于 Maven 的构建是 Apache Spark 的参考构建。构建 Spark 需要 Maven 3.3.9 或者更高版本和 Java 8+ 。请注意，Spark 2.2.0 对于 Java 7 的支持已经删除了。

### 设置 Maven 的内存使用

您需要通过设置 `MAVEN_OPTS` 将 Maven 配置为比通常使用更多的内存：

    export MAVEN_OPTS="-Xmx2g -XX:ReservedCodeCacheSize=512m"

(`ReservedCodeCacheSize` 设置是可选的，但是建议使用。)
如果您不讲这些参数加入到 `MAVEN_OPTS` 中，则可能会看到以下错误和警告：

    [INFO] Compiling 203 Scala sources and 9 Java sources to /Users/me/Development/spark/core/target/scala-{{site.SCALA_BINARY_VERSION}}/classes...
    [ERROR] Java heap space -> [Help 1]

您可以通过如前所述设置 `MAVEN_OPTS` 变量来解决这些问题。

**注意:**

* 如果使用 `build/mvn` 但是没有设置 `MAVEN_OPTS` ，那么脚本会自动地将以上选项添加到 `MAVEN_OPTS` 环境变量。
* 即使不适用 `build/mvn` ，Spark 构建的 `test` 阶段也会自动将这些选项加入到 `MAVEN_OPTS` 中。   

### build/mvn

Spark 现在与一个独立的 Maven 安装包封装到了一起，以便从位于 `build/` 目录下的源代码构建和部署 Spark 。该脚本将自动下载并设置所有必要的构建需求 ([Maven](https://maven.apache.org/) ，[Scala](http://www.scala-lang.org/) 和 [Zinc](https://github.com/typesafehub/zinc)) 到本身的 `build/` 目录下。其尊重任何已经存在的 `mvn` 二进制文件，但是将会 pull down 其自己的 Scala 和 Zinc 的副本，不管是否满足适当的版本需求。`build/mvn` 的执行作为传递到 `mvn` 的调用，允许从以前的构建方法轻松转换。例如，可以像下边这样构建 Spark 的版本：

    ./build/mvn -DskipTests clean package

其他的构建例子可以在下边找到。

## 构建一个可运行的 Distribution 版本

想要创建一个像 [Spark 下载](http://spark.apache.org/downloads.html) 页面中的 Spark distribution 版本，并且使其能够运行，请使用项目 root 目录下的 `./dev/make-distribution.sh` 。它可以使用 Maven 的配置文件等等进行配置，如直接使用 Maven 构建。例如：

    ./dev/make-distribution.sh --name custom-spark --pip --r --tgz -Psparkr -Phadoop-2.7 -Phive -Phive-thriftserver -Pmesos -Pyarn

这将构建 Spark distribution 以及 Python pip 和 R 包。有关使用情况的更多信息，请运行 `./dev/make-distribution.sh --help`

## 指定 Hadoop 版本并启用 YARN

您可以通过 `hadoop.version` 属性指定要编译的 Hadoop 的确切版本。如果未指定， Spark 将默认构建为 Hadoop 2.6.X 。

您可以启用 `yarn` 配置文件，如果与 `hadoop.version` 不同的话，可以选择设置 `yarn.version` 属性。

示例：

    # Apache Hadoop 2.6.X
    ./build/mvn -Pyarn -DskipTests clean package

    # Apache Hadoop 2.7.X and later
    ./build/mvn -Pyarn -Phadoop-2.7 -Dhadoop.version=2.7.3 -DskipTests clean package

## 使用 Hive 和 JDBC 支持构建

要启用 Spark SQL 及其 JDBC server 和 CLI 的 Hive 集成，将 `-Phive` 和 `Phive-thriftserver` 配置文件添加到现有的构建选项中。
默认情况下， Spark 将使用 Hive 1.2.1 绑定构建。

    # With Hive 1.2.1 support
    ./build/mvn -Pyarn -Phive -Phive-thriftserver -DskipTests clean package

## 打包没有 Hadoop 依赖关系的 YARN

默认情况下，由 `mvn package` 生成的 assembly directory （组件目录）将包含所有的Spark 依赖，包括 Hadoop 及其一些生态系统项目。 在 YARN 部署中，导致这些的多个版本显示在执行器 classpaths 上： Spark 组件的打包的版本和每个节点上的版本，包含在 `yarn.application.classpath` 中。
`hadoop-provided` 配置文件构建了不包括 Hadoop 生态系统项目的组件，就像 ZooKeeper 和 Hadoop 本身。

## 使用 Mesos 构建

    ./build/mvn -Pmesos -DskipTests clean package

## 使用 Scala 2.10 构建
要使用 Scala 2.10 编译的 Spark 软件包，请使用 `-Dscala-2.10` 属性：

    ./dev/change-scala-version.sh 2.10
    ./build/mvn -Pyarn -Dscala-2.10 -DskipTests clean package

请注意，Scala 2.10 的支持已经不再适用于 Spark 2.1.0 ，并且可能在 Spark 2.2.0 中被删除。

## 单独构建子模块

可以使用 `mvn -pl` 选项来构建 Spark 的子模块。

例如，您可以使用下面打代码来构建 Spark Streaming 模块：

    ./build/mvn -pl :spark-streaming_2.11 clean install

其中 `spark-streaming_2.11` 是在 `streaming/pom.xml` 文件中定义的 `artifactId` 。

## Continuous Compilation（连续编译）

我们使用支持增量和连续编译的 scala-maven-plugin 插件。例如：

    ./build/mvn scala:cc

这里应该运行连续编译（即等待更改）。然而这并没有得到广泛的测试。有几个需要注意的事情：

* 它只扫描路径 `src/main` 和 `src/test` （查看 [docs](http://scala-tools.org/mvnsites/maven-scala-plugin/usage_cc.html)），所以其只对含有该结构的某些子模块起作用。

* 您通常需要在工程的根目录运行 `mvn install` 在编译某些特定子模块时。这是因为依赖于其他子模块的子模块需要通过 `spark-parent` 模块实现。

因此，运行 `core` 子模块的连续编译总流程会像下边这样：

    $ ./build/mvn install
    $ cd core
    $ ../build/mvn scala:cc

## 使用 SBT 构建

Maven 是推荐用于打包 Spark 的官方构建工具，是 *build of reference* 。但是 SBT 支持日常开发，因为它可以提供更快的迭代编译。更多高级的开发者可能希望使用 SBT 。SBT 构建是从 Maven POM 文件导出的，因此也是相同的 Maven 配置文件和变量可以设置来控制 SBT 构建。 例如：

    ./build/sbt package

为了避免在每次需要重新编译时启动 sbt 的开销，可以启动 sbt 在交互模式下运行 `build/sbt`，然后在命令中运行所有构建命令提示。

## 加速编译

经常编译 Spark 的开发人员可能希望加快编译速度; 例如通过使用 Zinc（对于使用 Maven 构建的开发人员）或避免重新编译组件 JAR （对于使用 SBT 构建的开发人员）。 有关如何执行此操作的更多信息，请参阅
[有用的开发工具页面](http://spark.apache.org/developer-tools.html#reducing-build-times) 。

## 加密文件系统

当在加密文件系统上构建时（例如，如果您的 home 目录是加密的），则 Spark 构建可能会失败，并导致 "Filename too long" 错误。 作为解决方法，在项目 `pom.xml` 中的 `scala-maven-plugin` 的配置参数中添加以下内容：

    <arg>-Xmax-classfile-name</arg>
    <arg>128</arg>

在 `project/SparkBuild.scala` 添加:

    scalacOptions in Compile ++= Seq("-Xmax-classfile-name", "128"),

到 `sharedSettings` val。 如果您不确定添加这些行的位置，请参阅 [this PR](https://github.com/apache/spark/pull/2883/files) 。

## IntelliJ IDEA 或 Eclipse

有关设置 IntelliJ IDEA 或 Eclipse 用于 Spark 开发和故障排除的帮助，请参阅 [有用的开发工具页面](http://spark.apache.org/developer-tools.html) 。


# 运行测试

默认情况下通过 [ScalaTest Maven plugin](http://www.scalatest.org/user_guide/using_the_scalatest_maven_plugin) 进行测试。请注意，测试不应该以 root 用户或者 admin 用户运行。

以下是运行测试的命令示例：

    ./build/mvn test

## 使用 SBT 测试

以下是运行测试的命令示例：

    ./build/sbt test

## 运行独立测试

有关如何运行独立测试的信息，请参阅 [有用的开发工具页面](http://spark.apache.org/developer-tools.html#running-individual-tests) 。

## PySpark pip 可安装

如果您正在构建 Spark 以在 Python 环境中使用，并且希望对其进行 pip 安装，那么您将首先需要如上所述构建 Spark JARs 。 然后，您可以构建适合于 setup.py 和 pip 可安装软件包的 sdist 软件包。

    cd python; python setup.py sdist

**注意:** 由于打包要求，您无法直接从 Python 目录中进行 pip 安装，而是必须首先按照上述方式构建 sdist 包。

或者，您还可以使用 --pip 选项运行 make-distribution 。

## 使用 Maven 进行 PySpark 测试

如果您正在构建 PySpark 并希望运行 PySpark 测试，则需要使用 Hive 支持构建 Spark 。

    ./build/mvn -DskipTests clean package -Phive
    ./python/run-tests

run-tests 脚本也可以限于特定的 Python 版本或者特定的模块。

    ./python/run-tests --python-executables=python --modules=pyspark-sql

**注意:** 您还可以使用 sbt 构建来运行 Python 测试，只要您使用 Hive 支持构建 Spark 。

## 运行 R 测试

要运行 SparkR 测试，您需要首先安装 [knitr](https://cran.r-project.org/package=knitr), [rmarkdown](https://cran.r-project.org/package=rmarkdown), [testthat](https://cran.r-project.org/package=testthat), [e1071](https://cran.r-project.org/package=e1071) and [survival](https://cran.r-project.org/package=survival) 包:

    R -e "install.packages(c('knitr', 'rmarkdown', 'testthat', 'e1071', 'survival'), repos='http://cran.us.r-project.org')"

您可以使用以下命令只运行 SparkR 测试：

    ./R/run-tests.sh

## 运行基于 Docker 的集成测试套装

为了运行 Docker 集成测试，你必须在你的 box 上安装 `docker` engine （引擎）。
有关安装说明，请参见[ Docker 站点](https://docs.docker.com/engine/installation/)。
一旦安装，如果还没有运行 Docker 服务，`docker` service 就需要启动。
在 Linux 上，这可以通过 `sudo service docker start` 来完成。

    ./build/mvn install -DskipTests
    ./build/mvn test -Pdocker-integration-tests -pl :spark-docker-integration-tests_2.11

或者

    ./build/sbt docker-integration-tests/test
