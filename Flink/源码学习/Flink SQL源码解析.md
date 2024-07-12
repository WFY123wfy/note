[TOC]

# 1 StreamTableEnvironment创建流程

~~~java
StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
// 入口
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv);
~~~

![image-20240711140557875](./picture/image-20240711140557875.png)

## 1.1 创建CatalogManager：管理元数据

![image-20240711140836312](./picture/image-20240711140836312.png)

![image-20240711141009281](./picture/image-20240711141009281.png)

## 1.2 创建Executor：transformations -> StreamGraph && execute

![image-20240711141453964](./picture/image-20240711141453964.png)

### 1.2.1 核心方法

![image-20240711142044971](./picture/image-20240711142044971.png)

#### 1 createPipeline：将transformations 转为 Pipeline(StreamGraph)

![image-20240711142251641](./picture/image-20240711142251641.png)

**createPipeline在DefaultExecutor中的具体实现：**

![image-20240711142545603](./picture/image-20240711142545603.png)

![image-20240711142815835](./picture/image-20240711142815835.png)

#### 2 execute、executeAsync：同步或异步触发执行Pipeline(StreamGraph)

![image-20240711142924395](./picture/image-20240711142924395.png)

## 1.3 创建Planner：SQL -> Operation  -> Transformation

**两大功能：**

- 通过getParser()方法获取SQL解析器，将SQL转换成Table API特定的Operation类型tree
- 优化、转化Operation类型tree为可执行的Transformation类型

![image-20240711145013953](./picture/image-20240711145013953.png)

### 1.3.1 核心方法

- getParser：获取SQL解析器
- translate：

![image-20240711145649243](./picture/image-20240711145649243.png)

![image-20240711151102196](./picture/image-20240711151102196.png)

### 1.3.2 创建流程

**由DefaultPlannerFactory的create方法创建，DefaultPlannerFactory对象是通过SPI机制创建的**

![image-20240711153706509](./picture/image-20240711153706509.png)

这里的PlannerFactory实现类为DefaultPlannerFactory：

![image-20240711181000154](./picture/image-20240711181000154.png)

DefaultPlannerFactory中的create方法，根据运行模式不同，可以创建StreamPlanner和BatchPlanner

![image-20240711181326196](./picture/image-20240711181326196.png)

# 2 Flink SQL是如何触发执行的

## 2.1 代码示例

~~~java
StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
streamEnv.setParallelism(1);
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv);

String createSourceTableSQL = "CREATE TABLE source_table ( " +
  "`id` INT, " +
  "`name` STRING," +
  " age INT ) WITH ( " +
  "    'connector' = 'datagen', " +
  "    'rows-per-second' = '1'" +
  ")";

String createSinkTableSQL = "CREATE TABLE print_table ( " +
  "`id` INT, " +
  "`name` STRING," +
  " age INT ) WITH ( " +
  "'connector' = 'print'" +
  ")";
tableEnv.executeSql(createSourceTableSQL);
tableEnv.executeSql(createSinkTableSQL);

tableEnv.executeSql("insert into print_table select id,name,age from source_table where id>100 and UPPER(name)='test'");
~~~

## 2.2 执行流程

**执行executeSql方法，会触发flink任务的执行，入口为TableEnvironmentImpl类。执行过程中会进行SQL解析 -> 校验 -> 转换 -> 优化 -> 代码生成 -> StreamGraph生成 -> 执行的各个步骤：**

![image-20240711143745780](./picture/image-20240711143745780.png)

**根据SQL的类型不同，有不同的执行方式：**

- ModifyOperation：INSERT INTO类型
- CreateTableOperation：CREATE TABLE类型
- DropTableOperation：DROP TABLE类型

![image-20240711144018234](./picture/image-20240711144018234.png)

这里以INSERT INTO类型举例：会将transfromations 转换成 StreamGrap，在触发执行

![image-20240711143240333](./picture/image-20240711143240333.png)

# 3 Flink SQL解析流程

> 参考：https://blog.csdn.net/hiliang521/article/details/134861469?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-134861469-blog-132372907.235%5Ev43%5Epc_blog_bottom_relevance_base7&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-1-134861469-blog-132372907.235%5Ev43%5Epc_blog_bottom_relevance_base7&utm_relevant_index=2

![image-20240712152539957](/Users/wufeiye1/code/study/note/Flink/源码学习/picture/image-20240712152539957.png)

## 3.1 SQL解析器的Java代码生成

![image-20240711155101189](./picture/image-20240711155101189.png)

**Flink源码中，涉及SQL解析器生成的目录如下：**

![image-20240711155330577](./picture/image-20240711155330577.png)

生成的SQL解析器类名为**org.apache.flink.sql.parser.impl.FlinkSqlParserImpl**，相关配置如下：

![image-20240711155504780](./picture/image-20240711155504780.png)

