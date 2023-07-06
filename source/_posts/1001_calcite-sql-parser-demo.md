---
title: 手把手教你使用calcite解析SQL
date: 2023-06-22 17:12:17
tags: [calcite,编译原理,database]
---

大家好，我又腆着大脸来更新了，也是知道自己鸽了很久很久，也就不找说辞了，尽管确实是有点遭不住996了。还是恭祝大家端午安康吧。对于最近越发火爆的`calcite`,作为一个18年就开始翻译官方文档的作者来说，真是很欣慰，大家都用他来干什么呢？很多同学都是想用他来重写SQL，或者做优化。反正无论如何，还是需要对他有一定的了解，今天我来和大家分享一下，如何从代码端来解析SQL。
<!-- more -->

## 添加依赖
我们使用csv来做数据源
```xml
    <dependency>
      <groupId>org.apache.calcite</groupId>
      <artifactId>calcite-core</artifactId>
      <version>1.34.0</version>
    </dependency>

    <dependency>
      <groupId>org.apache.calcite</groupId>
      <artifactId>calcite-example-csv</artifactId>
      <version>1.21.0</version>
    </dependency>
```

## 构建元数据
这里我们使用2个csv来模拟2张数据表，订单表和消费者表
orders.csv
```
id,goods,price,amount,user_id
1,book,100,100,1
```

consumers.csv
```
id,firstname,lastname,birth
1,li,jacky,1984
2,li,doudou,2019
3,li,maimai,2019
```

## 加载元数据

```java
        SchemaPlus rootSchema = Frameworks.createRootSchema(true);
        String csvPath = "src\\main\\resources\\db";
        CsvSchema csvSchema = new CsvSchema(new File(csvPath), CsvTable.Flavor.SCANNABLE);
        rootSchema.add("orders", csvSchema.getTable("orders"));
        rootSchema.add("consumers", csvSchema.getTable("consumers"));
```

## 定义sql
```sql
SELECT o.id, o.goods, o.price, o.amount, c.firstname, c.lastname FROM orders AS o LEFT OUTER JOIN consumers c ON o.user_id = c.id WHERE o.amount > 30 ORDER BY o.id LIMIT 5
```

## 验证SQL
```java
        JavaTypeFactoryImpl sqlTypeFactory = new JavaTypeFactoryImpl();
        Properties properties = new Properties();
        properties.setProperty(CalciteConnectionProperty.CASE_SENSITIVE.camelName(), "false");
        // reader 接收 schema，用于检测字段名、字段类型、表名等是否存在和一致
        CalciteCatalogReader catalogReader = new CalciteCatalogReader(
                CalciteSchema.from(rootSchema),
                CalciteSchema.from(rootSchema).path(null),
                sqlTypeFactory,
                new CalciteConnectionConfigImpl(properties));
        // 简单示例，大部分参数采用默认值即可
        SqlValidator validator = SqlValidatorUtil.newValidator(
                SqlStdOperatorTable.instance(),
                catalogReader,
                sqlTypeFactory,
                SqlValidator.Config.DEFAULT);
        // validate: SqlNode -> SqlNode
        SqlNode sqlNodeValidated = validator.validate(sqlNodeParsed);
```

## 打印逻辑计划与物理计划
```java
        RexBuilder rexBuilder = new RexBuilder(sqlTypeFactory);
        HepProgramBuilder hepProgramBuilder = new HepProgramBuilder();
        hepProgramBuilder.addRuleInstance(CoreRules.FILTER_INTO_JOIN);

        HepPlanner hepPlanner = new HepPlanner(hepProgramBuilder.build());
        hepPlanner.addRelTraitDef(ConventionTraitDef.INSTANCE);

        RelOptCluster relOptCluster = RelOptCluster.create(hepPlanner, rexBuilder);
        SqlToRelConverter sqlToRelConverter = new SqlToRelConverter(
                // 没有使用 view
                new RelOptTable.ViewExpander() {
                    @Override
                    public RelRoot expandView(RelDataType rowType, String queryString, List<String> schemaPath,  List<String> viewPath) {
                        return null;
                    }
                },
                validator,
                catalogReader,
                relOptCluster,
        // 均使用标准定义即可
        StandardConvertletTable.INSTANCE,
                SqlToRelConverter.config());
        RelRoot logicalPlan = sqlToRelConverter.convertQuery(sqlNodeValidated, false, true);

        System.out.println();
        System.out.println(RelOptUtil.dumpPlan("[Logical plan]", logicalPlan.rel, SqlExplainFormat.TEXT, SqlExplainLevel.NON_COST_ATTRIBUTES));

        hepPlanner.setRoot(logicalPlan.rel);
        RelNode phyPlan = hepPlanner.findBestExp();
        System.out.println(RelOptUtil.dumpPlan("[Physical plan]", phyPlan, SqlExplainFormat.TEXT, SqlExplainLevel.NON_COST_ATTRIBUTES));
```


