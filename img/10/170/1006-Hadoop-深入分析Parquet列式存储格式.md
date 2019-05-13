#### 对比测试
==以下测试方法不一定科学完善，但有一定代表性。压缩率和访问性能上ORC和Parquet表现突出==。

##### 测试1
Hive + Impala + Impala分区、Parquet查询性能对比：<br />
<img src="https://richie-leo.github.io/ydres/img/10/170/1006/hive-impala-parquet-vs.png" style="max-width:600px;width:99%;" />

##### 测试2
Parquet和Sequence File的存储和查询性能对比：<br />
<img src="https://richie-leo.github.io/ydres/img/10/170/1006/parquet-vs-sequnce-file-disk.png" style="max-width:620px;width:99%;" />

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/parquet-vs-sequnce-file-perf.png" style="max-width:550px;width:99%;" />

##### 测试3
参考[Hive的几种常见压缩格式（ORC，Parquet，Sequencefile，RCfile，Avro）的读写查询性能测试](https://blog.csdn.net/weixin_36714575/article/details/80091578)，对100G的JSON日志文件进行测试对比，结果如下：

存储格式 | ORC | Sequencefile | Parquet | RCfile | Avro
---|---|---|---|---|---
数据压缩后大小 | 1.8G | 67.0G | 11G | 63.8G | 66.7G
存储耗费时间 | 535.7s | 625.8s | 537.3s | 543.48 | 544.3
SQL查询响应速度 | 19.63s | 184.07s | 24.22s | 88.5s | 281.65s

--------
> 原文：[深入分析 Parquet 列式存储格式](https://www.infoq.cn/article/in-depth-analysis-of-parquet-column-storage-format)

#### 简介
Parquet 是面向分析型业务的列式存储格式，由 Twitter 和 Cloudera 合作开发，2015 年 5 月从 Apache 的孵化器里毕业成为 Apache 顶级项目，最新的版本是 1.8.0。

#### 列式存储
列式存储和行式存储相比有哪些优势呢？

1. 可以跳过不符合条件的数据，只读取需要的数据，降低 IO 数据量。
2. 压缩编码可以降低磁盘存储空间。由于同一列的数据类型是一样的，可以使用更高效的压缩编码（例如 Run Length Encoding 和 Delta Encoding）进一步节约存储空间。
3. 只读取需要的列，支持向量运算，能够获取更好的扫描性能。

当时 Twitter 的日增数据量达到压缩之后的 100TB+，存储在 HDFS 上，工程师会使用多种计算框架（例如 MapReduce, Hive, Pig 等）对这些数据做分析和挖掘；日志结构是复杂的嵌套数据类型，例如一个典型的日志的 schema 有 87 列，嵌套了 7 层。所以需要设计一种列式存储格式，既能支持关系型数据（简单数据类型），又能支持复杂的嵌套类型的数据，同时能够适配多种数据处理框架。

关系型数据的列式存储，可以将每一列的值直接排列下来，不用引入其他的概念，也不会丢失数据。关系型数据的列式存储比较好理解，而嵌套类型数据的列存储则会遇到一些麻烦。如图 1 所示，我们把嵌套数据类型的一行叫做一个记录（record)，嵌套数据类型的特点是一个 record 中的 column 除了可以是 Int, Long, String 这样的原语（primitive）类型以外，还可以是 List, Map, Set 这样的复杂类型。在行式存储中一行的多列是连续的写在一起的，在列式存储中数据按列分开存储，例如可以只读取 A.B.C 这一列的数据而不去读 A.E 和 A.B.D，那么如何根据读取出来的各个列的数据重构出一行记录呢？

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/01.png" style="max-width:500px;width:99%;" /><br />
图 1 行式存储和列式存储

Google 的[Dremel](http://research.google.com/pubs/pub36632.html)系统解决了这个问题，核心思想是使用“record shredding and assembly algorithm”来表示复杂的嵌套数据类型，同时辅以按列的高效压缩和编码技术，实现降低存储空间，提高 IO 效率，降低上层应用延迟。Parquet 就是基于 Dremel 的数据模型和算法实现的。

#### Parquet 适配多种计算框架
Parquet 是语言无关的，而且不与任何一种数据处理框架绑定在一起，适配多种语言和组件，能够与 Parquet 配合的组件有：

查询引擎: Hive, Impala, Pig, Presto, Drill, Tajo, HAWQ, IBM Big SQL

计算框架: MapReduce, Spark, Cascading, Crunch, Scalding, Kite

数据模型: Avro, Thrift, Protocol Buffers, POJOs

那么 Parquet 是如何与这些组件协作的呢？这个可以通过图 2 来说明。数据从内存到 Parquet 文件或者反过来的过程主要由以下三个部分组成：

1. 存储格式 (storage format) <br />
   [parquet-format](https://github.com/apache/parquet-format)项目定义了Parquet内部的数据类型、存储格式等。
2. 对象模型转换器 (object model converters) <br />
   这部分功能由[parquet-mr](https://github.com/apache/parquet-mr)项目来实现，主要完成外部对象模型与 Parquet 内部数据类型的映射。
3. 对象模型 (object models)
   对象模型可以简单理解为内存中的数据表示，Avro, Thrift, Protocol Buffers, Hive SerDe, Pig Tuple, Spark SQL InternalRow 等这些都是对象模型。Parquet也提供了一个[example object model](https://github.com/Parquet/parquet-mr/tree/master/parquet-column/src/main/java/parquet/example)帮助大家理解。

例如parquet-mr项目里的 parquet-pig 项目就是负责把内存中的 Pig Tuple 序列化并按列存储成 Parquet 格式，以及反过来把 Parquet 文件的数据反序列化成 Pig Tuple。

这里需要注意的是 Avro, Thrift, Protocol Buffers 都有他们自己的存储格式，但是 Parquet 并没有使用他们，而是使用了自己在parquet-format项目里定义的存储格式。所以如果你的应用使用了 Avro 等对象模型，这些数据序列化到磁盘还是使用的parquet-mr定义的转换器把他们转换成 Parquet 自己的存储格式。

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/02.png" style="max-width:700px;width:99%;" /><br />
图 2 Parquet 项目的结构

#### Parquet 数据模型
理解 Parquet 首先要理解这个列存储格式的数据模型。我们以一个下面这样的 schema 和数据为例来说明这个问题。

```parquet
message AddressBook {
 required string owner;
 repeated string ownerPhoneNumbers;
 repeated group contacts {
   required string name;
   optional string phoneNumber;
 }
}
```

这个 schema 中每条记录表示一个人的 AddressBook。有且只有一个 owner，owner 可以有 0 个或者多个 ownerPhoneNumbers，owner 可以有 0 个或者多个 contacts。每个 contact 有且只有一个 name，这个 contact 的 phoneNumber 可有可无。这个 schema 可以用图 3 的树结构来表示。

每个 schema 的结构是这样的：根叫做 message，message 包含多个 fields。每个 field 包含三个属性：repetition, type, name。repetition 可以是以下三种：required（出现 1 次），optional（出现 0 次或者 1 次），repeated（出现 0 次或者多次）。type 可以是一个 group 或者一个 primitive 类型。

Parquet 格式的数据类型没有复杂的 Map, List, Set 等，而是使用 repeated fields 和 groups 来表示。例如 List 和 Set 可以被表示成一个 repeated field，Map 可以表示成一个包含有 key-value 对的 repeated field，而且 key 是 required 的。

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/03.png" style="max-width:491px;width:99%;" /><br />
图 3 AddressBook 的树结构表示

#### Parquet 文件的存储格式
那么如何把内存中每个 AddressBook 对象按照列式存储格式存储下来呢？

在 Parquet 格式的存储中，一个 schema 的树结构有几个叶子节点，实际的存储中就会有多少 column。例如上面这个 schema 的数据存储实际上有四个 column，如图 4 所示。

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/04.png" style="max-width:700px;width:99%;" /><br />
图 4 AddressBook 实际存储的列

Parquet 文件在磁盘上的分布情况如图 5 所示。所有的数据被水平切分成 Row group，一个 Row group 包含这个 Row group 对应的区间内的所有列的 column chunk。一个 column chunk 负责存储某一列的数据，这些数据是这一列的 Repetition levels, Definition levels 和 values（详见后文）。一个 column chunk 是由 Page 组成的，Page 是压缩和编码的单元，对数据模型来说是透明的。一个 Parquet 文件最后是 Footer，存储了文件的元数据信息和统计信息。Row group 是数据读写时候的缓存单元，所以推荐设置较大的 Row group 从而带来较大的并行度，当然也需要较大的内存空间作为代价。一般情况下推荐配置一个 Row group 大小 1G，一个 HDFS 块大小 1G，一个 HDFS 文件只含有一个块。

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/05.png" style="max-width:601px;width:99%;" /><br />
图 5 Parquet 文件格式在磁盘的分布

拿我们的这个 schema 为例，在任何一个 Row group 内，会顺序存储四个 column chunk。这四个 column 都是 string 类型。这个时候 Parquet 就需要把内存中的 AddressBook 对象映射到四个 string 类型的 column 中。如果读取磁盘上的 4 个 column 要能够恢复出 AddressBook 对象。这就用到了我们前面提到的 “record shredding and assembly algorithm”。

#### Striping/Assembly 算法
对于嵌套数据类型，我们除了存储数据的 value 之外还需要两个变量 Repetition Level(R), Definition Level(D) 才能存储其完整的信息用于序列化和反序列化嵌套数据类型。Repetition Level 和 Definition Level 可以说是为了支持嵌套类型而设计的，但是它同样适用于简单数据类型。在 Parquet 中我们只需定义和存储 schema 的叶子节点所在列的 Repetition Level 和 Definition Level。

##### ==Definition Level==
嵌套数据类型的特点是有些 field 可以是空的，也就是没有定义。如果一个 field 是定义的，那么它的所有的父节点都是被定义的。从根节点开始遍历，当某一个 field 的路径上的节点开始是空的时候我们记录下当前的深度作为这个 field 的 Definition Level。如果一个 field 的 Definition Level 等于这个 field 的最大 Definition Level 就说明这个 field 是有数据的。对于 required 类型的 field 必须是有定义的，所以这个 Definition Level 是不需要的。在关系型数据中，optional 类型的 field 被编码成 0 表示空和 1 表示非空（或者反之）。

##### ==Repetition Level==
记录该 field 的值是在哪一个深度上重复的。只有 repeated 类型的 field 需要 Repetition Level，optional 和 required 类型的不需要。Repetition Level = 0 表示开始一个新的 record。在关系型数据中，repetion level 总是 0。

下面用 AddressBook 的例子来说明 Striping 和 assembly 的过程。

对于每个 column 的最大的 Repetion Level 和 Definition Level 如图 6 所示。

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/06.png" style="max-width:627px;width:99%;" /><br />
图 6 AddressBook 的 Max Definition Level 和 Max Repetition Level

下面这样两条 record：

```parquet
AddressBook {
 owner: "Julien Le Dem",
 ownerPhoneNumbers: "555 123 4567",
 ownerPhoneNumbers: "555 666 1337",
 contacts: {
   name: "Dmitriy Ryaboy",
   phoneNumber: "555 987 6543",
 },
 contacts: {
   name: "Chris Aniszczyk"
 }
}
AddressBook {
 owner: "A. Nonymous"
}
```

以 contacts.phoneNumber 这一列为例，"555 987 6543"这个 contacts.phoneNumber 的 Definition Level 是最大 Definition Level=2。而如果一个 contact 没有 phoneNumber，那么它的 Definition Level 就是 1。如果连 contact 都没有，那么它的 Definition Level 就是 0。

下面我们拿掉其他三个 column 只看 contacts.phoneNumber 这个 column，把上面的两条 record 简化成下面的样子：

```parquet
AddressBook {
 contacts: {
   phoneNumber: "555 987 6543"
 }
 contacts: {
 }
}
AddressBook {
}
```

这两条记录的序列化过程如图 7 所示：

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/07.png" style="max-width:577px;width:99%;" /><br />
图 7 一条记录的序列化过程

如果我们要把这个 column 写到磁盘上，磁盘上会写入这样的数据（图 8）：

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/08.png" style="max-width:211px;width:99%;" /><br />
图 8 一条记录的磁盘存储

注意：NULL 实际上不会被存储，如果一个 column value 的 Definition Level 小于该 column 最大 Definition Level 的话，那么就表示这是一个空值。

下面是从磁盘上读取数据并反序列化成 AddressBook 对象的过程：

1. 读取第一个三元组 R=0, D=2, Value=”555 987 6543” <br />
   ==R=0表示是一个新的record==，要根据schema创建一个新的nested record直到Definition Level=2。<br />
   ==D=2说明Definition Level=Max Definition Level，那么这个Value就是contacts.phoneNumber这一列的值，赋值操作contacts.phoneNumber=”555 987 6543”==。
2. 读取第二个三元组 R=1, D=1 <br />
   ==R=1表示不是一个新的record，是上一个record中一个新的contacts==。<br />
   ==D=1 表示contacts定义了，但是contacts的下一个级别也就是phoneNumber没有被定义，所以创建一个空的contacts==。
3. 读取第三个三元组 R=0, D=0 <br />
   R=0表示一个新的record，根据schema创建一个新的nested record直到Definition Level=0，也就是创建一个AddressBook根节点。

可以看出在 Parquet 列式存储中，对于一个 schema 的所有叶子节点会被当成 column 存储，而且叶子节点一定是 primitive 类型的数据。对于这样一个 primitive 类型的数据会衍生出三个 sub columns (R, D, Value)，也就是从逻辑上看除了数据本身以外会存储大量的 Definition Level 和 Repetition Level。那么这些 Definition Level 和 Repetition Level 是否会带来额外的存储开销呢？实际上这部分额外的存储开销是可以忽略的。因为对于一个 schema 来说 level 都是有上限的，而且非 repeated 类型的 field 不需要 Repetition Level，required 类型的 field 不需要 Definition Level，也可以缩短这个上限。例如对于 Twitter 的 7 层嵌套的 schema 来说，只需要 3 个 bits 就可以表示这两个 Level 了。

对于存储关系型的 record，record 中的元素都是非空的（NOT NULL in SQL）。Repetion Level 和 Definition Level 都是 0，所以这两个 sub column 就完全不需要存储了。所以在存储非嵌套类型的时候，Parquet 格式也是一样高效的。

上面演示了一个 column 的写入和重构，那么在不同 column 之间是怎么跳转的呢，这里用到了有限状态机的知识，详细介绍可以参考Dremel。

#### 数据压缩算法
列式存储给数据压缩也提供了更大的发挥空间，除了我们常见的 snappy, gzip 等压缩方法以外，由于列式存储同一列的数据类型是一致的，所以可以使用更多的压缩算法。


压缩算法 | 使用场景
---|---
Run Length Encoding | 重复数据
Delta Encoding | 有序数据集，例如 timestamp，自动生成的 ID，以及监控的各种 metrics
Dictionary Encoding | 小规模的数据集合，例如 IP 地址
Prefix Encoding | Delta Encoding for strings

#### 性能
Parquet 列式存储带来的性能上的提高在业内已经得到了充分的认可，特别是当你们的表非常宽（column 非常多）的时候，Parquet 无论在资源利用率还是性能上都优势明显。具体的性能指标详见参考文档。

Spark 已经将 Parquet 设为默认的文件存储格式，Cloudera 投入了很多工程师到 Impala+Parquet 相关开发中，Hive/Pig 都原生支持 Parquet。Parquet 现在为 Twitter 至少节省了 1/3 的存储空间，同时节省了大量的表扫描和反序列化的时间。这两方面直接反应就是节约成本和提高性能。

如果说 HDFS 是大数据时代文件系统的事实标准的话，Parquet 就是大数据时代存储格式的事实标准。

#### 参考文档
http://parquet.apache.org/ <br />
https://blog.twitter.com/2013/dremel-made-simple-with-parquet <br />
http://blog.cloudera.com/blog/2015/04/using-apache-parquet-at-appnexus/ <br />
http://blog.cloudera.com/blog/2014/05/using-impala-at-scale-at-allstate/

--------
> 原文：[新一代列式存储格式Parquet的最佳实践](https://blog.csdn.net/yu616568/article/details/50993491)

数据模型：
```parquet
message Document {
    required int64 DocId;
    optional group Links {
        repeated int64 Backward;
        repeated int64 Forward;
    }
    repeated group Name {
        repeated group Language {
            required string Code;
            optional string Country;
        }
        optional string Url;
    }
}
```

#### Repetition Levels
为了支持repeated类型的节点，在写入的时候该值等于它和前面的值在哪一层节点是不共享的。在读取的时候根据该值可以推导出哪一层上需要创建一个新的节点，例如对于这样的一个schema和两条记录。

```parquet
message nested {
     repeated group leve1 {
          repeated string leve2;
     }
}
r1:[[a,b,c,] , [d,e,f,g]]
r2:[[h] , [i,j]]
```

计算repetition level值的过程如下：
- value=a是一条记录的开始，和前面的值(已经没有值了)在根节点(第0层)上是不共享的，所以repeated level=0.
- value=b它和前面的值共享了level1这个节点，但是level2这个节点上是不共享的，所以repeated level=2.
- 同理value=c, repeated level=2.
- value=d和前面的值共享了根节点(属于相同记录)，但是在level1这个节点上是不共享的，所以repeated level=1.
- value=h和前面的值不属于同一条记录，也就是不共享任何节点，所以repeated level=0.

根据以上的分析每一个value需要记录的repeated level值如下：

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/10.png" style="max-width:232px;width:99%;" />

在读取的时候，顺序的读取每一个值，然后根据它的repeated level创建对象，当读取value=a时repeated level=0，表示需要创建一个新的根节点(新记录)，value=b时repeated level=2，表示需要创建一个新的level2节点，value=d时repeated level=1，表示需要创建一个新的level1节点，当所有列读取完成之后可以创建一条新的记录。本例中当读取文件构建每条记录的结果如下：

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/11.png" style="max-width:620px;width:99%;" />

可以看出repeated level=0表示一条记录的开始，并且repeated level的值只是针对路径上的repeated类型的节点，因此在计算该值的时候可以忽略非repeated类型的节点，在写入的时候将其理解为该节点和路径上的哪一个repeated节点是不共享的，读取的时候将其理解为需要在哪一层创建一个新的repeated节点，这样的话每一列最大的repeated level值就等于路径上的repeated节点的个数（不包括根节点）。减小repeated level的好处能够使得在存储使用更加紧凑的编码方式，节省存储空间。

#### Definition Levels
有了repeated level我们就可以构造出一个记录了，为什么还需要definition levels呢？由于repeated和optional类型的存在，可能一条记录中某一列是没有值的，假设我们不记录这样的值就会导致本该属于下一条记录的值被当做当前记录的一部分，从而造成数据的错误，因此对于这种情况需要一个占位符标示这种情况。

definition level的值仅仅对于空值是有效的，表示在该值的路径上第几层开始是未定义的，对于非空的值它是没有意义的，因为非空值在叶子节点是定义的，所有的父节点也肯定是定义的，因此它总是等于该列最大的definition levels。例如下面的schema。

```parquet
message ExampleDefinitionLevel {
  optional group a {
    optional group b {
      optional string c;
    }
  }
}
```

它包含一个列a.b.c，这个列的的每一个节点都是optional类型的，当c被定义时a和b肯定都是已定义的，当c未定义时我们就需要标示出在从哪一层开始时未定义的，如下面的值：

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/12.png" style="max-width:500px;width:99%;" />

由于definition level只需要考虑未定义的值，而对于repeated类型的节点，只要父节点是已定义的，该节点就必须定义（例如Document中的DocId，每一条记录都该列都必须有值，同样对于Language节点，只要它定义了Code必须有值），所以计算definition level的值时可以忽略路径上的required节点，这样可以减小definition level的最大值，优化存储。

#### 一个完整的例子
本节我们使用Dremel论文中给的Document示例和给定的两个值r1和r2展示计算repeated level和definition level的过程，这里把未定义的值记录为NULL，使用R表示repeated level，D表示definition level。

<img src="https://richie-leo.github.io/ydres/img/10/170/1006/13.png" style="max-width:800px;width:99%;" />

首先看DocuId这一列，对于r1，DocId=10，由于它是记录的开始并且是已定义的，所以R=0，D=0，同样r2中的DocId=20，R=0，D=0。

对于Links.Forward这一列，在r1中，它是未定义的但是Links是已定义的，并且是该记录中的第一个值，所以R=0，D=1，在r1中该列有两个值，value1=10，R=0(记录中该列的第一个值)，D=2(该列的最大definition level)。

对于Name.Url这一列，r1中它有三个值，分别为`url1='http://A'`，它是r1中该列的第一个值并且是定义的，所以R=0，D=2；`value2='http://B'`，和上一个值value1在Name这一层是不相同的，所以R=1，D=2；value3=NULL，和上一个值value2在Name这一层是不相同的，所以R=1，但它是未定义的，而Name这一层是定义的，所以D=1。r2中该列只有一个值`value3='http://C'`，R=0，D=2.

最后看一下`Name.Language.Code`这一列，r1中有4个值，`value1='en-us'`，它是r1中的第一个值并且是已定义的，所以R=0，D=2(由于Code是required类型，这一列repeated level的最大值等于2)；`value2='en'`，它和value1在Language这个节点是不共享的，所以R=2，D=2；value3=NULL，它是未定义的，但是它和前一个值在Name这个节点是不共享的，在Name这个节点是已定义的，所以R=1，D=1；`value4='en-gb'`，它和前一个值在Name这一层不共享，所以R=1，D=2。在r2中该列有一个值，它是未定义的，但是Name这一层是已定义的，所以R=0，D=1.
