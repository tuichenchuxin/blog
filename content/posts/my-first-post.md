---
title: "ShardingSphere Hint 实用指南"
date: 2022-04-01T19:33:02+08:00
draft: false
---

## 背景
Apache ShardingSphere 基于用户的实际使用场景，为用户打造了多种实用功能，包括数据分片、读写分离等。在数据分片功能中，Apache ShardingSphere 提供了标准分片、复合分片等多种实用的分片策略，在各种分片策略中，用户又可以配置相关分片算法，从而解决数据分片的问题。在读写分离功能中，ShardingSphere 为用户提供了静态和动态的两种读写分离类型以及丰富的负载均衡算法以满足用户实际需求。
可以看到 ShardingSphere 的分片和读写分离功能已经非常丰富，不过用户的真实使用场景是千变万化的。以多租户场景为例，用户期望按照登录账号所属租户进行分片，但是租户信息却并不是存在于每条业务 SQL 中，这时从 SQL 中提取分片字段的算法将无法发挥作用。再以读写分离为例，大部分场景下用户都希望能够将查询操作路由到从库上执行，但是在某些实时性要求很高的场景下，用户希望将 SQL 强制路由到主库执行，这时读写分离就无法满足业务要求。
基于以上痛点 Apache ShardingSphere 为用户提供了 Hint 功能，用户可以结合实际业务场景，利用 SQL 外部的逻辑进行强制路由或者分片。目前 ShardingSphere 为用户提供了两种 Hint 方式，一种通过 Java API 手动编程，利用 HintManager 进行强制路由和分片，这种方式对采用 JDBC 编程的应用非常友好，只需要少量的代码编写，就能够轻松实现不依赖 SQL 的分片或者强制路由功能。另外一种方式对于不懂开发的 DBA 而言更加友好，ShardingSphere 基于分布式 SQL 提供的使用方式，利用 SQL HINT 和 DistSQL HINT， 为用户提供了无需编码就能实现的分片和强制路由功能。接下来，让我们一起了解下这两种使用方式。
## 基于 HintManager 的手动编程
ShardingSphere 主要通过 HintManager 对象来实现强制路由和分片的功能。利用 HintManager，用户的分片将不用再依赖 SQL。它可以极大地扩展用户的使用场景，让用户可以更加灵活地进行数据分片或者强制路由。目前通过 HintManager，用户可以配合 ShardingSphere 内置的或者自定义的 Hint 算法实现分片功能，还可以通过设置指定数据源或者强制主库读写，实现强制路由功能。在学习 HintManager 的使用之前，让我们先来简单地了解一下它的实现原理，这有助于我们更好地使用它。
## HintManager 实现原理
其实通过查看 HintManager 代码，我们可以快速地了解它的原理。
``` java 
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class HintManager implements AutoCloseable {
    
    private static final ThreadLocal<HintManager> HINT_MANAGER_HOLDER = new ThreadLocal<>();
}
```
正如你所看到的，ShardingSphere 通过 ThreadLocal 来实现 HintManager 的功能，只要在同一个线程中，用户的分片设置都会得以保留。因此，只要用户在执行 SQL 之前调用 HintManager 相关功能，ShardingSphere 就能在当前线程中获取用户设置的分片或强制路由条件，从而进行分片或者路由操作。了解了 HintManager 的原理之后，让我们一起来学习一下它的使用。
## HintManager 的使用
### 使用 Hint 分片
Hint 分片算法需要用户实现 org.apache.shardingsphere.sharding.api.sharding.hint.HintShardingAlgorithm 接口。 Apache ShardingSphere 在进行路由时，将会从 HintManager 中获取分片值进行路由操作。
#### 参考配置如下：
```
rules:
- !SHARDING
  tables:
    t_order:
      actualDataNodes: demo_ds_${0..1}.t_order_${0..1}
      databaseStrategy:
        hint:
          algorithmClassName: xxx.xxx.xxx.HintXXXAlgorithm
      tableStrategy:
        hint:
          algorithmClassName: xxx.xxx.xxx.HintXXXAlgorithm
  defaultTableStrategy:
    none:
  defaultKeyGenerateStrategy:
    type: SNOWFLAKE
    column: order_id

props:
    sql-show: true
```

#### 获取 HintManager 实例
HintManager hintManager = HintManager.getInstance();

