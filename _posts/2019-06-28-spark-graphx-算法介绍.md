---
layout:     post
title:      spark graphx 算法介绍 
subtitle:   elasticsearch启动流程分析
date:       2019-06-28
author:     chencoder
catalog: 	true
tags:
    - spark
    - graph
---

spark graphx 是基于pregel模型实现的各种图像算法工具包。


## pregel

Pregel是Google提出的大规模分布式图计算平台，专门用来解决网页链接分析、社交数据挖掘等实际应用中涉及的大规模分布式图计算问题。

Pregel在编程模型上遵循以图节点为中心的模式，在超级步S中。每一个图节点能够汇总从超级步S-1中其它节点传递过来的消息，改变图节点自身的状态。并向其它节点发送消息。这些消息经过同步后。会在超级步S+1中被其它节点接收并做出处理。用户仅仅须要自己定义一个针对图节点的计算函数F(vertex),用来实现上述的图节点计算功能。至于其它的任务，比方任务分配、任务管理、系统容错等都交由Pregel系统来实现。

## spark 中的 pregel

spark graphx 中通过Pregel类实现pregel模型，是一个基础的工具类。

Pregel.scala
```
  def apply[VD: ClassTag, ED: ClassTag, A: ClassTag]
     (graph: Graph[VD, ED],
      initialMsg: A,
      maxIterations: Int = Int.MaxValue,
      activeDirection: EdgeDirection = EdgeDirection.Either)
     (vprog: (VertexId, VD, A) => VD,
      sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexId, A)],
      mergeMsg: (A, A) => A)
```

vprog：
vprog是用户定义的顶点程序，会运行在每一个顶点上，该vprog函数的功能是负责接收入站的message，
并计算出的顶点的新属性值。
在首轮迭代时，在所有的顶点上都会调用程序vprog函数，传人默认的defaultMessage；在次轮迭代时，只有接收到message消息的顶点才会调用vprog函数。

sendMsg：
用户提供的函数，应用于以当前迭代计算收到消息的顶点为源顶点的边edges；sendMsg函数的功能
是发送消息，消息的发送方向默认是沿着出边反向（向边的目的顶点发送消息）。

mergeMsg：
用户提供定义的函数，将具有相同目的地的消息合并成一个；如果一个顶点，收到两个以上的A类型的消息message，该函数将他们合并成一个A类型消息。 这个函数必须是可交换的和关联的。理想情况下，A类型的message的size大小不应增加。

基本计算过程：
重复调用vprog方法处理顶点，发送消息，合并消息， 直到没有消息可以发送（没有活动的顶点）

## graphx 中实现的图算法

以下内容来自 https://www.jianshu.com/p/289cf0fc4a98, 总结的非常全面

### TriangleCount(数三角形)
#### 简介
统计每个顶点所在的三角形个数.
对网络图中进行三角形个数计数可以根据三角形数量反应网络中的稠密程度和质量。
#### 应用场景
* 用于社区发现。如微博中你关注的人也关注你，大家的关注关系中有很多三角形，说明社区很强很稳定，大家联系比较紧密；如果一个人只关注了很多人，却没有形成三角形，则说明社交群体很小很松散。
* 衡量社群耦合关系的紧密程度， 通过三角形数量来反应社区内部的紧密程度，作为一项参考指标。

#### 算法思路 
计算规则: 如果一条边的两个顶点有共同的邻居，则这三个点构成三角形。

计算步骤：
1.  为每个节点计算邻居集合
2.  对于每条边，计算两端节点邻居集合的交集，将交集中元素个数告知两端节点，该个数即对应着节点关联的三角形数。
3.  对每个节点合并三角形数目统计总数，由于三角形中一个顶点关联两条边，所以对于同一个三角形而言，一个顶点计算了两次，故最终结果需要除以2.

