> 参考：https://liebing.org.cn/apache-calcite-relational-algebra.html
>
> 关系代数最早由E. F. Codd在1970年的论文”[A Relational Model of Data for Large Shared Data Banks](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf)“中提出, 是关系型数据库查询语言的基础, 也是查询优化技术的理论基础。
>
> 随着关系代数和关系模型的不断发展和完善, 目前几乎所有对外支持SQL访问的系统, 都会将SQL转化为等价的关系代数表达, 并基于此进行查询优化。
>
> 在Calcite内部, 同样会将SQL查询转化为一颗等价的关系算子树, 并在此基础上进行查询优化.

## 1 关系代数

主要的关系代数运算如下表所示：

| **类别**     | **名称**                    | **符号** | **示例**                                                     |
| :----------- | :-------------------------- | :------- | :----------------------------------------------------------- |
| **一元运算** | 选择(Select)                | σ𝜎       | 符号: σcondition(R)𝜎𝑐𝑜𝑛𝑑𝑖𝑡𝑖𝑜𝑛(𝑅) SQL: `SELECT * FROM R WHERE condition` |
|              | 投影(Project)               | ΠΠ       | 符号: Πx,y(R)Π𝑥,𝑦(𝑅) SQL: `SELECT x, y FROM R`               |
|              | 赋值(Assignment)            | ←←       | 符号:t←Πx,y(σcondition(R))𝑡←Π𝑥,𝑦(𝜎𝑐𝑜𝑛𝑑𝑖𝑡𝑖𝑜𝑛(𝑅)) SQL: `CREATE VIEW t AS SELECT x, y FROM R WHERE condition` |
|              | 重命名(Rename)              | ρ𝜌       | 符号: ρS(x1,y1)(R)𝜌𝑆(𝑥1,𝑦1)(𝑅) SQL: `SELECT * FROM (SELECT x AS x1, y AS y1 FROM R) AS S` |
| **二元运算** | 并(Union)                   | ∪∪       | 符号: R∪S𝑅∪𝑆 SQL: `SELECT * FROM R UNION SELECT * FROM S`    |
|              | 交(Intersection)            | ∩∩       | 符号: R∩S=R−(R−S)𝑅∩𝑆=𝑅−(𝑅−𝑆) SQL: `SELECT * FROM R WHERE x NOT IN (SELECT x FROM R WHERE x NOT IN (SELECT x FROM S))` |
|              | 差(Difference)              | −−       | 符号: R−S𝑅−𝑆 SQL: `SELECT * FROM R WHERE x NOT IN (SELECT x FROM S)` |
|              | 笛卡儿积(Cartesian-product) | ××       | 符号: R×S𝑅×𝑆 SQL: `SELECT * FROM R, S`                       |
|              | 除(Divide)                  | ÷÷       | 符号: R÷S𝑅÷𝑆 SQL: `SELECT DISTINCT r1.x FROM R AS r1 WHERE NOT EXISTS (SELECT S.y FROM S WHERE NOT EXISTS (SELECT * FROM R AS r2 WHERE r2.x = r1.x AND r2.y = S.y))` |
|              | 连接(Join)                  | ⋈⋈       | 符号: R⋈conditionS𝑅⋈𝑐𝑜𝑛𝑑𝑖𝑡𝑖𝑜𝑛𝑆 SQL: `SELECT * FROM R JOIN S ON condition` |