---
title: "ShardingSphere 元数据能力增强解读与实战"
date: 2023-03-06T09:14:47+08:00
draft: false
categories: ["Tech"]
---

## Apache ShardingSphere 元数据介绍
Apache ShardingSphere 的元数据主要包括规则、数据源、表结构等信息。规则信息可能包含分片、加密、读写分离、事务、高可用等。
数据源信息存储的是需要通过 ShardingSphere 来进行管理的底层数据库资源。表结构信息主要就是底层数据源的表结构，包括表的 column 信息、索引信息等。

Apache ShardingSphere 通过这些元数据信息配合治理中心的能力，例如 zookeeper、etcd 的存储和通知能力，可以实现集群内配置的共享和变更，从而实现计算节点的水平扩展。同时元数据信息对于 ShardingSphere 而言也是至关重要的，以表的数据结构为例，ShardingSphere 利用表的数据结构可以对采用了加密规则的 SQL 进行正确的改写，内核中的 federation 引擎也会利用表结构信息进行 SQL 优化。

既然 ShardingSphere 的元数据如此重要，那么我们该怎么入手了解元数据呢？

## Apache ShardingSphere 三层元数据结构
ShardingSphere 的三层元数据结构是个了解元数据信息的很好入口。我们可以启动 ShardingSphere--proxy 的 cluster 模式，这样可以在 zookeeper 中直观的看到 ShardingSphere 的三层元数据结构。
如下结构展示了 ShardingSphere 元数据在 zookeeper 中的结构
```yaml
governance_ds
--metadata (元数据信息)
----sharding_db (逻辑库名称)
------active_version (当前生效的版本)
------versions
--------0
----------data_sources (底层数据库信息)
----------rules (逻辑库的规则信息，例如分片规则，加密规则等)
------schemas (表、视图信息)
--------sharding_db
----------tables
------------t_order
------------t_single
----------views
----shardingsphere (内置元数据库)
------schemas
--------shardingsphere
----------tables
------------sharding_table_statics (分片统计信息表)
------------cluster_information （版本信息）
----performance_schema (模拟 mysql 数据库)
------schemas
--------performance_schema
----------tables
------------accounts
----information_schema (模拟 mysql 数据库)
------schemas
--------information_schema
----------tables
------------tables
------------schemata
------------columns
------------engines
------------routines
------------parameters
------------views
----mysql
----sys
--sys_data (内置元数据库的具体行信息)
----shardingsphere
------schemas
--------shardingsphere
----------tables
------------sharding_table_statistics
--------------79ff60bc40ab09395bed54cfecd08f94
--------------e832393209c9a4e7e117664c5ff8fc61
------------cluster_information
--------------d387c4f7de791e34d206f7dd59e24c1c
```
上图展示了 ShardingSphere 的元数据结构信息，内容非常丰富，不过我们也不用担心，只要了解了大的节点信息，就可以根据逻辑推测出其它相关信息。

我们首先关注下 metadata 节点，下方的 active_version 中展示了当前生效的元数据版本，然后在 versions 节点下我们可以找到生效的版本 0 ， 在这个生效版本节点下存储了规则和数据库的连接信息。
schemas 节点下存储了该逻辑库下的表和视图的具体信息，值得一提的是，ShardingSphere 存储的表结构信息是经过规则装饰后的表结构信息，例如分片表只会根据其中一片真实表获取结构，然后替换表名称，如果是加密规则，也不会在表结构中展示真实的加密列信息。这样做的目的是为了让用户完全面向逻辑库来进行相关操作。

对于 metadata 下的 shardingsphere 节点，该节点也跟逻辑库的结构类似，只不过这里存储的是 ShardingSphere 内置的一些表结构，例如  sharding_table_statics （分片统计信息表）、cluster_information （版本信息表）等。这一块内容后面的内置元数据库会再进一步展开讲讲。

