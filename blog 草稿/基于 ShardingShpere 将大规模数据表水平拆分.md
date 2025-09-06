假设商铺有 2000 万，平均每个商户有 10 张优惠券，那么优惠券的规模将达到亿级。假设将 1 亿条数据存储在 MySQL 单表中，查询性能会降低，维护成本变高，因此引入了水平分表将大规模数据表水平拆分。

# 分库分表
分库分表涉及水平拆分和垂直拆分两种方式，垂直拆分主要用于模块拆分，水平拆分主要用于拆解大规模数据。

## 垂直拆分
以数据库视角来看，垂直拆分是将不同业务模块的数据表放到不同数据库中；以单表视角来看，垂直拆分单表时，将单表字段拆为两个表，适用于分离不经常被访问的字段。

## 水平拆分
以数据库视角来看，水平分库将一个大规模数据库根据一定规则拆分为多个小规模数据库，从单表来看，水平分表是将一张大规模数据表根据一定规则拆分为多个分布在不同数据库中的小规模数据表中。

查询数据或者插入数据时，根据指定的分片键和分片路由算法计算出某条数据具体的数据库分片以及数据表分片。

可解决规模问题，支持水平扩展提升查询/写入性能（分片并行执行）但是由于涉及多个物理库，就需要解决跨**分片查询**、**分布式事务**、**分片策略**等一系列问题。

# ShardingSphere 实现水平分库分表

现在主要介绍如何基于 ShardingSphere 数据库中间件实现水平分库分表。

ShardingSphere 有多种工作模式，支持嵌入应用，独立部署，为了减轻部署负担，采用了嵌入应用方式演示。

## 核心概念
ShardingShpere 核心概念包含了分片键、分片策略、读写分离以及分布式事务。其中分片键和分片策略的选择是实现水平分库分表的关键，分片键应该尽量保障均匀分布，同时还要选择合适的分片算法，避免出现数据倾斜问题。由于涉及多个数据库，还需要解决分布式事务，暂不讨论。

## 内置分片算法
ShardingSphere 内置**取模分片**（MOD / HASH 分片）和**范围分片**（RANGE 分片两种默认分片算法。**取模分片**对分片键取模，根据余数路由到不同的数据库或表，对分布均匀的分片键分片效果较好，但是当分片数量改变时，需要重新分片。**范围分片**
根据分片键值所在的区间选择目标数据节点，热点问题明显，暂不考虑。

## 注意事项
ShardingSphere 使用时应该避免出现 **读扩散** 和 **跨分片查询** 等一些列问题。 

ShardingSphere 水平分库分表仅仅围绕分片键和分片。倘若查询数据时不带上分片键，会出现读扩散问题，有 N 个分片，性能下降至单表的 1/N。倘若基于分片键进行范围查询，在默认配置下，ShardingSphere 可能直接抛出异常，提示无法路由到唯一分片，因为 ShardingSphere 的核心目标是**精准路由**，而不是无限制的全路由广播。

### 读扩散
当 SQL **没有带上分片键**，ShardingSphere 无法判断数据在哪个分片，就只能把查询 **广播到所有分片**，然后再在应用层或代理层做聚合。这会导致所有分片都被查询，压力增大，响应时间取决于最慢的分片。

```MySQL
-- 假设你用 `user_id % 4` 取模分片：

-- `name` 不是分片键，ShardingSphere 无法知道哪一张表存了 Alice。
-- 结果：会向 `t_user_0`、`t_user_1`、`t_user_2`、`t_user_3` 全部发送同样 的查询，然后聚合结果。
SELECT * FROM t_user WHERE name = 'Alice';


-- 因为 `user_id` 是分片键，可直接定位到 `t_user_1`，不会读扩散。
SELECT * FROM t_user WHERE user_id = 1001;
```

 **常见解决方案**
1. **查询时强制使用分片键**：在应用层保证路由精准。
2. **分片键冗余或联合索引**：避免非分片键查询。
3. **配合全局表 / 广播表**：对于少量数据，所有库存一份，避免跨分片查询。
4. **引入搜索引擎或缓存**：如 Elasticsearch 或 Redis 做非分片键查询。

 对于高并发场景，广播查询会极大增加数据库压力。读放大：一次查询需要访问 N 个分片，成本是单表查询的 N 倍。结果集聚合需要额外的内存和计算。

