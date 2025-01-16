---
title: "Calcite支持benchmark函数"
date: 2024-11-29T11:38:14+08:00
draft: false
---
# 需求
想支持下 https://dev.mysql.com/doc/refman/8.4/en/information-functions.html#function_benchmark

```sql
mysql> SELECT BENCHMARK(1000000,AES_ENCRYPT('hello','goodbye'));
+---------------------------------------------------+
| BENCHMARK(1000000,AES_ENCRYPT('hello','goodbye')) |
+---------------------------------------------------+
|                                                 0 |
+---------------------------------------------------+
1 row in set (4.74 sec)
```

得先了解下嵌套函数的实现

又折腾了一周 polardbx 的启动，终于能启动了。

试一下 它是否能支持。
嗯，暂不支持，不过错误提示的挺好的。

```
mysql> SELECT BENCHMARK(1000000,AES_ENCRYPT('hello','goodbye'));
ERROR 4998 (HY000): [1900df69ae400000][192.168.200.77:8527][sharding_db]ERR-CODE: [PXC-4998][ERR_NOT_SUPPORT] function BENCHMARK not support yet!
```

接下来做些啥呢？ 看看嵌套函数咋实现的。

这个嵌套函数可以实现

```
mysql> select GREATEST(max(1),min(2));
+--------------------------+
| GREATEST(max(1), min(2)) |
+--------------------------+
|                        2 |
+--------------------------+
1 row in set (0.18 sec)
```

# 优化逻辑

经过解析和 validate. polardb-x 会通过 coverter 将 sql 转化成 RelNode

```
rel#503:LogicalProject.NONE.[].any.[](input=LogicalAggregate#502,GREATEST(max(1), min(2))=GREATEST($0, $1))
```
## RBO 优化

### HepPlanner 

添加很多变换规则。

setRoot() 时会创建 graph

可以看到创建了edge（边）和对应的 vertex . 边记录了这些 vertex 的联系。

![](/img/065EF8B788F579A5F5E6B24ACAC076C0.png)

executeProgram()

循环各个vertex 并应用所有规则

这个case 没有找到应用的规则，看起来是应用规则，并 transform vertex

最终构建出

```
rel#513:LogicalProject.NONE.[].any.[](input=LogicalAggregate#511,GREATEST(max(1), min(2))=GREATEST($0, $1))
```

## CBO 优化

addCBORules 根据配置添加规则

### VolcanoPlanner

setRoot()
- 调用 registerImpl() 并且计算代价保存到 relsubset ，并且筛选规则

drive()
- 应用规则
当前这个case 走的是 TopDownRuleDriver 

task.perform() 的过程中会不断产生 task。中间使用规则时，会进行 transform ，并且重新计算 cost。

我的粗浅理解就是使用规则再算一次 cost,从而方便找到最小代价的执行计划。
```java
@Override public void drive() {
    TaskDescriptor description = new TaskDescriptor();

    // Starting from the root's OptimizeGroup task.
    tasks.push(
        new OptimizeGroup(
            requireNonNull(planner.root, "planner.root"),
            planner.infCost));

    // Ensure materialized view roots get explored.
    // Note that implementation rules or enforcement rules are not applied
    // unless the mv is matched.
    exploreMaterializationRoots();

    try {
      // Iterates until the root is fully optimized.
      while (!tasks.isEmpty()) {
        Task task = tasks.pop();
        description.log(task);
        task.perform();
      }
    } catch (VolcanoTimeoutException ex) {
      LOGGER.warn("Volcano planning times out, cancels the subsequent optimization.");
    }
  }
```

CBO 优化之后的结果
```
rel#534:PhysicalProject.DRDS.[].any.[](input=HashAgg#533,GREATEST(max(1), min(2))=GREATEST($0, $1))
```

# 执行逻辑

遍历 pysicalPlan 根据节点的不同，产生不同的执行器。

LocalExecutionPlanner.plan() 方法会遍历节点生成 PipelineDepTree

这里存着调用顺序已经相互依赖的关系

通过
![](/img/60B29B578274C8DC33D11C180AD86E3F.png)

TaskExecutor.enqueueSplits()   开始实际调度执行

调用之后，内外部如何传递结果暂时还没找到在哪儿搞的

# 回到如何实现 benchmark 上来

select concat(1, max(10));

