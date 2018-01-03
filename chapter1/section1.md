# 前奏

首先，我们需要明确一个概念：Apache Calcite 是一个动态的数据管理框架。

Calcite管理了很多种典型的数据库，但是他并没有这些数据库具有的关键能力：数据存储、数据处理算法、元数据的存储。也可以这样说：Calcite只是对各种数据库（不同的数据源）的查询进行了封装，并对外提供了统一的查询入口。

Calcite是有意的屏蔽这些基本的数据库所具有的能力。我们将看到这种“有意识”的做法对于业务应用（applications）与一种或多种数据源的数据存储和处理引擎之间的平衡是多么聪明的做法。同时，这种做法也为数据库添加数据（除查询外唯一的一种数据库操作）提供了坚实的基础。

让我们举个例子：我们先创建一个空的Calcite实例然后添加一些数据：

```
public static class HrSchema {
  public final Employee[] emps = 0;
  public final Department[] depts = 0;
}
Class.forName("org.apache.calcite.jdbc.Driver");
Properties info = new Properties();
info.setProperty("lex", "JAVA");
Connection connection =
    DriverManager.getConnection("jdbc:calcite:", info);
CalciteConnection calciteConnection =
    connection.unwrap(CalciteConnection.class);
SchemaPlus rootSchema = calciteConnection.getRootSchema();
Schema schema = ReflectiveSchema.create(calciteConnection,
    rootSchema, "hr", new HrSchema());
rootSchema.add("hr", schema);
Statement statement = calciteConnection.createStatement();
ResultSet resultSet = statement.executeQuery(
    "select d.deptno, min(e.empid)\n"
    + "from hr.emps as e\n"
    + "join hr.depts as d\n"
    + "  on e.deptno = d.deptno\n"
    + "group by d.deptno\n"
    + "having count(*) > 1");
print(resultSet);
resultSet.close();
statement.close();
connection.close();
```



