# 代数

关系代数是Calcite的核心。每个查询都被表示为一颗关联操作树。你可以将SQL翻译成关联代数，或者直接建立关联操作树。

规划器规则使用保留语义的数学标识来转换表达式树。例如，如果过滤器不引用来自其他输入的列，则将过滤器推入内部联接的输入是有效的。Calcite是通过不停的将计划规则应用到关联表达式来达到优化查询语句的目的。成本模型在这个过程中起到指导的作用，并且计划引擎会生成一个拥有相同语义但是耗时更少的可替代的表达式来执行。这个计划过程是可以扩展的，你可以添加你自己的执行计划，优化规则，成本模型和统计。

## 代数创建

最简单的创建关联表达式的方式是使用代数方式来创建，[RelBuilder](http://calcite.apache.org/apidocs/org/apache/calcite/tools/RelBuilder.html)，下面给出例子：

### tablescan

```
final FrameworkConfig config;
final RelBuilder builder = RelBuilder.create(config);
final RelNode node = builder
  .scan("EMP")
  .build();
System.out.println(RelOptUtil.toString(node));
```

([RelBuilderExample.java](https://github.com/apache/calcite/blob/master/core/src/test/java/org/apache/calcite/examples/RelBuilderExample.java) 在这个类中你能够找到这个上面的例子和其他的例子的完整代码)



这个代码的是：

```
LogicalTableScan(table=[[scott, EMP]])
```

这行代码扫描整个的`EMP`表，等同于下面的SQL语句：

```
SELECT * FROM scott.EMP;
```



### 添加一个项目

现在，添加一个项目，等同于一下语句：

```
SELECT ename, deptno FROM scott.EMP;
```

我们只是在调用`build`方法之前添加一个调用`project`方法：

```
final RelNode node = builder
  .scan("EMP")
  .project(builder.field("DEPTNO"), builder.field("ENAME"))
  .build();
System.out.println(RelOptUtil.toString(node));
```

并且输出是：

```
LogicalProject(DEPTNO=[$7], ENAME=[$1])
LogicalTableScan(table=[[scott, EMP]])
```

对`builder.field`的两次调用创建简单表达式，该表达式返回来自输入的关系表达式中的字段，即由`scan`调用创建的TableScan。Calcite已按顺序\$7到\$1将它们转换为字段引用。



### 添加过滤器和聚合

下面是一个带有过滤和聚合的查询：

```
final RelNode node = builder
  .scan("EMP")
  .aggregate(builder.groupKey("DEPTNO"),
      builder.count(false, "C"),
      builder.sum(false, "S", builder.field("SAL")))
  .filter(
      builder.call(SqlStdOperatorTable.GREATER_THAN,
          builder.field("C"),
          builder.literal(10)))
  .build();
System.out.println(RelOptUtil.toString(node));
```

它等同于下面的sql语句：

```
SELECT deptno, count(*) AS c, sum(sal) AS s FROM emp GROUP BY deptno HAVING count(*) > 10
```

并且会产生如下的代码：

```
LogicalFilter(condition=[>($1, 10)])
LogicalAggregate(group=[{7}], C=[COUNT()], S=[SUM($5)])
LogicalTableScan(table=[[scott, EMP]])
```



### 压栈和出栈

构建器使用堆栈来存储一步生成的关系表达式，并将其作为输入传递给下一步。这将允许生成关联表达式的方法来生成一个构建器。大部分情况下，你将唯一使用的栈方法是`build()`，以获取最后一个关联表达式，即树的根。有时候堆栈变得如此之深以至于会变得混乱。为了保持(逻辑)清晰，你可以从堆栈中删除表达式。例如，我们能够创建一个非常复杂的join关联：

```
.
               join
             /      \
        join          join
      /      \      /      \
CUSTOMERS ORDERS LINE_ITEMS PRODUCTS
```

创建上面的表达式树，我们能够通过三个阶段来完成。用`left`和`right`变量来存储中间变量，并在创建最终`Join`时使用`push()`将它们放回堆栈。

```
final RelNode left = builder
  .scan("CUSTOMERS")
  .scan("ORDERS")
  .join(JoinRelType.INNER, "ORDER_ID")
  .build();

final RelNode right = builder
  .scan("LINE_ITEMS")
  .scan("PRODUCTS")
  .join(JoinRelType.INNER, "PRODUCT_ID")
  .build();

final RelNode result = builder
  .push(left)
  .push(right)
  .join(JoinRelType.INNER, "ORDER_ID")
  .build();
```



### 字段名称和序号

您可以通过名称或序号来引用字段。序数是基于零的，每步操作都必须保证它输出字段的顺序。例如，Project返回每个标量表达式生成的字段。每个操作的名称必须保证是唯一的，但有时候这也会导致这些名称并不完全符合你的期望。例如，当你加入EMP和EPT，输出列中的其中一个被称为DEPTNO并且另一个可能会被称为DEPTNO_1。

下面的这些关系表达式方法可以更好地控制字段名称：

- `project`可以让你使用`alias(expr, fieldName)`来封装表达式。它会移除这些封装但是会保持建议的名称(只要这些名称是唯一的)。
- `values(String[] fieldNames, Object... values)`允许一系列的字段名称。这些字段名称中只要有一个是null，构建器都会生成唯一的名称。

如果表达式隐射了输入字段或输入字段的强制类型，它将使用该输入字段的名称。一旦唯一的名称已经被分配了，那么名称将不会再改变。如果你有自定义的RelNode实例，你可以依赖这些不变的字段名称。事实上，在整个关联表达式中，这些字段名称都是唯一不变的。

但是，如果一个关联表达式已经通过了几个重新规则（参考[RelOptRule](http://calcite.apache.org/apidocs/org/apache/calcite/plan/RelOptRule.html)）结果表达式中的字段名称将看起来不像原始的那样。在这一点上，最好按顺序引用字段。在构建接受多个输入的关系表达式时，您需要构建将其考虑在内的字段引用。这种情况最多发生在join条件中。

假设你现在在EMP和DEPT表格上创建一个join条件，EMP表格有8个字段 [EMPNO, ENAME, JOB, MGR, HIREDATE, SAL, COMM, DEPTNO]，DEPT表格有3个字段 [DEPTNO, DNAME, LOC]。在内部，Calcite会将这些字段表达成一个有11个字段的混合输入列的偏移量，并且正确的输入中的第一个字段是字段 #8。

但是通过构建器API，您可以指定哪个输入的哪个字段。要引用"SAL"(内部是 列#5)，可以写成`builder.field(2, 0, "SAL")`, `builder.field(2, "EMP", "SAL")`，或者`builder.field(2, 0, 5)`。这就意味着字段#5是两个输入中的输入#0。(为什么需要知道有两个输入？因为它们是被存储在栈中：输入#1是在栈顶，输入#0是在它的下面。如果我们不告诉构建器是两个输入，那么它不知道输入#0在栈中具体的位置)。类似的，要引用"DNAME"(内部是列#9(8+1))，可以写成`builder.field(2, 1, "DNAME")`, `builder.field(2, "DEPT", "DNAME")`,或者`builder.field(2, 1, 1)`。