源码解析 :
```
object TriangleCount {

  def run[VD: ClassTag, ED: ClassTag](graph: Graph[VD, ED]): Graph[Int, ED] = {
    // Transform the edge data something cheap to shuffle and then canonicalize
    //得到的是一个无自连边且无重复边的、边是从小id指向大id的图
    val canonicalGraph = graph.mapEdges(e => true).removeSelfEdges().convertToCanonicalEdges()
    // Get the triangle counts
    val counters = runPreCanonicalized(canonicalGraph).vertices
    // Join them bath with the original graph
    graph.outerJoinVertices(counters) { (vid, _, optCounter: Option[Int]) =>
      optCounter.getOrElse(0)
    }
  }


  def runPreCanonicalized[VD: ClassTag, ED: ClassTag](graph: Graph[VD, ED]): Graph[Int, ED] = {
    // 构建邻居集合
    val nbrSets: VertexRDD[VertexSet] =
      // 收集邻居节点，边方向为Either，保证点的入边和出边连接的邻居点都会被收集
      graph.collectNeighborIds(EdgeDirection.Either).mapValues { (vid, nbrs) =>
        val set = new VertexSet(nbrs.length)
        var i = 0
        while (i < nbrs.length) {
          // prevent self cycle
          if (nbrs(i) != vid) {
            set.add(nbrs(i))
          }
          i += 1
        }
        set
      }

    // 更新图中顶点的属性为邻居点集合
    val setGraph: Graph[VertexSet, ED] = graph.outerJoinVertices(nbrSets) {
      (vid, _, optSet) => optSet.getOrElse(null)
    }

    def edgeFunc(ctx: EdgeContext[VertexSet, ED, Int]) {
      // 在边上操作源点和终点的邻居集合是，遍历较小的集合，加快遍历速度
      val (smallSet, largeSet) = if (ctx.srcAttr.size < ctx.dstAttr.size) {
        (ctx.srcAttr, ctx.dstAttr)
      } else {
        (ctx.dstAttr, ctx.srcAttr)
      }
      val iter = smallSet.iterator
      var counter: Int = 0
      while (iter.hasNext) {
        val vid = iter.next()
        if (vid != ctx.srcId && vid != ctx.dstId && largeSet.contains(vid)) {
          counter += 1
        }
      }
      ctx.sendToSrc(counter)
      ctx.sendToDst(counter)
    }

    // 沿着图中的边计算两个顶点的邻居集合的交集，并为每个顶点合并消息（消息为三角形个数）
    val counters: VertexRDD[Int] = setGraph.aggregateMessages(edgeFunc, _ + _)
    graph.outerJoinVertices(counters) { (_, _, optCounter: Option[Int]) =>
      val dblCount = optCounter.getOrElse(0)
      // 算法为每个三角形计算了两次，所以结果是偶数
      require(dblCount % 2 == 0, "Triangle count resulted in an invalid number of triangles.")
      dblCount / 2        //注意最后需要除以2，每个三角形被计算了两遍
    }
  }
}
```

### PageRank
PageRank是谷歌提出的用于解决链接分析中网页排名问题的算法，目的是为了对互联网中数以亿计的网页进行排名。

#### 简介
美国斯坦福大学的Larry Page和Sergey Brin在研究网页排序问题时采用学术界评判论文重要性的方法即看论文的引用量以及引用该论文的论文质量，对应于网页的重要性有两个假设：
  1. 数量假设：如果一个网页A被很多其他网页链接到，则该网页比较重要；
  2. 质量假设如果一个很重要的网页链接到网页A，则该网页的重要性会被提高。

#### 应用场景
* 社交应用的相似度内容推荐: 过对微博微信等社交应用进行社交网络分析，可以基于pagerank算法根据用户通常浏览的信息以及停留时间实现基于用户的相似度的内容推荐.

* 分析用户社交影响力: 在社交网络分析时根据用户的PageRank值进行用户影响力分析；

* 文献重要性研究 ： 根据文献的PageRank值评判该文献的质量，PageRank算法就是基于评判文献质量的想法来实现设计。

#### 算法思路
1. 为每个节点（网页）设置一个同样的初始PageRank值；

2. 第一次迭代:每个节点得到一个新的PageRank值；

3. 第二次迭代：用这组新的PageRank按上述公式形成另一组新的PageRank。

### 标签传播(LabelPropagation)

标签传播是将自己的标签信息传播给所有的邻居节点，邻居节点根据收到的标签信息选择出现最多的那个标签来更新自己的标签，并不断传播下去，直到图中节点的标签不再变动。

#### 简介
标签传播是为了在网络中发现社区，通过将自身标签传递给邻居节点以期形成一个具有同样标签的社区团体，标签传播合适于非重叠社区的发现。

#### 应用场景
* 社区发现: 标签传播进行社区发现即聚类，可以发现社交网络中的团体、诈骗犯罪团伙.
* 节点预测: 根据邻居节点的标签预测当前节点所属类别，可以进行分类预测、标签预测等。
* 信息分类。

