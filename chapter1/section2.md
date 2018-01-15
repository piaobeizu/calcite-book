# 快速入门

本章主要介绍一个简单的CSV适配器如何一步一步的创建和连接到Calcite。这个适配器能够将一个目录下面的csv文件表现成一个包含各种表的schema。Calcite已经实现了rest接口，也提供了一套完整的SQL接口。

Calcite-example-CSV是一个功能完整的适配器，他能够使Calcite读CSV格式的文件。值得注意的是，几百行Java代码就足以提供完整的SQL查询功能。

CSV还可以作为构建适配器到其他数据格式的模板。尽管没有多少行代码，但它涵盖了几个重要的概念：

* 通过SchemaFactory和Schema 接口自定义schema

* 用json模型文件描述schema

* 用json模型文件描述视图view

* 通过表接口自定义表

* 确定表的记录类型

* 创建一个表的简单方法是使用`ScannableTable`接口直接列举所有的行

* 创建一个表的更高级的方法是实现`FilterableTable`接口，并且能够根据一些简单的判断来进行过滤

* 创建一个表的高级的方法是使用`TranslatableTable`类，这个类能使用规划规则转换为关系操作符。

### 下载安装

##### **环境准备：** java版本\(1.7或更高，最好1.8），git和maven（3.2.1或更高）

```
$ git clone https://github.com/apache/calcite.git
$ cd calcite
$ mvn install -DskipTests -Dcheckstyle.skip=true
$ cd example/csv
```

### 开始：

现在我们需要用到sqline来连接Calcite，这个工程里面包含了`SQL shell`脚本.

```
$ ./sqlline
```

执行这个命令后：

```
[INFO] 
[INFO] Calcite ............................................ SUCCESS [  1.003 s]
[INFO] Calcite Linq4j ..................................... SUCCESS [  0.021 s]
[INFO] Calcite Core ....................................... SUCCESS [  0.213 s]
[INFO] Calcite Cassandra .................................. SUCCESS [  0.136 s]
[INFO] Calcite Druid ...................................... SUCCESS [  0.061 s]
[INFO] Calcite Elasticsearch .............................. SUCCESS [  0.237 s]
[INFO] Calcite Elasticsearch5 ............................. SUCCESS [  0.269 s]
[INFO] Calcite Examples ................................... SUCCESS [  0.006 s]
[INFO] Calcite Example CSV ................................ SUCCESS [  0.099 s]
[INFO] Calcite Example Function ........................... SUCCESS [  0.019 s]
[INFO] Calcite File ....................................... SUCCESS [  0.090 s]
[INFO] Calcite MongoDB .................................... SUCCESS [  0.020 s]
[INFO] Calcite Pig ........................................ SUCCESS [  0.370 s]
[INFO] Calcite Piglet ..................................... SUCCESS [  0.011 s]
[INFO] Calcite Plus ....................................... SUCCESS [  0.115 s]
[INFO] Calcite Server ..................................... SUCCESS [  0.020 s]
[INFO] Calcite Spark ...................................... SUCCESS [  0.242 s]
[INFO] Calcite Splunk ..................................... SUCCESS [  0.014 s]
[INFO] Calcite Ubenchmark ................................. SUCCESS [  0.029 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.785 s
[INFO] Finished at: 2018-01-08T22:27:32+08:00
[INFO] Final Memory: 36M/359M
[INFO] ------------------------------------------------------------------------
sqlline version 1.3.0
sqlline>
```

继续执行如下命令：

```
sqlline> !connect jdbc:calcite:model=target/test-classes/model.json admin admin
```

（windows下请执行sqlline.bat命令）

执行元数据查询：

```
0: jdbc:calcite:model=target/test-classes/mod> !tables
+-----------+-------------+------------+------------+---------+----------+------------+-----------+---------------------------+----------------+
| TABLE_CAT | TABLE_SCHEM | TABLE_NAME | TABLE_TYPE | REMARKS | TYPE_CAT | TYPE_SCHEM | TYPE_NAME | SELF_REFERENCING_COL_NAME | REF_GENERATION |
+-----------+-------------+------------+------------+---------+----------+------------+-----------+---------------------------+----------------+
|           | SALES       | DEPTS      | TABLE      |         |          |            |           |                           |                |
|           | SALES       | EMPS       | TABLE      |         |          |            |           |                           |                |
|           | metadata    | COLUMNS    | SYSTEM_TABLE |         |          |            |           |                           |                |
|           | metadata    | TABLES     | SYSTEM_TABLE |         |          |            |           |                           |                |
+-----------+-------------+------------+------------+---------+----------+------------+-----------+---------------------------+----------------+
```

（温馨提示：在执行sqline的`!tables`命令后，后台执行了[`DatabaseMetaData.getTables()`](http://docs.oracle.com/javase/7/docs/api/java/sql/DatabaseMetaData.html#getTables%28java.lang.String, java.lang.String, java.lang.String, java.lang.String[]%29)。相同的元数据查询命令有`!columns`和`!describe`）

你现在能够看到在这个系统中有5个表：`EMPS`，`DEPTS`在当前的`SALES` schema中，并且`COLUMNS`和`TABLES`是在系统`metadata`的schema中。系统表始终存在于Calcite中，但是其他表都是由特定的schema的实现来生成；例如：`EMPS`和`DEPTS`表是基于`target/test-classes`下的`EMPS.CSV`和`DEPTS.csv`文件来生成的。

让我们在这些表的基础上做些查询操作来展示Calcite是怎样提供完整的SQL查询的实现。先来扫描一张表：

```
0: jdbc:calcite:model=target/test-classes/mod> select * from emps;
+-------+------+--------+--------+------+-------+-----+---------+---------+----------+
| EMPNO | NAME | DEPTNO | GENDER | CITY | EMPID | AGE | SLACKER | MANAGER | JOINEDAT |
+-------+------+--------+--------+------+-------+-----+---------+---------+----------+
| 100   | Fred | 10     |        |      | 30    | 25  | true    | false   | 1996-08-03 |
| 110   | Eric | 20     | M      | San Francisco | 3     | 80  |         | false   | 2001-01-01 |
| 110   | John | 40     | M      | Vancouver | 2     | null | false   | true    | 2002-05-03 |
| 120   | Wilma | 20     | F      |      | 1     | 5   |         | true    | 2005-09-07 |
| 130   | Alice | 40     | F      | Vancouver | 2     | null | false   | true    | 2007-01-01 |
+-------+------+--------+--------+------+-------+-----+---------+---------+----------+
```

加入 JION 和 GROUP BY：

```
0: jdbc:calcite:model=target/test-classes/mod>  SELECT d.name, COUNT(*)
. . . . . . . . . . . . . . . . . . . . . . .> FROM emps AS e JOIN depts AS d ON e.deptno = d.deptno
. . . . . . . . . . . . . . . . . . . . . . .> GROUP BY d.name;
+------+---------------------+
| NAME |       EXPR$1        |
+------+---------------------+
| Sales | 1                   |
| Marketing | 2               |
+------+---------------------+|
```

最后，VALUES操作能够生成单独的一行，这是一种非常方便的方式来测试表达式和内置的SQL函数：

```
0: jdbc:calcite:model=target/test-classes/mod> VALUES CHAR_LENGTH('Hello, ' || 'world!');
+------------+
|   EXPR$0   |
+------------+
| 13         |
+------------+
```

Calcite有很多种其他的SQL特征。我们不需要完整的介绍他们。你们可以写一些查询来测试。



