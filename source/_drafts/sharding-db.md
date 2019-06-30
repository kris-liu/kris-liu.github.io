---
title: 基于Spring+MyBatis实现一个分库分表、读写分离功能的工具库
date: 2019-06-01 22:00:00
categories: 架构
tags: 
  - 架构
  - Spring
  - SpringBoot
  - MyBatis
---

一般随着业务的发展，业务量越来越大，业务越来越复杂，数据量也越来越大，数据库层面的优化通常会使用分库分表、读写分离的策略进行扩展。


## 分表的优缺点

### 垂直拆分：

垂直拆分分表指的是将一张大表根据业务、字段冷热、大字段等因素，拆分成多个结构不同的表。

#### 优点：

1. 优化查询性能，减少IO消耗。数据库索引通常以页位单位加载数据，单行数据越小，一页中包含的数据就越多，内存能加载更多数据，命中率更高。

#### 缺点：

1. 产生多表之间的关联查询，一般在业务层面进行多表数据组装，增加了一定的复杂性。

### 水平拆分：

水平拆分分表指的是按一定的分片算法，将同一张表的数据拆分到多个表，每个表结构相同。

#### 优点：

1. 降低单表数据量，优化查询性能，一般来说MYSQL建议单表数据量在1000W以内，一般预估数据有效时间或热点时间内的数据量单表不超过1000W即可，历史数据进行归档。

#### 缺点：

1. 若使用场景存在多个分表键，往往需要根据各个分表键进行一定的数据冗余存储。冗余一般分为冗余索引关系表和冗余全量表，需要根据性能，存储成本，维护成本等方面进行选择。
2. 复杂查询支持度不高。一般分表会通过某个业务键比如uid进行分表，非uid维度的查询，比如进行一些聚合查询，分页查询，排序，统计等SQL，将难以执行。解决方案一般通过添加额外存储结构进行处理，常用的ES+HBase方案，通过ES将索引字段进行存储，通过主键或唯一键，关联到HBase进行全量数据查询查询。

## 分库的优缺点

### 垂直拆分：

垂直拆分分库指的是根据业务模块，将不同业务、关联度不高的表拆分到不同的数据库中，关联度高的表集中在一个库。

#### 优点：

1. 业务隔离，不同业务的库中只包含该业务所属的表，减少业务系统相互之间的影响。
2. 优化数据库库性能，减少数据库压力，避免磁盘存储容量不足。

#### 缺点：

