---
layout: global
displayTitle: Spark 安全
title: 安全
---

Spark 当前支持使用 shared secret（共享密钥）来 authentication（认证）。可以通过配置 `spark.authenticate` 参数来开启认证。这个参数用来控制 Spark 的通讯协议是否使用共享密钥来执行认证。这个认证是一个基本的握手，用来保证两侧具有相同的共享密钥然后允许通讯。如果共享密钥无法识别，他们将不能通讯。创建共享密钥的方法如下:

* 对于 Spark on [YARN](running-on-yarn.html)  的部署方式，配置 `spark.authenticate` 为 `true`，会自动生成和分发共享密钥。每个 application 会使用一个单独的共享密钥.
* 对于 Spark 的其他部署方式, 需要在每一个 nodes 配置 `spark.authenticate.secret` 参数. 这个密码将会被所有的 Master/Workers 和 aplication 使用.

## Web UI

Spark UI 可以通过 `spark.ui.filters` 设置使用 [javax servlet filters](http://docs.oracle.com/javaee/6/api/javax/servlet/Filter.html) 和采用通过 [SSL settings](security.html#ssl-configuration) 启用 [https/SSL](http://en.wikipedia.org/wiki/HTTPS) 来进行安全保护。

### 认证

当 UI 中有一些不允许其他用户看到的数据时，用户可能想对 UI 进行安全防护。用户指定的 javax servlet filter 可以对用户进行身份认证，用户一旦登入，Spark 可以比较与这个用户相对应的视图 ACLs 来确认是否授权用户访问 UI。配置项 `spark.acls.enable`,`spark.ui.view.acls` 和 `spark.ui.view.acls.groups` 控制 ACLs 的行为。注意，启动 application 的用户具有使用 UI 视图的访问权限。在 YARN 上，Spark UI 使用标准的 YARN web application 代理机制可以通过任何已安装的 Hadoop filters 进行认证。

Spark 也支持修改 ACLs 来控制谁具有修改一个正在运行的 Spark application 的权限。这些权限包括 kill 这个 application 或者一个 task。以上功能通过配置 `spark.acls.enable`，`spark.modify.acls` 和 `spark.modify.acls.groups` 来控制。注意如果你已经认证登入 UI，为了使用 UI 上的 kill 按钮，必须先添加这个用户到 modify acls 和 view acls。在 YARN 上，modify acls 通过 YARN 接口传入并控制谁具有 modify 权限。Spark 允许在 acls 中指定多个对所有的 application 都具有 view 和 modify 权限的管理员，通过选项 `spark.admin.acls` 和 `spark.admin.acls.groups` 来控制。这样做对于一个有帮助用户调试 applications 的管理员或者支持人员的共享集群，非常有帮助。

## 事件日志

如果你的 applications 使用事件日志，事件日志的的位置必须手动创建同时具有适当的访问权限（`spark.eventLog.dir`）。如果想对这些日志文件做安全保护，目录的权限应该设置为 `drwxrwxrwxt`。目录的属主应该是启动 history server 的高级用户，同时组权限应该仅限于这个高级用户组。这将允许所有的用户向目录中写入文件，同时防止删除不是属于自己的文件（权限字符t为防删位）。事件日志将会被 Spark 创建，只有特定的用户才就有读写权限。


## 加密

Spark 支持 HTTP SSL 协议。SASL 加密用于块传输服务和 RPC 端。也可以加密 Shuffle 文件。

### SSL 配置

SSL 配置是分级组织的。用户可以对所有的通讯协议配置默认的 SSL 设置，除非被特定协议的设置覆盖掉。这样用户可以很容易的为所有的协议提供通用的设置，无需禁用每个单独配置的能力。通用的 SSL 设置在 Spark 配置文件的 `spark.ssl` 命名空间中。下表描述了用于覆盖特定组件默认配置的命名空间:

<table class="table">
  <tr>
    <th>Config Namespace（配置空间）</th>
    <th>Component（组件）</th>
  </tr>
  <tr>
    <td><code>spark.ssl.fs</code></td>
    <td>文件下载客户端 (用于从启用了 HTTPS 的服务器中下载 jars 和 files).</td>
  </tr>
  <tr>
    <td><code>spark.ssl.ui</code></td>
    <td>Spark application Web UI</td>
  </tr>
  <tr>
    <td><code>spark.ssl.standalone</code></td>
    <td>Standalone Master / Worker Web UI</td>
  </tr>
  <tr>
    <td><code>spark.ssl.historyServer</code></td>
    <td>History Server Web UI</td>
  </tr>
</table>


所有的SSL选项可以在 [配置页面](configuration.html) 里查询到。SSL 必须在所有的节点进行配置，对每个使用特定协议通讯的组件生效。

### YARN mode
key-store 可以在客户端侧准备好，然后作为 application 的一部分通过 executor 进行分发和使用。因为用户能够通过使用 `spark.yarn.dist.files` 或者 `spark.yarn.dis.archives` 配置选项，在 YARN上 启动 application 之前部署文件。加密和传输这些文件的责任在 YARN 一侧，与 Spark 无关。

为了一个长期运行的应用比如 Spark Streaming 应用能够写 HDFS，需要通过分别设置 `--principal` 和 `--keytab` 参数传递 principal 和 keytab 给 `spark-submit`。传入的 keytab 会通过 Hadoop Distributed Cache 拷贝到运行 Application 的 Master（这是安全的 - 如果 YARN 使用 SSL 配置同时启用了 HDFS 加密）。Kerberos 会使用这个 principal 和 keytab 周期的更新登陆信息，同时 HDFS 需要的授权 tokens 会周期生成，因此应用可以持续的写入 HDFS。

### Standalone mode
用户需要为 master 和 slaves 提供 key-stores 和相关配置选项。必须通过附加恰当的 Java 系统属性到 `SPARK_MASTER_OPTS` 和 `SPARK_WORKER_OPTS` 环境变量中，或者只在 `SPARK_DAEMON_JAVA_OPTS` 中。在这种模式下，用户可以允许 executors 使用从派生它的 worker 继承来的 SSL 设置。可以通过设置 `spark.ssl.useNodeLocalConf` 为 `true` 来完成。如果设置这个参数，用户在客户端侧提供的设置不会被 executors 使用。

### 准备 key-stores
可以使用 `keytool` 程序生成 key-stores。这个工具的使用说明在 [这里](https://docs.oracle.com/javase/7/docs/technotes/tools/solaris/keytool.html)。为 standalone 部署方式配置 key-stores 和 trust-store 最基本的步骤如下:

* 为每个 node 生成 keys pair
* 在每个 node 上导出 key pair 的 public key（公钥）到一个文件中
* 导入以上生成的所有的公钥到一个单独的 trust-store
* 在所有的节点上分发 trust-store

### 配置 SASL 加密

当认证（`sprk.authenticate`）开启的时候，块传输支持 SASL 加密。通过在 application 配置中设置 `spark.authenticate.enableSaslEncryption` 为 `true` 来开启 SASL 加密。

当使用外部 shuffle 服务，需要在 shuffle 服务配置中通过设置 `spark.network.sasl.serverAlwaysEncrypt` 为 `true` 来禁用未加密的连接。如果这个选项开启，未设置使用 SASL 加密的 application 在链接到 shuffer 服务的时候会失败。

## 针对网络安全配置端口

Spark 严重依赖 network，同时一些环境对使用防火墙有严格的要求。下面使一些 Spark 网络通讯使用的主要的端口和配置方式.

### Standalone mode only

<table class="table">
  <tr>
    <th>From</th><th>To</th><th>Default Port（默认端口）</th><th>Purpose（目的）</th><th>Configuration
    Setting（配置设置）</th><th>Notes（注意）</th>
  </tr>
  <tr>
    <td>Browser</td>
    <td>Standalone Master</td>
    <td>8080</td>
    <td>Web UI</td>
    <td><code>spark.master.ui.port /<br> SPARK_MASTER_WEBUI_PORT</code></td>
    <td>Jetty-based. Standalone mode only.</td>
  </tr>
  <tr>
    <td>Browser</td>
    <td>Standalone Worker</td>
    <td>8081</td>
    <td>Web UI</td>
    <td><code>spark.worker.ui.port /<br> SPARK_WORKER_WEBUI_PORT</code></td>
    <td>Jetty-based. Standalone mode only.</td>
  </tr>
  <tr>
    <td>Driver /<br> Standalone Worker</td>
    <td>Standalone Master</td>
    <td>7077</td>
    <td>Submit job to cluster /<br> Join cluster</td>
    <td><code>SPARK_MASTER_PORT</code></td>
    <td>Set to "0" to choose a port randomly. Standalone mode only.</td>
  </tr>
  <tr>
    <td>Standalone Master</td>
    <td>Standalone Worker</td>
    <td>(random)</td>
    <td>Schedule executors</td>
    <td><code>SPARK_WORKER_PORT</code></td>
    <td>Set to "0" to choose a port randomly. Standalone mode only.</td>
  </tr>
</table>

### All cluster managers

<table class="table">
  <tr>
    <th>From</th><th>To</th><th>Default Port（默认端口）</th><th>Purpose（目的）</th><th>Configuration
    Setting（配置设置）</th><th>Notes（注意）</th>
  </tr>
  <tr>
    <td>Browser</td>
    <td>Application</td>
    <td>4040</td>
    <td>Web UI</td>
    <td><code>spark.ui.port</code></td>
    <td>Jetty-based</td>
  </tr>
  <tr>
    <td>Browser</td>
    <td>History Server</td>
    <td>18080</td>
    <td>Web UI</td>
    <td><code>spark.history.ui.port</code></td>
    <td>Jetty-based</td>
  </tr>
  <tr>
    <td>Executor /<br> Standalone Master</td>
    <td>Driver</td>
    <td>(random)</td>
    <td>Connect to application /<br> Notify executor state changes</td>
    <td><code>spark.driver.port</code></td>
    <td>Set to "0" to choose a port randomly.</td>
  </tr>
  <tr>
    <td>Executor / Driver</td>
    <td>Executor / Driver</td>
    <td>(random)</td>
    <td>Block Manager port</td>
    <td><code>spark.blockManager.port</code></td>
    <td>Raw socket via ServerSocketChannel</td>
  </tr>
</table>


更多关于安全配置参数方面的细节请参考 [配置页面](configuration.html)。安全方面的实施细节参考 <a href="{{site.SPARK_GITHUB_URL}}/tree/master/core/src/main/scala/org/apache/spark/SecurityManager.scala">
<code>org.apache.spark.SecurityManager</code></a>.

