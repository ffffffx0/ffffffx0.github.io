---
title: Hive源码流程分析
tagline: ""
category : [Java, Hive]
layout: post
tags : [Java, Hive]
---

#### 主要流程

词法分析，生成AST树，ParseDriver完成。   
分析AST树，AST拆分成查询子块，信息记录在QB，这个QB在下面几个阶段都需要用到，SemanticAnalyzer.doPhase1完成。   
从metastore中获取表的信息，SemanticAnalyzer.getMetaData完成。   
生成逻辑执行计划，SemanticAnalyzer.genPlan完成。   
优化逻辑执行计划，Optimizer完成，ParseContext作为上下文信息进行传递。   
生成物理执行计划，SemanticAnalyzer.generateTaskTree完成。   
物理计划优化，PhysicalOptimizer完成，PhysicalContext作为上下文信息进行传递


#### Parse:  解析sql生成ASTNode

```
Driver.run()->Driver.runInternal()-->Driver.compileInternal()-->Driver.compile()-->ParseUtils.parse()
-->ParseDriver.parse()-->

```
生成ASTNode

```
//Driver.java
ASTNode tree;
try {
  tree = ParseUtils.parse(command, ctx);//  1，parse sql  to ASTNode
} catch (ParseException e) {}
//ParseUtils.java
/** Parses the Hive query. */
public static ASTNode parse(
    String command, Context ctx, String viewFullyQualifiedName) throws ParseException {
  ParseDriver pd = new ParseDriver();
  ASTNode tree = pd.parse(command, ctx, viewFullyQualifiedName);
  tree = findRootNonNullToken(tree);
  handleSetColRefs(tree);
  return tree;
}
```
ParseDriver.parse代码(省略部分代码)

```
public ASTNode parse(String command, Context ctx, String viewFullyQualifiedName){
  HiveLexerX lexer = new HiveLexerX(new ANTLRNoCaseStringStream(command));
  TokenRewriteStream tokens = new TokenRewriteStream(lexer);
  HiveParser parser = new HiveParser(tokens);
  
  if (ctx != null) {
  parser.setHiveConf(ctx.getConf());
  }
  parser.setTreeAdaptor(adaptor);
  
  HiveParser.statement_return r = null;
  try {
  r = parser.statement();
  } catch (RecognitionException e) {}
  
  ASTNode tree = (ASTNode) r.getTree();
  tree.setUnknownTokenBoundaries();
  return tree;
}
```
HiveParser的代码太长了，暂且不看

#### 抽象语法树(AST)变成一个QB（Query Block） 

```
//Driver.java
BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(queryState, tree);

if (!retrial) {
  openTransaction();
  generateValidTxnList();
}

sem.analyze(tree, ctx);// 2,BaseSemanticAnalyzer.analyze

//BaseSemanticAnalyzer.java
public void analyze(ASTNode ast, Context ctx) throws SemanticException {
  initCtx(ctx);
  init(true);
  analyzeInternal(ast);
}
//BaseSemanticAnalyzer.java
public void init(boolean clearPartsCache) {
  // clear most members
  reset(clearPartsCache);

  // init
  QB qb = new QB(null, null, false);
  this.qb = qb;
}
```

#### 生成逻辑执行计划(Operator)

```
Operator sinkOp = genOPTree(ast, plannerCtx);

Operator genOPTree(ASTNode ast, PlannerContext plannerCtx) throws SemanticException {
  // fetch all the hints in qb
  List<ASTNode> hintsList = new ArrayList<>();
  getHintsFromQB(qb, hintsList);
  getQB().getParseInfo().setHintList(hintsList);
  return genPlan(qb);
}
public Operator genPlan(QB qb) throws SemanticException {
  return genPlan(qb, false);
}

public Operator genPlan(QB qb, boolean skipAmbiguityCheck){
...
Operator<?> operator = genPlan(qb, qbexpr);
...
}
```

#### 逻辑执行计划 Optimizer

```
Optimizer optm = new Optimizer();
optm.setPctx(pCtx);
optm.initialize(conf);
pCtx = optm.optimize();
```
Optimizer.initialize()初始化添加各种优化器

```
public void initialize(HiveConf hiveConf) {
  ...
  transformations = new ArrayList<Transform>();
  transformations.add(new GroupByOptimizer());
  transformations.add(new SkewJoinOptimizer());
  ...
}
```
Optimizer.optimize()  各优化组件优化操作

```
public ParseContext optimize() throws SemanticException {
  for (Transform t : transformations) {
    t.beginPerfLogging();
    pctx = t.transform(pctx);
    t.endPerfLogging(t.toString());
  }
  return pctx;
}
```

#### 生成物理执行计划

```
// 9. Optimize Physical op tree & Translate to target execution engine (MR,
// TEZ..)
if (!ctx.getExplainLogical()) {
  TaskCompiler compiler = TaskCompilerFactory.getCompiler(conf, pCtx);
  compiler.init(queryState, console, db);
  compiler.compile(pCtx, rootTasks, inputs, outputs);
  fetchTask = pCtx.getFetchTask();
}
```
这里的TaskCompiler根据引擎不同可以是MR,Tez,Spark,TaskCompilerFactory的实现

```
public static TaskCompiler getCompiler(HiveConf conf, ParseContext parseContext) {
  if (HiveConf.getVar(conf, HiveConf.ConfVars.HIVE_EXECUTION_ENGINE).equals("tez")) {
    return new TezCompiler();
  } else if (HiveConf.getVar(conf, HiveConf.ConfVars.HIVE_EXECUTION_ENGINE).equals("spark")) {
    return new SparkCompiler();
  } else {
    return new MapReduceCompiler();
  }
}
```

TaskCompiler.compile的主要流程

```
optimizeOperatorPlan(pCtx, inputs, outputs); 再次优化op?
generateTaskTree(rootTasks, pCtx, mvTask, inputs, outputs);生成物理计划
optimizeTaskPlan(rootTasks, pCtx, ctx);优化物理计划


```

####  执行task 

```
//Driver.execute()
TaskRunner runner = launchTask(task, queryId, noName, jobname, jobs, driverCxt);
```