1. 跨库的复杂查询，需要业务层面进行数据组装，增加了复杂性。
2. 跨库的事务引起的分布式事务问题。解决方案见：[分布式事务](http://blogxin.cn/2018/04/23/Distributed-Transaction/)。不过还是建议一般情况下通过一定的系统设计，避免分布式事务的问题。

### 水平拆分：

水平拆分分库指的是按一定的分片算法，先将同一张表的数据拆分到多个表，每个表结构相同，再将这多个表拆分到多个数据库中。

#### 优点：

1. 优化单机和单库的性能瓶颈，比如CPU、IO、内存等瓶颈，提高数据库集群整体的吞吐量，提高稳定性。
3. 避免磁盘存储容量不足。

#### 缺点：

1. 若使用场景存在多个分表键，往往需要根据各个分表键进行一定的数据冗余存储。冗余一般分为冗余索引关系表和冗余全量表，需要根据性能，存储成本，维护成本等方面进行选择。
2. 复杂查询支持度不高。一般分库分表会通过某个业务键比如uid进行分库分表，非uid维度的查询，比如进行一些聚合查询，分页查询，排序，统计等SQL，将难以执行，一般通过添加额外存储结构进行处理，常用的ES+HBase方案，通过ES将索引字段进行存储，通过主键或唯一键，关联到HBase进行全量数据查询查询。
3. 跨分片的事务引起的分布式事务问题。解决方案见：[分布式事务](http://blogxin.cn/2018/04/23/Distributed-Transaction/)。不过还是建议一般情况下通过一定的系统设计，避免分布式事务的问题。


## 读写分离的优缺点

#### 优点：

1. 减少主库压力，一般系统都是读多写少，将部分一致性要求低的查询放到从库，可以有效降低主库的压力，尤其是一些统计类SQL。

#### 缺点：

1. 主从之间数据存在延迟，查询从库有可能读到脏数据，需要根据实际查询场景，决定是否可以查从库。


## 如何自己实现一个分库分表、读写分离功能的工具库

这里主要介绍水平分库分表的工具和实现原理：

### 常见的有两大类型

* Client模式：该模式是提供一个jar包集成到业务框架中，执行SQL时通过该jar包提供的扩展，直接在客户端完成SQL进行解析，重写，路由的能力。比如Sharding-Sphere。

* Proxy模式：该模式是提供一个中间层Proxy，业务系统像访问数据库一样对Proxy进行访问，由Proxy对SQL进行解析，重写，路由到实际数据库等功能。比如DBProxy。

Client模式架构简单，没有中间层，性能较好，减少了中间层Proxy的运维成本。Proxy模式对多语言的支持较好，不必为每种语言都进行一次应用框架客户端的封装和维护。相对来说个人比较喜欢Client模式。


### 基于Client模式封装一个应用客户端jar

一般有两种实现方式：

1. 代理DataSource、Connection、Statement等jdbc协议中的接口，内部对原始DataSource，Connection，Statement等进行增强，添加SQL解析，重写，路由的能力。

2. 基于spring和mybatis的扩展和插件，进行SQL的拦截，解析SQL，进行重写和路由。

第一种方式对所属容器，ORM框架没有依赖，兼容性更好，第二种方式强依赖spring和mybatis，由于我们日常开发基本都是使用spring和mybatis，所以这里主要介绍第二种方式的实现原理，并亲自动手实现一个工具库：

#### 实现分表能力

首先我们需要实现分表的能力：

1. 通过mybatis插件对SQL进行拦截，需要在创建Statement之前进行拦截，可以使用该拦截器：

	```java
	@Intercepts({@Signature(method = "prepare", type = StatementHandler.class, args = {Connection.class, Integer.class})})
	public class ShardingInterceptor implements Interceptor {
	
	}
	```

2. 由于需要知道哪些表的SQL需要被拦截重写，一般可以通过注解进行元数据的标记，定义如下注解，并将该注解标记在mybatis的mapper接口上，便可以在拦截器中获取分库分表规则：

	```java
	@Retention(RetentionPolicy.RUNTIME)
	@Target({ElementType.TYPE})
	public @interface Sharding {
	
	    /**
	     * 是否分表
	     */
	    boolean sharding() default true;
	
	    /**
	     * 库名
	     */
	    String databaseName();
	
	    /**
	     * 基础表名
	     */
	    String tableName();
	
	    /**
	     * @see ShardingStrategy
	     * 分表策略
	     */
	    String strategy();
	
	    /**
	     * 分表数量
	     */
	    int count();
	
	}
	```

3. 对SQL进行重写时，需要根据一定的策略进行计算数据所在分表，定义如下分表策略接口，并提供一些基本实现：

	```java
	public interface ShardingStrategy {
	
	    String UNDERLINE = "_";
	
	    /**
	     * 获取分表位的实际表名
	     *
	     * @param sharding    Sharding信息
	     * @param shardingKey 分库分表因子
	     * @return 带分表位的实际表名
	     */
	    String getTargetTableName(Sharding sharding, String shardingKey);
	
	    /**
	     * 计算分表
	     *
	     * @param sharding    Sharding信息
	     * @param shardingKey 分库分表因子
	     * @return 计算分表
	     */
	    Integer calculateTableSuffix(Sharding sharding, String shardingKey);
	}
	
	```
	
	```java
	public abstract class AbstractShardingStrategyWithDataBase implements ShardingStrategy {
	
	    protected static Map<String, ShardingDataSourceInfo> shardingDataSourceInfoMap = Maps.newHashMap();
	
	    public static void setShardingDataSourceInfoMap(Map<String, ShardingDataSourceInfo> shardingDataSourceInfoMap) {
	        AbstractShardingStrategyWithDataBase.shardingDataSourceInfoMap = shardingDataSourceInfoMap;
	    }
	
	    @Override
	    public String getTargetTableName(Sharding sharding, String shardingKey) {
	        Integer tableSuffix = calculateTableSuffix(sharding, shardingKey);
	        return getTableName(sharding.tableName(), tableSuffix);
	    }
	
	    public String getTableName(String tableName, Integer shardingKey) {
	        return tableName + UNDERLINE + shardingKey;
	    }
	
	}
	```
	
	```java
	public class HashShardingStrategyWithDataBase extends AbstractShardingStrategyWithDataBase {
	
	    @Override
	    public Integer calculateTableSuffix(Sharding sharding, String shardingKey) {
	        return Math.abs(shardingKey.hashCode()) % sharding.count();
	    }
	
	}
	```

4. 在拦截器中，获取当前sql对应的mybatis元信息，从元信息中获取对应mapper接口上的标记注解，用来获取分库分表信息，进行SQL的重写：

	```
		@Override
	    public Object intercept(Invocation invocation) throws Throwable {
	        StatementHandler statementHandler = (StatementHandler) realTarget(invocation.getTarget());
	        MetaObject metaObject = SystemMetaObject.forObject(statementHandler);
	
	        String id = (String) metaObject.getValue(DELEGATE_MAPPED_STATEMENT_ID);
	        String className = id.substring(0, id.lastIndexOf(POINT));
	        Sharding sharding = Class.forName(className).getDeclaredAnnotation(Sharding.class);
	        if (sharding != null && sharding.sharding()) {
	            String sql = (String) metaObject.getValue(DELEGATE_BOUND_SQL_SQL);
	            sql = sql.replaceAll(sharding.tableName(), getTargetTableName(metaObject, sharding));
	            metaObject.setValue(DELEGATE_BOUND_SQL_SQL, sql);
	        }
	        return invocation.proceed();
	    }
	    
	    private String getTargetTableName(MetaObject metaObject, Sharding sharding) throws Exception {
	        String shardingKey = getShardingKey(metaObject);
	        String targetTableName;
	        if (!StringUtils.isEmpty(shardingKey)) {
	            targetTableName = getShardingStrategy(sharding).getTargetTableName(sharding, shardingKey);
	        } else if (StringUtils.isEmpty(shardingKey) && !StringUtils.isEmpty(ShardingContext.getShardingTable())) {
	            targetTableName = DEFAULT_SHARDING_STRATEGY.getTargetTableName(sharding, ShardingContext.getShardingTable());
	        } else {
	            throw new RuntimeException("没有找到分表信息。shardingKey=" + shardingKey + "，ShardingContext=" + ShardingContext.getShardingTable());
	        }
	        return targetTableName;
	    }
		
	    /**
	     * 默认取第一个参数作为分表键
	     * @param metaObject
	     * @return
	     */
	    private String getShardingKey(MetaObject metaObject) {
	        String shardingKey = null;
	        Object parameterObject = metaObject.getValue(DELEGATE_PARAMETER_HANDLER_PARAMETER_OBJECT);
	        if (parameterObject instanceof String) {
	            shardingKey = (String) parameterObject;
	        } else if (parameterObject instanceof Map) {
	            Map<String, Object> parameterMap = (Map<String, Object>) parameterObject;
	            Object param1 = parameterMap.get(PARAM_1);
	            if (param1 instanceof String) {
	                shardingKey = (String) param1;
	            }
	        }
	        return shardingKey;
	    }
	```

5. 考虑到有些扫表类sql并不包含分表键，所以提供如下扩展类，通过ThreadLocal进行路由填充，当sql中不存在分表键时，使用该扩展类处理：

	```java
	public class ShardingContext {
	
	    /**
	     * 分表Key 表名后缀
	     * 直接填充分表位，主要用于按表进行扫描，使用完成后必须及时调用clear方法清空上下文
	     */
	    private static final ThreadLocal<String> SHARDING_TABLE = new ThreadLocal<>();
	
	    public static String getShardingTable() {
	        return SHARDING_TABLE.get();
	    }
	
	    public static void setShardingTable(String shardingTable) {
	        SHARDING_TABLE.set(shardingTable);
	    }
	
	    public static void clear() {
	        ShardingContext.SHARDING_TABLE.remove();
	    }
	}
	```

这样我们就实现了一个具备分表能力的插件了。

### 实现分库能力

基于分表能力，进行增强，实现分库能力：

1. 进行分库可以通过spring提供的`AbstractRoutingDataSource`抽象类进行扩展，该类内部维护了一个DataSource的map，需要自行实现`determineCurrentLookupKey`方法获取分库key，然后根据该key来获取对应的DataSource：

	```java
	    public static class ShardingDataSource extends AbstractRoutingDataSource {
	
	        /**
	         * ShardingContext.getShardingDatabase() 为库名+分库序号
	         *
	         * @return
	         */
	        @Override
	        protected Object determineCurrentLookupKey() {
	            return ShardingContext.getShardingDatabase();
	        }
	    }
	```
	
	```java
	public class ShardingContext {

	    /**
	     * 分库key 库名+分库序号
	     * 用于获取对应库名序号的dataSource
	     * 使用分库插件时必须及时调用clear方法清空上下文
	     */
	    private static final ThreadLocal<String> SHARDING_DATABASE = new ThreadLocal<>();
	
	    /**
	     * 分表Key 表名后缀
	     * 直接填充分表位，主要用于按表进行扫描，使用完成后必须及时调用clear方法清空上下文
	     */
	    private static final ThreadLocal<String> SHARDING_TABLE = new ThreadLocal<>();

	    public static String getShardingDatabase() {
	        return SHARDING_DATABASE.get();
	    }
	
	    public static void setShardingDatabase(String shardingDatabase) {
	        SHARDING_DATABASE.set(shardingDatabase);
	    }
	
	    public static String getShardingTable() {
	        return SHARDING_TABLE.get();
	    }
	
	    public static void setShardingTable(String shardingTable) {
	        SHARDING_TABLE.set(shardingTable);
	    }
	
	    public static void clear() {
	        ShardingContext.SHARDING_DATABASE.remove();
	        ShardingContext.SHARDING_TABLE.remove();
	    }
	}
	```

2. 通过`AbstractRoutingDataSource`进行分库需要我们获取数据库连接前计算好所属分库，首先需要自定义如下配置格式信息用于配置各个分库相关信息，配置文件中需要定义各个分库的分库策略、分库数量、每个数据库的连接池配置：

	```java
	sharding.databases.test.shardingStrategy=cn.blogxin.sharding.plugin.strategy.database.DefaultShardingDataBaseStrategy
	sharding.databases.test.shardingCount=2
	
	sharding.databases.test.dataSource.master.0.driverClassName=com.mysql.jdbc.Driver
	sharding.databases.test.dataSource.master.0.url=jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&characterEncoding=utf-8
	sharding.databases.test.dataSource.master.0.username=root
	sharding.databases.test.dataSource.master.0.password=
	
	sharding.databases.test.dataSource.master.1.driverClassName=com.mysql.jdbc.Driver
	sharding.databases.test.dataSource.master.1.url=jdbc:mysql://127.0.0.1:3306/test1?useUnicode=true&characterEncoding=utf-8
	sharding.databases.test.dataSource.master.1.username=root
	sharding.databases.test.dataSource.master.1.password=
	```

3. 需要根据一定的策略进行分库的计算，定义如下分库策略接口，并提供一些基本实现：

	```java
	public interface ShardingDataBaseStrategy {
	
	    /**
	     * 计算获取对应分库序号
	     *
	     * @param sharingDataBaseCount    分库数量
	     * @param sharingTableCount       分表数量
	     * @param currentShardingTableKey 当前分表位
	     * @return 分库序号
	     */
	    Integer calculate(int sharingDataBaseCount, int sharingTableCount, int currentShardingTableKey);
	}
	```
	
	```java
	/**
	 * 默认分库策略，将分表从小到大均匀分配至各分库中
	 * 比如：
	 * 2个库，10个表
	 * 0-4表在0库，5-9表在1库
	 *
	 * @author kris
	 */
	public class DefaultShardingDataBaseStrategy implements ShardingDataBaseStrategy {
	
	    @Override
	    public Integer calculate(int sharingDataBaseCount, int sharingTableCount, int currentShardingTableKey) {
	        if (sharingTableCount >= sharingDataBaseCount && sharingTableCount % sharingDataBaseCount == 0) {
	            int base = sharingTableCount / sharingDataBaseCount;
	            return currentShardingTableKey / base;
	        }
	        throw new RuntimeException("分库分表规则配置错误");
	    }
	}
	```

4. 加载该分库配置文件信息，并初始化`ShardingDataSource`：

	```java
	@Configuration
	@ConfigurationProperties(prefix = "sharding")
	public class ShardingProperties {
	
	    private Map<String, Database> databases;
	
	    public Map<String, Database> getDatabases() {
	        return databases;
	    }
	
	    public void setDatabases(Map<String, Database> databases) {
	        this.databases = databases;
	    }
	
	}
	```

	```java
	public class Database {
	
	    /**
	     * 分库策略
	     */
	    private String shardingStrategy = "";
	
	    /**
	     * 分库数量
	     */
	    private Integer shardingCount;
	
	    /**
	     * key：分库位
	     * value：分库对应的dataSource配置
	     */
	    private Map<String, Map<Integer, DataSourceProperties>> dataSource;
	
	    public String getShardingStrategy() {
	        return shardingStrategy;
	    }
	
	    public void setShardingStrategy(String shardingStrategy) {
	        this.shardingStrategy = shardingStrategy;
	    }
	
	    public Integer getShardingCount() {
	        return shardingCount;
	    }
	
	    public void setShardingCount(Integer shardingCount) {
	        this.shardingCount = shardingCount;
	    }
	
	    public Map<String, Map<Integer, DataSourceProperties>> getDataSource() {
	        return dataSource;
	    }
	
	    public void setDataSource(Map<String, Map<Integer, DataSourceProperties>> dataSource) {
	        this.dataSource = dataSource;
	    }
	}
	```
	
	```java
    @Resource
    private ShardingProperties shardingProperties;
	
    private DataSource shardingDataSource() {
        Map<String, Database> databases = shardingProperties.getDatabases();
        Preconditions.checkArgument(!CollectionUtils.isEmpty(databases), "不存在分库配置");
	
        Map<String, ShardingDataSourceInfo> shardingDataSourceInfoMap = Maps.newHashMap();
        Map<Object, Object> targetDataSources = Maps.newHashMap();
        DataSource dataSource = null;
	
        for (Map.Entry<String, Database> entry : databases.entrySet()) {
            String dataBaseName = entry.getKey();
            Database database = entry.getValue();
	
            ShardingDataSourceInfo shardingDataSourceInfo = new ShardingDataSourceInfo();
            shardingDataSourceInfo.setShardingCount(database.getShardingCount());
            shardingDataSourceInfo.setShardingDataBaseStrategy(createShardingDataBaseStrategy(database.getShardingStrategy()));
            shardingDataSourceInfoMap.put(dataBaseName, shardingDataSourceInfo);
	
            Set<Map.Entry<String, Map<Integer, DataSourceProperties>>> entries = database.getDataSource().entrySet();
            for (Map.Entry<String, Map<Integer, DataSourceProperties>> masterSlave : entries) {
                String masterSlaveKey = masterSlave.getKey();
                Map<Integer, DataSourceProperties> masterSlaveValue = masterSlave.getValue();
                for (Map.Entry<Integer, DataSourceProperties> propertiesEntry : masterSlaveValue.entrySet()) {
                    String shardingDataBaseKey = dataBaseName + masterSlaveKey + propertiesEntry.getKey();
                    dataSource = createDataSource(propertiesEntry.getValue(), HikariDataSource.class);
                    targetDataSources.put(shardingDataBaseKey, dataSource);
                }
            }
        }
	
        Preconditions.checkArgument(MapUtils.isNotEmpty(targetDataSources), "找不到database配置");
        Preconditions.checkNotNull(dataSource, "找不到database配置");
	
        AbstractShardingStrategyWithDataBase.setShardingDataSourceInfoMap(shardingDataSourceInfoMap);
	
        ShardingDataSource shardingDataSource = new ShardingDataSource();
        shardingDataSource.setTargetDataSources(targetDataSources);
        /**
         * 用于创建LazyConnectionDataSourceProxy时获取真实数据库连接，来获取实际数据库的自动提交配置和隔离级别
         */
        shardingDataSource.setDefaultTargetDataSource(dataSource);
        shardingDataSource.setLenientFallback(false);
        shardingDataSource.afterPropertiesSet();
        return shardingDataSource;
    }

	```

5. 计算分表时，需要额外计算好分库位，扩展分表插件的抽象类`AbstractShardingStrategyWithDataBase`，在计算分表时，根据分表位，获取分表所属的分库key，设置到`ShardingContext.setShardingDatabase`中，用于在`ShardingDataSource.determineCurrentLookupKey()`中获取分库key，来获取所属的真实DataSource：

	```java
	public abstract class AbstractShardingStrategyWithDataBase implements ShardingStrategy {
	
	    private static Map<String, ShardingDataSourceInfo> shardingDataSourceInfoMap = Maps.newHashMap();
	
	    public static void setShardingDataSourceInfoMap(Map<String, ShardingDataSourceInfo> shardingDataSourceInfoMap) {
	        AbstractShardingStrategyWithDataBase.shardingDataSourceInfoMap = shardingDataSourceInfoMap;
	    }
	
	    @Override
	    public String getTargetTableName(Sharding sharding, String shardingKey) {
	        Integer tableSuffix = calculateTableSuffix(sharding, shardingKey);
	        ShardingDataSourceInfo shardingDataSourceInfo = shardingDataSourceInfoMap.get(sharding.databaseName());
	        if (shardingDataSourceInfo != null) {
	            int databaseNum = shardingDataSourceInfo.getShardingDataBaseStrategy().calculate(shardingDataSourceInfo.getShardingCount(), sharding.count(), tableSuffix);
	            ShardingContext.setShardingDatabase(sharding.databaseName() + ShardingContext.getMasterSalve() + databaseNum);
	        }
	        return getTableName(sharding.tableName(), tableSuffix);
	    }
	
	    private String getTableName(String tableName, Integer shardingKey) {
	        return tableName + UNDERLINE + shardingKey;
	    }
	
	}
	```

6. 由于进行SQL重写路由时使用的是创建Statement前阶段的mybatis插件，此时已经获取到了一个真实的数据库连接，无法重新进行分库级别的路由了，那么我们只能将路由阶段提前或者将获取数据库连接阶段延后，提前的话有可能我们拿不到待执行的SQL，所以我们只能选择将获取连接阶段延后。可以使用spring提供的`LazyConnectionDataSourceProxy`，使用该延迟类型数据库连接池在创建连接时，并不会真正创建连接，而是产生一个`Connection`代理类，对该代理`Connection`进行属性设置，比如开启事务、设置隔离级别等配置修改时，只是在内存中进行记录，只有需要执行sql时，才会真正创建一个`Connection`，并将之前内存中的事务属性、隔离级别属性应用到该真实`Connection`上。通过该延迟类型连接池，我们就可以在创建Statement前阶段的mybatis插件中，通过`ShardingContext.setShardingDatabase`进行分库key的设置，从而在`ShardingDataSource`中获取该SQL所属的分库对应的`DataSource`，来获取对应的`Connection`：

	```java
	@Configuration
	@AutoConfigureBefore(DataSourceAutoConfiguration.class)
	@ConditionalOnProperty(name = "sharding.databases", havingValue = "enable")
	@EnableConfigurationProperties(ShardingProperties.class)
	public class ShardingDataSourceConfiguration {
	
	    @Resource
	    private ShardingProperties shardingProperties;
	
	    private DataSource shardingDataSource() {
	        Map<String, Database> databases = shardingProperties.getDatabases();
	        Preconditions.checkArgument(!CollectionUtils.isEmpty(databases), "不存在分库配置");
	
	        Map<String, ShardingDataSourceInfo> shardingDataSourceInfoMap = Maps.newHashMap();
	        Map<Object, Object> targetDataSources = Maps.newHashMap();
	        DataSource dataSource = null;
	
	        for (Map.Entry<String, Database> entry : databases.entrySet()) {
	            String dataBaseName = entry.getKey();
	            Database database = entry.getValue();
	
	            ShardingDataSourceInfo shardingDataSourceInfo = new ShardingDataSourceInfo();
	            shardingDataSourceInfo.setShardingCount(database.getShardingCount());
	            shardingDataSourceInfo.setShardingDataBaseStrategy(createShardingDataBaseStrategy(database.getShardingStrategy()));
	            shardingDataSourceInfoMap.put(dataBaseName, shardingDataSourceInfo);
	
	            Set<Map.Entry<String, Map<Integer, DataSourceProperties>>> entries = database.getDataSource().entrySet();
	            for (Map.Entry<String, Map<Integer, DataSourceProperties>> masterSlave : entries) {
	                String masterSlaveKey = masterSlave.getKey();
	                Map<Integer, DataSourceProperties> masterSlaveValue = masterSlave.getValue();
	                for (Map.Entry<Integer, DataSourceProperties> propertiesEntry : masterSlaveValue.entrySet()) {
	                    String shardingDataBaseKey = dataBaseName + masterSlaveKey + propertiesEntry.getKey();
	                    dataSource = createDataSource(propertiesEntry.getValue(), HikariDataSource.class);
	                    targetDataSources.put(shardingDataBaseKey, dataSource);
	                }
	            }
	        }
	
	        Preconditions.checkArgument(MapUtils.isNotEmpty(targetDataSources), "找不到database配置");
	        Preconditions.checkNotNull(dataSource, "找不到database配置");
	
	        AbstractShardingStrategyWithDataBase.setShardingDataSourceInfoMap(shardingDataSourceInfoMap);
	
	        ShardingDataSource shardingDataSource = new ShardingDataSource();
	        shardingDataSource.setTargetDataSources(targetDataSources);
	        /**
	         * 用于创建LazyConnectionDataSourceProxy时获取真实数据库连接，来获取实际数据库的自动提交配置和隔离级别
	         */
	        shardingDataSource.setDefaultTargetDataSource(dataSource);
	        shardingDataSource.setLenientFallback(false);
	        shardingDataSource.afterPropertiesSet();
	        return shardingDataSource;
	    }
	
	    @Bean
	    public DataSource dataSource() {
	        LazyConnectionDataSourceProxy dataSourceProxy = new LazyConnectionDataSourceProxy();
	        dataSourceProxy.setTargetDataSource(shardingDataSource());
	        return dataSourceProxy;
	    }
	    
	}
	```
	
通过spring提供的`AbstractRoutingDataSource`以及`LazyConnectionDataSourceProxy`扩展类，便可以快速实现分库能力了。

### 实现读写分离能力



### 使用springboot的starter自动装配



### 实现DOME

https://github.com/kris-liu/Sharding-DB

