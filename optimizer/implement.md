# 1.expression

## 1.1 expression tree

expression tree由UnaryExpression，BinaryExpression，LeafExpression组成。其中UnaryExpression和BinaryExpressions属于内部节点，LeafExpression属于叶子节点。叶子节点的种类不多，有Literal，和Attribute。

BinaryOperator和BInaryExpression的区别是：第一个区别是，BinaryOperator是中缀表达式,a SYMBOL b,例如a+b,a&gt;b。

第二个区别是，BinaryOperator的两个输入操作数的数据类型必须相同，在进行求值之前，会对输入数据类型进行检查，如果数据类型不相同 ，会抛出一个异常。



expression tree层级如下：

+-- Expression

      +-- UnaryExpression,一元表达式

      +-- BinaryExpression，二元表达式

           +-- BinaryOperator，二元操作符

      +-- TernaryExpression，三元表达式

      +-- LeafExpression，叶子表达式

   

给定一个表达式，求值过程是递归求值的。对于大多数表达式而言，如果输入是null，那么输出也是null.

例如SparkSql中的BinaryExpression的eval如下：



```
  override def eval(input: InternalRow): Any = {
    val value1 = left.eval(input)
    if (value1 == null) {
      null
    } else {
      val value2 = right.eval(input)
      if (value2 == null) {
        null
      } else {
        nullSafeEval(value1, value2)
      }
    }
  }

```



 

只有极个别的表达式不是这样，例如三值逻辑表达式，例如EqualNullSafe

```
  override def eval(input: InternalRow): Any = {
    val value = child.eval(input)
    if (value == null) {
      null
    } else {
      nullSafeEval(value)
    }
  }
```



### 1.1.1 Literal,属于LeafExpression



### 1.1.2 算术表达式

算术表达式主要主要执行正/负，加减乘除求模等算术运算，主要有一元算术表达式和二元算术表达式。二元算术表达式和普通的二元表达式的区别是，二元算术表达式的两个输入操作数的数据类型是相同的。



算术表达式层级如下：

+-- UnaryExpression

      +-- UnaryMinus\(-1\)

      +-- UnaryPositive\(+1\)

      +-- Abs

+-- BinaryArithmetric

      +-- Add\(1+2\)

      +-- Subtract\(1-2\)

      +-- Miltiply\(1\*2\)

      +-- Divide\(1/2\)

      +-- Remainder\(%\)



### 1.1.3 谓词表达式

谓词表达式的计算结果是bool，尤其需要注意的是，谓词表达式是三值逻辑的。其真值表如下所示：

 ![](/assets/expression-predicate-equal-true-table.png)![](/assets/expression-predicate-logical-true-table.png)

* Not  ，三值逻辑

* And

```
 override def eval(input: InternalRow): Any = {
    val input1 = left.eval(input)
    if (input1 == false) {
       false
    } else {
      val input2 = right.eval(input)
      if (input2 == false) {
        false
      } else {
        if (input1 != null && input2 != null) {
          true
        } else {
          null
        }
      }
    }
  }
```

*  Or,三值逻辑

```
override def eval(input: InternalRow): Any = {
    val input1 = left.eval(input)
    if (input1 == true) {
      true
    } else {
      val input2 = right.eval(input)
      if (input2 == true) {
        true
      } else {
        if (input1 != null && input2 != null) {
          false
        } else {
          null
        }
      }
    }
  }
```

* In
* EqualTo\(二值逻辑）
* EqualNullSafe（三值逻辑）

```
  override def eval(input: InternalRow): Any = {
    val input1 = left.eval(input)
    val input2 = right.eval(input)
    if (input1 == null && input2 == null) {
      true
    } else if (input1 == null || input2 == null) {
      false
    } else {
      ordering.equiv(input1, input2)
    }
```



* LessThan\(a&lt;b\)
* LessThanOrEqual\(a&lt;=b\)
* GreaterThan\(a&gt;b\)
* GreaterThanOrEqual\(a&gt;=b\)



### 1.1.4 数学表达式

+-- LeafMathExpression

    +-- E

    +-- Pi

+-- UnaryMathExpression

    +-- Log

    +-- Sin

    +-- ...

+-- BinaryMathExpression

    +-- Pow

### 1.1.5 条件表达式

### 1.1.6 类型转换

### 1.1.7 日期表达式和字符串表达式

### 1.1.8 命名表达式和属性绑定

 Attribute

 Alias

 AttibuteReference

 BoundAttribute

### 1.1.9 聚合表达式



### 1.1.10 子查询

 SubQueryExpresion



## 1.2 重要属性

### 1.2.1 reference

 expression中的free variable.例如x+2+3+y,中的free  variable是x和y。这些free variable在进行实际求值之前，需要绑定到tuple中对应的属性。

这个属性非常重要。

### 1.2.2 foldable

如果一个expression是foldable的，那么可以直接替换为其计算结果。

### 1.2.3 isnull

expression的结果是否为null。

### 1.2.4 deterministic

同一个输入，多次求值结果是否相同。

## 1.3 表达式求值

 eval\(tuple\)

## 1.4 optimization

1.常量折叠.\(\(2+x\)+1\)+\(y+\(1+2\)\) =&gt;\(\(2+x\)+1\)+\(y+3\)，对1+2执行常量折叠。需要注意的是，\(2+x\)+1这个表达式不可折叠，因为x+2不是可折叠的。

2.对具有结合律的运算执行常量折叠：例如

\(\(2+x\)+1\)+\(y+\(1+2\)\) =&gt;2+1+1+2+\(x+y\)=&gt;6+x+y

3.简化bool表达式

简化：a AND true =&gt; a，a AND false=&gt;false。。。。



消除not：!\(a&gt;b\)=&gt;a&lt;=b

！！\(a&gt;b\)=&gt;a&gt;b

提取通用逻辑表达式，主要是应用&&和\|\|分配律来做的：

\( a OR b OR c OR e\) AND \(a OR b OR d OR f\)=&gt;\(a OR b\) OR \(\(c OR e\) AND \(d OR f\)\)

4.like化简

## 1.5 工具类

### 1.5.1 split condition to CNF

### 1.5.2 split condition to DNF

### 1.5.3 subsetof 



  

