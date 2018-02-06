很多研究证明，列存储查询性能要比行存储好，那么一个很自然的问题就是，在传统的行数据库中，我们采用vertical partitioning，index-only等方式来模拟一个列存储，性能是否要比行存储好。Column-Stores vs. Row-Stores: How Different Are They Really? 曾对这个问题做过深刻的讨论。



# 1.emulate column storage in row storage

作者在System X上分别采用vertical partitioning，index-only，materialized view三种方式来模拟列存储，最终得出的结论如下：在vertical partitioning中，在SSBM基准测试中，当查询超过4个列时，vertical partitioning性能就比行存储差很多了，原因是tuple overhead和join cost太高了。在index-only中，性能更加糟糕。



System X在traditional和materialized view下支持partitioning，在SSBM的性能测试中，我们在orderdate建立了分区\(partition\)，因此如果在orderdate上有谓词，那么将显著的提高性能，但是System X在vertical partitioning和index-only等方式中，不支持 partitioning，因此，在从图6中可以发现：traditional 平均性能要优于vertical partitionging和index-only。进一步的，仔细观察图6，除了fligh1，3，4中，traditional性能要都优于vertical partitioning，因为我们利用了System X的partitioning。而在flight2中，因为Q2.1，Q2.2,Q2.3都没有用到orderdate，所以没有用到orderdate分区，traditional 和vertical partitioning性能相差不多。

![](/assets/行存储中模拟列存储性能.png)



作者也说了，如果不使用System X的partitioning功能，那么同样的查询，性能将降低一半。因此我们可以假设在vertical partitioning中，如果采用partitioning，那么性能也能提高一倍。从而我们可以得出如下结论：

* 1.即使在行存储中，如果我们采用列存储方式来存储数据，查询性能也要优于行存储，当然了，优于tuple overhead和join cost等因素，查询的列不能太多，否则性能急剧下降。
* 2.要想发挥列存储的性能，不能采用“在行存储中模拟列存储”这种方式，我们必须将存储层完全替换为列存储，也就是一个column一个文件。事实上，  为了发挥列存储的最佳性能，只是将存储层替换为列存储是远远不够的，我们还要使用面向列的查询执行引擎\(query execution engin\)。

## Vertical Partitioning

每一个列存储为一个物理表，为了重组元祖，需要添加一个类似tuple identifer的东西，用于唯一表示一个元祖。tuple identifier 可以是primary key，也可以是position。一般来说使用position要更好一些，因为position占用的存储空间要比primary key更小。为了提高性能，每一个物理表都在position上建立一个聚集索引，这样可以可以使用index-join来重组元祖。



vertical partitioning最致命的问题：tuple overheads和join cost太高。

在 行存储中，tuple存储方式逻辑上如下：tuple header,position,value，每一个tuple都有一个tuple header，这个tuple header占用的空间可能要比\[position,value\]的存储空间还要大。

> 在row storage中， linerorder表有17 column，6\* 10^7个 tuple, 6GB\(decompression\),4GB\(compression\)，在vertical partitioning中one column,8Byte\(tuple header\) + 4Byte\(position\)+4\(value\) \* 10^7=960MB  。可以发现，在vertical partitioning中，当访问4个column时，IO开销就和row storage一样了，其优势已经荡然无存。

## Index-only



vertical partitioning方法的缺失是太浪费存储空间了，每一个列中都要存储position，此外，每一个值还有tuple header的开销。

在Index-only方法中，base ralation 还按照row方式存储，但是呢，我们在每一个column上建立一个unclustered B-tree索引，用来将column value映射到tid，这里注意的时，在unclustered b-tree中，我们不在存储primary key或者position了。



当我们执行查询时，如果有多个predicate，例如a&gt;13 and b&lt;10，那么可以在利用column a上执行一个index scan，返回所有a&gt;13的&lt;value,tid&gt;， 同样的，在column b上执行一个index scan，返回b&lt;10的&lt;value,tid&gt;。然后对这两组&lt;tid,vlaue&gt;按照tid执行一个join即可。当有多个predicate时，可以predicate对应的unclusterd b-tree上执行查询，然后在内存中对返回的的&lt;tid,column value&gt;做一个join即可。  如果没有谓词，那么就糟糕了，例如select age ,salary from tmp。此时需要在分别在age，salary上执行一个full index scan，然后在对他们执行join。



作为一个优化方式使用composite index。



这里最耗时的就是对两组&lt;tid,value&gt;执行join操作。System X中，这个join为hash join，非常慢。

## Materialized Views