```
rel#741:PhysicalProject.DRDS.[].any.[](input=HashAgg#739,concat(1, max(10))=CONCAT(?0, $0))
```

## 先把 benchmark 当成 concat 来实现

搜索 concat 相关关键词，然后仿照着增加 benchmark。

哈哈，走通了

```
No connection. Trying to reconnect...
Connection id:    2
Current database: sharding_db

+-----------------------------------------------------+
| BENCHMARK(1000000, AES_ENCRYPT('hello', 'goodbye')) |
+-----------------------------------------------------+
|                                                   0 |
+-----------------------------------------------------+
```

接下来就是要调整下逻辑适配一下

```
rel#523:PhysicalProject.DRDS.[].any.[](input=LogicalValues#514,BENCHMARK(1000000, AES_ENCRYPT('hello', 'goodbye'))=BENCHMARK(?0, AES_ENCRYPT(?1, ?2)))
```

现在这个执行逻辑是先执行 AES_ENCRYPT 再将结果放到 BENCHMARK 方法中去执行。

但是我需要在 BENCHMARK 中执行 AES_ENCRYPT。

再找找已有的实现，嵌套后执行的。

找了一圈也没有找到已有的实现方式，应该是一个回调实现，感觉跟执行顺序有关。

debug 发现如下逻辑, 它会遍历内部的参数，如果是方法的话，会调用计算结果并返回。

所以考虑根据 benchmark 生成特定的执行类，而不是当前的这个，从而控制流程。

```java
public class ScalarFunctionExpression extends AbstractExpression {

    private List<IExpression> args;
    private IScalarFunction function;
    private ExprContextProvider contextHolder;

    public ScalarFunctionExpression() {
    }

    public List<IExpression> getArgs() {
        return args;
    }

    @Override
    public Object eval(Row row, ExecutionContext ec) {

        if (function != null) {
            Object[] actualArgs = new Object[args.size()];
            for (int i = 0; i < args.size(); i++) {
                actualArgs[i] = args.get(i).eval(row, ec);
            }
//            function.setArgs(Arrays.asList(actualArgs));
            Object result = function.compute(actualArgs, ec);
            DataType returnType = function.getReturnType();
            return returnType.convertFrom(result);
        } else {
            GeneralUtil.nestedException("invoke function of null");
        }
        return null;
    }
```

## 调整逻辑, 成功实现

修改 compute 逻辑，将 callback 类型的不计算，直接传入

![](/img/9B098C78A90DB5C31F9E1AE24FD9ADB2.png)

修改 benchmark 逻辑
```java
public class Benchmark extends AbstractCallBackScalarFunction {
    public Benchmark(List<DataType> operandTypes, DataType resultType) {
        super(operandTypes, resultType);
    }

    @Override
    public String[] getFunctionNames() {
        return new String[]{"BENCHMARK"};
    }

    @Override
    public Object compute(Object[] args, ExecutionContext ec) {

        if (args[1] instanceof ScalarFunctionExpression && args[0] instanceof Number) {
            for (int i = 0; i < ((Number) args[0]).intValue(); i++) {
                ((ScalarFunctionExpression) args[1]).eval(null, ec);
            }
            // TODO support args[1] is select
        }
        return 0;
    }
}
```

## 考虑 select 的实现

先尝试下这个查询

```sql
SELECT BENCHMARK(1000000, (SELECT 1));
```

```sql
SELECT BENCHMARK(2, (SELECT id from sbtest_sharding_c limit 1));
```
发现这两个查询都可以正常执行，但是很奇怪断点没有走到 benchmark 函数中。

```
rel#1300:PhysicalProject.DRDS.[].any.[](input=LogicalCorrelate#1298,BENCHMARK(3, ( SELECT id FROM sbtest_sharding_c LIMIT 1 ))=BENCHMARK(?0, $1),variablesSet=[$cor0])
```

看起来没走到是因为 benchmark 的参数类型跟函数实际的不匹配。

benchmark 应该也是属于是流程控制类函数。

## polarDB-X benchmark 的实现
我去，居然发现了 polardbx 的 benchmark 有一些实现

```
BenchmarkVectorizedExpression
```
现在倒是走通了，但是好像是歪路啊。

接下来应该是把我的修改给去掉一部分。然后实现debug 看它原来支持的 benchmark.