#### 添加分片键
- 使用 hintManager.addDatabaseShardingValue 来添加数据源分片键值。
- 使用 hintManager.addTableShardingValue 来添加表分片键值。
分库不分表情况下，强制路由至某一个分库时，可使用 hintManager.setDatabaseShardingValue 方式添加分片。
清除分片键值
分片键值保存在 ThreadLocal 中，所以需要在操作结束时调用 hintManager.close() 来清除 ThreadLocal 中的内容。
#### 完整代码示例
``` java
String sql = "SELECT * FROM t_order";
try (HintManager hintManager = HintManager.getInstance();
     Connection conn = dataSource.getConnection();
     PreparedStatement preparedStatement = conn.prepareStatement(sql)) {
    hintManager.addDatabaseShardingValue("t_order", 1);
    hintManager.addTableShardingValue("t_order", 2);
    try (ResultSet rs = preparedStatement.executeQuery()) {
        while (rs.next()) {
            // ...
        }
    }
}

String sql = "SELECT * FROM t_order";
try (HintManager hintManager = HintManager.getInstance();
     Connection conn = dataSource.getConnection();
     PreparedStatement preparedStatement = conn.prepareStatement(sql)) {
    hintManager.setDatabaseShardingValue(3);
    try (ResultSet rs = preparedStatement.executeQuery()) {
        while (rs.next()) {
            // ...
        }
    }
}
```

### 使用 Hint 强制主库路由
#### 获取 HintManager
与基于 Hint 的数据分片相同。
#### 设置主库路由
- 使用 hintManager.setWriteRouteOnly 设置主库路由。
#### 清除分片键值
与基于 Hint 的数据分片相同。
#### 完整代码示例
``` java
String sql = "SELECT * FROM t_order";
try (HintManager hintManager = HintManager.getInstance();
     Connection conn = dataSource.getConnection();
     PreparedStatement preparedStatement = conn.prepareStatement(sql)) {
    hintManager.setWriteRouteOnly();
    try (ResultSet rs = preparedStatement.executeQuery()) {
        while (rs.next()) {
            // ...
        }
    }
}
```

### 使用 Hint 路由至指定数据库
#### 获取 HintManager
与基于 Hint 的数据分片相同。
#### 设置路由至指定数据库
- 使用 hintManager.setDataSourceName 设置数据库名称。
#### 完整代码示例
``` java 
String sql = "SELECT * FROM t_order";
try (HintManager hintManager = HintManager.getInstance();
     Connection conn = dataSource.getConnection();
     PreparedStatement preparedStatement = conn.prepareStatement(sql)) {
    hintManager.setDataSourceName("ds_0");
    try (ResultSet rs = preparedStatement.executeQuery()) {
        while (rs.next()) {
            // ...
        }
    }
}
```
#### 清除强制路由值
与基于 Hint 的数据分片相同。