## DEMO及执行结果

### 代码文件`CalciteRelCase.java`
```java
package com.dafei1288;


import org.apache.calcite.adapter.csv.*;
import org.apache.calcite.config.CalciteConnectionConfigImpl;
import org.apache.calcite.config.CalciteConnectionProperty;
import org.apache.calcite.jdbc.CalciteSchema;
import org.apache.calcite.jdbc.JavaTypeFactoryImpl;
import org.apache.calcite.plan.ConventionTraitDef;
import org.apache.calcite.plan.RelOptCluster;
import org.apache.calcite.plan.RelOptTable;
import org.apache.calcite.plan.RelOptUtil;
import org.apache.calcite.plan.hep.HepPlanner;
import org.apache.calcite.plan.hep.HepProgramBuilder;
import org.apache.calcite.prepare.CalciteCatalogReader;
import org.apache.calcite.rel.RelNode;
import org.apache.calcite.rel.RelRoot;
import org.apache.calcite.rel.rules.CoreRules;
import org.apache.calcite.rel.type.*;
import org.apache.calcite.rex.RexBuilder;
import org.apache.calcite.schema.SchemaPlus;
import org.apache.calcite.sql.*;
import org.apache.calcite.sql.fun.SqlStdOperatorTable;
import org.apache.calcite.sql.parser.SqlParser;
import org.apache.calcite.sql.validate.SqlValidator;
import org.apache.calcite.sql.validate.SqlValidatorUtil;
import org.apache.calcite.sql2rel.SqlToRelConverter;
import org.apache.calcite.sql2rel.StandardConvertletTable;
import org.apache.calcite.tools.Frameworks;


import java.io.File;
import java.util.List;
import java.util.Properties;

/**
 * CalciteRelCase
 *
 */
public class CalciteRelCase {
    public static void main( String[] args ) throws Exception{

        // Convert query to SqlNode
        String sql = "SELECT o.id, o.goods, o.price, o.amount, c.firstname, c.lastname FROM orders AS o LEFT OUTER JOIN consumers c ON o.user_id = c.id WHERE o.amount > 30 ORDER BY o.id LIMIT 5";
        SqlParser.Config config = SqlParser.configBuilder().setCaseSensitive(false).build();
        SqlParser parser = SqlParser.create(sql, config);

        SqlNode sqlNodeParsed = parser.parseQuery();
        System.out.println("[parsed sqlNode]");
        System.out.println(sqlNodeParsed);

        SchemaPlus rootSchema = Frameworks.createRootSchema(true);
        String csvPath = "src\\main\\resources\\db";
        CsvSchema csvSchema = new CsvSchema(new File(csvPath), CsvTable.Flavor.SCANNABLE);


//        CsvTableFactory csvTableFactory = new CsvTableFactory();
//        Map<String,Object> operand = new HashMap<>();
//        operand.put("file",authorPath);
//        operand.put("flavor", "scannable");
//        CsvTable aC = csvTableFactory.create(rootSchema,"author",operand,null);
//        CsvTableFactory csvTableFactoryB = new CsvTableFactory();
//        Map<String,Object> operandB = new HashMap<>();
//        operandB.put("file",bookPath);
//        operandB.put("flavor", "scannable");
//        CsvTable bC = csvTableFactoryB.create(rootSchema,"book",operandB,null);


        rootSchema.add("orders", csvSchema.getTable("orders"));
        rootSchema.add("consumers", csvSchema.getTable("consumers"));

        JavaTypeFactoryImpl sqlTypeFactory = new JavaTypeFactoryImpl();
        Properties properties = new Properties();
        properties.setProperty(CalciteConnectionProperty.CASE_SENSITIVE.camelName(), "false");
        // reader 接收 schema，用于检测字段名、字段类型、表名等是否存在和一致
        CalciteCatalogReader catalogReader = new CalciteCatalogReader(
                CalciteSchema.from(rootSchema),
                CalciteSchema.from(rootSchema).path(null),
                sqlTypeFactory,
                new CalciteConnectionConfigImpl(properties));
        // 简单示例，大部分参数采用默认值即可
        SqlValidator validator = SqlValidatorUtil.newValidator(
                SqlStdOperatorTable.instance(),
                catalogReader,
                sqlTypeFactory,
                SqlValidator.Config.DEFAULT);
        // validate: SqlNode -> SqlNode
        SqlNode sqlNodeValidated = validator.validate(sqlNodeParsed);
        System.out.println();
        System.out.println("[validated sqlNode]");
        System.out.println(sqlNodeValidated);




        RexBuilder rexBuilder = new RexBuilder(sqlTypeFactory);
        HepProgramBuilder hepProgramBuilder = new HepProgramBuilder();
        hepProgramBuilder.addRuleInstance(CoreRules.FILTER_INTO_JOIN);

        HepPlanner hepPlanner = new HepPlanner(hepProgramBuilder.build());
        hepPlanner.addRelTraitDef(ConventionTraitDef.INSTANCE);

        RelOptCluster relOptCluster = RelOptCluster.create(hepPlanner, rexBuilder);
        SqlToRelConverter sqlToRelConverter = new SqlToRelConverter(
                // 没有使用 view
                new RelOptTable.ViewExpander() {
                    @Override
                    public RelRoot expandView(RelDataType rowType, String queryString, List<String> schemaPath,  List<String> viewPath) {
                        return null;
                    }
                },
                validator,
                catalogReader,
                relOptCluster,
        // 均使用标准定义即可
        StandardConvertletTable.INSTANCE,
                SqlToRelConverter.config());
        RelRoot logicalPlan = sqlToRelConverter.convertQuery(sqlNodeValidated, false, true);

        System.out.println();
        System.out.println(RelOptUtil.dumpPlan("[Logical plan]", logicalPlan.rel, SqlExplainFormat.TEXT, SqlExplainLevel.NON_COST_ATTRIBUTES));





        hepPlanner.setRoot(logicalPlan.rel);
        RelNode phyPlan = hepPlanner.findBestExp();
        System.out.println(RelOptUtil.dumpPlan("[Physical plan]", phyPlan, SqlExplainFormat.TEXT, SqlExplainLevel.NON_COST_ATTRIBUTES));


    }
}

```

