# sharding-jdbc-learning


## sharding-jdbc数据源

数据源DataSource

以github上示例代码分库分表`ShardingOnlyWithDatabasesAndTables.java`为例，看下数据源是如何创建的？

java代码比较容易切入，不看配置文件方式，本质上相同

```java
    /**
     * 数据源的创建过程
     */
    private static DataSource getDataSource() throws SQLException {
        /**
         * 分库分表规则配置类
         */
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        shardingRuleConfig.getTableRuleConfigs().add(getOrderTableRuleConfiguration());
        shardingRuleConfig.getTableRuleConfigs().add(getOrderItemTableRuleConfiguration());
        shardingRuleConfig.getBindingTableGroups().add("t_order, t_order_item");
        /**
         * 设置分片规则
         * 支持inline表达式
         * 分片算法可以自行实现（如有需要）
         */
        shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(new InlineShardingStrategyConfiguration("user_id", "demo_ds_${user_id % 2}"));
        shardingRuleConfig.setDefaultTableShardingStrategyConfig(new StandardShardingStrategyConfiguration("order_id", new ModuloShardingTableAlgorithm()));
        /**
         * 工厂方法创建数据源
         */
        return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), shardingRuleConfig, new HashMap<String, Object>(), new Properties());
    }
```

看下规则类里的成员变量，即可以设置这些属性，对于配置文件同样适用这些属性

- ShardingRuleConfiguration

```java
    public final class ShardingRuleConfiguration {

        /**
         * 默认数据源
         */
        private String defaultDataSourceName;
        /**
         * 分表规则配置
         */
        private Collection<TableRuleConfiguration> tableRuleConfigs = new LinkedList<>();
        /**
         * 绑定表规则配置
         */
        private Collection<String> bindingTableGroups = new LinkedList<>();
        /**
         * 数据源分片默认规则配置
         */
        private ShardingStrategyConfiguration defaultDatabaseShardingStrategyConfig;
        /**
         * 表分片默认规则配置
         */
        private ShardingStrategyConfiguration defaultTableShardingStrategyConfig;
        /**
         * 自增主键
         */
        private KeyGenerator defaultKeyGenerator;
        /**
         * 读写分离规则配置
         */
        private Collection<MasterSlaveRuleConfiguration> masterSlaveRuleConfigs = new LinkedList<>();
    }
```

工厂方法创建数据源

```java
    return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), shardingRuleConfig, new HashMap<String, Object>(), new Properties());
```

看下`ShardingDataSourceFactory`

```java
    /**
     * 传入的参数：
     * 1.Map<String, DataSource> 多数据源KV
     * 2.ShardingRuleConfiguration  分库分表规则配置  已配置好的数据塞入
     * 3.Map<String, Object> configMap 用户自定义配置
     * 4.Properties 有日志等的配置
     * 也就是说sharding-jdbc里的数据源扩展了这么多内容
     */
    public static DataSource createDataSource(
            final Map<String, DataSource> dataSourceMap, final ShardingRuleConfiguration shardingRuleConfig, final Map<String, Object> configMap, final Properties props) throws SQLException {
        return new ShardingDataSource(dataSourceMap, new ShardingRule(shardingRuleConfig, dataSourceMap.keySet()), configMap, props);
    }
```

- ShardingDataSource

分开来看下：

第一个参数（`Map<String, DataSource> dataSourceMap`）：

```java
    /**
     * hashmap键值对保存对应多个数据源
     * 跟平常用的也没太大区别
     */
    private static Map<String, DataSource> createDataSourceMap() {
        Map<String, DataSource> result = new HashMap<>();
        result.put("demo_ds_0", DataSourceUtil.createDataSource("demo_ds_0"));
        result.put("demo_ds_1", DataSourceUtil.createDataSource("demo_ds_1"));
        return result;
    }

    public class DataSourceUtil {

    private static final String HOST = "127.0.0.1";

    private static final int PORT = 3306;

    private static final String USER_NAME = "test";

    private static final String PASSWORD = "test123";

    public static DataSource createDataSource(final String dataSourceName) {
        BasicDataSource result = new BasicDataSource();
        result.setDriverClassName(com.mysql.jdbc.Driver.class.getName());
        result.setUrl(String.format("jdbc:mysql://127.0.0.1:3306/test?useUnicode=true&characterEncoding=utf-8&useSSL=false", HOST, PORT, dataSourceName));
        result.setUsername(USER_NAME);
        result.setPassword(PASSWORD);
        return result;
    }
}
```

