#### Kylin架构

<img src="https://richie-leo.github.io/ydres/img/10/170/1005/kylin_diagram.png" style="max-width:750px; width:99%;" />

1. 左边是数据源，从这里读取源数据，通过Hadoop的MapReduce预计算生成OLAP Cube，数据源支持Hive、Kafka(流式数据)、RDBMS。<br />
   ==Kylin采用星型模型(Star Schema Data)，只支持一个事实表，数据和业务场景较复杂时，需要在Hive中进行处理，例如建立大宽表、视图等==。
2. OLAP Cube：预计算结果，存储在HBase中
3. Query Engine：使用开源的Calcite框架实现SQL解析，生成语法树(逻辑计划)
4. Routing：将SQL执行计划翻译为对Cube的查询，即HBase的Scan操作，Group By用来找到cuboid，过滤条件转换成Scan的开始和结束位置。有些操作无法命中Cube，会转向查询原始数据(速度慢) <br />
   <img src="https://richie-leo.github.io/ydres/img/10/170/1005/sql-plan-on-precaculated-cube.png" style="max-width:450px; width:99%;" />
5. Metadata：元数据，包括Project、Model、Cube的定义，Job的定义和输出等。存储在HBase中

#### Cube原理
Kylin通过执行MapReduce/Spark任务构建Cube，对业务所需的维度组合和度量进行预聚合，当查询到达时直接访问预计算聚合结果，省去对大数据的扫描和运算，这就是Kylin高性能查询的基本实现原理。

##### Cube结构
- Cuboid：维度的任意组合成为一个cuboid；
- Cube：所有维度的组合形成一个Cube；

下图是一个4维Cube `[time, item, location, supplier]`的逻辑结构示意图：每个节点代表一个Cuboid，整个图代表一个Cube：<br />
<img src="https://richie-leo.github.io/ydres/img/10/170/1005/cube-structure.png" style="max-width:500px; width:99%;" />

其中，==所有维度都有的组合称为Base Cuboid==。Base Cuboid下面并不是单一一个组合值，而是所有维度的全部唯一值的组合。<br />
对于`A, B, C, D`四维cube，base cuboid与```select A, B, C, D, count(1) from xxx group by A, B, C, D```基本等价。

##### Cube存储
Cube使用key-value形式存储在HBase中，key是每个成员的组合值，不同cuboid下面的key结构不一样，例如：`cuboid = { time, location, product }`下面的某个key可能是`2016:Nanjing:iPhone6`。如果纬度值平均长度比较大，这种方式很浪费空间。Kylin将每个维度的唯一值提取出来做成数据字典(字典树Trie)，使用数据字典的索引以二进制形式存储到HBase中。这对应于设置界面中Rowkeys Encoding的dict方式。

##### ==Layered Cubing 逐层算法==
<img src="https://richie-leo.github.io/ydres/img/10/170/1005/layered-cubing.jpg" style="max-width:480px; width:99%;" />

上图是一个四维Cube，有维度A、B、C、D，它需要五轮Map-Reduce来完成：第一轮MR的输入是源数据，这一步会对维度列的值进行编码，并计算ABCD组合的结果。接下来的MR以上一轮的输出为输入，向上聚合计算三个维度的组合：ABC, BCD, ABD, 和ACD，依此类推，直到算出所有的维度组合。

- 优点：简单。Cube聚合的过程就是把要聚合掉的维度从key中减掉组成新的key交给Map-Reduce，由Map-Reduce框架对新key做排序和再聚合，计算结果写到HDFS。这个算法很好地利用了Map-Reduce框架。
- 缺点：
  - 数据量大的时候，Hadoop主要利用磁盘做排序，数据在Mapper和Reducer之间还需要洗牌(shuffle)。在计算Cube的时候，集群的IO使用率往往很高; 在运行一些大的任务时，瓶颈会出现在网络传输和磁盘读写上，而CPU和内存的使用率比较低。
  - 此外, 因为需要递交N+1次Map-Reduce任务，每次递交任务，都需要检查集群是否有可用的节点能否满足资源要求，如果没有还需等待其它任务释放资源。反复的任务递交，给Hadoop集群带来额外的调度开销。特别是当集群比较繁忙的时候，等待的时间常常会非常可观，这些都导致 了Cube构建的时间比较长。

##### ==Fast Cubing 快速数据立方算法==
<img src="https://richie-leo.github.io/ydres/img/10/170/1005/fast-cubing.jpg" style="max-width:550px; width:99%;" />

核心思想是最大化利用Mapper端的CPU和内存，对分配的数据块，将需要的组合全都做计算后再输出给Reducer，由Reducer再做一次合并(merge)，从而计算出完整数据的所有组合。如此，经过一轮Map-Reduce就完成了以前需要N轮的Cube计算。

在Mapper内部，也有一些优化，下图是一个典型的四维Cube的生成树。第一步计算Base Cuboid，再基于它计算减少一个维度的组合。基于parent节点计算child节点，可以重用之前的计算结果。当计算child节点时，需要parent节点的值尽可能留在内存中。如果child节点还有child，那么递归向下，所以它是一个深度优先遍历。当有一个节点没有child，或者它的所有child都已经计算完，这时候它就可以被输出，占用的内存就可以释放。

<img src="https://richie-leo.github.io/ydres/img/10/170/1005/cube-build-tree-on-mapper.jpg" style="max-width:550px; width:99%;" />