至于 performance_schema、information_schema、mysql、sys 等节点都是用来模拟 mysql 的数据字典而建立的。当然如果用户的前端协议是 PostgreSQL， 那么这些数据字典也会变更为 PostgreSQL 的数据字典。目前数据字典这边主要用于支持各种客户端工具直接连接 proxy，未来会进一步增加数据收集从而支持对于这些数据字典的查询。这一块也会在后面的元数据库内容中做进一步介绍。

可能有同学看到我们的数据结构会比较好奇，为什么在 sharding_db 的逻辑库下还有一层 sharding_db 呢？为什么不直接把 tables 节点放置在 sharding_db 下呢？其实这也就是 ShardingSphere 三层元数据结构的由来，ShardingSphere 是一个数据库的上层平台，因此需要兼容多种数据库格式，MySQL 的确是两层结构的，但是对于 PostgreSQL 而言，它是三层结构的，由 实例 和 database 以及 schema 组成，因此为了兼容性，ShardingSphere 使用了三层数据库结构，对于 MySQL，ShardingSphere 增加了一个相同的逻辑 schema 层，从而保证逻辑的统一性。

在了解了 metadata 的结构之后，相信细心的同学也发现了 sys_data 节点。那么这个节点的作用是什么呢？让我们走进 ShardingSphere 的内置元数据库。


## Apache ShardingSphere 内置元数据库

### 内置元数据库简介
内置元数据库是什么？其实我们从上面的节点中就能猜出来，在 metadata 节点下有一个 shardingsphere 节点，该节点下目前存在着两张表，sharding_table_statics （分片信息收集表），cluster_information （版本信息表）。可能有同学会说他们都是 ShardingSphere 内部信息的收集表，的确 ShardingSphere 内置元数据库的设计目标之一就是存储内部收集信息服务于功能和用户，另外，内置元数据库还可以用来存储用户设置的信息（暂未实现）。

说了这么多，肯定有同学好奇，sys_data 节点的作用是什么。ShardingSphere 将表的结构信息存储在 metadata 中，表的行信息存储在 sys_data 中。我们可以看到 sys_data 节点下的多个 id，其实他们就是该表的每一行，具体行的内容就存储在该节点之下。

通过在 metadata 中存储表结构，在 sys_data 中存储表的内容，我们就可以实现通过 sql 直接查询内置元数据库表的信息了。例如我们可以通过如下 SQL 直接查询出当前的版本信息。
```
mysql> select * from shardingsphere.cluster_information;
+----------------+
| version        |
+----------------+
| 5.3.2-SNAPSHOT |
+----------------+
1 row in set (3.35 sec)
```
我们还可以直接查询出某个真实表的统计信息。
```
mysql> select * from shardingsphere.sharding_table_statistics where actual_table_name = "t_order_1"\G
*************************** 1. row ***************************
                  id: 2
 logic_database_name: sharding_db
    logic_table_name: t_order
actual_database_name: ds_0
   actual_table_name: t_order_1
           row_count: 0
                size: 16384
*************************** 2. row ***************************
                  id: 4
 logic_database_name: sharding_db
    logic_table_name: t_order
actual_database_name: ds_1
   actual_table_name: t_order_1
           row_count: 0
                size: 16384
2 rows in set (0.39 sec)
```

那么元数据库的功能是如何实现的呢？

### 内置数据库的工作原理
内置元数据库功能的实现主要依赖两个方面，一是数据收集，二是查询实现。