#### 算法思路
核心思想：将一个节点的邻居节点的标签中数量最多的那个标签作为自身的标签，如果数量最多的标签多于一个，则随机选择一个最多的标签作为自身的标签。

1. 初始化，给每个节点一个唯一的标签，Graphx将节点的自身id作为自己的标签；
2. 每个节点将自身标签以及对应的数值1组成map<VertexId, Long>发送给邻居节点；
3. 每个节点将会收到来自邻居的标签信息，对收到的具有同样的标签进行汇总统计，对收到Map结构中的第二个元素进行相加求和，即得到节点收到的标签以及收到同样标签的数量；
4. 节点将收到的消息进行处理，根据标签数量选择数量最大的标签及数量作为自己新的属性。
   
不断迭代上述第3,4步。

#### 源码分析

```
 def run[VD, ED: ClassTag](graph: Graph[VD, ED], maxSteps: Int): Graph[VertexId, ED] = {
    require(maxSteps > 0, s"Maximum of steps must be greater than 0, but got ${maxSteps}")

    // 初始化消息，每个节点将自身id作为自己的初始标签
    val lpaGraph = graph.mapVertices { case (vid, _) => vid }
    // 发送消息，每条边上分别将两端节点的标签信息及标签数量1发送给对方节点
    def sendMessage(e: EdgeTriplet[VertexId, ED]): Iterator[(VertexId, Map[VertexId, Long])] = {
      Iterator((e.srcId, Map(e.dstAttr -> 1L)), (e.dstId, Map(e.srcAttr -> 1L)))
    }
    //合并消息， 节点收到的多个消息，将相同标签的数量相加
    def mergeMessage(count1: Map[VertexId, Long], count2: Map[VertexId, Long])
      : Map[VertexId, Long] = {
      (count1.keySet ++ count2.keySet).map { i =>  //i表示标签
        val count1Val = count1.getOrElse(i, 0L) //如果count1消息中有标签i，则拿到1，否则0
        val count2Val = count2.getOrElse(i, 0L)  //如果count2消息中有标签i，则拿到1，否则0
        i -> (count1Val + count2Val) 	//将消息中标签i出现的次数相加
      }.toMap 
    }
    // 如果没有收到消息则维持原来的标签，否则将收到的最多的标签作为自己新的标签
    def vertexProgram(vid: VertexId, attr: Long, message: Map[VertexId, Long]): VertexId = {
      if (message.isEmpty) attr else message.maxBy(_._2)._1
    }
    //初始化消息，空的标签Map信息
    val initialMessage = Map[VertexId, Long]()

    //调用 Pregel计算
    Pregel(lpaGraph, initialMessage, maxIterations = maxSteps)(
      vprog = vertexProgram,
      sendMsg = sendMessage,
      mergeMsg = mergeMessage)
  }

```


### 最短路径
graphx的最短路径算法只针对非带权图（即边的权重相等），采用迪杰斯特拉算法。

#### 简介
最短路径算法用来计算任意两个节点之间的最短距离，给定一组节点集合，求图中所有节点与集合中节点的最短路径。

#### 应用场景
交通路线查询:貌似最短路径算法是图计算工具普遍提供出来的算法，但好像直接使用它的业务场景相对较少，了解有限。

#### 算法流程
1. 初始化，给每个节点一个空的Map用来存储与顶点t的最短距离，而顶点t与自身的距离设置为0；
2. 每个节点将自己与顶点t的距离+1之后发送给所有的邻居节点；
3. 每个节点 i 将会收到来自邻居认为的 i 与顶点 t 的最短距离，对收到有同样key值的Map信息进行汇总，根据Map中标识最短距离的key值大小取最小值作为自己要接收的消息；
4. 节点将收到的消息进行处理，同样是比较收到消息中顶点t的最短距离与自己当前到顶点t的最短距离，取最小者作为自己到顶点t的新的距离。
   
  不断迭代上述2,3,4步。

