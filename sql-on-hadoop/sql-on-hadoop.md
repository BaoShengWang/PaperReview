author：王宝生

e-mail：franciswbs@163.com

github:[https://github.com/BaoShengWang](https://github.com/BaoShengWang)



# ![](/assets/profile.png) {#1-sql-on-hadoop-分类}

# 1. SQL On Hadoop 分类 {#1-sql-on-hadoop-分类}

1.1 查询延时分类

AtScale 在 2016 年的一篇名为 \[15\]The Business Intelligence for Hadoop Benchmark 的 SQL On Hadoop 性能测评报告中指出：受查询数据量大小，查询类型 \(join 表个数，表大小，是否聚合\)，并发用户量等因素影响，没有一个 SQL On Hadoop 系统能够在所有场景下胜出。 比如 Impala 和 Presto 在并发场景下性能比较优越，Spark SQL 大表 Join 性能比较好。然而对于所有 SQL On Hadoop 而言，大表 Join 都比较慢。

在众多的 SQL On Hadoop 系统中，有必要对其进行一个分类。一般而言，用户更关心的是查询时延，根据用户提交查询到结果返回的时间长短，将 SQL 查询分为如下三类：batch SQL，interactive SQL，operation SQL, 如图 1。

![](https://mmbiz.qpic.cn/mmbiz_png/cokWkYcF4DekTUVbaiabavaliaaGJN4VZ5FfhEz7JrwAZ8GYybiajsibFCickvL1tJE0AfWicME1oxKEsGcjPRLlqxRQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

图 1 SQL On Hadoop 分类, 摘自文献 \[14\]

* **Batch SQL**，Batch SQL 的查询时间通常在分钟，小时级别，一般用于复杂的 ETL 处理，数据挖掘，高级分析。由于 Batch SQL 的查询延时比较高，因此支持查询内 \(Intra-query\) 容错是该类系统必须具备的属性，查询内容错是指，当节点宕机或者查询内部某个 Task 失败时，系统必须能够重新提交该 task 而不是重新提交整个查询来进行容错。Batch SQL 中最典型的系统是 Hive。Spark SQL 也可以归类到该系统。

* **Interactive SQL**，Interactive SQL 也叫做交互式 SQL 查询，用户通常在同一个表上反复的执行不同的查询，Interactive SQL 的查询时间通常在毫秒级或者秒级以内，一般不超过分钟级别。由于该类系统主要追求低延迟，而不过分强调查询内部容错，所以当某个 task 失败时，可以重新提交该查询以便进行容错，因为重新提交一个 SQL 查询的执行时间通常很短。Interactive SQL 在实现上通常采用 MPP 架构，并且将热点数据缓存到内存中，比如 Presto，Impala，Drill，HAWQ。鉴于 Spark SQL 也具有非常高效的查询速度，Spark SQL 也可以归类到 Interactive SQL 中。

* **Operation SQL**, 通常是单点查询，延时要求小于 1 秒，该类系统主要是 HBase。

1.2 架构分类

1.2.1 MPP 架构

MPP 架构的优点是查询速度快，通常在秒计甚至毫秒级以内就可以返回查询结果，这也是为何很多强调低延迟的系统采用 MPP 架构的原因。

下面重点看下 MPP 架构的缺点，MPP 架构最主要的缺点是不支持细粒度的容错，集群节点数量很难扩展到 100 个以上，如果集群出现落后节点，那么将影响整个系统的查询性能，此外不管 MPP 节点数量的多少，并发查询的数量通常只能达到 20 个左右。

**容错**，MPP 架构的容错特点是粗粒度容错，不能处理落后节点 \(Straggler node\)。粗粒度容错是指，某个 task 执行失败将导致整个查询失败，然后系统重新提交整个查询来获取结果。这种容错方式只适用于 Iterative SQL 这种低延迟的工作负载，而不适合 Batch SQL 场景，因为 Batch SQL 查询时间通常在分钟小时级别，重新提价一个查询代价太高。

落后节点，当一个节点执行速度慢于其他节点时，将导致整个系统的查询性能下降。

**扩展性**：受落后节点的影响，MPP 架构很难扩展到 100 个节点以上。如果某个节点慢于其他节点，那么整个系统的查询性能将受限于这个最慢的节点，而与集群节点数量无关。需要注意的是，在大型集群中落后节点是普遍存在的，随着集群节点数量的增加，落后节点出现的概率也增加，\[13\] 针对磁盘故障概率的统计如下：

![](https://mmbiz.qpic.cn/mmbiz_png/cokWkYcF4DekTUVbaiabavaliaaGJN4VZ5WkUibZ3mC4arDPQyVFz2tNFTsrDLokwTzznicgVeVayN5TqQ4ic2SVDwA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

如果集群包含 1000 个未使用一年的磁盘，那么每年将有大约 20 磁盘出现故障，平均每两周就会出现一个故障。当磁盘使用超过一年后，每年磁盘故障出现的概率将达到 8% 左右，平均每周将出现大约两次故障。由于这个原因，MPP 架构很难扩展到 100 个节点以上，一般在 50 个节点左右。

**并发**，MPP 架构的并发查询数量和集群节点数量无关。MPP 是对称结构，当执行一个查询时，该查询将被调度到集群中的每一个节点执行，这意味着一个包含 4 个节点的 MPP 集群和一个包含 400 个节点的 MPP 集群所支持的并发查询数量是相同的，也就是说，并发查询数量和集群节点数量无关，一般而言，当并发查询个数达到 20 左右时，整个系统的吞吐已经达到满负荷状态。

综上所述，MPP 架构不适合大规模部署，如果需要大规模部署，可以考虑 Spark Sql 这样的系统。

1.2.2 非 MPP 架构

典型的非 MPP 架构有 Hive，Spark Sql。他们分别构建在 MR 和 Spark 之上，优点是集群节点数量可以扩展到几百甚至上千个，支持细粒度容错。缺点是查询速度可能不如 MPP 架构。

# 2. 运行引擎的设计 {#2-运行引擎的设计}

## 2.1. 优化器 {#21-优化器}

目前 SQL On Hadoop 的查询优化器主要有两种：基于规则的 \(Rule-Based Optimizer\) 和基于代价的 \(Cost-Based Optimizer CBO\)。基于规则的优化器简单，易于实现，通过内置的一组规则来决定如何执行查询计划，这里不做介绍。

设计一个好的 CBO 优化器非常具有挑战性，一个好的 CBO 依赖于详细可靠的统计信息，比如每个列的最大值，最小值，表大小，表分区信息，桶信息，然而在 SQL On Hadoop 中，通常缺乏可靠的统计结果，代价估计代数，这使得在 SQL On Hadoop 中引入 CBO 很困难。尽管如此，鉴于 CBO 在运行可以更加智能的进行查询优化，仍然有越来越多的 SQL On Hadoop 开始支持 CBO，比如 Hive，Spark SQL\(计划中\)。

CBO 主要用来优化 shuffle，join，如何尽可能的避免 shuffle，提高 join 执行速度是 CBO 主要关注的问题，其中 Join 的实现方式和 Join 顺序是重点考虑的。在 SQL On Hadoop 主要有四种 join 实现方式：shuffle hash join,broadcast join,Bucket join，cartesian join：

* **shuffle hash join**，在 map 阶段按照 join key 对两个表执行 hash shuffle，这样拥有相同 join key 的元组将 shuffle 到同一个节点，在 reduce 阶段对表进行 join。

* **broadcast join**，当一个大表 join 一个小表时，并且小表可以完全放到内存中，此时可以将小表广播到大表所在的每一个计算节点，然后执行 join。这种 join 方式叫做 broadcast join 或者 map join。Broadcast join 优点是避免了 shuffle，提高 join 性能。

* **Bucket join**, 假设表 A 和表 B 使用 bucket 分区策略存储，并且表 A 和表 B 的 bucket 个数为 n，此时可以按照如下方式 join:bucket 1 of A join bucet 1 of B,......,bucket n of A join bucket n of B。

  Bucket join 优点是可以对两个大表执行 join，并且不需要将数据放到内存中，在 Hive 和 Spark2.0 中都支持 Bucket join。

* **cartesian join**，也叫做笛卡儿积 join，对两个表执行笛卡儿积 join，结果集中元素的数量是两个表大小的乘积。比如表 A 有 10 万行，表 B 有 10 万行，那么笛卡儿积 join 之后的表大小将达到 100 万条数据。因此除非到万不得已，否则不会使用笛卡儿积 join。

表的 join 顺序 \(Join order\) 主要有两种：left-deep tree（下图左）,bushy tree\(下图右\)。一个好的 CBO 应该能够根据 SQL 语句的特点，来自动选择使用 Left-deep tree 还是 bushy tree 执行 join。

* **Left-deep tree**, 如果对 A，B，C，D 执行 join，那么首先 A join B 得到一个临时表 AB 并 AB 物化到磁盘，然后 AB join C 得到中间临时表 ABC 并物化到磁盘，最后 ABC joinD 得到最终结果。可以发现，这种 join 顺序非常简单，缺点是只能串行 join，并且由于产生了大量的中间临时表，因此不太适合 OLAP 中的星型和雪花模型。

* **bushy tree**, 采用 bushy tree 方式，可以并行执行 A join B 和 C joinD。然后将二者的结果 AB 和 CD 进行 join 得到最终结果。Bushy tree 优点是可以并行 join，并且能够很好的处理星型模型和雪花模型。

![](https://mmbiz.qpic.cn/mmbiz_png/cokWkYcF4DekTUVbaiabavaliaaGJN4VZ5WgaI90UmD7OzE85nRvfvmXicAqsD8sBkTfrdJATnskPECZC2h9lRIibw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

图 2left-deep tree 和 bushy tree, 摘自文献 \[16\]

## 2.2. 查询执行引擎 {#22-查询执行引擎}

查询执行引擎 \(query execution engine\) 是 SQL On Hadoop 的核心组件。查询执行引擎的好坏对查询性能的影响非常大。目前主要有两种查询执行：火山执行模型和向量化执行引擎。在后面的向量化执行引擎章节中有详细的介绍。

# 3. 性能优化 {#3-性能优化}

从硬件资源角度将性能优化分为 3 个部分：

* **磁盘优化**：数据本地化，减少中间结果的物化，数据压缩，列存储文件，分区，块级索引

* **CPU 优化**：向量化执行引擎，动态代码生成，轻量级压缩算法，任务启动优化

* **内存和 CPU 缓存**：内存压缩列存储，堆外存储，缓存敏感数据结构和算法

## 3.1 数据本地化 {#31-数据本地化}

SQL On Hadoop 设计的一个基本原则是：将计算任务移动到数据所在的节点而不是反过来。这主要出于网络优化的目的，因为数据分布在不同的节点，如果移动数据那么将会产生大量的低效的网络数据传输。数据本地化一般分为三种：节点局部性 \(Node Locality\), 机架局部性 \(Rack Locality\) 和全局局部性 \(Global Locality\)。节点局部性是指将计算任务分配到数据所在的节点上，此时无需任何数据传输，效率最佳。机架局部性是指将计算任务移动到数据所在的机架，虽然计算任务和数据分属不同的计算节点，但是因为机架内部网络传输速度明显高于机架间网络传输，所以机架局部性也是一种不错的方式。其他的情况属于全局局部性，此时需要跨机架进行网络传输，会产生非常大的网络传输开销。

调度系统在进行任务调度时，应该尽可能的保证节点局部性，然后是机架局部性，如果以上两者都不能满足，调度系统也会通过网络传输将数据移动到计算任务所在的节点，虽然性能相对低效，但也比资源空置比较好。

为了实现数据本地化调度，调度系统会结合延迟调度算法来进行任务调度。核心思想是优先将计算任务调度到数据所在的节点 i，如果节点 i 没有足够的计算资源，那么等待几秒钟后如果节点 i 依然没有计算资源可用，那么就放弃数据本地化将该计算任务调度到其他计算节点。

## 3.2 减少中间结果的物化 {#32-减少中间结果的物化}

在一个追求低延迟的 SQL On Hadoop 系统中，尽可能的减少中间结果的磁盘物化可以极大的提高查询性能。 如下图，Hive 执行引擎采用 pull 获取数据，其优点是可以进行细粒度的容错，缺点是下游的 MapReduce 必须等待上游 MapReduce 完全将数据写入到磁盘后才能开始 pull 数据。Presto 采用 push 方式获取数据，数据完全以流的方式在不同 stage 之间进行传输，中间结果不需要物化到磁盘，从而使得 presto 具有非常高效的执行速度，缺点是不能支持细粒度的容错。

进行传输，中间结果不需要物化到磁盘，从而使得 presto 具有非常高效的执行速度，缺点是不能支持细粒度的容错。

![](https://mmbiz.qpic.cn/mmbiz_png/cokWkYcF4DekTUVbaiabavaliaaGJN4VZ5ibicEpS0vpM3kCyel0wRsgRcS7v85icuz0ibrS9MyCND19xPdoEBel84jA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

图 3push 和 pull

## 3.3 列存储 {#33-列存储}

传统的关系存储模型将一个元组的列连续存储，即使只查询一个列，也需要将整个元组读取出来，可以发现，当查询只有少量列时，性能非常低。

列存储的思想是将元组垂直划分为列族集合，每一个列族独立存储，列族可以退化为只仅包含一个列的平凡列族。当查询少量列时，列存储模型可以极大的减少磁盘 IO 操作，提高查询性能。当查询的列跨越多个列族时，需要将存储在不同列族中列数据拼接成原始数据，由于不同列族存储在不同的 HDFS 节点上，导致大量的数据跨越网络传输，从而降低查询性能。因此在实际使用列族时，通常根据业务查询特点，将频繁访问的列放在一个列族中。

在传统的数据库领域中，人们已经对列存储进行了非常深刻的研究，并且很多研究成果已经被应用到工业领域，其中包括轻量级压缩算法，直接操作压缩数据，延迟物化，向量化执行引擎。可是纵观目前 SQL On Hadoop 系统，这些技术的应用仍然远远的落后于传统数据库，在最近的一些 SQL On Hadoop 中已经添加了向量化执行引擎，轻量级压缩算法，但是诸如直接操作压缩数据，延迟解压等技术还没有被应用到 SQL on Hadop 系统。关于列存储的更多内容可以参见 \[20\]。

**列存储压缩**

列存储压缩算法具有如下特点：

**压缩比**列存储模型具有非常高的压缩比，通常可以达到 10：1，而行存储压缩比通常只有 4：1。如图 4：

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

图 4 重量级压缩算法

**轻量级压缩算法 \(Leight-Weight Compression\)**轻量级压缩算法是 CPU 友好的。行存储模型只能使用 zip，lzo，snappy 等重量级压缩算法，这些算法最大的缺点是压缩和解压缩速度比较慢，通常每秒只能解压至多几百兆数据。相反，列存储模型不仅可以使用重量级压缩算法，还可以使用一些非常轻量级的压缩算法，比如 Run-length encode，Bit Vector。轻量级压缩算法不仅具有较好的压缩比，而且还具有非常高的压缩和解压速度。目前在 ORC File 和 Parquet 存储中，已经支持 Bit packing,Run-length enode,Dictionary encode 等轻量级压缩算法。

**直接操作压缩数据 \(Operating Directly on Compressed Data\)**当使用轻量级压缩算法时，可能无需解压即可直接获取计算结果。例如:Run Length Encode 算法将连续重复的字符压缩为字符个数和字符，比如 aaaaaabbccccaaaa 将被压缩为 6a2b4c4a，其中 6a 表示有连续 6 个字符 a。现在假设一个某列包含上述压缩的字符串，当执行 select count\(\*\) from table where columnA=’a’时，不需要解压 6a2b4c4a，就能够知道 a 的个数是 10。

需要注意的是，由于行存储只能使用重量级压缩算法，所以直接操作压缩数据不能被应用到行存储。

**延迟解压**parquet 中的数据按块存储，每个块存储了最小值，最大值等轻量级索引，比如某个块的最小值最大值分别是 100 和 120，这表明该块中的任意一条数据都介于 100 到 120 之间，因此当我们执行 select column a from table where v&gt;120 时，执行引擎可以跳过这个数据块，而不必将其解压再进行数据过滤。相反，在行存储中，必须将数据块完整的读取到内存中，解压，然后再进行数据过滤，导致不必要的磁盘读取操作。

## 3.4 块级索引 {#34-块级索引}

传统数据库使用索引来优化查询性能，然而受限于 HDFS block 的放置策略，使用索引来优化 SQL On Hadoop 不是一件容易的事情。目前大部分 SQL On Hadoop 系统都不支持全局索引，取而代之使用的是块级索引，比如 Hive Index，ORC File，Parquet。块级索引的思想是在每一个数据块中添加一些诸如最大值，最小值的轻量级索引，当 SQL 引擎扫描 HDFS 文件时，可以跳过不符合条件的 Block，从而减少磁盘 IO 提高查询性能。如下图，在 ORC File 中，每一个 Stripe 都包含一个 Index Data,Index Data 中存储了列的最大值，最小值。当执行引擎执行 filter 这种查询时，只需要读取 Index Data 就行，如果符合条件就读取 Row Data，否则可以直接跳过 Row Data 的读取，从而减少磁盘 IO，提高查询性能。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

图 3-3 ORC Storage

最大值，最小值这样的统计索引主要用于优化范围查询性能，对于单点查询通常可以使用布隆过滤器作为索引，布隆过滤器可以在数据量非常大的情况下快速的查询数据。

## 3.5 分区 {#35-分区}

MPP 数据库根据分区策略将一个表水平或者垂直切分为一个子表集合，不同的子表存储在不同的节点，这样可以并行的处理不同的子表。典型的分区策略有哈希，范围。

SQL On Hadoop 中也存在表分区的概念，一个表分区存储在一个 HDFS 文件目录下，文件目录以列名 = 列值方式存储。比如我们在 Hive 中执行如下 SQL：

```
CREATE TABLE test_table(id string,name int) PARTITION BY(ds string)。
```

当向 test\_table 中插入如下元组时：

```
(id=‘10010’，name=‘sql on hadoop’,ds=‘2017-05-31’)
(id=‘10010’，name=‘sql on hadoop’,ds=‘2017-05-32’)
```

HDFS 中将创建如下目录:

```
/user/hive/warehouse/test_table/ds=2017-05-31
/user/hive/warehouse/test_table/ds=2017-05-32
```

当执行 SELECT \* FROM test\_table WHERE ds=’2017-05-31’时，只需要扫描 ds=2017-05-31目录即可，这样可以跳过大量无关数据的扫描，从而加快数据查询速度。

目前大部分 SQL On Hadoop 都支持分区功能，比如 Hive，Presto，Impala，Spark SQL。

## 3.6 压缩 {#36-压缩}

一般情况下，压缩 HDFS 中的文件可以极大的提高查询性能。压缩能够减少数据所占用的存储空间，减少磁盘 IO 的读写，提高数据处理速度，此外，压缩还能够减少网络传输量，提高网络传输速度。在 SQL On Hadoop 中，压缩主要应用在 HDFS 中的数据源，shuffle 数据，最终计算结果。

如果应用程序是 io-bound 的，那么压缩数据可以提高数据处理速度，因为压缩后的数据变小了，所以可以增加数据读写速度。需要主要的是，压缩算法并不是压缩比越高越好，压缩率越高的算法压缩和解压缩速度就越慢，用户需要在 cpu 和 io 之间取得一个良好的平衡。例如 gzip2 拥有非常高的压缩比，但是其压缩和解压缩速度却非常慢，甚至可能超过数据未压缩时的读写时间，因此没有 SQL On Hadooop 系统使用 gzip2 算法，目前在 SQL On Hadoop 系统中比较流行的压缩算法主要有：Snappy，Lzo，Glib。

如果应用程序是 cpu-bound 的，那么选择一个可以 splittable 的压缩算法是很重要的，如果一个文件是 splittabe 的，那么这个文件可以被切分为多个可以并行读取的数据块，这样 MR 或者 Spark 在读取文件时，会为每一个数据块分配一个 task 来读取数据，从而提高数据查询速度。

## 3.7 向量化执行引擎 {#37-向量化执行引擎}

查询执行引擎 \(query execution engine\) 是数据库中的一个核心组件，用于将查询计划转换为物理计划，并对其求值返回结果。查询执行引擎对数据库系统性能影响很大，目前主要的执行引擎有如下四类：Volcano-style，Block-oriented processing，Column-at-a-time，Vectored iterator model。下面分别介绍这四种执行引擎。

**Volcano-style**, 最早的查询执行引擎是 Volcano-style execution engine\(火山执行引擎，火山模型\)，也叫做迭代模型 \(iterator model\)，或者 one-tuple-at-a-time。在这种模型中，查询计划是一个由 operator 组成的 tree 或者 DAG，其中每一个 operator 包含三个函数：open，next，close。Open 用于申请资源，比如分配内存，打开文件，close 用于释放资源，next 方法递归的调用子 operator 的 next 方法生成一个元组。图 1 描述了 select id,name,age from people where age &gt;30 的火山模型的查询计划，该查询计划包含 User，Project，Select，Scan 四个 operator，每个 operator 的 next 方法递归调用子节点的 next，一直递归调用到叶子节点 Scan operato，Scan Operator 的 next 从文件中返回一个元组。

![](https://mmbiz.qpic.cn/mmbiz_png/cokWkYcF4DekTUVbaiabavaliaaGJN4VZ56lmLFUFx2IiboGygkoRa9oic8gVf5jickU41zG2FVTOm2ZTNCpgE8ODlg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

图 3-4 火山模型 摘自文献 \[2,page 39\]

火山模型的主要缺点是昂贵的解释开销 \(interpretation overhead\) 和低下的 CPU Cache 命中率。首先，火山模型的 next 方法通常实现为一个虚函数，在编译器中，虚函数调用需要查找虚函数表, 并且虚函数调用是一个非直接跳转 \(indirect jump\), 会导致一次错误的 CPU 分支预测 \(brance misprediction\), 一次错误的分支预测需要十几个周期的开销。火山模型为了返回一个元组，需要调用多次 next 方法，导致昂贵的函数调用开销。\[\] 研究表明，在采用火山执行模型的 MySQL 中执行 TPC-H Q1 查询，仅有 10% 的时间用于真正的查询计算，其余的 90% 时间都浪费在解释开销 \(interpretation overhead\)。其次，next 方法一次只返回一个元组，元组通常采用行存储，如图 3-5 Row Format，如果顺序访问第一列 1，2，3，那么每次访问都将导致 CPU Cache 命中失败 \(假设该行不能完全放入 CPU Cache 中\)。如果采用 Column Format，那么只有在访问第一个值时才出现缓存命中失败，后续访问 2 和 3 时都将缓存命中成功, 从而极大的提高查询性能。

![](https://mmbiz.qpic.cn/mmbiz_png/cokWkYcF4DekTUVbaiabavaliaaGJN4VZ5hsBMFfoNexPfErFXibR3tia0g7PCqFYHw6dONLia4icPibLObAiaxYKmoPHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

图 3-6 行存储和列存储

**Block-oriented processing**，Block-oriented processing 模型是对火山模型的一个改进，该模型一次 next 调用返回一批元组, 元组个数在 100-1000 不等，next 内部使用一个循环来处理这批元组。在图 1 的火山模型中，Select operator next 方法可以如下实现:

```
def next():Array[Tuple]={
  // 调用子节点的 next 方法，返回一个元组向量，该向量包含 1024 个元组
  val tuples=child.next()
  val result=new ArrayBuffer[Tuple]
  for(i=0;i

<

tuples.length;i++){
    val age=tuples(i).age
    // 筛选年龄大于 30 的人
    If(age

>

30) result.append(tuples(i))
  }
  result// 返回结果
}
```

Block-oriented processing 模型的优点是一次 next 返回多个元组，减少了解释开销，同时也被证明增加了 CPU Cache 的命中率，当 CPU 访问元组中的某个列时会将该元组加载到 CPU Cache\(如果该元组大小小于 CPU Cache 缓存行的大小\), 访问后继的列将直接从 CPU Cache 中获取，从而具有较高的 CPU Cache 命中率，然而如果之访问一个列或者少数几个列时 CPU 命中率仍然不理想。该模型最大的一个缺点是不能充分利用现代编译器技术，比如在上面的循环中，很难使用 SIMD 指令处理数据。

**Column-at-a-time 模型**，向量化执行的最早历史可以追朔到 MonetDB\[\], 在 MonetDB 提出了一个叫做 Column-at-a-time 的查询执行模型，该模型中每一次 next 调用返回一个或者多个列，每个列以数组形式返回。该模型优点是具有非常高的查询效率，缺点是一个列数据需要被物化到内存甚至磁盘，导致很高的内存占用和 io 开销，同时数据不能放到 CPU Cache 中，导致较低的 CPU Cache 命中率。

**Vectored iterator model,VectorWise**提出了 Vectored iterator model 模型，该模型是对 Column-at-a-time 的改进，next 调用不是返回完整的一个列，而是返回一个可以放到 CPU Cache 的向量。该模型避免了 Column-at-a-tim CPU Cache 命中率低的缺点。Vectored iterator model 最大的优点是可以使用运行时编译器 \(JIT\) 动态的生成更适合现代处理器的指令，比如 JIT 可以生成 SIMD 指令来处理向量。考虑 TPC-H Q1 查询：SELECT l\_extprice\*\(1-l\_discount\)\*\(1+l\_tax\) FROM lineitem。该 SQL 查询的执行计划如下：

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

其中 Project operator 的 next 方法可以如下实现 \(scala 伪代码\):

```
def next():Array[Tuple]={
  val tuples=child.next()
  var result=new ArrayBuffer[Int]
  for(i=0;i

<

tuples.length;i++){
    val tuple=tuples(i)
    val r=tuples.l_extprice*(1-tuple.l_discount)*(1+tuple.l_tax) 
    result.append(r)
  }
  retult
}
```

近几年，一些 SQL On Hadoop 系统引入了向量化执行引擎，比如 Hive，Impala，Presto，Spark 等，尽管其实现细节不同，但核心思想是一致的：尽可能的在一次 next 方法调用返回多条数据，然后使用动态代码生成技术来优化循环，表达式计算从而减少解释开销，提高 CPU Cache 命中率，减少分支预测。

Impala 中的向量化执行引擎本质上属于 Block-oriented processing，imapla 的每次 next 调用返回一批元组，这种模型仍然具有较低的 CPU Cache 命中率，同时也很难使用 SIMD 等指令进行优化，为了缓解这个问题，Impala 使用动态代码生成技术，对于大循环，表达式计算等进行使用动态代码生成来进行优化。

在 Spark2.0 中，实现了基于 Parquet 的向量化执行引擎 \[12\]，该执行引擎属于 Vectored iterator model，引擎在调用 next 方法时以列存储格式返回一批元组，可以使用循环来处理该批元组。此外为了更充分的利用现代 CPU 特性，Spark 还支持整阶段代码生成技术，核心思想是将多个 operator 编译到一个方法中，从而减少解释开销。

## 3.8 动态代码生成 {#38-动态代码生成}

动态代码生成一般和向量化执行引擎结合使用，因为向量执行引擎的 next 方法内部可以使用 for 循环来处理元组向量或者列向量，使用动态代码生成技术可以在运行时对 next 方法生成更高效的执行代码。研究证明向量化执行引擎和动态代码生成可以减少解释开销 \(interpretation overhead\), 见文献 \[18\]，主要影响以下三个方面：

* **Select**, 当 select 语句中包含复杂的表达式计算时，比如 avg，sum，count，select 的计算性能主要受 CPU Cache 和 SIMD 指令影响。当数据不能放到 CPU Cache 时，CPU 大部分时间都在等待数据从内存加载到 CPU Cache，因此当 CPU 执行计算所需的数据在 CPU Cache 中时可以极大的提高计算性能。一条 SIMD 指令可以同时计算多个数据，因此使用 SIMD 指令执行表达式计算可以提高计算性能。

* **where**，与 Select 语句不同的是 Where 语句一般不需要复杂的计算，影响 where 性能更多的是分支预测。如果 CPU 分支预测错误，那么之前的 CPU 流水线将全被清洗，一次 CPU 分支预测错误可能至少浪费十几个指令周期的开销。通过使用动态代码生成技术，JIT 编译器能够自动的生成分支预测友好的指令。

* **Hash**，hash 算法影响 equal-join，group 的查询性能，hash 算法的 CPU Cache 命中率很低。\[18\] 描述了一种缓存友好的 hash 算法，可以显著的提高 hash 计算性能。

动态代码生成有两种：C++ 系和 java 系。其中 C++ 系可以直接生成本机可执行二进制代码，并且能够生成高效的 SIMD 指令，例如 Impala 使用 C++ 实现查询执行引擎，同时使用 LLVM 编译器动态的生成本机可执行二进制代码，LLVM 可以生成 SIMD 指令对表达式执行计算。Java 系利用反射机制动态的生成 java 字节码，一般而言，不能充分利用 SIMD 指令进行优化,Spark 使用反射机制动态的生成 java 字节码，通常很难直接利用 SIMD 进行表达式优化。此外在 Spark2.0 中所提供的整阶段代码生成 \(Whole-Stage Code Generation\) 技术也是动态代码生成技术将多个 Operator 编译成一个方法进行优化。

需要注意的是，动态代码生成技术并不总是万能药，在下图中，impala 的动态代码生成技术并没有提高 TPC-DS Q42，Q52，Q55 的查询速度，主要原因这些 SQL 语句的 SELECT 语句中并没有什么复杂的计算。

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 3.9 堆外存储 {#39-堆外存储}

使用 JVM 实现的查询执行引擎依赖于 GC 回收内存，每一次 Full GC 会暂停所有工作线程，一次 GC 通常在分钟级别以上，导致所有 SQL 查询计算停止，从而严重的影响查询性能并且可能会导致一些非常奇怪的异常出现，比如网络超时，shuffle 获取数据数据失败。为了减少 GC 对程序性能的影响，许多 SQL On Hadoop 使用堆外存储 \(off heap\) 来存储数据。

堆外存储所需的内存由操作系统管理而不是 Java GC，java.nio 提供了一些用于读写堆外存储的类，可以在堆外存储中存储 Int，Double 这种基元类型，也可以存储 map，struct 这种复合对象，当存储复合对象时需要将复合对象序列化存储到堆外存储，在读取时也需进行反序列化。因为序列化 / 反序列化会消耗大量的 CPU 计算，因此在使用堆外存储时需要在 GC 和 cpu 之间进行一个合理的平衡。

## 3.10 内存压缩列存储 {#310-内存压缩列存储}

在内存中缓存热点数据是提高查询性能的一个基本优化手段。在内存中缓存热点数据需要考虑至少考虑三个问题: 第一，如何减少数据的内存占用，第二，如何提高 CPU Cache 命中率，第三，如果使用 JVM 系统，还要考虑如何减少 GC 次数和 GC 时间。这里需要重点关注的是如何提高 CPU Cache 的命中率。

这三个问题可以通过使用内存压缩列存储来解决：

* **减少内存占用**，在内存列存储中，如果列元素类型是基元类型 \(Int,Double,Long 等\)，那么每一个列存储为一个数组，如果列元素是 Map，Struct 这种负责对象，可以将其序列化为一个字节数据进行存储。数组可以被压缩存储，需要注意的是，在选择压缩算法时，一般不会选择重量级压缩算法，虽然重量级压缩算法具有较高的压缩率，但是它在压缩和解压缩时非常慢，这将严重的影响查询性能。在内存压缩列存储中，轻量级压缩算法具有更高执行效率，这是因为轻量级压缩算法在进行压缩和解压时几乎不需要太多的 CPU 计算。在 Spark SQL 的内存压缩列存储中 \[10\]，使用的就是 Run length encode，dictionary encode 等轻量级压缩算法。

* **提高 CPU Cache 命中率**，内存列存储具有较好的 CPU Cache 命中率，因为列数据连续存储，所以当 CPU 访问数组中某个元素时可以将该元素临近的数据一起加载到 CPU Cache 缓存行中，这样 CPU 访问该元素的下一个元素时就不需要访问内存了，从而提高 CPU Cache 命中率，提高查询计算性能。

* **减少 GC 时间**，最后，内存列存储对于 JVM 系统也是友好的。首先，JVM 中每个对象都包含一个对象头，这个对象头的开销通常需要 12 个字节，如果我们将 Int 按行存储，那么每个 Int 都将至少浪费 12 个字节的存储空间占用。相反，如果将 Int 存储为一个数组，那么每个 Int 只需要 4 个字节，可以减少 3 倍的存储空间占用。内存列存储还可以减少 GC 时间，GC 时间主要和对象数量呈正相关，通过采用内存列存储，每个列作为一个数组对象存储，可以极大的减少对象数量，减少 GC 时间。

## 3.11 缓存敏感算法 {#311-缓存敏感算法}

自从 CPU Cache 出现以来，人们对于缓存敏感算法的研究就从未停止。所谓的缓存敏感算法，就是编写 CPU Cache 命中率高的算法。在这个领域已经有了大量的研究，比如磁盘索引 B-tree 的缓存敏感实现，内存索引 T-tree 的缓存敏感实现，链表，哈希表等等。

缓存敏感算法通常比较复杂，并且不易理解，因此将所有算法都设计成缓存敏感的是不明智的，事实上大部分 SQL 计算主要为排序，聚合，join，只需对这些算法进行优化即可。在 Spark SQL 实现了缓存敏感的 Sort 算法，该算法应用在基于 sort 的 shuffle，排序和 join，优化后的 Sort 性能至少提高了 3 倍。

# 4. 其他 {#4-其他}

目前在 SQL On Hadoop 领域中存在种类繁杂的开源软件，尽管其具体的实现细节和应用场景不同，但是仍然有一些共同的技术被广泛采用：列存储，向量化执行引擎，缓存热点数据，内存压缩列存储等。

由于设计决策，架构的不同，不同 SQL On Hadoop 仍然有许多不同的地方：

* **统一资源管理**，一个支持统一资源调度的 SQL On Hadoop 系统非常具有研究价值，因为在一个大型复杂的分布式集群中，不可能只有一种计算框架拥有数据，更多的是多种工作负载不同的计算框架同时部署在同一集群，比如 Spark，MR，Hive，SparkSql，Impala，为了避免不同计算框架之间的资源竞争，需要使用统一的资源调度框架进行资源管理，使用统一资源管理可以避免计算框架申请过多的资源导致集群，操作系统等出现不稳定状态，Yarn 和 Mesos 是两个最流行的开源资源管理框架。Impala，SparkSql 等都支持 Yarn 进行统一资源调度，presto 目前不支持 yarn。+

* **容错粒度**，Impala，Presto，drill 这些采用 MPP 架构的系统不支持细粒度的容错。Spark Sql，Hive 这些系统通过借鉴底层系统 MR 和 Spark 的容错机制，也实现了细粒度的容错。

* **JVM**， 大部分 SQL On Hadoop 都采用 JVM 语言来实现，部分系统采用非 Jvm，比如 Impala 使用 C++ 实现查询执行引擎。

最后，所有的 SQL On Hadoop 都应该尽可能的追求快速，易使用。查询速度越快，就越能适应更多的场景。支持 ANSI SQL 而不是其他方言可以减少用户学习曲线，避免用户陷入到过多的语言特性中。

