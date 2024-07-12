> 参考：https://liebing.org.cn/apache-calcite-sql-validator.html

## 1 相关概念

### 1.1 Schema

`Schema`是用于描述**数据库结构**(如包含哪些表, 函数等)的一个接口。在**关系型数据库管理系统中通常有Catalog, Schema, Table这样的层级结构来管理表和数据库**, 不过并非每种RDBMS都完全这么实现, 比如MySQL不支持Catalog, 并且用Database来替代Schema的位置。

### 1.2 Calcite Scope

 SQL语言中同样有作用域, 在Calcite中称为Scope。

~~~sql
SELECT expr1
FROM t1,
     t2,
     (SELECT expr2 FROM t3) AS q3
WHERE c1 IN (SELECT expr3 FROM t4)
ORDER BY expr4
~~~

在查询的各个位置可用的作用域如下:

- `expr1`只能看见`t1`, `t2`和`q3`, 也就是说`expr1`只能使用`t1`, `t2`, `q3`中存在的列名.
- `expr2`只能看见`t3`.
- `expr3`只能看见`t4`.
- `expr4`只能看见`t1`, `t2`, `q3`, 加上`SELECT`子句中定义的任何别名.

### 1.3 Namespace

SQL语句需要从一个源中获取数据, Calcite将数据源抽象为命名空间Namespace。

 Namespace是一个抽象概念, 它既**可以表示一个表、视图或子查询**。

上述SQL语句中有4个Namespace: 

- **`t1`**
-  **`t2`**
-  **`(SELECT expr2 FROM t3) AS q3`**
- **`(SELECT expr3 FROM t4)`** 

Calcite中使用`SqlValidatorNamespace`表示Namespace

## 2 Calcite SQL验证流程

**在SQL语句中, DDL语句是不需要验证的, DQL和DML语句都需要验证**

### 2.1 SQL重写

重写阶段所作的工作就是对解析树进行一些轻微的调整, 一般情况下这一阶段也不需要任何改动, 只要大概了解其流程即可. 其实如果在解析阶段就生成标准的格式, 就不需要重写了, 只不过这样会让解析器的代码变得冗长, 这应该也是Calcite把一些重写工作放到验证阶段的原因

### 2.2 注册Scope和Namespace

### 2.3 SQL语句验证

通过Visitor模式调用`SqlNode.validate()`方法进行验证