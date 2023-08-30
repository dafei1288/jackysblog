---
title: 如何使用calcite rule做SQL重写（下）
date: 2023-08-18 15:10:43
tags: [calcite,编译原理,database]
---

上一篇文章我们介绍了如何使用默认规则做条件下推，今天我们来尝试自定义规则，来实现对SQL的重写。
我们本期将会深入浅出的以修改查询表为例，进行Sql rewrite,比如 我们输入 `Select * from consumers` 实际查询则为 `Select * from consumers_1`，这个需求在分库分表里应该很常见了。
<!-- more -->


## 执行流程he

我们再来复习一下执行流程，借用 柳年思水 大佬的图，HepPlanner 在优化过程中，是先遍历规则，然后再对每个节点进行匹配转换，直到满足条件（超过限制次数或者规则遍历完一遍不会再有新的变化）

![](./img/blog/1004/11-hep.png)


## 准备数据
首先我们先准备数据，延用我们一直的案例，将`consumers.csv`再复制一份`consumers_1.csv`并修改里面的记录，用于区分效果。

### consumers.csv

```csv
id,firstname,lastname,birth
1,li,jacky,1984
2,li,doudou,2019
3,li,maimai,2019
```

### consumers_1.csv

```csv
id,firstname,lastname,birth
1,liu,jacky,1984
2,liu,doudou,2019
3,liu,maimai,2019
```

## 自定义规则

根据我们的需求，实际上只需要在扫描表的适合，对应的将源表替换为目标表即可，
所以，在`onMatch(final RelOptRuleCall call)`方法里，匹配到映射关系，将源表替换成重新生成的`RelNode`
```java
TableScan tableScan = call.rel(0);

for(String tn : tableScan.getTable().getQualifiedName()){
    if(srcName.equals(tn)){
        RelBuilder relBuilder = RelBuilder.create(frameworkConfig);
        RelNode relNode = relBuilder.scan(targetName).build();
         call.transformTo(relNode);
    }
}
```

构建配置的匹配规则
```java
default JackyTableRenameRule.Config withOperandFor(Class<? extends TableScan> tableScanClass) {
            return withOperandSupplier(b0 ->
                    b0.operand(tableScanClass).anyInputs())
                    .as(JackyTableRenameRule.Config.class);
        }
```

以下是完整代码

```java

package com.dafei1288;

import org.apache.calcite.plan.RelOptRuleCall;
import org.apache.calcite.plan.RelRule;
import org.apache.calcite.rel.RelNode;
import org.apache.calcite.rel.core.TableScan;
import org.apache.calcite.rel.rules.*;
import org.apache.calcite.tools.FrameworkConfig;
import org.apache.calcite.tools.RelBuilder;
import org.apache.calcite.tools.RelBuilderFactory;
import org.immutables.value.Value;

@Value.Enclosing
public class JackyTableRenameRule extends RelRule<JackyTableRenameRule.Config> implements TransformationRule {

    private String srcName;
    private String targetName;

    private FrameworkConfig frameworkConfig;

    public String getSrcName() {
        return srcName;
    }

    public void setSrcName(String srcName) {
        this.srcName = srcName;
    }

    public String getTargetName() {
        return targetName;
    }

    public void setTargetName(String targetName) {
        this.targetName = targetName;
    }

    public FrameworkConfig getFrameworkConfig() {
        return frameworkConfig;
    }

    public void setFrameworkConfig(FrameworkConfig frameworkConfig) {
        this.frameworkConfig = frameworkConfig;
    }

    protected JackyTableRenameRule(Config config) {
        super( config );

    }

    @Deprecated // to be removed before 2.0
    public JackyTableRenameRule(RelBuilderFactory relBuilderFactory) {
        this(JackyTableRenameRule.Config.DEFAULT.withRelBuilderFactory(relBuilderFactory)
                .as(JackyTableRenameRule.Config.class));
    }

    @Override
    public void onMatch(final RelOptRuleCall call) {
        TableScan tableScan = call.rel(0);

        for(String tn : tableScan.getTable().getQualifiedName()){
            if(srcName.equals(tn)){
                RelBuilder relBuilder = RelBuilder.create(frameworkConfig);
                RelNode relNode = relBuilder.scan(targetName).build();
                call.transformTo(relNode);
            }
        }
    }


    @Value.Immutable
    public interface Config extends RelRule.Config {

        static
        Config DEFAULT = ImmutableJackyTableRenameRule.Config.builder().build().withOperandFor(TableScan.class);

        @Override default JackyTableRenameRule toRule() {
            return new JackyTableRenameRule(this);
        }

        default JackyTableRenameRule toRule(String srcName, String targetName) {
            JackyTableRenameRule jtrr = new JackyTableRenameRule(this);
            jtrr.setSrcName(srcName);
            jtrr.setTargetName(targetName);
            return jtrr;
        }


        @Value.Default default boolean isAllowAlwaysTrueCondition() {
            return true;
        }

        default JackyTableRenameRule.Config withOperandFor(Class<? extends TableScan> tableScanClass) {
            return withOperandSupplier(b0 ->
                    b0.operand(tableScanClass).anyInputs())
                    .as(JackyTableRenameRule.Config.class);
        }
    }
}

```