在了解了基于 HintManager 的手动编程方式之后，让我们一起来了解 ShardingSphere 基于分布式 SQL 提供的另一种 Hint 的解决方案。
## 基于分布式 SQL 的 Hint
Apache ShardingSphere 的分布式 SQL HINT 主要由两种功能组成，一种叫做 SQL HINT，即基于 SQL 注释的方式提供的功能，另外一种是通过 DistSQL 实现的作用于 HintManager 的功能。
### SQL HINT
SQL HINT 就是通过在 SQL 语句上增加注释，从而实现强制路由的一种 Hint 方式。它降低了用户改造代码的成本，同时完全脱离了 Java API 的限制，不仅可以在 ShardingSphere-JDBC 中使用，也可以直接在 ShardingSphere-Proxy 上使用。
以下面 SQL 为例，即使用户配置了针对 t_order 的相关分片算法，该 SQL 也会直接在数据库 ds_0 上原封不动地执行，并返回执行结果。
``` sql
/* ShardingSphere hint: dataSourceName=ds_0 */
SELECT * FROM t_order;
```
通过注释的方式我们可以方便地将 SQL 直接送达指定数据库执行而无视其它分片逻辑。以多租户场景为例，用户不用再配置复杂的分库逻辑，也无需改造业务逻辑，只需要将指定库添加到注释信息中即可。在了解了 SQL HINT 的基本使用之后，让我们一起来了解一下 SQL HINT 的实现原理。
#### SQL HINT 的实现原理
其实了解 Apache ShardingSphere 的读者朋友们一定对 SQL 解析引擎不会感到陌生。SQL HINT 实现的第一步就是提取 SQL 中的注释信息。利用 antlr4 的通道功能，可以将 SQL 中的注释信息单独送至特定的隐藏通道，ShardingSphere 也正是利用该功能，在生成解析结果的同时，将隐藏通道中的注释信息一并提取出来了。具体实现如下方代码所示。
- 将 SQL 中的注释送入隐藏通道。
``` 
lexer grammar Comments;

import Symbol;

BLOCK_COMMENT:  '/*' .*? '*/' -> channel(HIDDEN);
INLINE_COMMENT: (('-- ' | '#') ~[\r\n]* ('\r'? '\n' | EOF) | '--' ('\r'? '\n' | EOF)) -> channel(HIDDEN);
```
- 访问语法树后增加对于注释信息的提取：
``` java 
public <T> T visit(final ParseContext parseContext) {
    ParseTreeVisitor<T> visitor = SQLVisitorFactory.newInstance(databaseType, visitorType, SQLVisitorRule.valueOf(parseContext.getParseTree().getClass()), props);
    T result = parseContext.getParseTree().accept(visitor);
    appendSQLComments(parseContext, result);
    return result;
}

private <T> void appendSQLComments(final ParseContext parseContext, final T visitResult) {
    if (!parseContext.getHiddenTokens().isEmpty() && visitResult instanceof AbstractSQLStatement) {
        Collection<CommentSegment> commentSegments = parseContext.getHiddenTokens().stream().map(each -> new CommentSegment(each.getText(), each.getStartIndex(), each.getStopIndex()))
                .collect(Collectors.toList());
        ((AbstractSQLStatement) visitResult).getCommentSegments().addAll(commentSegments);
    }
}
```
提取出用户 SQL 中的注释信息之后，我们就需要根据注释信息来进行相关强制路由了。既然是路由，那么自然就需要使用 Apache ShardingSphere 的路由引擎，我们在路由引擎上做了一些针对 HINT 的改造。
``` java 
public RouteContext route(final LogicSQL logicSQL, final ShardingSphereMetaData metaData) {
    RouteContext result = new RouteContext();
    Optional<String> dataSourceName = findDataSourceByHint(logicSQL.getSqlStatementContext(), metaData.getResource().getDataSources());
    if (dataSourceName.isPresent()) {
        result.getRouteUnits().add(new RouteUnit(new RouteMapper(dataSourceName.get(), dataSourceName.get()), Collections.emptyList()));
        return result;
    }
    for (Entry<ShardingSphereRule, SQLRouter> entry : routers.entrySet()) {
        if (result.getRouteUnits().isEmpty()) {
            result = entry.getValue().createRouteContext(logicSQL, metaData, entry.getKey(), props);
        } else {
            entry.getValue().decorateRouteContext(result, logicSQL, metaData, entry.getKey(), props);
        }
    }
    if (result.getRouteUnits().isEmpty() && 1 == metaData.getResource().getDataSources().size()) {
        String singleDataSourceName = metaData.getResource().getDataSources().keySet().iterator().next();
        result.getRouteUnits().add(new RouteUnit(new RouteMapper(singleDataSourceName, singleDataSourceName), Collections.emptyList()));
    }
    return result;
}
```
ShardingSphere 首先发现了符合定义的 SQL 注释，再经过基本的校验之后，就会直接返回用户指定的路由结果，从而实现强制路由功能。在了解了 SQL HINT 的基本原理之后，让我们一起学习如何使用 SQL HINT。
#### 如何使用 SQL HINT
SQL HINT 的使用非常简单，无论是 ShardingSphere-JDBC 还是 ShardingSphere-Porxy，都可以使用。
第一步打开注释解析开关。将 sqlCommentParseEnabled 设置为 true。
第二步在 SQL 上增加注释即可。目前 SQL HINT 支持指定数据源路由和主库路由。
- 指定数据源路由：目前只支持路由至一个数据源。 注释格式暂时只支持/* */，内容需要以ShardingSphere hint:开始，属性名为 dataSourceName。
/* ShardingSphere hint: dataSourceName=ds_0 */
SELECT * FROM t_order;

