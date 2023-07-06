---
title: 如何使用calcite构建SQL并执行查询
date: 2023-07-01 14:19:50
tags: [calcite,编译原理,database]
---
大家好，这是Calcite的第二篇文章了，我一直毫不掩饰对calcite的喜爱，而且一直在致力于为社区做一些贡献，如果你也喜欢这个项目的话，欢迎评论，转发，如果没看过第一篇的话，也欢迎移步去看看（手把手教你使用Calcite查看SQL执行计划）。

今天我要分享的主题是关于 Calcite 关系代数 以及 SQL 的那些事，Let's go !!!
<!-- more -->

## 关系代数

首先关系代数是 Calcite 的核心。每个查询都可以表示为一个 关系运算符树。你可以将 SQL 转换为关系代数，也可以直接构建关系运算符树。

优化器规则使用保持 相同语义 的 数学恒等式 来变换表达式树。例如，如果过滤器没有引用其他输入中的列，那么将过滤器推入到内部关联的输入则是有效的。

Calcite 通过反复地将优化器规则应用于关系表达式来优化查询。成本模型指导该过程，优化器引擎生成与原始语义相同，但成本较低的替代表达式。

优化过程是可扩展的。你可以添加自己的 关系运算符、优化器规则、成本模型 和 统计信息。


### 代数构建器

构建关系表达式的最简单方法是使用代数构建器 RelBuilder。下面是一个例子：

### 扫描表

```java
FrameworkConfig config;
RelBuilder builder = RelBuilder.create(config);
RelNode cnode = relBuilder.scan("consumers").build();
System.out.println("==> "+RelOptUtil.toString(cnode));
```
其执行结果如下：
```
==> LogicalTableScan(table=[[consumers]])
```
等价与SQL
```sql
SELECT * FROM consumers ;
```


### 添加投影
现在，让我们添加一个投影，相当于如下 SQL：
```sql
SELECT firstname,lastname FROM consumers ;
```

我们只需要在调用 build 方法前，添加一个 project 方法调用：
```java
cnode = relBuilder.scan("consumers").project(relBuilder.field("firstname"),
        relBuilder.field("lastname")).build();
System.out.println("==> "+RelOptUtil.toString(cnode));
```
其执行结果如下：
```
==> LogicalProject(firstname=[$1], lastname=[$2])
  LogicalTableScan(table=[[consumers]])
```

### 添加过滤聚合
下面是一个包含聚合和过滤的查询语句：
```java
cnode = relBuilder.scan("orders")
            .aggregate(
                    relBuilder.groupKey("goods"),
                    relBuilder.count(false, "c"),
                    relBuilder.sum(false, "p", relBuilder.field("price")))
            .filter(
                    relBuilder.call(
                        SqlStdOperatorTable.GREATER_THAN,
                                relBuilder.field("c"),
                                relBuilder.literal(1))
                    ).build();

System.out.println(RelOptUtil.toString(cnode));
```
相当于如下 SQL：
```sql
SELECT goods, count(*) AS count, sum(price) AS p FROM orders GROUP BY goods HAVING count(*) > 1
```
其执行结果如下：
```
LogicalFilter(condition=[>($1, 1)])
  LogicalAggregate(group=[{1}], c=[COUNT()], p=[SUM($2)])
    LogicalTableScan(table=[[orders]])
```

好了，接下来我们来看进一步的实例。

## 实例 CalciteRelBuilderCase 完整代码
```java
package com.dafei1288;

import org.apache.calcite.adapter.csv.CsvSchema;
import org.apache.calcite.adapter.csv.CsvTable;
import org.apache.calcite.plan.RelOptUtil;
import org.apache.calcite.rel.RelNode;
import org.apache.calcite.rel.core.JoinRelType;
import org.apache.calcite.schema.SchemaPlus;
import org.apache.calcite.sql.fun.SqlStdOperatorTable;
import org.apache.calcite.sql.parser.SqlParser;
import org.apache.calcite.tools.FrameworkConfig;
import org.apache.calcite.tools.Frameworks;
import org.apache.calcite.tools.RelBuilder;
import org.apache.calcite.tools.RelRunners;

import java.io.File;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.ResultSetMetaData;
import java.sql.SQLException;

public class CalciteRelBuilderCase {
    public static void main(String[] args) throws SQLException {
        SchemaPlus rootSchema = Frameworks.createRootSchema(true);
        String csvPath = "D:\\study\\codespace\\testcalcite\\src\\main\\resources\\db";
        CsvSchema csvSchema = new CsvSchema(new File(csvPath), CsvTable.Flavor.SCANNABLE);
        rootSchema.add("consumers", csvSchema.getTable("consumers"));
        rootSchema.add("orders", csvSchema.getTable("orders"));

        FrameworkConfig frameworkConfig = Frameworks.newConfigBuilder()
                .parserConfig(SqlParser.Config.DEFAULT)
                .defaultSchema(rootSchema)
                .build();

        RelBuilder relBuilder = RelBuilder.create(frameworkConfig);
//        id,goods,price,amount,user_id               orders
//        id,firstname,lastname,birth                 consumers
        RelNode node = relBuilder
                .scan("consumers")
                .scan("orders")
                .join(JoinRelType.INNER,
                        relBuilder.call(SqlStdOperatorTable.EQUALS,
                                relBuilder.field("id"),
                                relBuilder.field("user_id")))
                .filter(
                        relBuilder.call(SqlStdOperatorTable.EQUALS,
                                relBuilder.field("lastname"),
                                relBuilder.literal("jacky")))
                .project(
                        relBuilder.field("id"),
                        relBuilder.field("goods"),
                        relBuilder.field("price"),
                        relBuilder.field("firstname"),
                        relBuilder.field("lastname"))
                .sortLimit(0, 5, relBuilder.field("id"))
                .build();

        System.out.println(RelOptUtil.toString(node));




        System.out.println("[result:]");
        PreparedStatement run = RelRunners.run(node);
        ResultSet resultSet = run.executeQuery();

        ResultSetMetaData resultSetMetaData = resultSet.getMetaData();
        int colCount = resultSetMetaData.getColumnCount();
        for(int i = 1; i <= colCount ; i++){
            System.out.print(resultSetMetaData.getColumnName(i)+"\t");
        }
        System.out.println();

        while(resultSet.next()){
            for(int i = 1; i <= colCount ; i++){
                System.out.print(resultSet.getString(i)+"\t");
            }
            System.out.println();
        }
    }
}

```


## 执行结果
```
LogicalSort(sort0=[$0], dir0=[ASC], fetch=[5])
  LogicalProject(id=[$0], goods=[$5], price=[$6], firstname=[$1], lastname=[$2])
    LogicalFilter(condition=[=($2, 'jacky')])
      LogicalJoin(condition=[=($0, $4)], joinType=[inner])
        LogicalTableScan(table=[[consumers]])
        LogicalTableScan(table=[[orders]])

[result:]
id	goods	price	firstname	lastname	
1	book	100	li	jacky	

```

好了，今天的分享就到这里，下次我们来看看如果进行SQL改写。



## 参考连接

https://calcite.apache.org/docs/algebra.html
https://strongduanmu.com/wiki/calcite/algebra.html
https://zhuanlan.zhihu.com/p/623062311
https://blog.csdn.net/skyyws/article/details/124828049