## 测试代码

下面是实例化规则，并加入优化代码
```java
HepProgramBuilder hepProgramBuilder = HepProgram.builder();

JackyTableRenameRule jtrr = JackyTableRenameRule.Config.DEFAULT.toRule("consumers","consumers_1");
jtrr.setFrameworkConfig(frameworkConfig);

hepProgramBuilder.addRuleInstance(jtrr);

HepProgram program = hepProgramBuilder.build();
HepPlanner hepPlanner = new HepPlanner(program);
hepPlanner.setRoot(opTree);
RelNode r = hepPlanner.findBestExp();
```

完整测试代码：

```java
package com.dafei1288;


import org.apache.calcite.adapter.csv.CsvSchema;
import org.apache.calcite.adapter.csv.CsvTable;
import org.apache.calcite.plan.RelOptUtil;
import org.apache.calcite.plan.hep.HepPlanner;
import org.apache.calcite.plan.hep.HepProgram;
import org.apache.calcite.plan.hep.HepProgramBuilder;
import org.apache.calcite.rel.RelNode;
import org.apache.calcite.rel.RelWriter;
import org.apache.calcite.rel.core.JoinRelType;
import org.apache.calcite.rel.externalize.RelWriterImpl;
import org.apache.calcite.rel.rules.FilterJoinRule;
import org.apache.calcite.schema.SchemaPlus;
import org.apache.calcite.sql.fun.SqlStdOperatorTable;
import org.apache.calcite.sql.parser.SqlParser;
import org.apache.calcite.tools.FrameworkConfig;
import org.apache.calcite.tools.Frameworks;
import org.apache.calcite.tools.RelBuilder;
import org.apache.calcite.tools.RelRunners;

import java.io.File;
import java.io.PrintWriter;
import java.sql.ResultSet;


public class JackSqlRewriteCase {
    public static void main(String[] args) throws Exception {

        SchemaPlus rootSchema = Frameworks.createRootSchema(true);
        String csvPath = "src\\main\\resources\\db";
        CsvSchema csvSchema = new CsvSchema(new File(csvPath), CsvTable.Flavor.SCANNABLE);
        rootSchema.add("consumers", csvSchema.getTable("consumers"));
        rootSchema.add("consumers_1", csvSchema.getTable("consumers_1"));
        rootSchema.add("orders", csvSchema.getTable("orders"));

        FrameworkConfig frameworkConfig = Frameworks.newConfigBuilder()
                .parserConfig(SqlParser.Config.DEFAULT)
                .defaultSchema(rootSchema)
                .build();


        RelBuilder relBuilder = RelBuilder.create(frameworkConfig);

        RelNode cnode = relBuilder.scan("consumers").build();
        System.out.println("==> "+ RelOptUtil.toString(cnode));

        cnode = relBuilder.scan("consumers").project(relBuilder.field("firstname"),
                relBuilder.field("lastname")).build();
        System.out.println("==> "+RelOptUtil.toString(cnode));

        RelNode opTree = relBuilder
                .scan("consumers")
                .project(
                        relBuilder.field("id"),
                        relBuilder.field("firstname"),
                        relBuilder.field("lastname"))
                .build();

        RelWriter rw = new RelWriterImpl(new PrintWriter(System.out, true));
        opTree.explain(rw);


        System.out.println();


        HepProgramBuilder hepProgramBuilder = HepProgram.builder();

        JackyTableRenameRule jtrr = JackyTableRenameRule.Config.DEFAULT.toRule("consumers","consumers_1");
        jtrr.setFrameworkConfig(frameworkConfig);

        hepProgramBuilder.addRuleInstance(jtrr);

        HepProgram program = hepProgramBuilder.build();
        HepPlanner hepPlanner = new HepPlanner(program);
        hepPlanner.setRoot(opTree);
        RelNode r = hepPlanner.findBestExp();
        r.explain(rw);

        System.out.println();

        ResultSet result = RelRunners.run(r).executeQuery();
        int columns = result.getMetaData().getColumnCount();
        while (result.next()) {
            System.out.println(result.getString(1) + " " + result.getString(2) +  " " + result.getString(3));
        }
    }
}

```