### 执行结果
```
[parsed sqlNode]
SELECT `O`.`ID`, `O`.`GOODS`, `O`.`PRICE`, `O`.`AMOUNT`, `C`.`FIRSTNAME`, `C`.`LASTNAME`
FROM `ORDERS` AS `O`
LEFT JOIN `CONSUMERS` AS `C` ON `O`.`USER_ID` = `C`.`ID`
WHERE `O`.`AMOUNT` > 30
ORDER BY `O`.`ID`
FETCH NEXT 5 ROWS ONLY

[validated sqlNode]
SELECT `O`.`ID`, `O`.`GOODS`, `O`.`PRICE`, `O`.`AMOUNT`, `C`.`FIRSTNAME`, `C`.`LASTNAME`
FROM `ORDERS` AS `O`
LEFT JOIN `CONSUMERS` AS `C` ON `O`.`user_id` = `C`.`id`
WHERE CAST(`O`.`amount` AS INTEGER) > 30
ORDER BY `O`.`id`
FETCH NEXT 5 ROWS ONLY

[Logical plan]
LogicalSort(sort0=[$0], dir0=[ASC], fetch=[5]), id = 10
  LogicalProject(ID=[$0], GOODS=[$1], PRICE=[$2], AMOUNT=[$3], FIRSTNAME=[$6], LASTNAME=[$7]), id = 9
    LogicalFilter(condition=[>(CAST($3):INTEGER NOT NULL, 30)]), id = 6
      LogicalJoin(condition=[=($4, $5)], joinType=[left]), id = 5
        LogicalTableScan(table=[[orders]]), id = 1
        LogicalTableScan(table=[[consumers]]), id = 3

[Physical plan]
LogicalSort(sort0=[$0], dir0=[ASC], fetch=[5]), id = 19
  LogicalProject(ID=[$0], GOODS=[$1], PRICE=[$2], AMOUNT=[$3], FIRSTNAME=[$6], LASTNAME=[$7]), id = 17
    LogicalJoin(condition=[=($4, $5)], joinType=[left]), id = 24
      LogicalFilter(condition=[>(CAST($3):INTEGER NOT NULL, 30)]), id = 21
        LogicalTableScan(table=[[orders]]), id = 1
      LogicalTableScan(table=[[consumers]]), id = 3

```

好了，今天就分享到这了，接下会对`calcite`做更多解读，欢迎大家订阅，转发，谢了!!!