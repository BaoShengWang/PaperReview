# 1.物化策略的选择

## 1.1 late materialization优缺点

* 优点，延迟物化,尽可能晚的进行元祖重组，从而使得数据能够以列方式存储在内存中，这就使得query execution可以享受一切内存列存储所具有的优势，如果数据还使用了轻量级压缩索引，那么  优势就更多了。例如，在c-store中，因为数据是以压缩列存储方式存在于内存，所以我们可以使用直接操作压缩数据这个技术来改善查询性能，  再比如，c-store中的RLE Triple block&lt;value,position,run len&gt;就可以完全放到cpu cache line。

此外，延迟物化技术还可以减少不必要的元祖重组开销，尤其是在设计到聚合操作时。

**概况起来，延迟物化技术具有两大有点：operate on compression data，construct only relevent tuples.**



* 缺点，延迟物化的缺点也是显而易见的，例如，如果某个column需要多次，最经典的，我们应用predicate到column返回position list，然后这个position list  在figure 7 （a）中，首先应用predicate到DS1 for shipdate ,DS1 for linenum,分别返回position list for shipdate,position list for linenum，然后AND operator取这两个position  的交集position list,这个position list传递到DS3 for shipdate ,DS3 for linenum,然后DS3 for shipdate 获取position list对应的value。  如果shipdate的position不是有序的，并且也没有索引，那么我们只能再次顺序访问一遍shipdate了，使得这个重新访问的开销过于昂贵，从而影响性能。



## 1.2 Early Materialization优缺点

## 1.3 物化策略



# 2 late materialization的实现

## 2.1 base operator

我们需要实现下面几个base operator

** 1. datasource **

* CASE 1.输入predicate，返回满足predicate的postion list,
* CASE 2.输入predicate，返回满足predicate的{&lt;psotion,value&gt;}
* CASE 3.输入position list,返回position list对应的value list。
* CASE 4.输入{&lt;position,&lt;val1,val2...val m&gt;},predicate，返回{position,&lt;val1,val 2 ....,val m,val m+1&gt;}，val m+1是满足predicate的value。  CASE 4，相当于将一个column的value追加到tuple后面。

CASE 1,CAST 3，AND,MERGE 用于Late Materialization.CASE 2，CASE 4,SPC用于Early Materialization。

** 2.AND**

 **3.MERGE and SPC**

* MERGE，
* SPC，（scan,predicate,construct），就是说可以边scan多个列的同时，边construct 满足predicate的tuple。例如输入column是age，salary，predicate是age&gt;30,那么SPC可以返回满足age&gt;30的&lt;age,salary&gt;  ，SPC通常用在leaf operator上，它需要扫描输入的所有column，然后满足返回predicate的tuple。



## 2.2 逻辑执行顺序

** Late Materialization**

 1. **获取满足predicate 的position**，输入&lt;column a,predicate a&gt;，&lt;column b,predicate b&gt;，返回应用CASE 1，返回满足predicate a的position list b，同样的，返回满足predicate b的position list b.

  接下来，会应用AND取position list a 和position list b的交集，这个交集就是同时满足predicate a和predicate的position。。

 2.**获取value**，第1步中只得到了position list，还没有拿到对应的value，为了获取position对应的value，我们需要使用CASE 3来获取每一个column position list对应的value

 3.**拼接tuple**，第2步中只得到了独立的column value，为了将他们拼接成tuple，需要使用MERGE生成tuple。



逻辑执行顺序大致为：CASE 1-&gt;AND-&gt;CASE 3-&gt;MERGE



**Early Materialization**

Early Materialization 相对简单些，其思想要么是在第一步就将所有column拼接成tuple，或者是每次用到一个column时，追加这个column的value到之前的tuple。

1.**获取满足predicate的&lt;position ,value&gt;**，例如，输入&lt;column a ,predicate a&gt;,应用CASE 2，返回满足predicate a的&lt;position,value&gt;.

2.**动态追加column**,输入&lt;column b ,predicate b，&lt;position a ,value a&gt;&gt;,CASE 4可以返回满足predicate b的&lt;position,value a,value b&gt;

 

或者，直接使用SPC生成满足predicate a和predicate b的&lt;value a,value b&gt;。