## 结果展示

```
==> LogicalTableScan(table=[[consumers]])

==> LogicalProject(firstname=[$1], lastname=[$2])
  LogicalTableScan(table=[[consumers]])

4:LogicalProject(id=[$0], firstname=[$1], lastname=[$2])
  3:LogicalTableScan(table=[[consumers]])

6:LogicalProject(id=[$0], firstname=[$1], lastname=[$2])
  8:LogicalTableScan(table=[[consumers_1]])

1 liu jacky
2 liu doudou
3 liu maimai

Process finished with exit code 0
```


可以通过执行计划看到 `LogicalTableScan` 由 `consumers`
```
4:LogicalProject(id=[$0], firstname=[$1], lastname=[$2])
  3:LogicalTableScan(table=[[consumers]])
```
转变到了 `consumers_1`
```
6:LogicalProject(id=[$0], firstname=[$1], lastname=[$2])
  8:LogicalTableScan(table=[[consumers_1]])
```
查询出来的数据，也证明了执行计划的变更。


## SqlNode RelNode RexNode

首先我们补充一下，对SqlNode、RelNode、RexNode的理解

![](./img/blog/1004/ast2logicplan.png)

1. SqlNode 是 Parse、Validate 阶段的结果，对应 SQL 转换为语法树后的每个节点，例如 SqlSelect SqlJoin.
1. RelNode 是 SqlToRelConverter、Optimize 阶段的结果，对应语法树转换为关系运算符的节点，例如 LogicalProject LogicalJoin，这些节点操作的都是集合，是关系代数运算符的一种，即 relational expression.
1. RexNode 跟 RelNode 位于同一阶段，操作的是数据本身，例如limit 5里的 5 是RexLiteral，b.publish_year > 1830、b.author_id = a.id都是RexCall，对应常量、函数的表达式，即 Expression Node.

```
SqlNode is the abstract syntax tree that represents the actual structure of the query a user input. When a query is first parsed, it's parsed into a SqlNode. For example, a SELECT query will be parsed into a SqlSelect with a list of fields, a table, a join, etc. Calcite is also capable of generating a query string from a SqlNode as well.

RelNode represents a relational expression - hence "rel." RelNodes are used in the optimizer to decide how to execute a query. Examples of relational expressions are join, filter, aggregate, etc. Typically, specific implementations of RelNode will be created by users of Calcite to represent the execution of some expression in their system. When a query is first converted from SqlNode to RelNode, it will be made up of logical nodes like LogicalProject, LogicalJoin, etc. Optimizer rules are then used to convert from those logical nodes to physical ones like JdbcJoin, SparkJoin, CassandraJoin, or whatever the system requires. Traits and conventions are used by the optimizer to determine the set of rules to apply and the desired outcome, but you didn't ask about conventions :-)

RexNode represents a row expression - hence "Rex" - that's typically contained within a RelNode. The row expression contains operations performed on a single row. For example, a Project will contain a list of RexNodes that represent the projection's fields. A RexNode might be a reference to a field from an input to the RedNode, a function call (RexCall), a window (RexOver), etc. The operator within the RexCall defines what the node does, and operands define arguments to the operator. For example, 1 + 1 would be represented as a RexCall where the operator is + and the operands are 1 and 1.

What RelNode and RexNode together give you is a way to plan and implement a query. Systems typically use VolcanoPlanner (cost-based) or HepPlanner (heuristic) to convert the logical plan (RelNode) into a physical plan and then implement the plan by converting each RelNode and RexNode into whatever is required by the system for which the plan is generated. The optimizer might push down or pull up expressions or reorder joins to create a more optimal plan. Then, to implement that plan, e.g. the JDBC integration will convert an optimized RelNode back into a SqlNode and then a query string to be executed against some database over JDBC. The Spark implementation will convert RelNodes into operations on a Spark RDD. The Cassandra implementation will generate a CQL query, etc.
```


## 参考连接

https://lists.apache.org/thread/z3pvzy1fnl6t5m04gd3wv4tntwpf3g52

https://izualzhy.cn/calcite-example

http://matt33.com/2019/03/17/apache-calcite-planner/

https://zhuanlan.zhihu.com/p/61661909

