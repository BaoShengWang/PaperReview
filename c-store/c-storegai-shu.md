# 列存储中的物理计划



> SELECT count\(orderID\), suppKey
>
>  FROM lineitem
>
>  WHERE shipdate &gt; 1/1/05 GROUP BY suppkey





* row ,添加一个glue operator，屏蔽掉底层每一个column对应的access method。glue operator将所需要的column拼接成一个tuple  ，然后返回给上次operator。我们可以发现，selection operator在glue operator之上:selection operator是glue operator的parent node。  glue operator必须顺序访问每一个column，然后拼接成tuple返回给selection operator，selection operator在对tuple执行过滤操作，  predicate不能下推到glue节点，因为你不知道column之间的对应关系。



![](/assets/row-plan.png)



为了能够将predicate 下推到access method，引入第二个方式：

* column,为了将predicate下推到access method，我们要求每一个access method需要返回&lt;value,position&gt;或者position。  position可以在不同access method之间传递。  例如对于a&gt;2&&b&lt;3这个predicate。我们可以将a&gt;2下推到column a的access method，这个access method返回满足a&gt;3的position数组。  然后再将这个position数组传递到column b对应的access method，column b要求能够根据这些position快速的定位到value，然后在对返回  满足b&lt;3的value。

![](/assets/column-plan.png)

关键词，access metohd返回position，access method能够快速的找到position对应的value，operator操作&lt;value,position&gt;而不是tuple。



# 如何实现一个可扩展的面向列的直接操作压缩数据的查询执行引擎。

核心思想是通过添加block和DataSource抽象来实现的。添加block抽象，屏蔽掉底层的压缩细节，block提供接口返回&lt;value,position&gt;或者value,position等接口。添加DataSource抽象，用户可以将谓词，position下推到DataSource，DataSource返回满足条件的Block。operator通过调用调用DabaseSource来获取Block。Operator从block中读取数据执行自己内部的计算逻辑。



## 1.添加Block抽象

## 2.添加DataSource抽象。

  如下图，将DataSource添加到Operator体系中，让DataSource成为Operator的一个子类，DataSource负责从具体的column file中读取&lt;value,position&gt;返回给parent operator。DataSource平屏掉了底层的access method细节。



Operator

  DataSource

    RLEDataSource,使用RLEDecoder从disk page中解压出RLEBlock

	DeltaPosDataSource，使用DeltaPosDecoder从disk page中解压出DeltaBlock

	....

  Count

  Aggreagete



  

更具体的，每一种compression access method，都有一个对应的compression DataSource，DataSource是operator的一个子类，

这就相当于DataSource为parent Operator屏蔽掉了access method，使得每一个operator可以关注自己的计算逻辑。例如aggregate operator，不用关心Column是RLE compression，还是delta compression，aggregate operator可以直接使用DataSource接口来获取数据，DataSource在运行的时候，可以根据Column的压缩类型，动态的创建对应的的compression datasource，这个compression datasource知道如何解压column数据，更具体的，他用Decoder从disk page中解压出block，然后返回给parent operator。compression datasource，使用decoder从disk获取block。例如RLEDataSource,使用RLEDecoder从采用RLE压缩的Column中获取RLEBlock。



这样就实现了一个可以扩展的执行器架构。