第二个参数（`ShardingRuleConfiguration shardingRuleConfig`）：

在创建数据源之前将规则配置设置好，`new ShardingRule(shardingRuleConfig, dataSourceMap.keySet())`

- ShardingRule

```java
    /**
     * 将数据库中间件配置转化为可用的配置
     */
    public ShardingRule(final ShardingRuleConfiguration shardingRuleConfig, final Collection<String> dataSourceNames) {
        Preconditions.checkNotNull(dataSourceNames, "Data sources cannot be null.");
        Preconditions.checkArgument(!dataSourceNames.isEmpty(), "Data sources cannot be empty.");
        // 直接引用
        this.shardingRuleConfig = shardingRuleConfig;
        /**
         * 主要是为了区分数据分片+读写分离中的数据源
         * 方便接下来逻辑表与真实数据源+真实表名的映射配置
         * 最后有列出对应方法说明
         */
        shardingDataSourceNames = new ShardingDataSourceNames(shardingRuleConfig, dataSourceNames);
        /**
         * 表规则设置格式化 逻辑表对应的数据源和真实表映射关系设置
         */
        for (TableRuleConfiguration each : shardingRuleConfig.getTableRuleConfigs()) {
            tableRules.add(new TableRule(each, shardingDataSourceNames));
        }
        /**
         * 绑定表配置 debug里看出是将绑定表放在一个对象里，多个绑定表用list存储
         */
        for (String group : shardingRuleConfig.getBindingTableGroups()) {
            List<TableRule> tableRulesForBinding = new LinkedList<>();
            for (String logicTableNameForBindingTable : StringUtil.splitWithComma(group)) {
                tableRulesForBinding.add(getTableRule(logicTableNameForBindingTable));
            }
            bindingTableRules.add(new BindingTableRule(tableRulesForBinding));
        }
        /**
         * 获取默认分库规则
         * 判断是哪种分片
         * newInstance中会进行判断 5种分片规则（包含不分配规则） 不同分片规则进行不同的处理
         */
        defaultDatabaseShardingStrategy = null == shardingRuleConfig.getDefaultDatabaseShardingStrategyConfig()
                ? new NoneShardingStrategy() : ShardingStrategyFactory.newInstance(shardingRuleConfig.getDefaultDatabaseShardingStrategyConfig());
        /**
         * 获取默认分表规则 基本同上
         */
        defaultTableShardingStrategy = null == shardingRuleConfig.getDefaultTableShardingStrategyConfig()
                ? new NoneShardingStrategy() : ShardingStrategyFactory.newInstance(shardingRuleConfig.getDefaultTableShardingStrategyConfig());
        /**
        * 默认自增主键
        * 未设置默认 io.shardingsphere.core.keygen.DefaultKeyGenerator
        * 雪花算法
        */
        defaultKeyGenerator = null == shardingRuleConfig.getDefaultKeyGenerator() ? new DefaultKeyGenerator() : shardingRuleConfig.getDefaultKeyGenerator();
        /**
         * 读写分离配置
         * 将对应的逻辑数据源，主库，分库，负载均衡算法整合
         * debug里很清楚的能看到关系
         */
        for (MasterSlaveRuleConfiguration each : shardingRuleConfig.getMasterSlaveRuleConfigs()) {
            masterSlaveRules.add(new MasterSlaveRule(each));
        }
    }
```

第三个参数（`Map<String, Object> configMap`）：

用户自定义配置，暂时没看懂什么意思

第四个参数（`Properties props`）：

属性配置项，可以配置日志打印和线程数

4个参数配置完成之后，看下创建数据源过程：

```java
    public ShardingDataSource(final Map<String, DataSource> dataSourceMap, final ShardingRule shardingRule, final Map<String, Object> configMap, final Properties props) throws SQLException {
        super(dataSourceMap.values());
        /**
         * 自定义配置参数
         */
        if (!configMap.isEmpty()) {
            ConfigMapContext.getInstance().getShardingConfig().putAll(configMap);
        }
        /**
         * ShardingPropertiesConstant中的设置参数有两个
         * sql.show  /  executor.size
         * 主要就是判断这两个参数
         */
        shardingProperties = new ShardingProperties(null == props ? new Properties() : props);
        int executorSize = shardingProperties.getValue(ShardingPropertiesConstant.EXECUTOR_SIZE);
        /**
         * executor.size 实例化SQL执行引擎
         */
        executorEngine = new ExecutorEngine(executorSize);
        /**
         * 设置元数据 三个参数 数据源，分片规则，数据库类型
         */
        ShardingMetaData shardingMetaData = new JDBCShardingMetaData(dataSourceMap, shardingRule, getDatabaseType());
        /**
         * 获取表结构 将数据保存在元数据中
         * mysql最终执行MySQLShardingMetaDataHandler的getExistColumnMeta方法
         */
        shardingMetaData.init(shardingRule);
        /**
         * 判断是否开启日志打印功能
         */
        boolean showSQL = shardingProperties.getValue(ShardingPropertiesConstant.SQL_SHOW);
        /**
         * 上边构造的数据构成一个整体，类似spring上下文
         */
        shardingContext = new ShardingContext(dataSourceMap, shardingRule, getDatabaseType(), executorEngine, shardingMetaData, showSQL);
    }
```