- 主库路由：注释格式暂时只支持/* */，内容需要以ShardingSphere hint:开始，属性名为 writeRouteOnly。
/* ShardingSphere hint: writeRouteOnly=true */
SELECT * FROM t_order;

### DistSQL  HINT
Apache ShardingSphere 的  DistSQL 也提供了 Hint 相关功能，让用户可以通过 ShardingSphere-Proxy 来实现分片和强制路由功能。
#### DistSQL HINT 的实现原理
同前文一致，在学习使用 DistSQL HINT 功能之前，让我们一起来了解一下 DistSQL Hint 的实现原理。DistSQL HINT 的实现原理非常简单，其实就是通过操作 HintManager 实现的 HINT 功能。以读写分离 Hint 为例，当用户通过 ShardingSphere-Proxy 执行以下 SQL 时，其实 ShardingSphere 内部对 SQL 做了如下方代码所示的操作。
-- 强制主库读写
``` sql 
set readwrite_splitting hint source = write
```
``` java
@RequiredArgsConstructor
public final class SetReadwriteSplittingHintExecutor extends AbstractHintUpdateExecutor<SetReadwriteSplittingHintStatement> {
    
    private final SetReadwriteSplittingHintStatement sqlStatement;
    
    @Override
    public ResponseHeader execute() {
        HintSourceType sourceType = HintSourceType.typeOf(sqlStatement.getSource());
        switch (sourceType) {
            case AUTO:
                HintManagerHolder.get().setReadwriteSplittingAuto();
                break;
            case WRITE:
                HintManagerHolder.get().setWriteRouteOnly();
                break;
            default:
                break;
        }
        return new UpdateResponseHeader(new EmptyStatement());
    }
}

@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class HintManagerHolder {
    
    private static final ThreadLocal<HintManager> HINT_MANAGER_HOLDER = new ThreadLocal<>();
    
    /**
     * Get an instance for {@code HintManager} from {@code ThreadLocal},if not exist,then create new one.
     *
     * @return hint manager
     */
    public static HintManager get() {
        if (HINT_MANAGER_HOLDER.get() == null) {
            HINT_MANAGER_HOLDER.set(HintManager.getInstance());
        }
        return HINT_MANAGER_HOLDER.get();
    }
    
    /**
     * remove {@code HintManager} from {@code ThreadLocal}.
     */
    public static void remove() {
        HINT_MANAGER_HOLDER.remove();
    }
}
```

用户执行 SQL 之后，DistSQL 解析引擎会首先识别出该 SQL 是读写分离 Hint 的 SQL，同时会提取出用户想要自动路由或者强制到主库的字段。之后它会采用 SetReadwriteSplittingHintExecutor 执行器去执行 SQL，从而将正确操作设置到 HintManager 中，进而实现强制路由主库的功能。
#### DistSQL HINT 的使用
下表为大家展示了 DistSQL Hint 的相关语法。
| 语句 | 说明 | 示例 |
| -----|---- |-----|
| set readwrite_splitting hint source = [auto / write] | 针对当前连接，设置读写分离的路由策略（自动路由或强制到写库）| set readwrite_splitting hint source = write|
|set sharding hint database_value = yy |针对当前连接，设置 hint 仅对数据库分片有效，并添加分片值，yy：数据库分片值|set sharding hint database_value = 100|
|add sharding hint database_value xx = yy|针对当前连接，为表 xx 添加分片值 yy，xx：逻辑表名称，yy：数据库分片值|add sharding hint database_value t_order= 100|
|add sharding hint table_value xx = yy|针对当前连接，为表 xx 添加分片值 yy，xx：逻辑表名称，yy：表分片值|add sharding hint table_value t_order = 100|
|clear hint|针对当前连接，清除 hint 所有设置|clear hint|
|clear [sharding hint / readwrite_splitting hint]|针对当前连接，清除 sharding 或 readwrite_splitting 的 hint 设置|clear readwrite_splitting hint|
|show [sharding / readwrite_splitting] hint status|针对当前连接，查询 sharding 或 readwrite_splitting 的 hint 设置|show readwrite_splitting hint status

本文详细介绍了 Hint 使用的两种方式以及基本原理，相信通过本文，读者朋友们对 Hint 都有了一些基本了解了，大家可以根据自己的需求来选择使用合适的方式。如果在使用过程中遇到任何问题，或者有任何建议想法，都欢迎来社区反馈。
