# 如何设计一个列存储数据库

作者：王宝生

邮箱：franciswbs@163.com

github:https://github.com/BaoShengWang

![](/assets/微信.jpg)



本文内容主要摘自 Column-Stores vs. Row-Stores: How Different Are They Really?

本论文主要结论概况如下：

* 首先，尽管我们可以在行存储中通过vertical partitioning ，index only技术模拟列存储，但是这种模拟不会带来很好的性能提升，原因是tuple overhead开销太的了，在SSBM中，当访问超过4个列时，vertical partitioning的IO开销就和行存储一样了，此外join cost也非常高。
* 其次，观察Figure7\(a\)可以看到，要想获得查询性能量的提升，我们必须同时使用column storage和column query execution。compression和late materilization对性能提高最大，late materialization可以平均提高查询性能3倍左右。compression平均提高查询性能2倍，如果column是sort的，那么性能可以提升一个量级，比如SSBM基准测试中的flight1查询.
* 此外， 单纯的使用block-processin而没有开启compression，对查询性能的提升并不明显，
  block-processing大概提升查询性能5-50%左右,具体提升多少，取决于我们是否使用了compression，
   如果没有使用compression，那么block所节省的cpu好处并不明显。
  invisible join可以提高性能50-70%，因为Cstore使用late-materialized join技术

![](/assets/row vs column-performance in c-store.png)

如何设计一个列存储数据库系统呢，毫无疑问，最快速，最简单，成本最小的方式就是充分利用现有的行数据库系统，例如，我们可以在postgresql中模拟列存储，最直观的模拟方式就是，将表垂直切分为多个column，每一个column存储为一个物理表，这样当查询column a和column b时，可以只去column a和columnb对应的物理表中查询数据，而不用查询所有列。这种方式最大的有点就是可以充分利用postgresql中所有的组件：storage manager，query execution，query optimizer等等。我们几乎不用编写什么新的代码就能利用列存储的优势：只从磁盘查询所需的列。**事实上，早期就有人这么做过，并且查询性能还行：只有查询非常少数的几个列时，vertical partitioning才比行存储性能好。**

vertical partitioning可以认为是伪列存储。那么真正意义的列存储是什么样的呢？从目前的观点来看，现代真正意义上的列存储必须包含两个部分：

* column storage，列存储，每一个column存储为一个独立的文件，column物理存储格式可以看作：

  ```
      &lt;column header,value array&gt;

   column header描述了这个column的元数据信息，比如压缩编码，最大值，最小值等等。value array是一个连续存储的value数组。
  ```

* column query execution，查询执行引擎是面向列的，典型的查询执行引擎会用到下面几个技术：direct operate compression date，vectoried execution，late materilized。

在这篇paper中，我们会看到，同时使用column storage和column query execution是最佳方式，所得到的查询性能提升是高的，当然了，只使用column storage也能提高查询性能。

# 1.emulate column storage in row storage

作者在System X上分别采用vertical partitioning，index-only，materialized view三种方式来模拟列存储，最终得出的结论如下：**在vertical partitioning中，在SSBM基准测试中，当查询超过4个列时，vertical partitioning性能就比行存储差很多了，原因是  
tuple overhead和join cost太高了。在index-only中，性能更加糟糕。**

在查看paper中的性能图片时，尤其要注意的是，System X在traditional和materialized view下支持partitioning，而在vertical partitioning方式下不支持partitioning。除了flight2查询，其他查询基本都利用到了System X的partitioning功能，所以在看性能比较时，发现大多数情况下，traditional 方式要比vertical partitioning快很多。

但是呢，paper中也有提高，如果不利用System X的partitioning功能，那么traditional和materialized方式性能要下降一半，所以按照这个假设，如果在vertical partitioning方式中也支持了partitioning功能，那么其性能也会提高一倍。这就提醒我们在看性能图片时可以将flight1，flight3，flight4中的vertical partitioning性能提高一倍在做比较。