至此数据源的创建算是完成，主要的参数配置也都将加载完毕

附（部分源码说明）：

- ShardingDataSourceNames

```java
    public ShardingDataSourceNames(final ShardingRuleConfiguration shardingRuleConfig, final Collection<String> rawDataSourceNames) {
        this.shardingRuleConfig = shardingRuleConfig;
        dataSourceNames = getAllDataSourceNames(rawDataSourceNames);
    }
    /**
     * 返回数据源对应的集合（读写分离下返回逻辑数据源）
     */
    private Collection<String> getAllDataSourceNames(final Collection<String> dataSourceNames) {
        Collection<String> result = new LinkedHashSet<>(dataSourceNames);
        /**
         * 读写分离只记录逻辑数据源名称 不设置读写分离不执行下面操作
         */
        for (MasterSlaveRuleConfiguration each : shardingRuleConfig.getMasterSlaveRuleConfigs()) {
            result.remove(each.getMasterDataSourceName());
            result.removeAll(each.getSlaveDataSourceNames());
            result.add(each.getName());
        }
        return result;
    }
```

- newInstance

```java
    /**
     * 实例化分片策略
     */
    public static ShardingStrategy newInstance(final ShardingStrategyConfiguration shardingStrategyConfig) {
        if (shardingStrategyConfig instanceof StandardShardingStrategyConfiguration) {
            return new StandardShardingStrategy((StandardShardingStrategyConfiguration) shardingStrategyConfig);
        }
        if (shardingStrategyConfig instanceof InlineShardingStrategyConfiguration) {
            return new InlineShardingStrategy((InlineShardingStrategyConfiguration) shardingStrategyConfig);
        }
        if (shardingStrategyConfig instanceof ComplexShardingStrategyConfiguration) {
            return new ComplexShardingStrategy((ComplexShardingStrategyConfiguration) shardingStrategyConfig);
        }
        if (shardingStrategyConfig instanceof HintShardingStrategyConfiguration) {
            return new HintShardingStrategy((HintShardingStrategyConfiguration) shardingStrategyConfig);
        }
        return new NoneShardingStrategy();
    }
    /**
     * inline表达式分片策略
     */
    public InlineShardingStrategy(final InlineShardingStrategyConfiguration inlineShardingStrategyConfig) {
        Preconditions.checkNotNull(inlineShardingStrategyConfig.getShardingColumn(), "Sharding column cannot be null.");
        Preconditions.checkNotNull(inlineShardingStrategyConfig.getAlgorithmExpression(), "Sharding algorithm expression cannot be null.");
        /**
         * 获取分片列名
         */
        shardingColumn = inlineShardingStrategyConfig.getShardingColumn();
        /**
         * inline表达式
         */
        String algorithmExpression = InlineExpressionParser.handlePlaceHolder(inlineShardingStrategyConfig.getAlgorithmExpression().trim());
        /**
         * groovy 先放着 有点复杂
         */
        closure = (Closure) new GroovyShell().evaluate(Joiner.on("").join("{it -> \"", algorithmExpression, "\"}"));
    }
```

- MySQLShardingMetaDataHandler

```java
    /**
     * 获取物理表对应的列信息
     * 执行desc命令获取表的列信息
     * 将信息抽出保存
     */
    public List<ColumnMetaData> getExistColumnMeta(final Connection connection) throws SQLException {
        List<ColumnMetaData> result = new LinkedList<>();
        try (Statement statement = connection.createStatement()) {
            statement.execute(String.format("desc %s;", getActualTableName()));
            try (ResultSet resultSet = statement.getResultSet()) {
                while (resultSet.next()) {
                    result.add(new ColumnMetaData(resultSet.getString("Field"), resultSet.getString("Type"), resultSet.getString("Key")));
                }
            }
            return result;
        }
    }
```