### 跨分片查询异常

```MySQL
-- 分片规则：order_id % 4
-- 假设使用取模分片：
-- t_order_0, t_order_1, t_order_2, t_order_3 

-- 执行：
SELECT * FROM t_order WHERE order_id > 20;
```

`order_id > 20` 是一个范围查询 ShardingSphere 无法在 SQL 解析阶段判断具体落在哪个分它需要把查询广播到所有分片，再聚合结果**在默认配置下，ShardingSphere 可能直接抛出异常，提示无法路由到唯一分片**。因为 ShardingSphere 的核心目标是**精准路由**，而不是无限制的全路由广播。

 **常见解决方案**
1. **使用支持范围的分片算法**
    - 比如 **BoundedRangeShardingAlgorithm** 或自定义范围算法
    - 这种算法可以根据范围条件判断需要访问哪些分片（不再强制全表扫描或报错）
2. **应用层分片**
    - 手动在应用中将查询拆分为多个等值查询，再合并结果。
    - 例如：先计算范围内所有分片键，再对每个分片发起精准查询。

# ShardingShpere 实践
假设商铺有 2000 万，平均每个商户有 10 张优惠券，那么优惠券的规模将达到亿级。假设将 1 亿条数据存储在 MySQL 单表中，查询性能会降低，维护成本变高，因此引入了水平分表将大规模数据表水平拆分。

为了支持亿级优惠券存储，水平拆分为两个数据库，每个数据库有 0 到 7 共8 张表，两库共计 16 张表。现在来考虑关键问题，如何选择分片键于分片算法？

## Hash_MOD 分片算法
由于优惠券隶属于商铺，根据商铺查询优惠券是以主要场景，为了避免读扩散问题，适用商铺 Id 作为分片键，商铺 Id 使用雪花算法生成，相对均匀，分片算法自然就适用自带的 Hash_MOD 哈希取模分片。

```yaml
# 数据源集合  
dataSources:  
  # 自定义数据源名称，可以是 ds_0 也可以叫 datasource_0 都可以  
  ds_0:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/one_coupon_rebuild_0?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai  
    username: root  
    password: root  
  ds_1:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/one_coupon_rebuild_1?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai  
    username: root  
    password: root  
  
rules:  
  - !SHARDING  
    tables: # 需要分片的数据库表集合  
      t_coupon_template: # 优惠券模板表  
        # 真实存在数据库中的物理表  
        actualDataNodes: ds_${0..1}.t_coupon_template_${0..8}  
        databaseStrategy: # 分库策略  
          standard: # 单分片键分库  
            shardingColumn: shop_number # 分片键  
            shardingAlgorithmName: coupon_template_database_mod # 库分片算法名称，对应 rules[0].shardingAlgorithms        tableStrategy: # 分表策略  
          standard: # 单分片键分表  
            shardingColumn: shop_number # 分片键  
            shardingAlgorithmName: coupon_template_table_mod # 表分片算法名称，对应 rules[0].shardingAlgorithms    shardingAlgorithms: # 分片算法定义集合  
      coupon_template_database_mod: # 优惠券分库算法定义  
        type: HASH_MOD # 基于 Hash 方式分片  
        props:  
          sharding-count: 2 # 一共有 2 个库  
      coupon_template_table_mod: # 优惠券分表算法定义  
        type: HASH_MOD # 基于 Hash 方式分片  
        props:  
          sharding-count: 8 # 单库 8 张表  
  
props:  
  # 配置 ShardingSphere 默认打印 SQL 执行语句  
  sql-show: true
```

### 分片不均问题
![[attachments/Pasted image 20250902162851.png]]

经过测试，发现出现了严重的数据倾斜问题，0 物理库偶数表有数据，1 物理库技术表有数据。初次怀疑由于分片键由雪花算法生成，低位变化不明显，导致取模时可能出现数据倾斜，但是若成立，不应该出现明显的奇偶分离，故排除该原因。经过分析，无论是分库还是分表，都使用了自带的 Hash_MOD 算法，分库时已经天然将 ID 分为了奇偶两类，分表时全是偶数或者奇数，自然全部落到各数据库的奇数表或者偶数表中。

## 自定义分片算法
参考操作系统页表寻址思想，将两个物理库中 0_7 下标物理表的统一整合为 0_15 的物理表，相当于构造了下标 0_15 的连续表。步长为表个数除以物理库个数，即 16 / 2 = 8。