数据收集主要需要考虑如何将数据采集到内存，同时还需要考虑如何将内存信息同步到治理中心来保证集群间的同步。如何将数据采集到内存中，以 sharding_table_statistics 表的数据采集为例。从 `ShardingSphereDataCollector` 接口出发，我们看到该类有一个数据收集的方法。
```java
/**
 * ShardingSphere data collector.
 */
@SingletonSPI
public interface ShardingSphereDataCollector extends TypedSPI {
    
    /**
     * Collect.
     *
     * @param databaseName database name
     * @param table table
     * @param shardingSphereDatabases ShardingSphere databases
     * @return ShardingSphere table data
     * @throws SQLException sql exception
     */
    Optional<ShardingSphereTableData> collect(String databaseName, ShardingSphereTable table, Map<String, ShardingSphereDatabase> shardingSphereDatabases) throws SQLException;
```
查询该接口的调用方，我们可以看到是 `ShardingSphereDataCollectorRunnable` 定时任务在调用。没错，当前的实现方式是在 proxy 端启动定时任务来进行数据收集，根据内置元数据表来区分不同的数据收集器进行数据采集。未来会根据社区用户反馈，可能将这一部分做成 e-job 触发的方式来进行收集。
`ShardingStatisticsTableCollector` 类中具体展示了收集信息的逻辑。主要就是利用底层数据源和分片规则去查询数据库信息从而获取统计信息。