- 优点：总的IO量比以前大大减少；可以脱离Map-Reduce，很容易地在其它场景或框架下执行，例如Streaming 和Spark。<br />
- 缺点：
  - 代码比Layered Cubing复杂，并且引入多线程机制，同时还要估算JVM可用内存，当内存不足时需要将数据暂存到磁盘，所有这些都增加复杂度。
  - 对Hadoop资源要求较高，用户应尽可能在Mapper上多分配内存。如果内存很小，该算法需要频繁借助磁盘，性能优势就会较弱。在极端情况下(如数据量很大同时维度很多)，任务可能会由于超时等原因失败;

Cube查询和构建优化，参考[Apache Kylin 深入Cube和查询优化](http://www.uml.org.cn/bigdata/201707052.asp?artid=19592)、[Apache Kylin v1.5版本中的快速数据立方算法揭秘](http://www.sohu.com/a/110492992_116235)。

#### 部署
Kylin的部署，与Hadoop、Hive、HBase的架构无关，只需在Kylin中配置好相关的访问信息即可，另外Kylin服务器上需要有Hive、HBase客户端。

Kylin服务器节点是无状态的，只需每个节点指向相同的HBase，即组成一个集群。==同一个集群中只能有1个`kylin.metadata.url`为job或者all的节点，用于构建Cube，其余节点全部设为query，提供基于Cube的只读查询==。

通过Nginx、LVS为kylin集群配置负载均衡。

Cube构建时资源消耗比较严重，==可以将Hive和HBase的Hadoop集群分开部署，将HDFS加载原始数据以及MapReduce计算的负载隔离开，降低构建时影响查询性能==。

##### deploy.env
配置kylin的用途，可选DEV、QA(默认)、PROD。PROD模式下禁止创建Cube，DEV模式下可以。生产环境的Cube，使用kylin提供的工具由开发、测试环境导入。

#### 创建Cube

1. 在Hive中建立好事实表、维度表，导入数据；
   若必要，在Hive创建宽表、视图；数据导入可选Kettle、Sqoop、Flume等；
2. 在Kylin中创建Project；
3. 建立数据源；
   - Load Hive Table：手工方式填写Hive表，多个表明逗号分隔；
   - Load Hive Table From Tree：图形界面选择Hive表；
   - ==Add Streaming Table==：基于Kafka定义流式表，Kylin以一定时间间隔从Kafka读取数据，增量构建Cube，以此实现==准实时Cube构建==；
4. 建立Model；
   - 选择事实表、维度表，确定关联方式、关联条件等，确定度量Measures；
   - 设置Cube增量刷新分区字段，若分区字段为空，则每次全量刷新Cube；
5. 创建Cube；

##### 维度聚合组 Aggregation Groups
将维度分组，不在分组内的组合是不可能出现的，这样可以降低cuboid数量。

##### 维度类型
- ==Normal==：普通类型，所有维度组合构成Cube；
- ==Mandatory==：在每一次查询中都会用到的维度，维度组合中没有出现该维度的都无需计算和保存，因此Cuboid可以减少一半<br />
  <img src="https://richie-leo.github.io/ydres/img/10/170/1005/mandatory-dimension.png" />
- ==Hierarchy==：带层级的维度，比如：省市区，一般只会出现`group by 省`、`group by 省、市`、`group by 省、市、区`，像`group by 市、区`(没有省份)是无意义的，无需计算保存<br />
  <img src="https://richie-leo.github.io/ydres/img/10/170/1005/hierarchy-dimension.png" />
- ==Derived==：该维度与维度表的主键一一对应(事实表用外键关联维度表主键)，因此可以使用主键值代替维度表上的多个维度，降低cuboid数量，详细的解释参看[这里](http://kylin.apache.org/docs/howto/howto_optimize_cubes.html)。<br />
  <img src="https://richie-leo.github.io/ydres/img/10/170/1005/derived-dimension.png" />
- ==Joint==：关联维度，多个维度必然一起出现，不可能单独存在；

##### Cube合并
增量cube可以进行多次build，每次生成一个segment，每个segment对应一个时间区间的cube，这些segment的时间区间连续不重合，对拥有多个segment的cube可以合并成一个cube，减少存储空间，提升查询性能。

#### 构建Cube流程
1. 生成Hive临时表 (Create Intermediate Flat Hive Table) <br />
   创建Hive中间临时表，关联事实表、维度表，选择需要的字段，将数据插入临时表中
2. 为事实表提取distinct column (Extract Fact Table Distinct Columns) <br />
   Hive中间表为输入，出现在事实表中的每个维度和度量提取唯一值，按列名写入文件
3. 构建维度词典 (Build Dimension Dictionary) <br />
   步骤2的结果+所有维度表为输入，计算维度字典。对每个Hive维度表，在HBase中生成一个SnapshotTable，保存字典(TrieDictionary)ID和值的对应关系。
4. 保存cuboid统计信息 (Save Cuboid Statistics)
5. 创建HTable (Create HTable) <br />
   HBase中保存cuboid的表
6. 计算生成Base Cuboid数据文件 (Build Base Cuboid Data)
7. 计算N层Cuboid (Build N-Dimension Cuboid Data) <br />
   使用`Layered Cubing`、`Fast Cubing`等算法
8. 将Cuboid数据转换为HFile (Convert Cuboid Data to HFile)
9. 将HFile导入HBase表 (Load HFile to HBase Table)
10. 更新Cube信息
11. 清理中间表
