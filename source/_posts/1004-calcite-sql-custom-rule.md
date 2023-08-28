---
title: 如何使用calcite rule做SQL重写（下）
date: 2023-08-18 15:10:43
tags: [calcite,编译原理,database]
---


## 准备数据

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



        for(String tn : call.rel(0).getTable().getQualifiedName()){
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