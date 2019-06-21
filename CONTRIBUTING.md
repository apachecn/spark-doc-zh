# 贡献指南

> 请您勇敢地去翻译和改进翻译。虽然我们追求卓越，但我们并不要求您做到十全十美，因此请不要担心因为翻译上犯错——在大部分情况下，我们的服务器已经记录所有的翻译，因此您不必担心会因为您的失误遭到无法挽回的破坏。（改编自维基百科）

* 为了使项目更加便于维护，我们将文档格式全部转换成了 Markdown，同时更换了页面生成器。后续维护工作将完全在 Markdown 上进行。
* 小部分格式仍然存在问题，主要是链接和表格。需要大家帮忙找到，并提 PullRequest 来修复。

可能有用的链接：

* [2.0.2 中文版](http://spark.apachecn.org)
* [历史版本: Apache Spark 2.0.2 官方文档中文版](http://cwiki.apachecn.org/pages/viewpage.action?pageId=2883613)

负责人：

* [@965](https://github.com/wangweitong): 1097828409

## 章节列表

+   [Spark 概述](docs/1.md)
+   [编程指南](docs/2.md)
    +   [快速入门](docs/3.md)
    +   [Spark 编程指南](docs/4.md)
    +   [构建在 Spark 之上的模块](docs/5.md)
        +   [Spark Streaming 编程指南](docs/6.md)
        +   [Spark SQL, DataFrames and Datasets Guide](docs/7.md)
        +   [MLlib](docs/8.md)
        +   [GraphX Programming Guide](docs/9.md)
+   [API 文档](docs/10.md)
+   [部署指南](docs/11.md)
    +   [集群模式概述](docs/12.md)
    +   [Submitting Applications](docs/13.md)
    +   [部署模式](docs/14.md)
        +   [Spark Standalone Mode](docs/15.md)
        +   [在 Mesos 上运行 Spark](docs/16.md)
        +   [Running Spark on YARN](docs/17.md)
        +   [其它](docs/18.md)
+   [更多](docs/19.md)
    +   [Spark 配置](docs/20.md)
    +   [Monitoring and Instrumentation](docs/21.md)
    +   [Tuning Spark](docs/22.md)
    +   [作业调度](docs/23.md)
    +   [Spark 安全](docs/24.md)
    +   [硬件配置](docs/25.md)
    +   [Accessing OpenStack Swift from Spark](docs/26.md)
    +   [构建 Spark](docs/27.md)
    +   [其它](docs/28.md)
+   [外部资源](docs/29.md)
+   [Spark RDD（Resilient Distributed Datasets）论文](docs/paper.md)
+   [翻译进度](docs/30.md)

## 流程

### 一、认领

首先查看[整体进度](https://github.com/apachecn/spark-doc-zh/issues/189)，确认没有人认领了你想认领的章节。
 
然后回复 ISSUE，注明“章节 + QQ 号”（一定要留 QQ）。

### 二、翻译

可以合理利用翻译引擎（例如[谷歌](https://translate.google.cn/)），但一定要把它变得可读！

可以参照之前版本的中文文档，如果有用的话。

如果遇到格式问题，请随手把它改正。

### 三、提交

**提交的时候不要改动文件名称，即使它跟章节标题不一样也不要改，因为文件名和原文的链接是对应的！！！**

+   `fork` Github 项目
+   将译文放在`docs/2.0.2`文件夹下
+   `push`
+   `pull request`

请见 [Github 入门指南](https://github.com/apachecn/kaggle/blob/master/docs/GitHub)。