```
object ShortestPaths extends Serializable {
  /** Stores a map from the vertex id of a landmark to the distance to that landmark. */
  type SPMap = Map[VertexId, Int]

  // _*表示变长参数，表示前面的x参数是一个参数序列，这里用来存储最终结果：每个节点与给定landmark集中节点的最短路径
  private def makeMap(x: (VertexId, Int)*) = Map(x: _*)

  // 向前走一步
  private def incrementMap(spmap: SPMap): SPMap = spmap.map { case (v, d) => v -> (d + 1) }

  // 每个节点i对于收到的多个邻居节点认为的i与指定landmark集中节点的路径长度信息，取最短的路径。
  private def addMaps(spmap1: SPMap, spmap2: SPMap): SPMap = {
    (spmap1.keySet ++ spmap2.keySet).map {
      k => k -> math.min(spmap1.getOrElse(k, Int.MaxValue), spmap2.getOrElse(k, Int.MaxValue))
    }(collection.breakOut) // more efficient alternative to [[collection.Traversable.toMap]]
  }

  /**
   * 计算图中所有节点与给定节点集合中每个顶点的最短路径
   * landmarks是进行求最短路径顶点id的集合，会计算landmarks集合中每个元素
   * 返回结果是一个图，每个顶点的属性是到landmarks点之间的最短距离
   */
  def run[VD, ED: ClassTag](graph: Graph[VD, ED], landmarks: Seq[VertexId]): Graph[SPMap, ED] = {
    // 将图中landmarks集合中的节点标记为自身到自身的最短距离为0，其余顶点属性初始化为空的Map()
    val spGraph = graph.mapVertices { (vid, attr) =>
      if (landmarks.contains(vid)) makeMap(vid -> 0) else makeMap()
    }

    // 初始化时给每个节点传入一个空的Map属性
    val initialMessage = makeMap()

    def vertexProgram(id: VertexId, attr: SPMap, msg: SPMap): SPMap = {
      addMaps(attr, msg)
    }

    // 向邻居发送消息时，会把自身与指定节点的距离＋1之后再发送
    def sendMessage(edge: EdgeTriplet[SPMap, _]): Iterator[(VertexId, SPMap)] = {
      val newAttr = incrementMap(edge.dstAttr)
      // 如果src节点与指定节点距离和dst＋1之后的距离相等，则不发送消息，否则向src节点发送新距离信息
      if (edge.srcAttr != addMaps(newAttr, edge.srcAttr)) Iterator((edge.srcId, newAttr))
      else Iterator.empty
    }

    Pregel(spGraph, initialMessage)(vertexProgram, sendMessage, addMaps)
  }
}
```

### 连通分量 ConnectedComponents
graphx的connectcomponent求解图中的连通体，在图中任意两个顶点之间存在路径可达，则该图是连通图，对应的极大连通子图即该算法要求的连通体。

#### 简介

Graphx用图中顶点的id来标识节点所属的连通体，同一个连通体的编号是采用该联通体中最小的节点id来标识的。

#### 应用场景
1. 社交网络的社区发现

2. 测试机器的连通性或进行网络连接的判断

#### 算法流程

核心思想: 用图中节点的id来表示连通分量，将自身id传递给邻居节点，能够发送消息的必然是在同一个连通分量中。

  1. 首先初始化图，将图中顶点id作为顶点的属性，开始状态是每个节点单独作为一个连通分量，分量id是节点id；
  2. 对于每条边，如果边两端节点属性相同（说明两个节点位于同一连通分量中），不需要发送消息，否则将较小的属性发送给较大属性的节点；
  3. 同一个节点对于收到的多个消息，只接收最小的消息；
  4. 节点将自身属性记录的id与收到的消息中的id进行比较，采用最小的id更新自己的属性。
   
  不断迭代上述2，3，4步。

```
object ConnectedComponents {
  /**
   *  返回图，图中节点的属性是当前连通分量中最小的顶点id
   * */
  def run[VD: ClassTag, ED: ClassTag](graph: Graph[VD, ED],
                                      maxIterations: Int): Graph[VertexId, ED] = {
    require(maxIterations > 0, s"Maximum of iterations must be greater than 0," +
      s" but got ${maxIterations}")

    // 初始化图：将图中顶点的id作为顶点属性
    val ccGraph = graph.mapVertices { case (vid, _) => vid }
    // 边上两个顶点，将id较小的顶点的属性发送给id较大的顶点（使得最终连通分支的id是分支上最小的节点id）
    // 如果边的两个顶点属性相同，则说明已经在同一个连通分支，不需要发送消息
    def sendMessage(edge: EdgeTriplet[VertexId, ED]): Iterator[(VertexId, VertexId)] = {
      if (edge.srcAttr < edge.dstAttr) {
        Iterator((edge.dstId, edge.srcAttr))
      } else if (edge.srcAttr > edge.dstAttr) {
        Iterator((edge.srcId, edge.dstAttr))
      } else {
        Iterator.empty
      }
    }
    // 初始化消息，因为节点在处理消息时接收最小的id更新自己的属性，所以初始时给每个节点发送一个超大的值
    val initialMessage = Long.MaxValue
    val pregelGraph = Pregel(ccGraph, initialMessage,
      maxIterations, EdgeDirection.Either)(
      vprog = (id, attr, msg) => math.min(attr, msg), // 取当前属性和收到消息的最小者更新属性
      sendMsg = sendMessage,
      mergeMsg = (a, b) => math.min(a, b))            // 接收多个消息中的最小者
    ccGraph.unpersist()
    pregelGraph
  } // end of connectedComponents


  def run[VD: ClassTag, ED: ClassTag](graph: Graph[VD, ED]): Graph[VertexId, ED] = {
    run(graph, Int.MaxValue)
  }
}
```