![image-20240711155616643](./picture/image-20240711155616643.png)

**FlinkSqlParserImpl的代码是通过JavaCC将Parser.jj转换生成的，对应关系举例如下：**

- **SqlParserImplFactory FACTORY**

![image-20240711160205422](./picture/image-20240711160205422.png)

![image-20240711160126654](./picture/image-20240711160126654.png)

- **SqlNode SqlStmt() 方法：将SQL语句解析成SqlNode**

![image-20240711160615685](./picture/image-20240711160615685.png)

![image-20240711160748409](./picture/image-20240711160748409.png)

![image-20240711160823899](./picture/image-20240711160823899.png)

![image-20240711160904628](./picture/image-20240711160904628.png)

## 3.2 SQL -> Operation

![image-20240711182401974](./picture/image-20240711182401974.png)

**获取Flink层面的解析器ParserImpl对象，用来将SQL -> Operation**

**流模式下的planner是StreamPlanner，调用StreamPlanner的getParser方法获取（这里的getParser方法实现在PlannerBase中）**

![image-20240711182552759](./picture/image-20240711182552759.png)

--->

![image-20240711182731751](./picture/image-20240711182731751.png)

--->

![image-20240711183329432](./picture/image-20240711183329432.png)

通过SPI机制扫描获取parserFactory，默认的实现类为DefaultParserFactory，

![image-20240711183207175](./picture/image-20240711183207175.png)

---> DefaultParserFactory.create：

![image-20240711183529236](./picture/image-20240711183529236.png)

**分别提供了validatorSupplier和calciteParserSupplier：**

![image-20240711184408468](./picture/image-20240711184408468.png)

- **validatorSupplier分析：后续会通过FlinkPlannerImpl.getOrCreateSqlValidator方法来创建validator**

![image-20240711185051635](./picture/image-20240711185051635.png)

![image-20240711185255471](./picture/image-20240711185255471.png)

- **calciteParserSupplier分析：创建了一个CalciteParser对象，CalcitePaser中会调用SqlParser.create创建Calcite生成的FlinkSqlParserImpl对象(.jj文件生成)，后续会通过FlinkSqlParserImpl将SQL转换为SQLNode**

![image-20240711185418104](./picture/image-20240711185418104.png)

---> 

![image-20240711185526413](./picture/image-20240711185526413.png)

--->

![image-20240711185737226](./picture/image-20240711185737226.png)



Parser创建完成：public class ParserImpl implements Parser，Parser核心方法如下：

![image-20240711183754216](./picture/image-20240711183754216.png)

 **Flink层面的解析器ParserImpl对象 -> parse流程**

![。](./picture/image-20240711190514669.png)

### 3.2.1 SQL -> SqlNode AST（未校验）

```java
SqlNode parsed = parser.parse(statement);
```

**这里创建的parser是FlinkSqlParserImpl对象，FlinkSqlParserImpl同过.jj文件生成，通过parseStmt()方法将SQL转换成抽象语法树，这里返回的SqlNode为抽象语法树的根节点，此时并没有对抽象语法树的各个节点做校验。**

![image-20240711190755487](/Users/wufeiye1/code/study/note/Flink/源码学习/picture/image-20240711190755487.png)

![image-20240711190954629](./picture/image-20240711190954629.png)

### 3.3.2 SqlNode -> Operation

```java
Operation operation =
        SqlToOperationConverter.convert(planner, catalogManager, parsed)
                .orElseThrow(() -> new TableException("Unsupported query: " + statement));
```

![image-20240711204024195](./picture/image-20240711204024195.png)

**（1）SqlNode -> validated SqlNode**

```java
final SqlNode validated = flinkPlanner.validate(sqlNode);
```

![image-20240711204726522](./picture/image-20240711204726522.png)

**补充：遍历SqlNode是通过访问者模式，调用SqlNode的accept方法，出入响应的SqlVisitor，对于SQL校验要传入validator**

![image-20240711204628478](./picture/image-20240711204628478.png)

SqlCall的accept实现：

![image-20240711205137686](./picture/image-20240711205137686.png)

**（2）根据不同的SqlNode类型，将validated SqlNode转换成不同的Operation**

![image-20240712134656130](./picture/image-20240712134656130.png)

## 3.3 Operation -> Transformation

### 3.3.1 转换：Operation -> RelNode



### 3.3.2 优化：RelNode -> RelNode

### 3.3.3 生成执行图execGraph：RelNode -> ExecNodeGraph

### 3.3.4 生成ransformations DAG：ExecNodeGraph -> Transformation

## 3.4 executeInternal

### 3.4.1 Transformation -> Pipeline（StreamGraph）

### 3.4.2 executeAsync pipeline

## 3.5





SPI机制的扫描文件

![image-20240711153706509](./picture/image-20240711153706509.png)