在SSBM的性能测试中，我们在orderdate建立了分区\(partition\)，因此如果在orderdate上有谓词，那么将显著的提高性能，但System X在vertical partitioning和index-only等方式中，不支持partitioning，因此，在从图6中可以发现：traditional 平均性能要优于vertical partitionging和index-only。进一步的，仔细观察图6，除了fligh1，3，4中，traditional性能要都优于vertical partitioning，因为我们利用了System X的partitioning。**而在flight2中，因为Q2.1，Q2.2,Q2.3都没有用到orderdate，所以没有用到orderdate分区,traditional 和vertical partitioning性能相差不多。**

作者也说了，如果不使用System X的partitioning功能，那么同样的查询，性能将降低一半。因此我们可以假设在vertical partitioning中，如果采用partitioning，那么性能也能提高一倍。从而我们可以得出如下结论：

* 1.即使在行存储中，如果我们采用列存储方式来存储数据，查询性能也要优于行存储，当然了，优于tuple overhead和join cost等因素，查询的列不能太多，否则性能急剧下降。
* 2.要想发挥列存储的性能，不能采用“在行存储中模拟列存储”这种方式，我们必须将存储层完全替换为列存储，也就是一个column一个文件。事实上，为了发挥列存储的最佳性能，只是将存储层替换为列存储是远远不够的，我们还要使用面向列的查询执行引擎\(query execution engin\)。

## Vertical Partitioning

每一个列存储为一个物理表，为了重组元祖，需要添加一个类似tuple identifer的东西，用于唯一表示一个元祖。tuple identifier 可以是primary key，也可以是position。一般来说使用position要更好一些，因为position占用的存储空间要比primary key更小。为了提高性能，每一个物理表都在position上建立一个聚集索引，这样可以可以使用index-join来重组元祖。

**vertical partitioning最致命的问题：tuple overheads和join cost太高。**

在 行存储中，tuple存储方式逻辑上如下：tuple header,position,value，每一个tuple都有一个tuple header，这个tuple header占用空间可能要比\[position,value\]的存储空间还要大。

> 在row storage中， linerorder表有17 column，6\* 10^7个 tuple, 6GB\(decompression\),4GB\(compression\)，在  
> vertical partitioning中one column,8Byte\(tuple header\) + 4Byte\(position\)+4\(value\) \* 10^7=960MB。可以发现，在vertical partitioning中，当访问4个column时，IO开销就和row storage一样了，其优势已经荡然无存。

## Index-only

vertical partitioning方法的缺失是太浪费存储空间了，每一个列中都要存储position，此外，每一个值还有tuple header的开销。在Index-only方法中，base ralation 还按照row方式存储，但是呢，我们在每一个column上建立一个unclustered B-tree索引，用来将column value映射到tid，这里注意的时，在unclustered b-tree中，我们不在存储primary key或者position了。

当我们执行查询时，如果有多个predicate，例如a&gt;13 and b&lt;10，那么可以在利用column a上执行一个index scan，返回所有a&gt;13的&lt;value,tid&gt;， 同样的，在column b上执行一个index scan，返回b&lt;10的&lt;value,tid&gt;。然后对这两组&lt;tid,vlaue&gt;按照tid执行一个join即可。

当有多个predicate时，可以predicate对应的unclusterd b-tree上执行查询，然后在内存中对返回的的&lt;tid,column value&gt;做一个join即可。如果没有谓词，那么就糟糕了，例如select age ,salary from tmp。此时需要在分别在age，salary上执行一个full index scan，然后在对他们执行join。

作为一个优化方式使用composite index。

这里最耗时的就是对两组&lt;tid,value&gt;执行join操作。System X中，这个join为hash join，非常慢。

## Materialized Views

采用物化视图方式的查询性能最好，因为这种方式查询的数据量最小。但是优于物化视图通常只能用于查询模式已知的情况，所以在模拟列存储时，物化视图方式实践意义不大。