### 强连通分量 StronglyConnectedComponents
强连通分量是指在有向图中，如果两个顶点 v_i、 v_j 之间有一条从 v_i 到  v_j 的有向路径，同时还有一条从  v_j 到  v_i 的有向路径，则这两个顶点是强连通的。如果有向图G的每两个顶点都强连通则 G 是一个强连通图。有向图的极大强连通子图是该图的强连通分量。


#### 简介
graphx的强连通分量算法是计算一个图中所有的强连通分支，节点属性用来标识该节点所属的强连通分支，连通分支的标识是该连通分支中最小的节点id作为连通分支的id。

#### 应用场景
社区发现 ： 根据连通性来识别图中的子社区
查找路径回路


####  算法流程
通过循环不断寻找剩余图中的强连图分支。
1. 对图中所有节点设定初始连通分支id，用自己的节点id作为所属连通分支的id；

2. 首先做循环，将 **只存在单向边的或者孤立的节点** 和 **已经确认且打好标记的强连通分量中的节点**从图中去除；

3. 为图中节点正向着色，先用节点id为自身着色，之后沿着出边向邻居节点发送自己的着色id（只有较小的着色id向较大的着色id的节点发送消息）。

4. 为着色完成的图中节点反向打标签（是否完成连通分支id标记）。

在着色完成的图中，节点id与节点所在连通分支id相同时表明该节点是着色的root节点，标记为true。

若一个节点对应的入边的另外一个节点是true，则该节点也被标记为true。

节点沿着入边由src节点向dst节点发送自身标记情况，只有收到true的消息则节点便标记为true。

只有着色相同(表示root节点能到当前节点)，且一条边上dst节点发消息者是true(表示src节点能够到达root)但是src节点收消息者是false时，dst节点才会向src节点发送消息。 