既然是连续的下标，只需先利用分片键取模得到其落到哪个物理表分组下标，利用该下标结合步长计算出分片键处于哪个物理分库即可。

```Java
/**  
 * 基于 HashMod 方式自定义分库算法  
 */  
public final class DBHashModShardingAlgorithm implements StandardShardingAlgorithm<Long> {  
  
    @Getter  
    private Properties props;  
  
    private int shardingCount;  
    private static final String SHARDING_COUNT_KEY = "sharding-count";  
  
    @Override  
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> shardingValue) {  
        long id = shardingValue.getValue();  
        int dbSize = availableTargetNames.size();  
        int mod = (int) hashShardingValue(id) % shardingCount / (shardingCount / dbSize);  
        int index = 0;  
        for (String targetName : availableTargetNames) {  
            if (index == mod) {  
                return targetName;  
            }  
            index++;  
        }  
        throw new IllegalArgumentException("No target found for value: " + id);  
    }  
  
    @Override  
    public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Long> shardingValue) {  
        // 暂无范围分片场景，默认返回空  
        return List.of();  
    }  
  
    @Override  
    public void init(Properties props) {  
        this.props = props;  
        shardingCount = getShardingCount(props);  
    }  
  
    private int getShardingCount(final Properties props) {  
        ShardingSpherePreconditions.checkState(props.containsKey(SHARDING_COUNT_KEY), () -> new ShardingAlgorithmInitializationException(getType(), "Sharding count cannot be null."));  
        return Integer.parseInt(props.getProperty(SHARDING_COUNT_KEY));  
    }  
  
    private long hashShardingValue(final Comparable<?> shardingValue) {  
        return Math.abs((long) shardingValue.hashCode());  
    }  
}
```

```Java
/**  
 * 基于 HashMod 方式自定义分表算法  
 */  
public final class TableHashModShardingAlgorithm implements StandardShardingAlgorithm<Long> {  
  
    @Override  
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> shardingValue) {  
        long id = shardingValue.getValue();  
        int shardingCount = availableTargetNames.size();  
        int mod = (int) hashShardingValue(id) % shardingCount;  
        int index = 0;  
        for (String targetName : availableTargetNames) {  
            if (index == mod) {  
                return targetName;  
            }  
            index++;  
        }  
        throw new IllegalArgumentException("No target found for value: " + id);  
    }  
  
    @Override  
    public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Long> shardingValue) {  
        // 暂无范围分片场景，默认返回空  
        return List.of();  
    }  
  
    private long hashShardingValue(final Comparable<?> shardingValue) {  
        return Math.abs((long) shardingValue.hashCode());  
    }  
}
```

```YAML
# 数据源集合  
dataSources:  
  # 自定义数据源名称，可以是 ds_0 也可以叫 datasource_0 都可以  
  ds_0:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/coupon_rebuild_0?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai  
    username: root  
    password: qwDFerAs1  
  ds_1:  
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource  
    driverClassName: com.mysql.cj.jdbc.Driver  
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/coupon_rebuild_1?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai  
    username: root  
    password: qwDFerAs1  
  
rules:  
  - !SHARDING  
    tables: # 需要分片的数据库表集合  
      t_coupon_settlement:  
        actualDataNodes: ds_${0..1}.t_coupon_settlement_${0..15}  
        databaseStrategy:  
          standard:  
            shardingColumn: user_id  
            shardingAlgorithmName: user_coupon_settlement_database_mod  
        tableStrategy:  
          standard:  
            shardingColumn: user_id  
            shardingAlgorithmName: user_coupon_settlement_table_mod  
    shardingAlgorithms: # 分片算法定义集合  
      user_coupon_settlement_database_mod:  
        type: CLASS_BASED  
        props:  
          algorithmClassName: com.school.coupon.settlement.dao.sharding.DBHashModShardingAlgorithm  
          sharding-count: 16  
          strategy: standard  
      user_coupon_settlement_table_mod:  
        type: CLASS_BASED  
        props:  
          algorithmClassName: com.school.coupon.settlement.dao.sharding.TableHashModShardingAlgorithm  
          strategy: standard  
props:  
  # 配置 ShardingSphere 默认打印 SQL 执行语句  
  sql-show: true
```

