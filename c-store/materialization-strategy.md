author：王宝生

e-mail：franciswbs@163.com

github:[https://github.com/BaoShengWang](https://github.com/BaoShengWang)

![](/assets/微信.jpg)

延迟物化是c-store查询执行引擎中极其核心的技术，平均可以提高查询性能3倍。在下面的几个paper中有详细的介绍：

> 2007 Materialization Strategies in a Column-Oriented DBMS. MIT CSAIL Technical Report. MIT-CSAIL-TR-2006-078
>
> 2008 Materialization Strategies in a Column-Oriented DBMS, Abadi, Myers, DeWitt,and Madden. ICDE
>
> 2013 The Design and Implementation of Modern Column-Oriented Database Systems

现在对他们总一个整理，方便以后学习。

paper中还对pipeline中的position reaccess的优化做了介绍，使用的思想是PAX，这里不做介绍。

paper中介绍了4中物化策略：EM-pipelined,EM-paralle,LM-pipelined,LM-paralle.

EM，即early materialization，尽可能早的提前进行tuple reconstruction，早期的列数据库使用的就是这种方式。

LM，late materialization，尽可能晚的tuple reconstruction，使得数据能够尽可能的以列方式存在内存中，从而可以应用operate directly on compression data技术，减少cpu cost。

_**从paper中可以看到\(figure 10 \(b\)\)，EM和LM之间的主要矛盾是operate directly on compression，也就是说，EM和LM之间的性能差异主要取决于是否是compression，如果column是compression的，那么LM要优于EM，因为此时LM可以使用operate directly on compression技术，避decompression cost，尤其是数据量非常大时，效果更明显。**_

所谓的pipelined，就是一个Datasource的输出Datasource之间顺序执行，例如，CASE 1的输出是CASE 3的输入\(CAST 1-&gt;position list-&gt;CASE 3-&gt;&lt;value&gt;\),CASE 2的输出是CASE 4的输入\(CASE 2-&gt;&lt;position,value&gt;-&gt;CASE 4-&gt;&lt;value1,value2...&gt;\),可以发现不管哪一种方式，后面的Datasource都存储reaccess开销，也就说后面的Datasource都要根据position来获取value，如果predicate生成的数据非常非常多，那么这个reaccess开销非常大。

而paralle方式，也就是说多个datasoruce可以同时执行。例如在EM-paralle中，使用SPC operator在执行之前就将多个column拼接出tuple，在LM-paralle中，使用AND operator对多个column position进行and操作，最后使用merge生成tuple。

_**pipeline和paralle之间的主要矛盾是position reaccess，pipeline存在position reaccess cost，当selectivity非常大时，此开销将非常高**_。

---

## 1 late materialization的实现

我们以 SELECT SHIPDATE, LINENUM FROM LINEITEM WHERE SHIPDATE为例。

我们先来看看EM实现。如图figure 6\(a\)，描述了EM-pipelined的执行流程。首先使用DS2\(CASE 2\)扫描shipdate列，生成满足shipdate &lt; X 的&lt;pos,val1&gt;流，然后传递给DS4,DS4遍历linenum列中pos对应的value，然后生成满足linenum &lt; Y  的&lt;shipdate,linenum&gt;。figure 6\(b\)描述了EM-paralle执行流程，首先SPC操作扫描shipdate和linenum列生成&lt;shipdate,linenum&gt;，然后生成满足shipdate&lt; X，linenum &lt;Y的元祖。

概括起来，EM执行流程如下：

**Early Materialization**

Early Materialization 相对简单些，其思想要么是在第一步就将所有column拼接成tuple，或者是每次用到一个column时，追加这个column的value到之前的tuple。

1.**获取满足predicate的&lt;position ,value&gt;**，例如，输入&lt;column a ,predicate a&gt;,应用CASE 2，返回满足predicate a的&lt;position,value&gt;.

2.**动态追加column**,输入&lt;column b ,predicate b，&lt;position a ,value a&gt;&gt;,CASE 4可以返回满足predicate b的&lt;position,value a,value b&gt;

或者，直接使用SPC生成满足predicate a和predicate b的&lt;value a,value b&gt;。

![](/assets/物化策略-EM.png)

figure 7描述了LM执行流程。概括起来LM执行流程如下：

**Late Materialization**

1. **获取满足predicate 的position**，输入&lt;column a,predicate a&gt;，&lt;column b,predicate b&gt;，返回应用CASE 1，返回满足predicate a的position list b，同样的，返回满足predicate b的position list b.

   接下来，会应用AND取position list a 和position list b的交集，这个交集就是同时满足predicate a和predicate的position。。

2. **获取value**，第1步中只得到了position list，还没有拿到对应的value，为了获取position对应的value，我们需要使用CASE 3来获取每一个column position list对应的value

3. **拼接tuple**，第2步中只得到了独立的column value，为了将他们拼接成tuple，需要使用MERGE生成tuple。

late materialization 逻辑执行顺序大致为：CASE 1-&gt;AND-&gt;CASE 3-&gt;MERGE

![](/assets/物化策略-LM.png)

**pipeline vs paralle **

> _**pipeline的问题是存在position reaccess cost，因此当selectivity比较小时，可以采用pipeline方式。**_
>
> 所谓的pipelined，就是一个Datasource的输出Datasource之间顺序执行，例如，CASE 1的输出是CASE 3的输入\(CAST 1-&gt;position list-&gt;CASE 3-&gt;&lt;value&gt;\),CASE 2的输出是CASE 4的输入\(CASE 2-&gt;&lt;position,value&gt;-&gt;CASE 4-&gt;&lt;value1,value2...&gt;\),可以发现不管哪一种方式，后面的Datasource都存储reaccess开销，也就说后面的Datasource都要根据position来获取value，如果predicate生成的数据非常非常多，那么这个reaccess开销非常大。
>
> 而paralle方式，也就是说多个datasoruce可以同时执行。例如在EM-paralle中，使用SPC operator在执行之前就将多个column拼接出tuple，在LM-paralle中，使用AND operator对多个column position进行and操作，最后使用merge生成tuple。EM-paralle和LM-paralle的区别是LM-paralle中，谓词可以下推到datasource，例如CASE 1.而在EM-paralle，此外因为数据是以column方式存在于内存中的，因此可以使用operate directly on compression data等技术，谓词不能下推到Datasource。

下面介绍几个base operator：datasource，and，merge,spc。

** 1. datasource**

* CASE 1.输入predicate，返回满足predicate的postion list,
* CASE 2.输入predicate，返回满足predicate的{&lt;psotion,value&gt;}
* CASE 3.输入position list,返回position list对应的value list。
* CASE 4.输入{&lt;position,&lt;val1,val2...val m&gt;},predicate，返回{position,&lt;val1,val 2 ....,val m,val m+1&gt;}，val m+1是满足predicate的value。CASE 4，相当于将一个column的value追加到tuple后面。

CASE 1,CAST 3，AND,MERGE 用于Late Materialization.CASE 2，CASE 4,SPC用于Early Materialization。

** 2.AND**

ANDoperator对多个position list取交集,用在LM中。**                    
**

**3.MERGE and SPC**

* MERGE，将多个column的值合并成tuple，用于LM。
* SPC，（scan,predicate,construct），就是说可以边scan多个列的同时，边construct 满足predicate的tuple。例如输入column是age，salary，predicate是age&gt;30,那么SPC可以返回满足age&gt;30的&lt;age,salary&gt;，SPC通常用在leaf operator上，它需要扫描输入的所有column，然后满足返回predicate的tuple。

# 2 Experiments

> query 1
>
> SELECT SHIPDATE, LINENUM
>
> FROM LINEITEM
>
> WHERE SHIPDATE &lt; X AND LINENUM &lt; Y

其中sort key是\(returnflag,shipdate,linenum\),其中returnflag，shipdate采用RLE编码，linenum没有采用任何压缩。

![](/assets/物化策略-性能对比.png)

figure 10\(a\),linenum是uncompression的，此时主要的矛盾是pipeline和paralle之间的，换句话说，uncompression主要影响  
pipeline和paralle之间性能，而不是EM和LM之间的。因为pipeline存在position reaccess开销，所以当selectivity比较高，这个  
position reaccess cost将占据主要地位。

figure 10\(b\)，linenum采用RLE编码\(query 1所涉及的column都是sort and RLE\)，此时主要的矛盾是EM和LM之间，  
仔细观察发现,LM要远远优于EM，selectivity越大，差距越明显，原因是EM在一开始就需要decompression RLE data然后tuple reconstruction，  
这个decompression cost可能很高\(相比operate directly on compression data\)，尤其是数据量非常大的情况下。

# 3.物化策略的选择

## late materialization优缺点

* 优点，延迟物化,尽可能晚的进行元祖重组，从而使得数据能够以列方式存储在内存中，这就使得query execution可以享受一切内存列存储所具有的优势，如果数据还使用了轻量级压缩索引，那么优势就更多了。例如，在c-store中，因为数据是以压缩列存储方式存在于内存，所以我们可以使用直接操作压缩数据这个技术来改善查询性能，再比如，c-store中的RLE Triple block&lt;value,position,run len&gt;就可以完全放到cpu cache line。

此外，延迟物化技术还可以减少不必要的元祖重组开销，尤其是在设计到聚合操作时。

**概况起来，延迟物化技术具有两大有点：operate on compression data，construct only relevent tuples.**

* 缺点,在figure 7 （a）中，首先应用predicate到DS1 for shipdate ,DS1 for linenum,分别返回position list for shipdate,position list for linenum，然后AND operator取这两个position的交集position list,这个position list传递到DS3 for shipdate ,DS3 for linenum,然后DS3 for shipdate 获取position list对应的value。如果shipdate的position不是有序的，并且也没有索引，那么我们只能再次顺序访问一遍shipdate了，使得这个重新访问的开销过于昂贵，从而影响性能。

## 物化策略

可以使用如下启发式规则来决定使用哪一种物化策略：

1. 如果input column是light-weight compression时，那么使用LM物化策略。
2. 如果selectivity非常少时，那么使用pipelined。
3. 如果输出是aggreagate，那么使用LM



