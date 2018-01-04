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

* 创建一个表的更高级的方法是实现` FilterableTable`接口，并且能够根据一些简单的判断来进行过滤

* 创建一个表的高级的方法是使用`TranslatableTable`类，这个类能使用规划规则转换为关系操作符。



