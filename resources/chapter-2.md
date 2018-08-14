# sharding-jdbc-learning


## sharding-jdbc分库分表处理流程（一）

继续接上一节来说

数据源已经准备好了，接下来代码就是SQL的增删改查操作，看下java代码的写法

```java
    private void queryWithEqual() throws SQLException {
        String sql = "SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.user_id=? AND o.order_id=?";
        try (
            Connection connection = dataSource.getConnection();//第一部分
            PreparedStatement preparedStatement = connection.prepareStatement(sql)) {  //第二部分
            preparedStatement.setInt(1, 1);
            preparedStatement.setLong(2, 10L);
            printQuery(preparedStatement);
        }
    }
    private void printQuery(final PreparedStatement preparedStatement) throws SQLException {
        try (ResultSet resultSet = preparedStatement.executeQuery()) { //第三部分
            while (resultSet.next()) {
                System.out.print("order_id:" + resultSet.getLong(1) + ", ");
                System.out.print("user_id:" + resultSet.getLong(2) + ", ");
                System.out.print("status:" + resultSet.getString(3));
                System.out.println();
            }
        }
    }
```

上边的写法，最简单的直接使用JDBC原有的方式进行操作，代码层基本不用改变，这里来一步一步看下

### 第一部分：

`dataSource.getConnection()`

这里的数据源已经变成了`ShardingDataSource`,实例化`ShardingConnection`，shardingContext将配置文件都包含在内

```java
    @Override
    public ShardingConnection getConnection() {
        return new ShardingConnection(shardingContext);
    }
```

### 第二部分：

紧接着执行`connection.prepareStatement`，实际上最终调用为`ShardingPreparedStatement.prepareStatement`

以上边例子为例，执行：

```java
    /**
     * ShardingConnection.prepareStatement
     */
    public PreparedStatement prepareStatement(final String sql) {
        return new ShardingPreparedStatement(this, sql);
    }
    /**
     * ShardingPreparedStatement
     * ResultSet.TYPE_FORWARD_ONLY 只能向前移动 读取过之后自动释放内存
     * ResultSet.CONCUR_READ_ONLY  只读
     * ResultSet.HOLD_CURSORS_OVER_COMMIT 表示修改提交时ResultSet不关闭
     * 基本是默认值
     */
    public ShardingPreparedStatement(final ShardingConnection connection, final String sql) {
        this(connection, sql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY, ResultSet.HOLD_CURSORS_OVER_COMMIT);
    }
    /**
     * 参数配置
     */
    public ShardingPreparedStatement(final ShardingConnection connection, final String sql, final int resultSetType, final int resultSetConcurrency, final int resultSetHoldability) {
        this.connection = connection;
        this.resultSetType = resultSetType;
        this.resultSetConcurrency = resultSetConcurrency;
        this.resultSetHoldability = resultSetHoldability;
        /**
         * 数据源，分片等信息保存 看下类的成员变量
         */
        ShardingContext shardingContext = connection.getShardingContext();
        /**
         * preparedStatement路由引擎
         */        
        routingEngine = new PreparedStatementRoutingEngine(
                sql, shardingContext.getShardingRule(), shardingContext.getShardingMetaData(), shardingContext.getDatabaseType(), shardingContext.isShowSQL());
    }
```

看下路由引擎设置的变量：

```java
    public PreparedStatementRoutingEngine(final String logicSQL, final ShardingRule shardingRule, final ShardingMetaData shardingMetaData, final DatabaseType databaseType, final boolean showSQL) {
        // 逻辑sql SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.user_id=? AND o.order_id=?
        this.logicSQL = logicSQL;
        // 分片路由
        shardingRouter = ShardingRouterFactory.createSQLRouter(shardingRule, shardingMetaData, databaseType, showSQL);
        // 读写路由 实例化ShardingMasterSlaveRouter
        masterSlaveRouter = new ShardingMasterSlaveRouter(shardingRule.getMasterSlaveRules());
    }
    /**
     * 分片路由
     * 先判断是否强制路由
     * 不强制路由 实例化ParsingSQLRouter作为shardingRouter
     * 强制路由这里先不介绍了
     */  
    public static ShardingRouter createSQLRouter(final ShardingRule shardingRule, final ShardingMetaData shardingMetaData, final DatabaseType databaseType, final boolean showSQL) {
        return HintManagerHolder.isDatabaseShardingOnly() ? new DatabaseHintSQLRouter(shardingRule, showSQL) : new ParsingSQLRouter(shardingRule, shardingMetaData, databaseType, showSQL);
    }
```

`preparedStatement.setInt`是如何执行的？

到`AbstractShardingPreparedStatementAdapter.setInt`查看

```java
    public final void setInt(final int parameterIndex, final int x) {
        setParameter(parameterIndex, x);
    }
    /**
     * 保存参数到parameters
     */
    private void setParameter(final int parameterIndex, final Object value) {
        if (parameters.size() == parameterIndex - 1) {
            parameters.add(value);
            return;
        }
        for (int i = parameters.size(); i <= parameterIndex - 1; i++) {
            parameters.add(null);
        }
        parameters.set(parameterIndex - 1, value);
    }
```

### 第三部分：

接下来到重点了，`preparedStatement.executeQuery()`

查询是如何路由执行的？

实际执行`ShardingPreparedStatement.executeQuery`

```java
    /**
     * 执行查询
     */
    public ResultSet executeQuery() throws SQLException {
        ResultSet result;
        try {
            /**
             * 路由SQL执行单元集合
             * 真实数据源，实际sql 参数等
             */
            Collection<PreparedStatementUnit> preparedStatementUnits = route();
            /**
             * 
             * executeQuery 执行查询 将分路由查询结果保存在resultSets
             */
            List<ResultSet> resultSets = new PreparedStatementExecutor(
                    getConnection().getShardingContext().getExecutorEngine(), routeResult.getSqlStatement().getType(), preparedStatementUnits).executeQuery();
            /**
             * 将查询结果保存在queryResults，便于处理
             */
            List<QueryResult> queryResults = new ArrayList<>(resultSets.size());
            for (ResultSet each : resultSets) {
                queryResults.add(new JDBCQueryResult(each));
            }
            /**
             * SQL合并引擎
             */
            MergeEngine mergeEngine = MergeEngineFactory.newInstance(connection.getShardingContext().getShardingRule(), queryResults, routeResult.getSqlStatement());
            /**
             * 合并结果集生成最终结果
             */       
            result = new ShardingResultSet(resultSets, mergeEngine.merge(), this);
        } finally {
            clearBatch();
        }
        currentResultSet = result;
        return result;
    }
```

`resultSet.next()` ，`resultSet.getLong` 都是`ShardingResultSet`重写的

整体流程如上