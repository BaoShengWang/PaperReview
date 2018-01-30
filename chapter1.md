# Performance Tradeoffs in Read-Optimized Databases {#performance-tradeoffs-in-read-optimized-databases}

![](https://baoshengwang.gitbooks.io/columnar-database/content/assets/%E5%88%97%E5%AD%98%E5%82%A8%E8%A1%8C%E5%AD%98%E5%82%A8%E6%80%A7%E8%83%BD%E5%B9%B3%E8%A1%A1.png)

Performance Tradeoffs in Read-Optimized Databases\(2006年\)是列存储领域非常经典的一篇论文。

在此之前，很多paper已经发现，列存储查询性能要优于行存储。然后这些paper中并没有详细的描述在什么样的条件下，列存储查询性能要由于行存储。这篇讨论的就是这个内容：在什么时候列存储要比行存储查询性能好，什么时候不好，哪些因此影响查询性能。

> 在大多数情况下,预取大小比较大时，列存储比行存储具有更好的磁盘带宽利用率，但是在某些特定场景，列存储的cpu性能要比行存储更糟糕，例如在选择率较低，选择列非常多等情况下。

# 1 测试配置 {#1-测试配置}

## 1.1 测试基准表 {#11-测试基准表}

![](https://baoshengwang.gitbooks.io/columnar-database/content/assets/%E8%A1%8C%E5%AD%98%E5%82%A8%E5%88%97%E5%AD%98%E5%82%A8%E6%B5%8B%E8%AF%95%E5%9F%BA%E5%87%86%E8%A1%A8.png)

lineitem表含有16个属性，每一行元祖大小是150字节，整个表大小为9.5G。可以看出，数据量是蛮大的。

Order表和LINEITEM具有相同的基数，即元祖个数相同。

## 1.2 机器配置 {#12-机器配置}

Pentium 4 3.2GH

16 KB L1 Cache

1MB L2 Cache

1GB RAM

Linux 2.6

RAID with 3 SATA disk,180MB/sec bandwidth,60MB/sec per disk。

128 KB IO unit,预取48个IO UNIT,也就是6MB.

## 1.3 扫描器 {#13-扫描器}

![](https://baoshengwang.gitbooks.io/columnar-database/content/assets/%E8%A1%8C%E5%88%97%E5%AD%98%E5%82%A8-%E6%89%AB%E6%8F%8F%E5%99%A8.png)

这里要注意的是，在列存储扫描器中，列存储扫描器采用流水线方式顺序执行node scan。每一个node scan生成一组满足用户查询条件的元祖&lt;position,value&gt;,图中的poslist1+col1，然后传递到下一个node scan，来获取position 对应的值，并应用谓词条件，图中的poslist+col1+col2。

可以发现，在列存储扫描器中需要打开多个文件，从一个文件切换到下一个文件时，需要一次disk seek ，为了抵消disk seek，通常使用较大的prefetch buffer，例如每次打开一个列文件，获取1MB的数据。

所有operator都使用block-iterator模式，每一个operator调用next方法将返回一组元祖。使用这种方式可以平摊一些函数调用开销。block size是一个可配置的系统参数，调整这个参数使得next返回的block尽可能的放到L1 Cache中。

## 1.4 性能影响因子 {#14-性能影响因子}

* 查询负载：增加projection attribute，减少selectivety对查询性能有何影响。
* 物理存储：元祖大小，压缩对查询性能的影响。
* 系统参数：预取大小对查询性能的影响。目前来看，增加预取大小，可以提高列存储和行存储的性能，原因是较大的预取大小，可以平摊磁盘寻到开销。尤其对于列存储而言。
* 容量规划：给定一个查询，使用几个cpu和几个磁盘，这个对列存储的性能影响比较明显。
  **cpu /disk这个比率影对列存储的响查询性能影响比较大**
  。这个比率叫做cpdb，其中分子表示cpu每秒能处理的元祖个数，分母表示磁盘每秒能返回的元祖个数。当cpu每秒能处理的元祖个数远远大于磁盘每秒能处理的元祖个数时，我们说这个计算时disk-bound，因此此时的性能瓶颈是磁盘。当cpu每秒能处理的的元祖个数远远小于磁盘每秒能处理的元祖个时，这个计算时CPU-bound的，因为此时cpu是性能瓶颈。
* 系统负载：system load，系统负载（也就是磁盘，cpu利用率）影响单个查询的相应时间，例如，两个同样disk-bound的查询，本来磁盘就是瓶颈，而两个查询由都是disk-bound的，所以导致两个查询都很慢。
  **负载对列存储的影响比较大，也就是说当出现磁盘或者cpu竞争时，对列存储的性能影响要比行存储大的多**

# 2. 测试用例 {#2-测试用例}

## 2.1 用例1 {#21-用例1}

> **select**L1, L2 …
>
> **from**LINEITEM
>
> **where**predicate \(L1\) yields 10% selectivity

性能如图6：

![](https://baoshengwang.gitbooks.io/columnar-database/content/assets/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20180128150214.png)

### IO开销 {#io开销}

在大部分时间内，列存储的查询性能都要优于行存储，当选择的列大小超过元祖的85%之后，列存储性能将不如行存储，因为此时列存储需要打开更多的磁盘文件，每一次切换都带来一次磁盘寻到开销，而行存储则享受完全的顺序扫描。我们这里使用48个IO Units预取大小，因此能够平摊部分磁盘寻到开销，但是当列数增加到一定程度后，磁盘寻到开销将超过预取所带来的好处。

### CPU 开销 {#cpu-开销}

图6右侧描述了行存储和列存储的cpu开销，条形图的高度表示所花费的cpu时间，单位是秒。其中蓝色部分表示system time，linux处理 IO 请求时间。usr-uop,表示纯用户计算时间，例如select中的谓词计算。usr-L2,usr-L1表示数据从main memory到L2 cache和L2 cache 到 L1 cache的时间。

* 随着查询列数的增加，system time也增加，甚至最后超过了行存储的system。但是大多数情况下，列存储的cpu time要小于行存储 。
* 列存储的缓存延时低于行存储的缓存延时，当查询列数增加到一定程度后，尤其是到11列之后，缓存延时突增。

因为随着列数的增加，所需额外的处理也增加，如果selectivety较少时结果会怎么样呢，此时额外的处理是否还处于一个主要的占比。这引出下面一个测试用例:将selectivity降低到0.1。

## 2.2 用例2 {#22-用例2}

> **select**L1, L2 …
>
> **from**LINEITEM
>
> **where**predicate \(L1\) yields 0.1% selectivity

cpu性能测试如图7所示：

![](https://baoshengwang.gitbooks.io/columnar-database/content/assets/%E9%80%89%E6%8B%A9%E7%8E%87%E4%BD%8E%E7%9A%84%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95%E8%A1%A8.png)

降低selectivety对IO时间没有影响。影响最大的是cpu时间。

依然是随着选择列数的增加，system time增加。但是值得注意的是，usr-L2和usr-L1基本消失了，usr-uop也变得非常小。原因是每一node scan生成一个1000个元祖到到内存，此时内存延时远远低于在L1执行计算的开销。

还要注意的是，仔细查询图7，当选择率非常低，列存储的usr-uop非常接近行存储的usr-uop，而当选择率变得很大时，列存储的usr-uop将大于行存储的。原因时这里采用的时流水线扫描器，如果采用非流水线扫描器结果可能更好一些。所谓的非流水线扫描器工作如下：将每个node scan的数据一起加载到内存中，然后迭代器从多个内存segemtn中扫描拼接成元祖，后面以元祖为单位来处理。

## 用例3，窄元祖 {#用例3，窄元祖}

> select O1, O2 …
>
> from ORDERS
>
> where predicate \(O1\) yields 10% selectivity

性能如图8所示。

IO时间趋势和用例1一致。不同的是cpu时间。首先内存延时开销在行存储和列存储中都消失不见了。其次，列存储中的usr-uop比行存储要大，此外，usr-uop占比非常高，原因是内存传输数据的速度超过了cpu处理数据的速度。

## 用例4 {#用例4}

这个不详细说了，增加预取大小，可以同时列存储和行存储查询性能。

# 总结 {#总结}

![](https://baoshengwang.gitbooks.io/columnar-database/content/assets/%E8%A1%8C%E5%AD%98%E5%82%A8%E5%92%8C%E5%88%97%E5%AD%98%E5%82%A8%E4%B9%8B%E5%8A%A0%E9%80%9F%E6%AF%94.png)

作者使用speed up来衡量列存储和行存储的相对性能加速比，如图2所示。其中x轴表示的是元祖的宽度。y中表示的是cpdb.

$$speedup=列存储时间/行存储时间$$

$$cpdb=clock/DiskBW$$

$$cpdb=clock/DiskBW$$

其中clock表示cpu 的每秒周期数\( where clock is the available cycles per second from the CPUs \(e.g., for our single-CPU machine, clock is 3.2 billion cycles per second\)+

DiskBW表示是磁盘带宽，磁盘每秒传输的字节数。

在本paper中，$$clock=3.2GHZ=3.2_1000MHZ=3.2_1000_1000KHZ=3.2_1000_1000_1000HZ，DiskBW=180MB/S=188743680B$$

$$cpdb=3.2_1000_1000\*1000/188743680=17$$