在数据收集完成后，`ShardingSphereDataScheduleCollector` 类会根据收集到的信息跟内存中的信息做比对，如果发现不一致，那么会通过 EVENTBUS 发送事件给治理中心模块，治理中心收到事件后，会更新其它节点的信息，并做内存同步。监听事件类的代码如下。
```java
/**
 * ShardingSphere schema data registry subscriber.
 */
@SuppressWarnings("UnstableApiUsage")
public final class ShardingSphereSchemaDataRegistrySubscriber {
    
    private final ShardingSphereDataPersistService persistService;
    
    private final GlobalLockPersistService lockPersistService;
    
    public ShardingSphereSchemaDataRegistrySubscriber(final ClusterPersistRepository repository, final GlobalLockPersistService globalLockPersistService, final EventBusContext eventBusContext) {
        persistService = new ShardingSphereDataPersistService(repository);
        lockPersistService = globalLockPersistService;
        eventBusContext.register(this);
    }
    
    /**
     * Update when ShardingSphere schema data altered.
     *
     * @param event schema altered event
     */
    @Subscribe
    public void update(final ShardingSphereSchemaDataAlteredEvent event) {
        String databaseName = event.getDatabaseName();
        String schemaName = event.getSchemaName();
        GlobalLockDefinition lockDefinition = new GlobalLockDefinition("sys_data_" + event.getDatabaseName() + event.getSchemaName() + event.getTableName());
        if (lockPersistService.tryLock(lockDefinition, 10_000)) {
            try {
                persistService.getTableRowDataPersistService().persist(databaseName, schemaName, event.getTableName(), event.getAddedRows());
                persistService.getTableRowDataPersistService().persist(databaseName, schemaName, event.getTableName(), event.getUpdatedRows());
                persistService.getTableRowDataPersistService().delete(databaseName, schemaName, event.getTableName(), event.getDeletedRows());
            } finally {
                lockPersistService.unlock(lockDefinition);
            }
        }
    }
}

```
如上述代码所示，在事件接受后，当前节点会更新治理中心的信息，当节点信息发生变更后，依赖 zookeeper/etcd 的通知能力，集群中的其它节点会收到治理中心的变更，并在如下代码中更新自己节点的内存信息。
```java
/**
 * ShardingSphere data changed watcher.
 */
public final class ShardingSphereDataChangedWatcher implements GovernanceWatcher<GovernanceEvent> {
    
    @Override
    public Collection<String> getWatchingKeys(final String databaseName) {
        return Collections.singleton(ShardingSphereDataNode.getShardingSphereDataNodePath());
    }
    
    @Override
    public Collection<Type> getWatchingTypes() {
        return Arrays.asList(Type.ADDED, Type.UPDATED, Type.DELETED);
    }
    
    @Override
    public Optional<GovernanceEvent> createGovernanceEvent(final DataChangedEvent event) {
        if (isDatabaseChanged(event)) {
            return createDatabaseChangedEvent(event);
        }
        if (isSchemaChanged(event)) {
            return createSchemaChangedEvent(event);
        }
        if (isTableChanged(event)) {
            return createTableChangedEvent(event);
        }
        if (isTableRowDataChanged(event)) {
            return createRowDataChangedEvent(event);
        }
        return Optional.empty();
    }
    
    private boolean isDatabaseChanged(final DataChangedEvent event) {
        return ShardingSphereDataNode.getDatabaseName(event.getKey()).isPresent();
    }
    
    private boolean isSchemaChanged(final DataChangedEvent event) {
        return ShardingSphereDataNode.getDatabaseNameByDatabasePath(event.getKey()).isPresent() && ShardingSphereDataNode.getSchemaName(event.getKey()).isPresent();
    }
    
    private boolean isTableChanged(final DataChangedEvent event) {
        Optional<String> databaseName = ShardingSphereDataNode.getDatabaseNameByDatabasePath(event.getKey());
        Optional<String> schemaName = ShardingSphereDataNode.getSchemaNameBySchemaPath(event.getKey());
        Optional<String> tableName = ShardingSphereDataNode.getTableName(event.getKey());
        return databaseName.isPresent() && schemaName.isPresent() && tableName.isPresent();
    }
    
    private boolean isTableRowDataChanged(final DataChangedEvent event) {
        return ShardingSphereDataNode.getRowUniqueKey(event.getKey()).isPresent();
    }
    
    private Optional<GovernanceEvent> createDatabaseChangedEvent(final DataChangedEvent event) {
        Optional<String> databaseName = ShardingSphereDataNode.getDatabaseName(event.getKey());
        Preconditions.checkState(databaseName.isPresent());
        if (Type.ADDED == event.getType() || Type.UPDATED == event.getType()) {
            return Optional.of(new DatabaseDataAddedEvent(databaseName.get()));
        }
        if (Type.DELETED == event.getType()) {
            return Optional.of(new DatabaseDataDeletedEvent(databaseName.get()));
        }
        return Optional.empty();
    }
    
    private Optional<GovernanceEvent> createSchemaChangedEvent(final DataChangedEvent event) {
        Optional<String> databaseName = ShardingSphereDataNode.getDatabaseNameByDatabasePath(event.getKey());
        Preconditions.checkState(databaseName.isPresent());
        Optional<String> schemaName = ShardingSphereDataNode.getSchemaName(event.getKey());
        Preconditions.checkState(schemaName.isPresent());
        if (Type.ADDED == event.getType() || Type.UPDATED == event.getType()) {
            return Optional.of(new SchemaDataAddedEvent(databaseName.get(), schemaName.get()));
        }
        if (Type.DELETED == event.getType()) {
            return Optional.of(new SchemaDataDeletedEvent(databaseName.get(), schemaName.get()));
        }
        return Optional.empty();
    }
    
    private Optional<GovernanceEvent> createTableChangedEvent(final DataChangedEvent event) {
        Optional<String> databaseName = ShardingSphereDataNode.getDatabaseNameByDatabasePath(event.getKey());
        Preconditions.checkState(databaseName.isPresent());
        Optional<String> schemaName = ShardingSphereDataNode.getSchemaNameBySchemaPath(event.getKey());
        Preconditions.checkState(schemaName.isPresent());
        return doCreateTableChangedEvent(event, databaseName.get(), schemaName.get());
    }
    
    private Optional<GovernanceEvent> doCreateTableChangedEvent(final DataChangedEvent event, final String databaseName, final String schemaName) {
        Optional<String> tableName = ShardingSphereDataNode.getTableName(event.getKey());
        Preconditions.checkState(tableName.isPresent());
        if (Type.ADDED == event.getType() || Type.UPDATED == event.getType()) {
            return Optional.of(new TableDataChangedEvent(databaseName, schemaName, tableName.get(), null));
        }
        if (Type.DELETED == event.getType()) {
            return Optional.of(new TableDataChangedEvent(databaseName, schemaName, null, tableName.get()));
        }
        return Optional.empty();
    }
    
    private Optional<GovernanceEvent> createRowDataChangedEvent(final DataChangedEvent event) {
        Optional<String> databaseName = ShardingSphereDataNode.getDatabaseNameByDatabasePath(event.getKey());
        Preconditions.checkState(databaseName.isPresent());
        Optional<String> schemaName = ShardingSphereDataNode.getSchemaNameBySchemaPath(event.getKey());
        Preconditions.checkState(schemaName.isPresent());
        Optional<String> tableName = ShardingSphereDataNode.getTableNameByRowPath(event.getKey());
        Preconditions.checkState(tableName.isPresent());
        Optional<String> rowPath = ShardingSphereDataNode.getRowUniqueKey(event.getKey());
        Preconditions.checkState(rowPath.isPresent());
        if (Type.ADDED == event.getType() || Type.UPDATED == event.getType() && !Strings.isNullOrEmpty(event.getValue())) {
            YamlShardingSphereRowData yamlShardingSphereRowData = YamlEngine.unmarshal(event.getValue(), YamlShardingSphereRowData.class);
            return Optional.of(new ShardingSphereRowDataChangedEvent(databaseName.get(), schemaName.get(), tableName.get(), yamlShardingSphereRowData));
        }
        if (Type.DELETED == event.getType()) {
            return Optional.of(new ShardingSphereRowDataDeletedEvent(databaseName.get(), schemaName.get(), tableName.get(), rowPath.get()));
        }
        return Optional.empty();
    }
}
```
经过上述流程，我们就完成了元数据库信息的收集。

