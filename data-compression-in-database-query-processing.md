# Data Compression in Database Query Processing

![](/assets/列存储压缩算法介绍.png)

# 1 Leight-Weight Compression

所谓的Leight-Weight Compression，其实并不是真正意义上的压缩技术，而是采用encoding来对数据进行编码。其思想是对于重复值尽可能的只存储一份，对于较大的值尽可能的用较小的值来代替\(例如delta encoding\)。encoding的好处不仅仅可以减少存储空间，还可以提高解压缩效率，encoding是cpu友好的，从encoding中读取数据的开销非常低，通常进行一次位操作就ok了。例如传统关系数据库中的压缩算法是基于page的，要想读取page数据，必须解压整个page，而对于encoding来说，encoding通常将几个值进行编码而不是整个page。



我们把传统的行存储中基于page的压缩算法叫做Heavy-Weight Compression。而采用encoding这种压缩方式的叫做Leight-Weight Compression。Leight-Weight Compression压缩算法有RLE，delta，bitmap。



## 1.1 dictionary encoding

### 1.1.1 Word-aligned dictionary encoding

### 1.1.2 Bit-aligned dictionary encoding

## 1.2 bitmap

bitmap index由一组bit-vector组成，每一个bit-vector对应一个distinct value,bi't-vector存储了其对应的distinct value在列中的位置。例如，在图7中，我们在ProductId上建立了一个bitmap index.ProductId有4个不同的值，分别为1，2，3，4.每一个不同的值都对应一个bit-vector，这个bit-vector存储了值所在ProductId的位置。

![](/assets/import.png)

bitmap index有两种：uncompression bitmap和compression bitmap，uncompression bitmap。**当列的基数比较小或者bitmap 个数比较少时，查询非常快，但是当二者变得很大时，bitmap的开销将非常高**。为了解决这个问题，需要使用compression bitmap index。compression bitmap index使用RLE压缩位图，有两种BBC和WAH,WAH性能最好。



compression bitmap非常有用，他不仅可以节省存储空间，还能提高查询性能。在很多应用场景中，compression bitmap要优于B-tree，但是对于join操作，还有很大的优化空间。



* bitmap能节省存储空间

* bitmap能提高查询性能

> SELECT OrderID
>
> FROM Order WHERE ProductID ≤ 2.



为了执行上面这个查询执行，只需执行**位或**操作b1\| b2就ok了。



> SELECT OrderID
>
> FROM Order
>
> WHERE ProductID≤ 2 and CustomerID ≥ 100,



假设在在ProductIdhe和CustomerID上都有bitmap,那么可以分别在这两列上的bitmap上获取中间结果，然后在对中间结果进行**位与**操作。



## 1.3 run-length encoding

run-length encoding 简写为RLE。**其思想是将连续重复的值压缩为一个值存储，当数据数有序的，并且重复值比较多时，RLE的压缩效果非常理想**。RLE在节省存储空间的同时，还能提高查询性能\(range query，count query时不用解压数据\)。

 

RLE有两个实现方式:第一种是将连续重复值压缩为&lt;value,count&gt;,value是重复值，count是重复值出现的次数，当查询设计到多个列时，该实现性能比较低。另外一种实现方式是将重复值压缩为&lt;value,count,position&gt;，其中position代表value第一次出现的位置，C-store中使用的就是该实现。改实现可以友好的处理多列查询。



![](/assets/RUE-encoding.png)

### 1.3.1 Standard run-length encoding

 standard run-length encoding将连续重复值压缩为&lt;value,count&gt;。如图6左所示。

Order.CustomerId有$$10^6$$个value，每个value是4Byte，总大小为$$10^6 * 4$$，4GB。压缩后的数据是$$10^6 * 8$$，等于8MB，压缩比非常高。RLE的优点不仅仅是节省存储空间那么简单，RLE还能提高查询性能。例如给定如下查询语句：

> SELECT Count\(\*\) FROM Order WHERE CustomerID= 2

 在执行上面这个查询时，可以不用解压数据，直接查询RLE编码数据就能得到结果。而压缩后数据只有8MB，因此将大大的提高擦好像性能。

### 1.3.2 Variant run-length encoding

标准RLE编码不支持随机访问，例如查询**select B from R where A=t**，假设A采用RLE编码，为了响应这个查询，可以在column A的RLE编码上直接查询A=t的值，但是我们只能得到t对应的count，而不能得到t在列A中的start position,因此也就不能知道对应的B列中值的position。为了解决这个问题，我们需要在RLE中加入value的position。有如下两种方式：&lt;value,start position,run length&gt;,&lt;value,start position&gt;,在第二种方式中，为了获得value的run length，可以使用下一个value的start position减去当前value的start position。

目前C-Store采用&lt;value,start position,run length&gt;这种方式编码数据。



