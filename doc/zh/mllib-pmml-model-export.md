---
layout: global
title: PMML model export - RDD-based API
displayTitle: PMML model export - RDD-based API
---

* Table of contents
{:toc}

## `spark.mllib` 支持的模型

`spark.mllib` 支持模型导出到预测模型标记语言 ([PMML](http://en.wikipedia.org/wiki/Predictive_Model_Markup_Language))。

下表列出了 `spark.mllib` 可以导出到PMML的模型及其等效的PMML模型。

<table class="table">
  <thead>
    <tr><th>`spark.mllib` 模型</th><th>PMML 模型</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>KMeansModel</td><td>ClusteringModel</td>
    </tr>    
    <tr>
      <td>LinearRegressionModel</td><td>RegressionModel (functionName="regression")</td>
    </tr>
    <tr>
      <td>RidgeRegressionModel</td><td>RegressionModel (functionName="regression")</td>
    </tr>
    <tr>
      <td>LassoModel</td><td>RegressionModel (functionName="regression")</td>
    </tr>
    <tr>
      <td>SVMModel</td><td>RegressionModel (functionName="classification" normalizationMethod="none")</td>
    </tr>
    <tr>
      <td>Binary LogisticRegressionModel</td><td>RegressionModel (functionName="classification" normalizationMethod="logit")</td>
    </tr>
  </tbody>
</table>

## 示例

<div class="codetabs">

<div data-lang="scala" markdown="1">

要将支持的 `模型`（见上表）导出到PMML，只需调用 `model.toPMML`。

除了将 PMML 模型导出为 String（ `model.toPMML` 如上例所示），您可以将 PMML 模型导出为其他格式。

有关 API 的详细信息，请参阅 [`KMeans` Scala 文档](api/scala/index.html#org.apache.spark.mllib.clustering.KMeans) 和 [VectorsScala文档](api/scala/index.html#org.apache.spark.mllib.linalg.Vectors$)。

这里是一个建立 KMeansModel 并以 PMML 格式打印出来的完整示例：

{% include_example scala/org/apache/spark/examples/mllib/PMMLModelExportExample.scala %}

对于不支持的模型，您将无法找到 `.toPMML` 方法或 `IllegalArgumentException` 抛出异常。

</div>

</div>