那么内置元数据库是如何支持用户查询的呢？其实就是利用了 ShardingSphere 内置的 federation 引擎实现的。federation 引擎借助 calcite 的能力，可以将内存中的数据结构注册到 calcite 上，通过 calcite 将 sql 查询转化为内存的查询，从而实现 SQL 语句直接返回结果。
在 `FilterableTableScanExecutor` 类中，我们可以看到如果用户查询的表在内置元数据库中，那么会采用内存查询。
```java
    private Enumerable<Object[]> executeByShardingSphereData(final String databaseName, final String schemaName, final ShardingSphereTable table) {
        Optional<ShardingSphereTableData> tableData = Optional.ofNullable(data.getDatabaseData().get(databaseName)).map(optional -> optional.getSchemaData().get(schemaName))
                .map(ShardingSphereSchemaData::getTableData).map(shardingSphereData -> shardingSphereData.get(table.getName()));
        return tableData.map(this::createMemoryEnumerator).orElseGet(this::createEmptyEnumerable);
    }
    
    private Enumerable<Object[]> createMemoryEnumerator(final ShardingSphereTableData tableData) {
        return new AbstractEnumerable<Object[]>() {
            
            @Override
            public Enumerator<Object[]> enumerator() {
                return new MemoryEnumerator<>(tableData.getRows());
            }
        };
    }
```
当然 federation 引擎还提供了其它更强大的功能，例如跨库查询、复杂子查询等，当前 federation 也在快速迭代中，欢迎社区小伙伴前来贡献。

在了解了内置元数据库的基本实现原理后，我们就可以利用内置元数据库实现更多更丰富的功能了。例如支持 PostgreSQL 客户端的 \d 查询。

### PostgreSQL \d 的查询支持

PostgreSQL \d 是 PG 客户端常用的命令之一，要实现 \d 的查询，其实就是需要实现它对应的 SQL，并且需要对数据做一定的装饰，例如分片表替换成逻辑表。

