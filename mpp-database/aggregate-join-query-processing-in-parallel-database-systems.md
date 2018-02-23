**Select** PARTS.City, AVG\(Qty\)

> **From** PARTS, SHIPMENT
>
> **Where** PARTS.P\# = SHIPMENT.P\#
>
> **Group By** PARTS.City
>
> **Having** AVG\(Qty\)&gt;500 AND AVG\(Qty\)

## Partitioning Method \(JPM\)

* 按照join attribute将两个表分区到N个processor上。例如，按照P\#，将PARTS和SHIPMENT分区到processor1-processor4，使processor1-processor4分别存储 pI-p99, p100- p199, p200-p299, and p300-399对应的数据。
* 在每一个processor上执行join，和local aggregation
* 将local aggregation后的结果distribution到N个processor上。
* 在每一个processor上执行global aggregation。
* 最后coordinator processor对每一个processor上的global aggregation结果做一个union返回即可。



![](/assets/JPM.png)

## Aggregate Partitioning Method \(APM\)

* 按照city attribute对表PART分区

![](/assets/APM.png)

## Hybrid Partitioning Method \(HPM\).