我把改动都去除之后，发现这个 sql 是可以走通的，那么就要探究下流程，然后看看把 BENCHMARK(2, AES_ENCRYPT('hello','goodbye')) 走通。

```sql
SELECT BENCHMARK(2, (SELECT id from sbtest_sharding_c limit 1));
```

```
rel#651:PhysicalProject.DRDS.[].any.[](input=LogicalCorrelate#649,BENCHMARK(3, ( SELECT id FROM sbtest_sharding_c LIMIT 1 ))=BENCHMARK(?0, $1),variablesSet=[$cor0])
```

为什么 debug 发现虽然走到了 eval， 但是在内部都没有实际进行计算，很奇怪
```java
public void eval(EvaluationContext ctx) {
        MutableChunk chunk = ctx.getPreAllocatedChunk();
        int batchSize = chunk.batchSize();
        boolean isSelectionInUse = chunk.isSelectionInUse();
        int[] sel = chunk.selection();

        // benchmark
        //   /     \
        // count  function
        VectorizedExpression child = children[1];
        for (int i = 0; i < count; i++) {
            // try count times
            child.eval(ctx);
        }

        RandomAccessBlock outputVectorSlot = chunk.slotIn(outputIndex, outputDataType);
        long[] res = (outputVectorSlot.cast(LongBlock.class)).longArray();

        if (isSelectionInUse) {
            for (int i = 0; i < batchSize; i++) {
                int j = sel[i];
                res[j] = 0L;
            }
        } else {
            for (int i = 0; i < batchSize; i++) {
                res[i] = 0L;
            }
        }
    }
```

```java
public class InputRefVectorizedExpression extends AbstractVectorizedExpression {
    private final int inputIndex;

    public InputRefVectorizedExpression(DataType<?> dataType, int inputIndex, int outputIndex) {
        super(dataType, outputIndex, new VectorizedExpression[0]);
        this.inputIndex = inputIndex;
    }

    @Override
    public void eval(EvaluationContext ctx) {
        // 总是相等的
        if (outputIndex != inputIndex) {
            MutableChunk chunk = ctx.getPreAllocatedChunk();
            boolean isSelectionInUse = chunk.isSelectionInUse();
            int[] selection = chunk.selection();
            chunk.slotIn(outputIndex).copySelected(isSelectionInUse, selection, chunk.batchSize(),
                chunk.slotIn(inputIndex));
        }
    }
}
```

看起来是架子搭好了，但是 children 没有正确的传入，另外，benchmark(3,AES_ENCRYPT('HELLO','WORLD')) 也没有走到这个场景。

目前还没具体方法，未知的太多，例如 vectoriedExpression 是否支持 select 等。

https://github.com/polardb/polardbx-sql/issues/228 我提了个issue，期待大佬回答指导下。

先看一下 concat 函数的实现逻辑

mysql> SELECT CONCAT(3, (SELECT id from sbtest_sharding_id limit 1));
+----------------------------------------------------------+
| CONCAT(3, ( SELECT id FROM sbtest_sharding_id LIMIT 1 )) |
+----------------------------------------------------------+
| 3100001                                                  |
+----------------------------------------------------------+
1 row in set (8.99 sec)

看执行流程，似乎是 SELECT id from sbtest_sharding_id limit 1 被作为子查询，查询后再进行 concat 计算了。

```
rel#1069:PhysicalProject.DRDS.[].any.[](input=LogicalCorrelate#1067,CONCAT(4, ( SELECT id FROM sbtest_sharding_id LIMIT 1 ))=CONCAT(?0, $1),variablesSet=[$cor0])
```

看起来这个和 concat 没啥差别呀

那么它为什么会走到子查询逻辑，而 benchmark 不行，需要探究下这点。

### benchmark 和 concat 现在逻辑有何区别

```sql
SELECT CONCAT(3, (SELECT id from sbtest_sharding_id limit 1));

SELECT BENCHMARK(3, (SELECT id from sbtest_sharding_id limit 1));
```

所以 benchmark 应该是要封装多次调用查询才行。

但是benchmark 也是跟concat 一样，先调用了 SELECT id from sbtest_sharding_id limit 1 并获取结果。

所以想要支持，核心是让他能多次调用。

也就是 VectorizedProjectExec 类，需要看是否还是使用这个类来执行才行。需要调整执行顺序。

VectorizedProjectExec 内部封装了 CorrelateExec 执行类。