下面以一个具体例子说明算法流程：
原始图:
![原始图](https://chenxh.github.io/img/scc-origin.png  "原始图")

对原始图进行该算法处理，每一步得到的结果展示如下图： 

![计算流程](https://chenxh.github.io/img/scc-process.png  "计算流程")

#### 源码分析

```
object StronglyConnectedComponents {
  def run[VD: ClassTag, ED: ClassTag](graph: Graph[VD, ED], numIter: Int): Graph[VertexId, ED] = {
    require(numIter > 0, s"Number of iterations must be greater than 0," +
      s" but got ${numIter}")

    // the graph we update with final SCC ids, and the graph we return at the end
    // 初始化图，将节点id作为节点属性，sccGraph是最后的返回结果图
    var sccGraph = graph.mapVertices { case (vid, _) => vid }
    // 在迭代中使用的图
    var sccWorkGraph = graph.mapVertices { case (vid, _) => (vid, false) }.cache()

    // 辅助变量prevSccGraph，用来unpersist缓存图
    var prevSccGraph = sccGraph

    var numVertices = sccWorkGraph.numVertices
    var iter = 0
    while (sccWorkGraph.numVertices > 0 && iter < numIter) {
      iter += 1
      // 此处循环内部工作：
      // 1.第一次循环进入时：将sccWorkGrpah图中只有单向边的节点或者孤立节点去掉； 后面循环进入时：将sccWorkGraph图中已经标识完成的强连通分量去掉。
      // 2.更新图中节点所属的强连通分支id
      // 只有在第一次进入第一层循环时，第一层循环内部的do-while循环才会循环多次，第2次以上只会只运行一次do{}的内容，因为后面图中不存在单向节点了。
      do {
        numVertices = sccWorkGraph.numVertices
        sccWorkGraph = sccWorkGraph.outerJoinVertices(sccWorkGraph.outDegrees) {
          (vid, data, degreeOpt) => if (degreeOpt.isDefined) data else (vid, true)
        }.outerJoinVertices(sccWorkGraph.inDegrees) {
          (vid, data, degreeOpt) => if (degreeOpt.isDefined) data else (vid, true)
        }.cache() //得到图中的有双向边的节点（vid，false）， 单向边或者孤立节点（vid，true），并且已经成功标记完连通分支的节点自身属性便是（vid，true）

        // 拿到图中只有单向边的节点和孤立节点
        val finalVertices = sccWorkGraph.vertices
            .filter { case (vid, (scc, isFinal)) => isFinal}
            .mapValues { (vid, data) => data._1}

        // write values to sccGraph
        //sccGraph[VertexId, ED]      finalVertices VertexRDD[VertexId]
        //外部第一次循环不会变动sccGraph节点的属性，只有在第二次开始才会将顶点所属的强连通分支id更新到图节点属性中。
        sccGraph = sccGraph.outerJoinVertices(finalVertices) {
          (vid, scc, opt) => opt.getOrElse(scc)
        }.cache()
        // materialize vertices and edges
        sccGraph.vertices.count()
        sccGraph.edges.count()
        // sccGraph materialized so, unpersist can be done on previous
        prevSccGraph.unpersist(blocking = false)
        prevSccGraph = sccGraph

        // 只保留属性attr._2为false的节点（这些节点是未完成连通分量打标签的节点，后面进入pregel重新着色）
        sccWorkGraph = sccWorkGraph.subgraph(vpred = (vid, data) => !data._2).cache()
      } while (sccWorkGraph.numVertices < numVertices) //图中存在单向边的节点，节点被删除变少了，则继续循环

      // 如果达到迭代次数则返回此时的sccGraph，将不再进入pregel进行下一步的着色和打标签。
      if (iter < numIter) {
        // 初始用vid为自身节点着色，每次重新进入pregel的图将重新着色
        sccWorkGraph = sccWorkGraph.mapVertices { case (vid, (color, isFinal)) => (vid, isFinal) }
        sccWorkGraph = Pregel[(VertexId, Boolean), ED, VertexId](
          sccWorkGraph, Long.MaxValue, activeDirection = EdgeDirection.Out)(
          // vprog： 节点在自己所属连通分支和邻居所属分支中取最小者更新自己。
          (vid, myScc, neighborScc) => (math.min(myScc._1, neighborScc), myScc._2),
          // sendMsg：正向（out）向邻居传播自身所属的连通分支（只有当自己所属连通分支比邻居小才会发送消息）
          e => {
            if (e.srcAttr._1 < e.dstAttr._1) {
              Iterator((e.dstId, e.srcAttr._1))
            } else {
              Iterator()
            }
          },
          // mergeMsg： 多条消息（邻居的连通分支）取最小者
          (vid1, vid2) => math.min(vid1, vid2))

        //第二个pregel：为着色后的节点打标签，final表示该节点的连通分支id已经标记完成。
        sccWorkGraph = Pregel[(VertexId, Boolean), ED, Boolean](
          sccWorkGraph, false, activeDirection = EdgeDirection.In)(
          // vprog： 如果节点id和所属连通分支id相同，则该节点是root
          //         root节点是完成连通分支标记的节点，是final （final是被标记为true）
          //         如果节点和final节点是邻居（收到的消息是final），则该节点也是final
          (vid, myScc, existsSameColorFinalNeighbor) => {
            val isColorRoot = vid == myScc._1
            (myScc._1, myScc._2 || isColorRoot || existsSameColorFinalNeighbor)
          },
          // 从完成着色的分量的root开始，反向（in）遍历节点，当一条边上两个节点的着色不同时则不发送消息。
          e => {
            val sameColor = e.dstAttr._1 == e.srcAttr._1
            val onlyDstIsFinal = e.dstAttr._2 && !e.srcAttr._2
            if (sameColor && onlyDstIsFinal) {
              Iterator((e.srcId, e.dstAttr._2))
            } else {
              Iterator()
            }
          },
          // mergeMsg
          (final1, final2) => final1 || final2)
      }
    }
    sccGraph
  }

}
```



