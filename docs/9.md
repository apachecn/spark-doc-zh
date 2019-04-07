# GraphX Programming Guide

*   [概述](#概述)
*   [入门](#入门)
*   [属性 Graph](#属性-graph)
    *   [示例属性 Graph](#示例属性-graph)
*   [Graph 运算符](#graph-运算符)
    *   [运算符的汇总表](#运算符的汇总表)
    *   [Property 运算符](#property-运算符)
    *   [Structural 运算符](#structural-运算符)
    *   [Join 运算符](#join-运算符)
    *   [邻域聚合](#邻域聚合)
        *   [聚合消息（aggregateMessages）](#聚合消息-aggregatemessages)
        *   [Map Reduce Triplets Transition Guide (Legacy)](#map-reduce-triplets-transition-guide-legacy)
        *   [计算级别信息](#计算级别信息)
        *   [收集相邻点](#收集相邻点)
    *   [Caching and Uncaching](#caching-and-uncaching)
*   [Pregel API](#pregel-api)
*   [Graph 建造者](#graph-建造者)
*   [Vertex and Edge RDDs](#vertex-and-edge-rdds)
    *   [VertexRDDs](#vertexrdds)
    *   [EdgeRDDs](#edgerdds)
*   [优化表示](#优化表示)
*   [Graph 算法](#graph-算法)
    *   [PageRank](#pagerank)
    *   [连接组件](#连接组件)
    *   [Triangle 计数](#triangle-计数)
*   [示例](#示例)

![GraphX](img/c3e273c3c14008f2f2e44d4effc09865.jpg "GraphX Logo") 

# 概述

GraphX 是 Spark 中用于图形和图形并行计算的新组件。在高层次上，GraphX 通过引入一个新的[图形](#property_graph)抽象来扩展 Spark [RDD](api/scala/index.html#org.apache.spark.rdd.RDD)：一种具有附加到每个顶点和边缘的属性的定向多重图形。为了支持图计算，GraphX 公开了一组基本运算符（例如： [subgraph](#structural_operators)，[joinVertices](#join_operators) 和 [aggregateMessages](#aggregateMessages)）以及 [Pregel](#pregel) API 的优化变体。此外，GraphX 还包括越来越多的图形[算法](#graph_algorithms) 和 [构建器](#graph_builders)，以简化图形分析任务。

# 入门

首先需要将 Spark 和 GraphX 导入到项目中，如下所示：

```
import org.apache.spark._
import org.apache.spark.graphx._
// To make some of the examples work we will also need RDD
import org.apache.spark.rdd.RDD
```

如果您不使用 Spark 外壳，您还需要一个 `SparkContext`。要了解有关 Spark 入门的更多信息，请参考 [Spark快速入门指南](quick-start.html)。

# 属性 Graph

[属性 Graph](api/scala/index.html#org.apache.spark.graphx.Graph) 是一个定向多重图形，用户定义的对象附加到每个顶点和边缘。定向多图是具有共享相同源和目标顶点的潜在多个平行边缘的有向图。支持平行边缘的能力简化了在相同顶点之间可以有多个关系（例如： 同事和朋友）的建模场景。每个顶点都由唯一的 64 位长标识符（`VertexId`）键入。GraphX 不对顶点标识符施加任何排序约束。类似地，边缘具有对应的源和目标顶点标识符。

属性图是通过 vertex（`VD`）和 edge（`ED`）类型进行参数化的。这些是分别与每个顶点和边缘相关联的对象的类型。

> 当它们是原始数据类型（例如： int，double 等等）时，GraphX 优化顶点和边缘类型的表示，通过将其存储在专门的数组中来减少内存占用。

在某些情况下，可能希望在同一个图形中具有不同属性类型的顶点。这可以通过继承来实现。例如，将用户和产品建模为二分图，我们可能会执行以下操作：

```
class VertexProperty()
case class UserProperty(val name: String) extends VertexProperty
case class ProductProperty(val name: String, val price: Double) extends VertexProperty
// The graph might then have the type:
var graph: Graph[VertexProperty, String] = null
```

像 RDD 一样，属性图是不可变的，分布式的和容错的。通过生成具有所需更改的新图形来完成对图表的值或结构的更改。请注意，原始图形的大部分（即，未受影响的结构，属性和索引）在新图表中重复使用，可降低此内在功能数据结构的成本。使用一系列顶点分割启发式方法，在执行器之间划分图形。与 RDD 一样，在发生故障的情况下，可以在不同的机器上重新创建图形的每个分区。

逻辑上，属性图对应于一对编码每个顶点和边缘的属性的类型集合（RDD）。因此，图类包含访问图形顶点和边的成员：

```
class Graph[VD, ED] {
  val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED]
}
```

`VertexRDD[VD]` 和 `EdgeRDD[ED]` 分别扩展了 `RDD[(VertexId, VD)]` 和 `RDD[Edge[ED]]` 的优化版本。`VertexRDD[VD]` 和 `EdgeRDD[ED]` 都提供了围绕图计算和利用内部优化的附加功能。我们在[顶点和边缘 RDD](#vertex_and_edge_rdds) 部分更详细地讨论了 [`VertexRDD`](api/scala/index.html#org.apache.spark.graphx.VertexRDD) 和 [`EdgeRDD`](api/scala/index.html#org.apache.spark.graphx.EdgeRDD) API，但现在它们可以被认为是 `RDD[(VertexId, VD)]` 和 `RDD[Edge[ED]]` 的简单 RDD。

### 示例属性 Graph

假设我们要构建一个由 GraphX 项目中的各种协作者组成的属性图。顶点属性可能包含用户名和职业。我们可以用描述协作者之间关系的字符串来注释边：

![The Property Graph](img/6953ce1edbabb581001c1b124db1d69d.jpg "The Property Graph") 

生成的图形将具有类型签名：

```
val userGraph: Graph[(String, String), String]
```

从原始文件，RDD 甚至合成生成器构建属性图有许多方法，这些在[图形构建器](#graph_builders)的一节中有更详细的讨论。最普遍的方法是使用 [Graph 对象](api/scala/index.html#org.apache.spark.graphx.Graph$)。例如，以下代码从 RDD 集合中构建一个图：

```
// Assume the SparkContext has already been constructed
val sc: SparkContext
// Create an RDD for the vertices
val users: RDD[(VertexId, (String, String))] =
  sc.parallelize(Array((3L, ("rxin", "student")), (7L, ("jgonzal", "postdoc")),
                       (5L, ("franklin", "prof")), (2L, ("istoica", "prof"))))
// Create an RDD for edges
val relationships: RDD[Edge[String]] =
  sc.parallelize(Array(Edge(3L, 7L, "collab"),    Edge(5L, 3L, "advisor"),
                       Edge(2L, 5L, "colleague"), Edge(5L, 7L, "pi")))
// Define a default user in case there are relationship with missing user
val defaultUser = ("John Doe", "Missing")
// Build the initial Graph
val graph = Graph(users, relationships, defaultUser)
```

在上面的例子中，我们使用了 [`Edge`](api/scala/index.html#org.apache.spark.graphx.Edge) case 类。边缘具有 `srcId` 和 `dstId` 对应于源和目标顶点标识符。此外，`Edge` 该类有一个 `attr` 存储边缘属性的成员。

我们可以分别使用 `graph.vertices` 和 `graph.edges` 成员将图形解构成相应的顶点和边缘视图。

```
val graph: Graph[(String, String), String] // Constructed from above
// Count all users which are postdocs
graph.vertices.filter { case (id, (name, pos)) => pos == "postdoc" }.count
// Count all the edges where src > dst
graph.edges.filter(e => e.srcId > e.dstId).count
```

> 注意，`graph.vertices` 返回一个 `VertexRDD[(String, String)]` 扩展 `RDD[(VertexId, (String, String))]`，所以我们使用 scala `case` 表达式来解构元组。另一方面，`graph.edges` 返回一个 `EdgeRDD` 包含 `Edge[String]` 对象。我们也可以使用 case 类型构造函数，如下所示：

```
graph.edges.filter { case Edge(src, dst, prop) => src > dst }.count
```

除了属性图的顶点和边缘视图之外，GraphX 还暴露了三元组视图。三元组视图逻辑上连接顶点和边缘属性，生成 `RDD[EdgeTriplet[VD, ED]]` 包含 [`EdgeTriplet`](api/scala/index.html#org.apache.spark.graphx.EdgeTriplet) 该类的实例。此 连接可以用以下SQL表达式表示：

```
SELECT src.id, dst.id, src.attr, e.attr, dst.attr
FROM edges AS e LEFT JOIN vertices AS src, vertices AS dst
ON e.srcId = src.Id AND e.dstId = dst.Id
```

或图形为：

![Edge Triplet](img/a0f9441a5c59b2607cccf63ae2887065.jpg "Edge Triplet") 

[`EdgeTriplet`](api/scala/index.html#org.apache.spark.graphx.EdgeTriplet) 类通过分别添加包含源和目标属性的 `srcAttr` 和 `dstAttr` 成员来扩展 [`Edge`](api/scala/index.html#org.apache.spark.graphx.Edge) 类。我们可以使用图形的三元组视图来渲染描述用户之间关系的字符串集合。

```
val graph: Graph[(String, String), String] // Constructed from above
// Use the triplets view to create an RDD of facts.
val facts: RDD[String] =
  graph.triplets.map(triplet =>
    triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1)
facts.collect.foreach(println(_))
```

# Graph 运算符

正如 RDDs 有这样的基本操作 `map`，`filter`，以及 `reduceByKey`，性能图表也有采取用户定义的函数基本运算符的集合，产生具有转化特性和结构的新图。定义了优化实现的核心运算符，并定义了 [`Graph`](api/scala/index.html#org.apache.spark.graphx.Graph) 表示为核心运算符组合的方便运算符 [`GraphOps`](api/scala/index.html#org.apache.spark.graphx.GraphOps)。不过，由于 Scala 的含义，操作员 `GraphOps` 可自动作为成员使用 `Graph`。例如，我们可以通过以下方法计算每个顶点的入度（定义 `GraphOps`）：

```
val graph: Graph[(String, String), String]
// Use the implicit GraphOps.inDegrees operator
val inDegrees: VertexRDD[Int] = graph.inDegrees
```

区分核心图形操作的原因 [`GraphOps`](api/scala/index.html#org.apache.spark.graphx.GraphOps) 是能够在将来支持不同的图形表示。每个图形表示必须提供核心操作的实现，并重用许多有用的操作 [`GraphOps`](api/scala/index.html#org.apache.spark.graphx.GraphOps)。

### 运算符的汇总表

以下是两个定义的功能的简要摘要，但为简单起见 [`Graph`](api/scala/index.html#org.apache.spark.graphx.Graph)，[`GraphOps`](api/scala/index.html#org.apache.spark.graphx.GraphOps) 它作为 Graph 的成员呈现。请注意，已经简化了一些功能签名（例如，删除了默认参数和类型约束），并且已经删除了一些更高级的功能，因此请参阅 API 文档以获取正式的操作列表。

```
/** Summary of the functionality in the property graph */
class Graph[VD, ED] {
  // Information about the Graph ===================================================================
  val numEdges: Long
  val numVertices: Long
  val inDegrees: VertexRDD[Int]
  val outDegrees: VertexRDD[Int]
  val degrees: VertexRDD[Int]
  // Views of the graph as collections =============================================================
  val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED]
  val triplets: RDD[EdgeTriplet[VD, ED]]
  // Functions for caching graphs ==================================================================
  def persist(newLevel: StorageLevel = StorageLevel.MEMORY_ONLY): Graph[VD, ED]
  def cache(): Graph[VD, ED]
  def unpersistVertices(blocking: Boolean = true): Graph[VD, ED]
  // Change the partitioning heuristic  ============================================================
  def partitionBy(partitionStrategy: PartitionStrategy): Graph[VD, ED]
  // Transform vertex and edge attributes ==========================================================
  def mapVertices[VD2](map: (VertexId, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapEdges[ED2](map: (PartitionID, Iterator[Edge[ED]]) => Iterator[ED2]): Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]
  def mapTriplets[ED2](map: (PartitionID, Iterator[EdgeTriplet[VD, ED]]) => Iterator[ED2])
    : Graph[VD, ED2]
  // Modify the graph structure ====================================================================
  def reverse: Graph[VD, ED]
  def subgraph(
      epred: EdgeTriplet[VD,ED] => Boolean = (x => true),
      vpred: (VertexId, VD) => Boolean = ((v, d) => true))
    : Graph[VD, ED]
  def mask[VD2, ED2](other: Graph[VD2, ED2]): Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD, ED]
  // Join RDDs with the graph ======================================================================
  def joinVertices[U](table: RDD[(VertexId, U)])(mapFunc: (VertexId, VD, U) => VD): Graph[VD, ED]
  def outerJoinVertices[U, VD2](other: RDD[(VertexId, U)])
      (mapFunc: (VertexId, VD, Option[U]) => VD2)
    : Graph[VD2, ED]
  // Aggregate information about adjacent triplets =================================================
  def collectNeighborIds(edgeDirection: EdgeDirection): VertexRDD[Array[VertexId]]
  def collectNeighbors(edgeDirection: EdgeDirection): VertexRDD[Array[(VertexId, VD)]]
  def aggregateMessages[Msg: ClassTag](
      sendMsg: EdgeContext[VD, ED, Msg] => Unit,
      mergeMsg: (Msg, Msg) => Msg,
      tripletFields: TripletFields = TripletFields.All)
    : VertexRDD[A]
  // Iterative graph-parallel computation ==========================================================
  def pregel[A](initialMsg: A, maxIterations: Int, activeDirection: EdgeDirection)(
      vprog: (VertexId, VD, A) => VD,
      sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexId,A)],
      mergeMsg: (A, A) => A)
    : Graph[VD, ED]
  // Basic graph algorithms ========================================================================
  def pageRank(tol: Double, resetProb: Double = 0.15): Graph[Double, Double]
  def connectedComponents(): Graph[VertexId, ED]
  def triangleCount(): Graph[Int, ED]
  def stronglyConnectedComponents(numIter: Int): Graph[VertexId, ED]
}
```

## Property 运算符

与 RDD `map` 运算符一样，属性图包含以下内容：

```
class Graph[VD, ED] {
  def mapVertices[VD2](map: (VertexId, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]
}
```

这些运算符中的每一个产生一个新的图形，其中顶点或边缘属性被用户定义的 `map` 函数修改。

> 请注意，在每种情况下，图形结构都不受影响。这是这些运算符的一个关键特征，它允许生成的图形重用原始图形的结构索引。以下代码段在逻辑上是等效的，但是第一个代码片段不保留结构索引，并且不会从 GraphX 系统优化中受益：

```
val newVertices = graph.vertices.map { case (id, attr) => (id, mapUdf(id, attr)) }
val newGraph = Graph(newVertices, graph.edges)
```

> 而是 [`mapVertices`](api/scala/index.html#org.apache.spark.graphx.Graph@mapVertices[VD2]((VertexId,VD)⇒VD2)(ClassTag[VD2]):Graph[VD2,ED]) 用来保存索引：

```
val newGraph = graph.mapVertices((id, attr) => mapUdf(id, attr))
```

这些运算符通常用于初始化特定计算或项目的图形以避免不必要的属性。例如，给出一个以度为顶点属性的图（我们稍后将描述如何构建这样一个图），我们为PageRank初始化它：

```
// Given a graph where the vertex property is the out degree
val inputGraph: Graph[Int, String] =
  graph.outerJoinVertices(graph.outDegrees)((vid, _, degOpt) => degOpt.getOrElse(0))
// Construct a graph where each edge contains the weight
// and each vertex is the initial PageRank
val outputGraph: Graph[Double, Double] =
  inputGraph.mapTriplets(triplet => 1.0 / triplet.srcAttr).mapVertices((id, _) => 1.0)
```

## Structural 运算符

目前GraphX只支持一套简单的常用结构运算符，我们预计将来会增加更多。以下是基本结构运算符的列表。

```
class Graph[VD, ED] {
  def reverse: Graph[VD, ED]
  def subgraph(epred: EdgeTriplet[VD,ED] => Boolean,
               vpred: (VertexId, VD) => Boolean): Graph[VD, ED]
  def mask[VD2, ED2](other: Graph[VD2, ED2]): Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD,ED]
}
```

该 [`reverse`](api/scala/index.html#org.apache.spark.graphx.Graph@reverse:Graph[VD,ED]) 运算符将返回逆转的所有边缘方向上的新图。这在例如尝试计算逆 PageRank 时是有用的。由于反向操作不会修改顶点或边缘属性或更改边缘数量，因此可以在没有数据移动或重复的情况下高效地实现。

在 [`subgraph`](api/scala/index.html#org.apache.spark.graphx.Graph@subgraph((EdgeTriplet[VD,ED])⇒Boolean,(VertexId,VD)⇒Boolean):Graph[VD,ED]) 操作者需要的顶点和边缘的谓词，并返回包含只有满足谓词顶点的顶点的曲线图（评估为真），并且满足谓词边缘边缘 _并连接满足顶点谓词顶点_。所述 `subgraph` 操作员可在情况编号被用来限制图形以顶点和感兴趣的边缘或消除断开的链接。例如，在以下代码中，我们删除了断开的链接：

```
// Create an RDD for the vertices
val users: RDD[(VertexId, (String, String))] =
  sc.parallelize(Array((3L, ("rxin", "student")), (7L, ("jgonzal", "postdoc")),
                       (5L, ("franklin", "prof")), (2L, ("istoica", "prof")),
                       (4L, ("peter", "student"))))
// Create an RDD for edges
val relationships: RDD[Edge[String]] =
  sc.parallelize(Array(Edge(3L, 7L, "collab"),    Edge(5L, 3L, "advisor"),
                       Edge(2L, 5L, "colleague"), Edge(5L, 7L, "pi"),
                       Edge(4L, 0L, "student"),   Edge(5L, 0L, "colleague")))
// Define a default user in case there are relationship with missing user
val defaultUser = ("John Doe", "Missing")
// Build the initial Graph
val graph = Graph(users, relationships, defaultUser)
// Notice that there is a user 0 (for which we have no information) connected to users
// 4 (peter) and 5 (franklin).
graph.triplets.map(
  triplet => triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1
).collect.foreach(println(_))
// Remove missing vertices as well as the edges to connected to them
val validGraph = graph.subgraph(vpred = (id, attr) => attr._2 != "Missing")
// The valid subgraph will disconnect users 4 and 5 by removing user 0
validGraph.vertices.collect.foreach(println(_))
validGraph.triplets.map(
  triplet => triplet.srcAttr._1 + " is the " + triplet.attr + " of " + triplet.dstAttr._1
).collect.foreach(println(_))
```

> 注意在上面的例子中只提供了顶点谓词。如果未提供顶点或边缘谓词，则 `subgraph` 运算符默认为 `true`。

在 [`mask`](api/scala/index.html#org.apache.spark.graphx.Graph@mask[VD2,ED2](Graph[VD2,ED2])(ClassTag[VD2],ClassTag[ED2]):Graph[VD,ED]) 操作者通过返回包含该顶点和边，它们也在输入图形中发现的曲线构造一个子图。这可以与 `subgraph` 运算符一起使用，以便根据另一个相关图中的属性限制图形。例如，我们可以使用缺少顶点的图运行连接的组件，然后将答案限制为有效的子图。

```
// Run Connected Components
val ccGraph = graph.connectedComponents() // No longer contains missing field
// Remove missing vertices as well as the edges to connected to them
val validGraph = graph.subgraph(vpred = (id, attr) => attr._2 != "Missing")
// Restrict the answer to the valid subgraph
val validCCGraph = ccGraph.mask(validGraph)
```

[`groupEdges`](api/scala/index.html#org.apache.spark.graphx.Graph@groupEdges((ED,ED)⇒ED):Graph[VD,ED]) 操作符将多边形中的平行边（即，顶点对之间的重复边）合并。在许多数值应用中，可以将平行边缘（它们的权重组合）合并成单个边缘，从而减小图形的大小。

## Join 运算符

在许多情况下，有必要使用图形连接来自外部收集（RDD）的数据。例如，我们可能有额外的用户属性，我们要与现有的图形合并，或者我们可能希望将顶点属性从一个图形拉到另一个。这些任务可以使用 _join_ 运算符完成。下面我们列出关键 join 运算符：

```
class Graph[VD, ED] {
  def joinVertices[U](table: RDD[(VertexId, U)])(map: (VertexId, VD, U) => VD)
    : Graph[VD, ED]
  def outerJoinVertices[U, VD2](table: RDD[(VertexId, U)])(map: (VertexId, VD, Option[U]) => VD2)
    : Graph[VD2, ED]
}
```

[`joinVertices`](api/scala/index.html#org.apache.spark.graphx.GraphOps@joinVertices[U](RDD[(VertexId,U)])((VertexId,VD,U)⇒VD)(ClassTag[U]):Graph[VD,ED]) 操作符将顶点与输入 RDD 相连，并返回一个新的图形，其中通过将用户定义的 `map` 函数应用于已连接顶点的结果而获得的顶点属性。RDD 中没有匹配值的顶点保留其原始值。

> 请注意，如果 RDD 包含给定顶点的多个值，则只能使用一个值。因此，建议使用以下命令使输入 RDD 变得独一无二，这也将对结果值进行 _pre-index_，以显着加速后续连接。

```
val nonUniqueCosts: RDD[(VertexId, Double)]
val uniqueCosts: VertexRDD[Double] =
  graph.vertices.aggregateUsingIndex(nonUnique, (a,b) => a + b)
val joinedGraph = graph.joinVertices(uniqueCosts)(
  (id, oldCost, extraCost) => oldCost + extraCost)
```

除了将用户定义的 `map` 函数应用于所有顶点并且可以更改顶点属性类型之外，更一般的 [`outerJoinVertices`](api/scala/index.html#org.apache.spark.graphx.Graph@outerJoinVertices[U,VD2](RDD[(VertexId,U)])((VertexId,VD,Option[U])⇒VD2)(ClassTag[U],ClassTag[VD2]):Graph[VD2,ED]) 的行为类似于 `joinVertices`。因为不是所有的顶点都可能在输入 RDD 中具有匹配的值，所以 `map` 函数采用 `Option` 类型。例如，我们可以通过使用 `outDegree` 初始化顶点属性来为 PageRank 设置一个图。

```
val outDegrees: VertexRDD[Int] = graph.outDegrees
val degreeGraph = graph.outerJoinVertices(outDegrees) { (id, oldAttr, outDegOpt) =>
  outDegOpt match {
    case Some(outDeg) => outDeg
    case None => 0 // No outDegree means zero outDegree
  }
}
```

> 您可能已经注意到上述示例中使用的多个参数列表（例如： `f(a)(b)` curried 函数模式。虽然我们可以将 `f(a)(b)` 同样地写成 `f(a,b)`，这意味着 `b` 上的类型推断不依赖于 `a`。因此，用户需要为用户定义的函数提供类型注释：

```
val joinedGraph = graph.joinVertices(uniqueCosts,
  (id: VertexId, oldCost: Double, extraCost: Double) => oldCost + extraCost)
```

## 邻域聚合

许多图形分析任务的关键步骤是聚合关于每个顶点邻域的信息。例如，我们可能想知道每个用户拥有的关注者数量或每个用户的追随者的平均年龄。许多迭代图表算法（例如：网页级别，最短路径，以及连接成分）相邻顶点（例如：电流值的 PageRank，最短到源路径，和最小可达顶点 ID）的重复聚合性质。

> 为了提高性能，主聚合操作员 `graph.mapReduceTriplets` 从新的更改 `graph.AggregateMessages`。虽然 API 的变化相对较小，但我们在下面提供了一个转换指南。

### 聚合消息 (aggregateMessages)

GraphX 中的核心聚合操作是 [`aggregateMessages`](api/scala/index.html#org.apache.spark.graphx.Graph@aggregateMessages[A]((EdgeContext[VD,ED,A])⇒Unit,(A,A)⇒A,TripletFields)(ClassTag[A]):VertexRDD[A])。该运算符将用户定义的 `sendMsg` 函数应用于图中的每个 _边缘三元组_，然后使用该 `mergeMsg` 函数在其目标顶点聚合这些消息。

```
class Graph[VD, ED] {
  def aggregateMessages[Msg: ClassTag](
      sendMsg: EdgeContext[VD, ED, Msg] => Unit,
      mergeMsg: (Msg, Msg) => Msg,
      tripletFields: TripletFields = TripletFields.All)
    : VertexRDD[Msg]
}
```

用户定义的 `sendMsg` 函数接受一个 [`EdgeContext`](api/scala/index.html#org.apache.spark.graphx.EdgeContext)，它将源和目标属性以及 edge 属性和函数 ([`sendToSrc`](api/scala/index.html#org.apache.spark.graphx.EdgeContext@sendToSrc(msg:A):Unit)，和 [`sendToDst`](api/scala/index.html#org.apache.spark.graphx.EdgeContext@sendToDst(msg:A):Unit)) 一起发送到源和目标属性。在 map-reduce 中，将 `sendMsg` 作为 _map_ 函数。用户定义的 `mergeMsg` 函数需要两个发往同一顶点的消息，并产生一条消息。想想 `mergeMsg` 是 map-reduce 中的 _reduce_ 函数。[`aggregateMessages`](api/scala/index.html#org.apache.spark.graphx.Graph@aggregateMessages[A]((EdgeContext[VD,ED,A])⇒Unit,(A,A)⇒A,TripletFields)(ClassTag[A]):VertexRDD[A]) 运算符返回一个 `VertexRDD[Msg]`，其中包含去往每个顶点的聚合消息（Msg类型）。没有收到消息的顶点不包括在返回的 `VertexRDD`[VertexRDD](api/scala/index.html#org.apache.spark.graphx.VertexRDD) 中。

另外，[`aggregateMessages`](api/scala/index.html#org.apache.spark.graphx.Graph@aggregateMessages[A]((EdgeContext[VD,ED,A])⇒Unit,(A,A)⇒A,TripletFields)(ClassTag[A]):VertexRDD[A]) 采用一个可选的`tripletsFields`，它们指示在 [`EdgeContext`](api/scala/index.html#org.apache.spark.graphx.EdgeContext) 中访问哪些数据（即源顶点属性，而不是目标顶点属性）。`tripletsFields` 定义的可能选项，[`TripletFields`](api/java/org/apache/spark/graphx/TripletFields.html) 默认值是 [`TripletFields.All`](api/java/org/apache/spark/graphx/TripletFields.html#All) 指示用户定义的 `sendMsg` 函数可以访问的任何字段[`EdgeContext`](api/scala/index.html#org.apache.spark.graphx.EdgeContext)。该 `tripletFields` 参数可用于通知 GraphX，只有部分 [`EdgeContext`](api/scala/index.html#org.apache.spark.graphx.EdgeContext) 需要允许 GraphX 选择优化的连接策略。例如，如果我们计算每个用户的追随者的平均年龄，我们只需要源字段，因此我们将用于 [`TripletFields.Src`](api/java/org/apache/spark/graphx/TripletFields.html#Src) 表示我们只需要源字段。

> 在早期版本的 GraphX 中，我们使用字节码检测来推断，[`TripletFields`](api/java/org/apache/spark/graphx/TripletFields.html) 但是我们发现字节码检查稍微不可靠，而是选择了更明确的用户控制。

在下面的例子中，我们使用 [`aggregateMessages`](api/scala/index.html#org.apache.spark.graphx.Graph@aggregateMessages[A]((EdgeContext[VD,ED,A])⇒Unit,(A,A)⇒A,TripletFields)(ClassTag[A]):VertexRDD[A]) 运算符来计算每个用户的资深追踪者的平均年龄。

```
import org.apache.spark.graphx.{Graph, VertexRDD}
import org.apache.spark.graphx.util.GraphGenerators

// Create a graph with "age" as the vertex property.
// Here we use a random graph for simplicity.
val graph: Graph[Double, Int] =
  GraphGenerators.logNormalGraph(sc, numVertices = 100).mapVertices( (id, _) => id.toDouble )
// Compute the number of older followers and their total age
val olderFollowers: VertexRDD[(Int, Double)] = graph.aggregateMessages[(Int, Double)](
  triplet => { // Map Function
    if (triplet.srcAttr > triplet.dstAttr) {
      // Send message to destination vertex containing counter and age
      triplet.sendToDst(1, triplet.srcAttr)
    }
  },
  // Add counter and age
  (a, b) => (a._1 + b._1, a._2 + b._2) // Reduce Function
)
// Divide total age by number of older followers to get average age of older followers
val avgAgeOfOlderFollowers: VertexRDD[Double] =
  olderFollowers.mapValues( (id, value) =>
    value match { case (count, totalAge) => totalAge / count } )
// Display the results
avgAgeOfOlderFollowers.collect.foreach(println(_))
```

<small>Find full example code at "examples/src/main/scala/org/apache/spark/examples/graphx/AggregateMessagesExample.scala" in the Spark repo.</small>

> `aggregateMessages` 当消息（和消息的总和）是恒定大小（例如：浮动和加法而不是列表和级联）时，该操作最佳地执行。

### Map Reduce Triplets Transition Guide (Legacy)

在早期版本的 GraphX 中，邻域聚合是使用 `mapReduceTriplets` 运算符完成的：

```
class Graph[VD, ED] {
  def mapReduceTriplets[Msg](
      map: EdgeTriplet[VD, ED] => Iterator[(VertexId, Msg)],
      reduce: (Msg, Msg) => Msg)
    : VertexRDD[Msg]
}
```

`mapReduceTriplets` 操作符接受用户定义的映射函数，该函数应用于每个三元组，并且可以使用用户定义的缩减函数来生成聚合的 _消息_。然而，我们发现返回的迭代器的用户是昂贵的，并且它阻止了我们应用其他优化（例如：局部顶点重新编号）的能力。在 [`aggregateMessages`](api/scala/index.html#org.apache.spark.graphx.Graph@aggregateMessages[A]((EdgeContext[VD,ED,A])⇒Unit,(A,A)⇒A,TripletFields)(ClassTag[A]):VertexRDD[A]) 中，我们引入了 EdgeContext，它暴露了三元组字段，并且还显示了向源和目标顶点发送消息的功能。此外，我们删除了字节码检查，而是要求用户指出三元组中实际需要哪些字段。

以下代码块使用 `mapReduceTriplets`：

```
val graph: Graph[Int, Float] = ...
def msgFun(triplet: Triplet[Int, Float]): Iterator[(Int, String)] = {
  Iterator((triplet.dstId, "Hi"))
}
def reduceFun(a: String, b: String): String = a + " " + b
val result = graph.mapReduceTriplets[String](msgFun, reduceFun)
```

可以使用 `aggregateMessages`：å

```
val graph: Graph[Int, Float] = ...
def msgFun(triplet: EdgeContext[Int, Float, String]) {
  triplet.sendToDst("Hi")
}
def reduceFun(a: String, b: String): String = a + " " + b
val result = graph.aggregateMessages[String](msgFun, reduceFun)
```

### 计算级别信息

常见的聚合任务是计算每个顶点的程度：与每个顶点相邻的边数。在有向图的上下文中，通常需要知道每个顶点的度数，外部程度和总程度。本 [`GraphOps`](api/scala/index.html#org.apache.spark.graphx.GraphOps) 类包含运营商计算度数每个顶点的集合。例如，在下面我们将计算最大值，最大和最大级别：

```
// Define a reduce operation to compute the highest degree vertex
def max(a: (VertexId, Int), b: (VertexId, Int)): (VertexId, Int) = {
  if (a._2 > b._2) a else b
}
// Compute the max degrees
val maxInDegree: (VertexId, Int)  = graph.inDegrees.reduce(max)
val maxOutDegree: (VertexId, Int) = graph.outDegrees.reduce(max)
val maxDegrees: (VertexId, Int)   = graph.degrees.reduce(max)
```

### 收集相邻点

在某些情况下，通过在每个顶点处收集相邻顶点及其属性可以更容易地表达计算。这可以使用 [`collectNeighborIds`](api/scala/index.html#org.apache.spark.graphx.GraphOps@collectNeighborIds(EdgeDirection):VertexRDD[Array[VertexId]]) 和 [`collectNeighbors`](api/scala/index.html#org.apache.spark.graphx.GraphOps@collectNeighbors(EdgeDirection):VertexRDD[Array[(VertexId,VD)]]) 运算符轻松实现。

```
class GraphOps[VD, ED] {
  def collectNeighborIds(edgeDirection: EdgeDirection): VertexRDD[Array[VertexId]]
  def collectNeighbors(edgeDirection: EdgeDirection): VertexRDD[ Array[(VertexId, VD)] ]
}
```

> 这些操作可能相当昂贵，因为它们重复信息并需要大量通信。如果可能，请直接使用 [`aggregateMessages`](api/scala/index.html#org.apache.spark.graphx.Graph@aggregateMessages[A]((EdgeContext[VD,ED,A])⇒Unit,(A,A)⇒A,TripletFields)(ClassTag[A]):VertexRDD[A]) 操作来表达相同的计算。

## Caching and Uncaching

在 Spark 中，默认情况下，RDD 不会保留在内存中。为了避免重新计算，在多次使用它们时，必须明确缓存它们（参见 [Spark Programming Guide](programming-guide.html#rdd-persistence)）。GraphX 中的图形表现方式相同。**当多次使用图表时，请务必先调用 [`Graph.cache()`](api/scala/index.html#org.apache.spark.graphx.Graph@cache():Graph[VD,ED])。**

在迭代计算中，_uncaching_ 也可能是最佳性能所必需的。默认情况下，缓存的 RDD 和图形将保留在内存中，直到内存压力迫使它们以 LRU 顺序逐出。对于迭代计算，来自先前迭代的中间结果将填满缓存。虽然它们最终被驱逐出来，但存储在内存中的不必要的数据会减慢垃圾收集速度。一旦不再需要中间结果，就会更有效率。这涉及每次迭代实现（缓存和强制）图形或 RDD，取消所有其他数据集，并且仅在将来的迭代中使用实例化数据集。然而，由于图形由多个 RDD 组成，所以很难将它们正确地分开。**对于迭代计算，我们建议使用 Pregel API，它可以正确地解析中间结果。**

# Pregel API

图形是固有的递归数据结构，因为顶点的属性取决于其邻居的属性，而邻居的属性又依赖于 _其_ 邻居的属性。因此，许多重要的图算法迭代地重新计算每个顶点的属性，直到达到一个固定点条件。已经提出了一系列图并行抽象来表达这些迭代算法。GraphX 公开了 Pregel API 的变体。

在高层次上，GraphX 中的 Pregel 运算符是 _限制到图形拓扑的_ 批量同步并行消息抽象。Pregel 操作符在一系列超级步骤中执行，其中顶点接收来自先前超级步骤的入站消息的 _总和_，计算顶点属性的新值，然后在下一个超级步骤中将消息发送到相邻顶点。与 Pregel 不同，消息作为边缘三元组的函数并行计算，消息计算可以访问源和目标顶点属性。在超级步骤中跳过不接收消息的顶点。Pregel 运算符终止迭代，并在没有剩余的消息时返回最终的图。

> 注意，与更多的标准 Pregel 实现不同，GraphX 中的顶点只能将消息发送到相邻顶点，并且使用用户定义的消息传递功能并行完成消息构造。这些约束允许在 GraphX 中进行额外优化。

以下是 [Pregel 运算符](api/scala/index.html#org.apache.spark.graphx.GraphOps@pregel[A](A,Int,EdgeDirection)((VertexId,VD,A)⇒VD,(EdgeTriplet[VD,ED])⇒Iterator[(VertexId,A)],(A,A)⇒A)(ClassTag[A]):Graph[VD,ED]) 的类型签名以及 其实现的 _草图_（注意：为了避免由于长谱系链引起的 stackOverflowError，pregel 支持周期性检查点图和消息，将 “spark.graphx.pregel.checkpointInterval” 设置为正数，说10。并使用 SparkContext.setCheckpointDir(directory: String)) 设置 checkpoint 目录）：

```
class GraphOps[VD, ED] {
  def pregel[A]
      (initialMsg: A,
       maxIter: Int = Int.MaxValue,
       activeDir: EdgeDirection = EdgeDirection.Out)
      (vprog: (VertexId, VD, A) => VD,
       sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexId, A)],
       mergeMsg: (A, A) => A)
    : Graph[VD, ED] = {
    // Receive the initial message at each vertex
    var g = mapVertices( (vid, vdata) => vprog(vid, vdata, initialMsg) ).cache()

    // compute the messages
    var messages = g.mapReduceTriplets(sendMsg, mergeMsg)
    var activeMessages = messages.count()
    // Loop until no messages remain or maxIterations is achieved
    var i = 0
    while (activeMessages > 0 && i < maxIterations) {
      // Receive the messages and update the vertices.
      g = g.joinVertices(messages)(vprog).cache()
      val oldMessages = messages
      // Send new messages, skipping edges where neither side received a message. We must cache
      // messages so it can be materialized on the next line, allowing us to uncache the previous
      // iteration.
      messages = GraphXUtils.mapReduceTriplets(
        g, sendMsg, mergeMsg, Some((oldMessages, activeDirection))).cache()
      activeMessages = messages.count()
      i += 1
    }
    g
  }
}
```

请注意，Pregel 需要两个参数列表（即：`graph.pregel(list1)(list2)`。第一个参数列表包含配置参数，包括初始消息，最大迭代次数以及发送消息的边缘方向（默认情况下为边缘）。第二个参数列表包含用于接收消息（顶点程序 `vprog`），计算消息（`sendMsg`）和组合消息的用户定义函数 `mergeMsg`。

在以下示例中，我们可以使用 Pregel 运算符来表达单源最短路径的计算。

```
import org.apache.spark.graphx.{Graph, VertexId}
import org.apache.spark.graphx.util.GraphGenerators

// A graph with edge attributes containing distances
val graph: Graph[Long, Double] =
  GraphGenerators.logNormalGraph(sc, numVertices = 100).mapEdges(e => e.attr.toDouble)
val sourceId: VertexId = 42 // The ultimate source
// Initialize the graph such that all vertices except the root have distance infinity.
val initialGraph = graph.mapVertices((id, _) =>
    if (id == sourceId) 0.0 else Double.PositiveInfinity)
val sssp = initialGraph.pregel(Double.PositiveInfinity)(
  (id, dist, newDist) => math.min(dist, newDist), // Vertex Program
  triplet => {  // Send Message
    if (triplet.srcAttr + triplet.attr < triplet.dstAttr) {
      Iterator((triplet.dstId, triplet.srcAttr + triplet.attr))
    } else {
      Iterator.empty
    }
  },
  (a, b) => math.min(a, b) // Merge Message
)
println(sssp.vertices.collect.mkString("\n"))
```

<small>Find full example code at "examples/src/main/scala/org/apache/spark/examples/graphx/SSSPExample.scala" in the Spark repo.</small>

# Graph 建造者

GraphX 提供了从 RDD 或磁盘上的顶点和边的集合构建图形的几种方法。默认情况下，图形构建器都不会重新分配图形边; 相反，边缘保留在其默认分区（例如 HDFS 中的原始块）中。[`Graph.groupEdges`](api/scala/index.html#org.apache.spark.graphx.Graph@groupEdges((ED,ED)⇒ED):Graph[VD,ED]) 需要重新分区图，因为它假定相同的边将被共同定位在同一分区上，因此您必须在调用 [`Graph.partitionBy`](api/scala/index.html#org.apache.spark.graphx.Graph@partitionBy(PartitionStrategy):Graph[VD,ED]) 之前调用 `groupEdges`。

```
object GraphLoader {
  def edgeListFile(
      sc: SparkContext,
      path: String,
      canonicalOrientation: Boolean = false,
      minEdgePartitions: Int = 1)
    : Graph[Int, Int]
}
```

[`GraphLoader.edgeListFile`](api/scala/index.html#org.apache.spark.graphx.GraphLoader$@edgeListFile(SparkContext,String,Boolean,Int):Graph[Int,Int]) 提供了从磁盘边缘列表中加载图形的方法。它解析以下形式的（源顶点 ID，目标顶点 ID）对的邻接列表，跳过以下开始的注释行 `#`：

```
# This is a comment
2 1
4 1
1 2 
```

它 `Graph` 从指定的边缘创建一个，自动创建边缘提到的任何顶点。所有顶点和边缘属性默认为1\. `canonicalOrientation` 参数允许在正方向（`srcId &lt; dstId`）重新定向边，这是[连接的组件](api/scala/index.html#org.apache.spark.graphx.lib.ConnectedComponents$)算法所要求的。该 `minEdgePartitions` 参数指定要生成的边缘分区的最小数量；如果例如 HDFS 文件具有更多块，则可能存在比指定更多的边缘分区。

```
object Graph {
  def apply[VD, ED](
      vertices: RDD[(VertexId, VD)],
      edges: RDD[Edge[ED]],
      defaultVertexAttr: VD = null)
    : Graph[VD, ED]

  def fromEdges[VD, ED](
      edges: RDD[Edge[ED]],
      defaultValue: VD): Graph[VD, ED]

  def fromEdgeTuples[VD](
      rawEdges: RDD[(VertexId, VertexId)],
      defaultValue: VD,
      uniqueEdges: Option[PartitionStrategy] = None): Graph[VD, Int]

}
```

[`Graph.apply`](api/scala/index.html#org.apache.spark.graphx.Graph$@apply[VD,ED](RDD[(VertexId,VD)],RDD[Edge[ED]],VD)(ClassTag[VD],ClassTag[ED]):Graph[VD,ED]) 允许从顶点和边缘的 RDD 创建图形。重复的顶点被任意挑选，并且边缘 RDD 中找到的顶点，而不是顶点 RDD 被分配了默认属性。

[`Graph.fromEdges`](api/scala/index.html#org.apache.spark.graphx.Graph$@fromEdges[VD,ED](RDD[Edge[ED]],VD)(ClassTag[VD],ClassTag[ED]):Graph[VD,ED]) 允许仅从 RDD 的边缘创建图形，自动创建边缘提到的任何顶点并将其分配给默认值。

[`Graph.fromEdgeTuples`](api/scala/index.html#org.apache.spark.graphx.Graph$@fromEdgeTuples[VD](RDD[(VertexId,VertexId)],VD,Option[PartitionStrategy])(ClassTag[VD]):Graph[VD,Int]) 允许仅从边缘元组的 RDD 创建图形，将边缘分配为值1，并自动创建边缘提到的任何顶点并将其分配给默认值。它还支持重复数据删除边缘; 重复数据删除，将`某些` [`PartitionStrategy`](api/scala/index.html#org.apache.spark.graphx.PartitionStrategy) 作为 `uniqueEdges` 参数传递（例如：`uniqueEdges = Some(PartitionStrategy.RandomVertexCut)`）。分区策略是必须的，以便在相同的分区上共同使用相同的边，以便可以进行重复数据删除。

# Vertex and Edge RDDs

GraphX 公开 `RDD` 了图中存储的顶点和边的视图。然而，由于 GraphX 在优化的数据结构中维护顶点和边，并且这些数据结构提供了附加功能，所以顶点和边分别作为[`VertexRDD`](api/scala/index.html#org.apache.spark.graphx.VertexRDD) 和 [`EdgeRDD`](api/scala/index.html#org.apache.spark.graphx.EdgeRDD) 返回。在本节中，我们将回顾一些这些类型中的其他有用功能。请注意，这只是一个不完整的列表，请参阅API文档中的正式操作列表。

## VertexRDDs

该 `VertexRDD[A]` 扩展 `RDD[(VertexId, A)]` 并增加了额外的限制，每个 `VertexId` 只发生一次。此外，`VertexRDD[A]` 表示一组顶点，每个顶点的属性类型A。在内部，这是通过将顶点属性存储在可重用的散列图数据结构中来实现的。因此，如果两个 `VertexRDD` 派生自相同的基础 [`VertexRDD`](api/scala/index.html#org.apache.spark.graphx.VertexRDD)（例如：`filter`或 `mapValues`），则可以在不使用散列评估的情况下连续连接。为了利用这个索引的数据结构，[`VertexRDD`](api/scala/index.html#org.apache.spark.graphx.VertexRDD) 公开了以下附加功能：

```
class VertexRDD[VD] extends RDD[(VertexId, VD)] {
  // Filter the vertex set but preserves the internal index
  def filter(pred: Tuple2[VertexId, VD] => Boolean): VertexRDD[VD]
  // Transform the values without changing the ids (preserves the internal index)
  def mapValues[VD2](map: VD => VD2): VertexRDD[VD2]
  def mapValues[VD2](map: (VertexId, VD) => VD2): VertexRDD[VD2]
  // Show only vertices unique to this set based on their VertexId's
  def minus(other: RDD[(VertexId, VD)])
  // Remove vertices from this set that appear in the other set
  def diff(other: VertexRDD[VD]): VertexRDD[VD]
  // Join operators that take advantage of the internal indexing to accelerate joins (substantially)
  def leftJoin[VD2, VD3](other: RDD[(VertexId, VD2)])(f: (VertexId, VD, Option[VD2]) => VD3): VertexRDD[VD3]
  def innerJoin[U, VD2](other: RDD[(VertexId, U)])(f: (VertexId, VD, U) => VD2): VertexRDD[VD2]
  // Use the index on this RDD to accelerate a `reduceByKey` operation on the input RDD.
  def aggregateUsingIndex[VD2](other: RDD[(VertexId, VD2)], reduceFunc: (VD2, VD2) => VD2): VertexRDD[VD2]
}
```

请注意，例如，`filter` 运算符 如何返回 [`VertexRDD`](api/scala/index.html#org.apache.spark.graphx.VertexRDD)。过滤器实际上是通过 `BitSet` 使用索引重新实现的，并保留与其他`VertexRDD` 进行快速连接的能力。同样，`mapValues` 运算符不允许 `map` 功能改变，`VertexId` 从而使相同的 `HashMap` 数据结构能够被重用。无论是 `leftJoin` 和 `innerJoin` 能够连接两个时识别 `VertexRDD` 来自同一来源的小号 `HashMap` 和落实线性扫描，而不是昂贵的点查找的加入。

`aggregateUsingIndex` 运算符对于从 `RDD[(VertexId, A)]` 有效构建新的 [`VertexRDD`](api/scala/index.html#org.apache.spark.graphx.VertexRDD) 非常有用。在概念上，如果我在一组顶点上构造了一个 `VertexRDD[B]`，这是一些 `RDD[(VertexId, A)]` 中的顶点的 _超集_，那么我可以重用索引来聚合然后再索引 `RDD[(VertexId, A)]`。例如：

```
val setA: VertexRDD[Int] = VertexRDD(sc.parallelize(0L until 100L).map(id => (id, 1)))
val rddB: RDD[(VertexId, Double)] = sc.parallelize(0L until 100L).flatMap(id => List((id, 1.0), (id, 2.0)))
// There should be 200 entries in rddB
rddB.count
val setB: VertexRDD[Double] = setA.aggregateUsingIndex(rddB, _ + _)
// There should be 100 entries in setB
setB.count
// Joining A and B should now be fast!
val setC: VertexRDD[Double] = setA.innerJoin(setB)((id, a, b) => a + b)
```

## EdgeRDDs

该 `EdgeRDD[ED]` 扩展 `RDD[Edge[ED]]` 使用其中定义的各种分区策略之一来组织块中的边 [`PartitionStrategy`](api/scala/index.html#org.apache.spark.graphx.PartitionStrategy)。在每个分区中，边缘属性和邻接结构分别存储，可以在更改属性值时进行最大限度的重用。

`EdgeRDD`[EdgeRDD](api/scala/index.html#org.apache.spark.graphx.EdgeRDD) 公开的三个附加功能是：

```
// Transform the edge attributes while preserving the structure
def mapValues[ED2](f: Edge[ED] => ED2): EdgeRDD[ED2]
// Reverse the edges reusing both attributes and structure
def reverse: EdgeRDD[ED]
// Join two `EdgeRDD`s partitioned using the same partitioning strategy.
def innerJoin[ED2, ED3](other: EdgeRDD[ED2])(f: (VertexId, VertexId, ED, ED2) => ED3): EdgeRDD[ED3]
```

在大多数应用中，我们发现在 `EdgeRDD`[EdgeRDD](api/scala/index.html#org.apache.spark.graphx.EdgeRDD) 上的操作是通过图形运算符完成的，或者依赖基 `RDD` 类中定义的操作。

# 优化表示

虽然在分布式图形的 GraphX 表示中使用的优化的详细描述超出了本指南的范围，但一些高级理解可能有助于可扩展算法的设计以及 API 的最佳使用。GraphX 采用顶点切分方式进行分布式图分割：

![Edge Cut vs. Vertex Cut](img/320231ef4777b1b8d73eb4aacd19bb40.jpg "Edge Cut vs. Vertex Cut") 

GraphX 不是沿着边沿分割图形，而是沿着顶点分割图形，这可以减少通信和存储开销。在逻辑上，这对应于将边缘分配给机器并允许顶点跨越多台机器。分配边缘的确切方法取决于 [`PartitionStrategy`](api/scala/index.html#org.apache.spark.graphx.PartitionStrategy) 各种启发式的几种折衷。用户可以通过与 [`Graph.partitionBy`](api/scala/index.html#org.apache.spark.graphx.Graph@partitionBy(PartitionStrategy):Graph[VD,ED]) 运算符重新分区图来选择不同的策略。默认分区策略是使用图形构建中提供的边的初始分区。然而，用户可以轻松切换到 GraphX 中包含的 2D 划分或其他启发式算法。

![RDD Graph Representation](img/dc17d028e115b035c452c7e82a5e6d5c.jpg "RDD Graph Representation") 

一旦边缘被划分，高效的图形并行计算的关键挑战就是有效地将顶点属性与边缘连接起来。因为真实世界的图形通常具有比顶点更多的边缘，所以我们将顶点属性移动到边缘。因为不是所有的分区都将包含邻近的所有顶点的边缘，我们内部维护标识在哪里执行所需的连接像操作时，广播顶点的路由表 `triplets` 和 `aggregateMessages`。

# Graph 算法

GraphX 包括一组简化分析任务的图算法。该算法被包含在 `org.apache.spark.graphx.lib` 包可直接作为方法来访问 `Graph` 通过 [`GraphOps`](api/scala/index.html#org.apache.spark.graphx.GraphOps)。本节介绍算法及其使用方法。

## PageRank

PageRank 测量在图中每个顶点的重要性，假设从边缘 _u_ 到 _v_ 表示的认可 _v_ 通过的重要性 _u_。例如，如果 Twitter 用户遵循许多其他用户，则用户将被高度排名。

GraphX 附带了 PageRank 的静态和动态实现方法作[`PageRank 对象`](api/scala/index.html#org.apache.spark.graphx.lib.PageRank$)上的方法。静态 PageRank 运行固定次数的迭代，而动态 PageRank 运行直到排列收敛（即，停止改变超过指定的公差）。[`GraphOps`](api/scala/index.html#org.apache.spark.graphx.GraphOps) 允许直接调用这些算法作为方法 `Graph`。

GraphX还包括一个可以运行 PageRank 的社交网络数据集示例。给出了一组用户 `data/graphx/users.txt`，并给出了一组用户之间的关系 `data/graphx/followers.txt`。我们计算每个用户的 PageRank 如下：

```
import org.apache.spark.graphx.GraphLoader

// Load the edges as a graph
val graph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt")
// Run PageRank
val ranks = graph.pageRank(0.0001).vertices
// Join the ranks with the usernames
val users = sc.textFile("data/graphx/users.txt").map { line =>
  val fields = line.split(",")
  (fields(0).toLong, fields(1))
}
val ranksByUsername = users.join(ranks).map {
  case (id, (username, rank)) => (username, rank)
}
// Print the result
println(ranksByUsername.collect().mkString("\n"))
```

<small>Find full example code at "examples/src/main/scala/org/apache/spark/examples/graphx/PageRankExample.scala" in the Spark repo.</small>

## 连接组件

连接的组件算法将图中每个连接的组件与其最低编号顶点的ID进行标记。例如，在社交网络中，连接的组件可以近似群集。GraphX包含[`ConnectedComponents object`](api/scala/index.html#org.apache.spark.graphx.lib.ConnectedComponents$) 中算法的实现，我们从 [PageRank 部分](#pagerank) 计算示例社交网络数据集的连接组件如下：

```
import org.apache.spark.graphx.GraphLoader

// Load the graph as in the PageRank example
val graph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt")
// Find the connected components
val cc = graph.connectedComponents().vertices
// Join the connected components with the usernames
val users = sc.textFile("data/graphx/users.txt").map { line =>
  val fields = line.split(",")
  (fields(0).toLong, fields(1))
}
val ccByUsername = users.join(cc).map {
  case (id, (username, cc)) => (username, cc)
}
// Print the result
println(ccByUsername.collect().mkString("\n"))
```

<small>Find full example code at "examples/src/main/scala/org/apache/spark/examples/graphx/ConnectedComponentsExample.scala" in the Spark repo.</small>

## Triangle 计数

顶点是三角形的一部分，当它有两个相邻的顶点之间有一个边。GraphX 在 [`TriangleCount 对象`](api/scala/index.html#org.apache.spark.graphx.lib.TriangleCount$) 中实现一个三角计数算法，用于确定通过每个顶点的三角形数量，提供聚类度量。我们从 [PageRank 部分](#pagerank) 计算社交网络数据集的三角形数。_需要注意的是 `TriangleCount` 边缘要处于规范方向（`srcId &lt; dstId`），而图形要使用 [`Graph.partitionBy`](api/scala/index.html#org.apache.spark.graphx.Graph@partitionBy(PartitionStrategy):Graph[VD,ED])。_

```
import org.apache.spark.graphx.{GraphLoader, PartitionStrategy}

// Load the edges in canonical order and partition the graph for triangle count
val graph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt", true)
  .partitionBy(PartitionStrategy.RandomVertexCut)
// Find the triangle count for each vertex
val triCounts = graph.triangleCount().vertices
// Join the triangle counts with the usernames
val users = sc.textFile("data/graphx/users.txt").map { line =>
  val fields = line.split(",")
  (fields(0).toLong, fields(1))
}
val triCountByUsername = users.join(triCounts).map { case (id, (username, tc)) =>
  (username, tc)
}
// Print the result
println(triCountByUsername.collect().mkString("\n"))
```

<small>Find full example code at "examples/src/main/scala/org/apache/spark/examples/graphx/TriangleCountingExample.scala" in the Spark repo.</small>

# 示例

假设我想从一些文本文件中构建图形，将图形限制为重要的关系和用户，在 sub-graph 上运行 page-rank，然后返回与顶级用户关联的属性。我可以用 GraphX 在几行内完成所有这些：

```
import org.apache.spark.graphx.GraphLoader

// Load my user data and parse into tuples of user id and attribute list
val users = (sc.textFile("data/graphx/users.txt")
  .map(line => line.split(",")).map( parts => (parts.head.toLong, parts.tail) ))

// Parse the edge data which is already in userId -> userId format
val followerGraph = GraphLoader.edgeListFile(sc, "data/graphx/followers.txt")

// Attach the user attributes
val graph = followerGraph.outerJoinVertices(users) {
  case (uid, deg, Some(attrList)) => attrList
  // Some users may not have attributes so we set them as empty
  case (uid, deg, None) => Array.empty[String]
}

// Restrict the graph to users with usernames and names
val subgraph = graph.subgraph(vpred = (vid, attr) => attr.size == 2)

// Compute the PageRank
val pagerankGraph = subgraph.pageRank(0.001)

// Get the attributes of the top pagerank users
val userInfoWithPageRank = subgraph.outerJoinVertices(pagerankGraph.vertices) {
  case (uid, attrList, Some(pr)) => (pr, attrList.toList)
  case (uid, attrList, None) => (0.0, attrList.toList)
}

println(userInfoWithPageRank.vertices.top(5)(Ordering.by(_._2._1)).mkString("\n"))
```

<small>Find full example code at "examples/src/main/scala/org/apache/spark/examples/graphx/ComprehensiveExample.scala" in the Spark repo.</small>