\d 实际的执行语句如下
```sql
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'i' THEN 'index' WHEN 'I' THEN 'global partition index' WHEN 'S' THEN 'sequence' WHEN 'L' THEN 'large sequence' WHEN 'f' THEN 'foreign table' WHEN 'm' THEN 'materialized view'  WHEN 'e' THEN 'stream' WHEN 'o' THEN 'contview' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner",
  c.reloptions as "Storage"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','v','m','S','L','f','e','o','')
      AND n.nspname <> 'pg_catalog'
      AND n.nspname <> 'db4ai'
      AND n.nspname <> 'information_schema'
      AND n.nspname !~ '^pg_toast'
      AND c.relname not like 'matviewmap\_%'
      AND c.relname not like 'mlog\_%'
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
```

需要实现该语句的查询，我们需要收集 `pg_catalog.pg_class` `pg_catalog.pg_namespace` 两张表的信息。另外我们还需要模拟返回如下两个函数的返回结果 `pg_catalog.pg_get_userbyid(c.relowner)`, `pg_catalog.pg_table_is_visible(c.oid)`。

表的收集同上面的 `sharding_table_statistics` 表的收集逻辑类似，这里就不再赘述，由于 pg_class 内容比较多，所以我们只收集 \d 涉及到的一些信息。另外在数据收集阶段，由于分片规则的存在，我们需要展示逻辑表名，因此需要对收集的信息做进一步的装饰，例如表名的替换。

查询的过程我们只需要模拟函数结果就可以，很幸运，calcite 提供了注册函数的能力。当然目前只是单纯的 mock，未来可以进一步扩展成为真实数据。

```java
    /**
     * Create catalog reader.
     *
     * @param schemaName schema name
     * @param schema schema
     * @param relDataTypeFactory rel data type factory
     * @param connectionConfig connection config
     * @return calcite catalog reader
     */
    public static CalciteCatalogReader createCatalogReader(final String schemaName, final Schema schema, final RelDataTypeFactory relDataTypeFactory, final CalciteConnectionConfig connectionConfig) {
        CalciteSchema rootSchema = CalciteSchema.createRootSchema(true);
        rootSchema.add(schemaName, schema);
        registryUserDefinedFunction(schemaName, rootSchema.plus());
        return new CalciteCatalogReader(rootSchema, Collections.singletonList(schemaName), relDataTypeFactory, connectionConfig);
    }
    
    private static void registryUserDefinedFunction(final String schemaName, final SchemaPlus schemaPlus) {
        if (!"pg_catalog".equalsIgnoreCase(schemaName)) {
            return;
        }
        schemaPlus.add("pg_catalog.pg_table_is_visible", ScalarFunctionImpl.create(SQLFederationPlannerUtil.class, "pgTableIsVisible"));
        schemaPlus.add("pg_catalog.pg_get_userbyid", ScalarFunctionImpl.create(SQLFederationPlannerUtil.class, "pgGetUserById"));
    }

    /**
     * Mock pg_table_is_visible function.
     *
     * @param oid oid
     * @return true
     */
    @SuppressWarnings("unused")
    public static boolean pgTableIsVisible(final Long oid) {
        return true;
    }
    
    /**
     * Mock pg_get_userbyid function.
     * 
     * @param oid oid
     * @return user name
     */
    @SuppressWarnings("unused")
    public static String pgGetUserById(final Long oid) {
        return "mock user";
    }
```

其它流程就基本跟 `sharding_table_statistics` 一致。那么我们来看下我们的成果吧。

我们执行 \d 结果如下（t_order 为分片表，t_single 为普通单表）。
```sql
sharding_db=> \d
                 List of relations
 Schema |   Name   |       Type        |   Owner
--------+----------+-------------------+-----------
 public | t_order  | table             | mock user
 public | t_single | table             | mock user
(2 rows)

```

目前 ShardingSphere 内置元数据库功能是试验性的功能，很多流程和实现方式还需要进一步完善，它是 ShardingSphere 社区对于元数据库的一次探索，它的进一步发展还依赖社区小伙伴的支持。读完了全完的你是不是也跃跃欲试了？欢迎来社区贡献，我们准备了丰富的任务列表等你来挑战。
https://github.com/apache/shardingsphere/issues/24378