所以需要着重看一下，这些执行器是如何构建的。

看起来执行器的构建是根据优化的结果来决定的。

群里问了些大佬，感觉还是需要从执行器这边来支持。

又看了下代码，如果想支持，就不要用 VectorizedProjectExec,例如下面这里，input 使用的 CorrelateExecutor, input.nextChunk() 
就会将 benchmark 中的内容计算并返回。

所以新的执行器应该是先进行 evaluate，然后在计算过程中，使用类似 CorrelateExec 来进行子查询，但是需要循环执行

```java
@Override
Chunk doNextChunk() {
    Chunk inputChunk = input.nextChunk();
    if (inputChunk == null) {
        return null;
    }

    for (int i = 0; i < expressions.size(); i++) {
        if (mappedColumnIndex[i] == -1) {
            evaluateExpression(i, inputChunk);
        }
    }
```

```java
@Override
Chunk doNextChunk() {
    if (closed || isFinish) {
        forceClose();
        return null;
    }
    if (currentChunk == null) {
        // read next chunk from input
        currentChunk = left.nextChunk();
        if (currentChunk == null) {
            isFinish = left.produceIsFinished();
            blocked = left.produceIsBlocked();
            return null;
        }
        applyRowIndex = 0;
    }

    if (hasConstantValue) {
        for (; applyRowIndex < currentChunk.getPositionCount(); applyRowIndex++) {
            // apply of applyRowIndex's row is finished, compute result
            for (int i = 0; i < left.getDataTypes().size(); i++) {
                currentChunk.getBlock(i).writePositionTo(applyRowIndex, blockBuilders[i]);
            }
            if (constantValue == RexDynamicParam.DYNAMIC_SPECIAL_VALUE.EMPTY) {
                blockBuilders[getDataTypes().size() - 1].writeObject(null);
            } else {
                blockBuilders[getDataTypes().size() - 1].writeObject(outColumnType.convertFrom(constantValue));
            }
        }
        currentChunk = null;
        return buildChunkAndReset();
    }

    // 这里就是查询逻辑，也就是说修改后，是需要多次执行的
    if (curSubqueryApply == null) {
        curSubqueryApply = createSubqueryApply(applyRowIndex);
        curSubqueryApply.prepare();
    }
    curSubqueryApply.process();
    if (curSubqueryApply.isFinished()) {
        curSubqueryApply.close();

        if ((leftConditions == null || leftConditions.size() == 0) && isValueConstant) {
            constantValue = curSubqueryApply.getResultValue();
            hasConstantValue = true;
        }

        // apply of applyRowIndex's row is finished, compute result
        for (int i = 0; i < left.getDataTypes().size(); i++) {
            currentChunk.getBlock(i).writePositionTo(applyRowIndex, blockBuilders[i]);
        }

        if (curSubqueryApply.getResultValue() == RexDynamicParam.DYNAMIC_SPECIAL_VALUE.EMPTY) {
            blockBuilders[getDataTypes().size() - 1].writeObject(null);
        } else {
            blockBuilders[getDataTypes().size() - 1].writeObject(
                getDataTypes().get(getDataTypes().size() - 1)
                    .convertFrom(curSubqueryApply.getResultValue()));
        }

        curSubqueryApply = null;
        if (++applyRowIndex == currentChunk.getPositionCount()) {
            applyRowIndex = 0;
            currentChunk = null;
            return buildChunkAndReset();
        }
    } else {
        blocked = curSubqueryApply.isBlocked();
    }
    return null;
}
```

# 结论
基于以上分析，要支持，可以考虑动优化的结果，不使用如下的内容，不过应该生成成哪种形式，还需要考虑，如果直接在计划树上，封装多次引用关系，这样也不友好，可能循环 100次计划树就特别长了。
```
rel#651:PhysicalProject.DRDS.[].any.[](input=LogicalCorrelate#649,BENCHMARK(3, ( SELECT id FROM sbtest_sharding_c LIMIT 1 ))=BENCHMARK(?0, $1),variablesSet=[$cor0])
```

第二种方式，就是还是动执行器，写一个针对benchmark 的执行器。来支持，但是这种魔改的方式，从我现在比较片面的角度来改造，还不太成熟。

基于这种考虑，我还是暂时放弃 benchmark 的支持，转而去看看 calcite 社区的一些任务，让我能更加熟悉这个技术框架。
