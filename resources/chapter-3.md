# sharding-jdbc-learning


## sharding-jdbc分库分表处理流程（二）

继续接上一节来说

整体流程我们看完，现在看下每步是如何处理的

来看下 `Collection<PreparedStatementUnit> preparedStatementUnits = route()`

=====> ShardingPreparedStatement.route

```java
    /**
     * 路由预执行SQL执行单元集合
     */
    private Collection<PreparedStatementUnit> route() throws SQLException {
        Collection<PreparedStatementUnit> result = new LinkedList<>();
        /**
         * =====》 PreparedStatementRoutingEngine.route
         * SQL路由处理结果
         */
        routeResult = routingEngine.route(getParameters());    // A部分
        
        for (SQLExecutionUnit each : routeResult.getExecutionUnits()) {
            /**
             *  each数据类似
             *  SQLExecutionUnit(dataSource=demo_ds_1, sqlUnit=SQLUnit(
             *  sql=SELECT i.* FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.user_id=? AND o.order_id=?, 
             *  parameterSets=[[1, 10]]))
             *  
             *  生成PreparedStatement
             */
            PreparedStatement preparedStatement = generatePreparedStatement(each);

            routedStatements.add(preparedStatement);
            /**
             * 将参数放入sql语句里
             */
            replaySetParameter(preparedStatement, each.getSqlUnit().getParameterSets().get(0));
            /**
             * 每个PreparedStatementUnit包含的数据类似：
             * 
             * sqlExecutionUnit=
             * SQLExecutionUnit(dataSource=demo_ds_1, sqlUnit=SQLUnit(
             * sql=SELECT i.* FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.user_id=? AND o.order_id=?, 
             * parameterSets=[[1, 10]]))
             * 
             * statement=
             * com.mysql.jdbc.JDBC42PreparedStatement@12cd9150: 
             * SELECT i.* FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.user_id=1 AND o.order_id=10
             * 
             */
            result.add(new PreparedStatementUnit(each, preparedStatement));
        }
        return result;
    }
```

### A部分

```java
    /**
     * PreparedStatementRoutingEngine.route
     */
    public SQLRouteResult route(final List<Object> parameters) {
        /**
         * 首次执行时需要进行sql路由解析
         * true 说明使用缓存
         */
        if (null == sqlStatement) {
            /**
             * ===========》 ParsingSQLRouter.parse
             */
            sqlStatement = shardingRouter.parse(logicSQL, true);   // AA部分
        }
        /**
         * 先进行分库分表的路由再执行读写分离的路由
         * ===========》 ParsingSQLRouter.route
         */
        return masterSlaveRouter.route(shardingRouter.route(logicSQL, parameters, sqlStatement));  // AB部分
    }
```

### AA部分:

```java
    /**
     * ParsingSQLRouter.parse
     * ===========》 SQLParsingEngine.parse
     */
    public SQLStatement parse(final String logicSQL, final boolean useCache) {
        return new SQLParsingEngine(databaseType, logicSQL, shardingRule, shardingMetaData).parse(useCache);
    }
    /**
     * SQLParsingEngine.parse
     */
    public SQLStatement parse(final boolean useCache) {
        Optional<SQLStatement> cachedSQLStatement = getSQLStatementFromCache(useCache); // AAA部分
        //判断是否为空
        if (cachedSQLStatement.isPresent()) {
            return cachedSQLStatement.get();
        }
        //缓存里没有 走正常流程
        //词法分析引擎
        LexerEngine lexerEngine = LexerEngineFactory.newInstance(dbType, sql);
        //分析首个单词 currentToken赋值 按照例子来说 lexerEngine.getCurrentToken().getType() = SELECT
        lexerEngine.nextToken();
        //解析器装载完毕后，AbstractSelectParser.parse方法去解析
        SQLStatement result = SQLParserFactory.newInstance(dbType, lexerEngine.getCurrentToken().getType(), shardingRule, lexerEngine, shardingMetaData).parse();
        //缓存需要
        if (useCache) {
            ParsingResultCache.getInstance().put(sql, result);
        }
        return result;
    }
```

### AAA部分：

```java
    /**
     * ========>  SQLParsingEngine.getSQLStatementFromCache
     * 
     * ParsingResultCache 很明显 缓存
     * Map<String, SQLStatement> cache
     * true 从缓存中获取
     * false null
     */
    private Optional<SQLStatement> getSQLStatementFromCache(final boolean useCache) {
        return useCache ? Optional.fromNullable(ParsingResultCache.getInstance().getSQLStatement(sql)) : Optional.<SQLStatement>absent();
    }
    /**
     * SQLParserFactory.newInstance
     * 
     * 根据提取的字符串判断执行语句的类型 SELECT DQL语句
     */
    public static SQLParser newInstance(final DatabaseType dbType, final TokenType tokenType, final ShardingRule shardingRule, final LexerEngine lexerEngine, final ShardingMetaData shardingMetaData) {
        if (isDQL(tokenType)) {
            return getDQLParser(dbType, shardingRule, lexerEngine, shardingMetaData);
        }
        if (isDML(tokenType)) {
            return getDMLParser(dbType, tokenType, shardingRule, lexerEngine, shardingMetaData);
        }
        if (isDDL(tokenType)) {
            return getDDLParser(dbType, tokenType, shardingRule, lexerEngine);
        }
        if (isTCL(tokenType)) {
            return getTCLParser(dbType, shardingRule, lexerEngine);
        }
        if (isDAL(tokenType)) {
            return getDALParser(dbType, (Keyword) tokenType, shardingRule, lexerEngine);
        }
        throw new SQLParsingUnsupportedException(tokenType);
    }
    
    private static SQLParser getDQLParser(final DatabaseType dbType, final ShardingRule shardingRule, final LexerEngine lexerEngine, final ShardingMetaData shardingMetaData) {
        return SelectParserFactory.newInstance(dbType, shardingRule, lexerEngine, shardingMetaData);
    }
    /**
     * MySQLSelectParser.newInstance
     * 
     * 根据数据库类型区分执行 这里mysql 需要执行的是这个方法
     */
    public MySQLSelectParser(final ShardingRule shardingRule, final LexerEngine lexerEngine, final ShardingMetaData shardingMetaData) {
        super(shardingRule, lexerEngine, new MySQLSelectClauseParserFacade(shardingRule, lexerEngine), shardingMetaData);
        selectOptionClauseParser = new MySQLSelectOptionClauseParser(lexerEngine);
        limitClauseParser = new MySQLLimitClauseParser(lexerEngine);
    }
```

基本的解析器装载完毕，执行`AbstractSelectParser.parse`方法

```java
    public final SelectStatement parse() {
        /**
         * 通过解析器解析SQL语句
         */
        SelectStatement result = parseInternal();
        /**
         * 合并子查询
         */
        if (result.containsSubQuery()) {
            result = result.mergeSubQueryStatement();
        }
        // TODO move to rewrite
        /**
         * order by 排序
         */
        appendDerivedColumns(result);
        appendDerivedOrderBy(result);
        return result;
    }
```

result结果：

```
SelectStatement(super=DQLStatement(super=AbstractSQLStatement(type=DQL, tables=Tables(tables=[Table(name=t_order, 
alias=Optional.of(o)), Table(name=t_order_item, alias=Optional.of(i))]), 
conditions=Conditions(orCondition=OrCondition(andConditions=[AndCondition(conditions=[Condition(column=Column(name=user_id, 
tableName=t_order), operator=EQUAL, positionValueMap={}, positionIndexMap={0=0}), Condition(column=Column(name=order_id, 
tableName=t_order), operator=EQUAL, positionValueMap={}, positionIndexMap={0=1})])])), 
sqlTokens=[TableToken(beginPosition=16, skippedSchemaNameLength=0, originalLiterals=t_order), 
TableToken(beginPosition=31, skippedSchemaNameLength=0, originalLiterals=t_order_item)], parametersIndex=2)), 
containStar=false, selectListLastPosition=11, groupByLastPosition=0, items=[StarSelectItem(owner=Optional.of(i))], 
groupByItems=[], orderByItems=[], limit=null, subQueryStatement=null)
```

接着`PreparedStatementRoutingEngine.route`最后读写分离+分库分表路由说

### AB部分：

```java
    /**
     * ParsingSQLRouter.route
     */
    public SQLRouteResult route(final String logicSQL, final List<Object> parameters, final SQLStatement sqlStatement) {
        /**
         * 主键配置 非insert语句无需执行
         * 
         * sqlstatament:
         * SelectStatement(super=DQLStatement(super=AbstractSQLStatement(type=DQL, tables=Tables(tables=[Table(name=t_order, 
         * alias=Optional.of(o)), Table(name=t_order_item, alias=Optional.of(i))]), 
         * conditions=Conditions(orCondition=OrCondition(andConditions=[AndCondition(conditions=[Condition(column=Column(name=user_id, 
         * tableName=t_order), operator=EQUAL, positionValueMap={}, positionIndexMap={0=0}), 
         * Condition(column=Column(name=order_id, tableName=t_order), operator=EQUAL, positionValueMap={}, 
         * positionIndexMap={0=1})])])), sqlTokens=[TableToken(beginPosition=16, skippedSchemaNameLength=0, 
         * originalLiterals=t_order), TableToken(beginPosition=31, skippedSchemaNameLength=0, originalLiterals=t_order_item)], 
         * parametersIndex=2)), containStar=false, selectListLastPosition=11, groupByLastPosition=0, 
         * items=[StarSelectItem(owner=Optional.of(i))], groupByItems=[], orderByItems=[], limit=null, subQueryStatement=null)
         */
        GeneratedKey generatedKey = null;
        if (sqlStatement instanceof InsertStatement) {
            generatedKey = getGenerateKey(shardingRule, (InsertStatement) sqlStatement, parameters);
        }
        //路由结果保存
        SQLRouteResult result = new SQLRouteResult(sqlStatement, generatedKey);
        /**
         * ===========》 OptimizeEngineFactory.newInstance  ABA部分
         * ===========》 QueryOptimizeEngine.optimize       ABB部分
         * 
         * ShardingCondition(shardingValues=[ListShardingValue(logicTableName=t_order, columnName=user_id, values=[1]), 
         * ListShardingValue(logicTableName=t_order, columnName=order_id, values=[10])])
         */
        ShardingConditions shardingConditions = OptimizeEngineFactory.newInstance(shardingRule, sqlStatement, parameters, generatedKey).optimize();
        // 增加主键部分
        if (null != generatedKey) {
            setGeneratedKeys(result, generatedKey);
        }
        /**
         * ===========》 private ParsingSQLRouter.route
         * 获取正确的路由信息
         * TableUnits(tableUnits=[TableUnit(dataSourceName=demo_ds_1, routingTables=[RoutingTable(logicTableName=t_order, 
         * actualTableName=t_order_0)])])
         */
        RoutingResult routingResult = route(parameters, sqlStatement, shardingConditions);   //ABC部分
        /**
         * SQL重写引擎
         * 将已有的参数拼装
         */
        SQLRewriteEngine rewriteEngine = new SQLRewriteEngine(shardingRule, logicSQL, databaseType, sqlStatement, shardingConditions, parameters);
        /**
         * 判断是否最终路由到单库单表
         */
        boolean isSingleRouting = routingResult.isSingleRouting();
        /**
         * 分页查询语句额外处理
         */
        if (sqlStatement instanceof SelectStatement && null != ((SelectStatement) sqlStatement).getLimit()) {
            processLimit(parameters, (SelectStatement) sqlStatement, isSingleRouting);
        }
        /**
         * sql语句改写
         * 这里就已经将实际的执行sql路由进行改写
         */
        SQLBuilder sqlBuilder = rewriteEngine.rewrite(!isSingleRouting);
        for (TableUnit each : routingResult.getTableUnits().getTableUnits()) {
            result.getExecutionUnits().add(new SQLExecutionUnit(each.getDataSourceName(), rewriteEngine.generateSQL(each, sqlBuilder)));
        }
        /**
         * 是否显示路由sql日志
         */
        if (showSQL) {
            SQLLogger.logSQL(logicSQL, sqlStatement, result.getExecutionUnits());
        }
        return result;
    }
    
```

### ABA部分：

```java
    /**
     * OptimizeEngineFactory.newInstance
     */
    public static OptimizeEngine newInstance(final ShardingRule shardingRule, final SQLStatement sqlStatement, final List<Object> parameters, final GeneratedKey generatedKey) {
        if (sqlStatement instanceof InsertStatement) {
            return new InsertOptimizeEngine(shardingRule, (InsertStatement) sqlStatement, parameters, generatedKey);
        }
        /**
         * 
         * 查询语句执行这个
         * 返回查询优化引擎
         * 分片表，分片字段，分片值
         */
        if (sqlStatement instanceof SelectStatement || sqlStatement instanceof DMLStatement) {
            return new QueryOptimizeEngine(sqlStatement.getConditions().getOrCondition(), parameters);
        }
        // TODO do with DDL and DAL
        return new QueryOptimizeEngine(sqlStatement.getConditions().getOrCondition(), parameters);
    }
```

### ABB部分：

```java
    /**
     * QueryOptimizeEngine.optimize
     * 
     * 分片规则优化
     * 逻辑表+分片列名+对应列的值
     */ 
    public ShardingConditions optimize() {
        List<ShardingCondition> result = new ArrayList<>(orCondition.getAndConditions().size());
        //每一个条件进行处理，便于java代码使用
        for (AndCondition each : orCondition.getAndConditions()) {
            result.add(optimize(each.getConditionsMap()));
        }
        return new ShardingConditions(result);
    }    
```

### ABC部分：

```java
    /**
     * private ParsingSQLRouter.route
     */
    private RoutingResult route(final List<Object> parameters, final SQLStatement sqlStatement, final ShardingConditions shardingConditions) {
        Collection<String> tableNames = sqlStatement.getTables().getTableNames();
        RoutingEngine routingEngine;
        if (sqlStatement instanceof UseStatement) {
            routingEngine = new IgnoreRoutingEngine();
        } else if (sqlStatement instanceof DDLStatement) {
            routingEngine = new TableBroadcastRoutingEngine(shardingRule, sqlStatement);
        } else if (sqlStatement instanceof ShowDatabasesStatement || sqlStatement instanceof ShowTablesStatement) {
            routingEngine = new DatabaseBroadcastRoutingEngine(shardingRule);
        } else if (shardingConditions.isAlwaysFalse()) {
            routingEngine = new UnicastRoutingEngine(shardingRule, tableNames);
        } else if (sqlStatement instanceof DALStatement) {
            routingEngine = new UnicastRoutingEngine(shardingRule, tableNames);
        } else if (tableNames.isEmpty() && sqlStatement instanceof SelectStatement) {
            routingEngine = new UnicastRoutingEngine(shardingRule, tableNames);
        } else if (tableNames.isEmpty()) {
            routingEngine = new DatabaseBroadcastRoutingEngine(shardingRule);
        //示例用的是这个路由引擎    
        } else if (1 == tableNames.size() || shardingRule.isAllBindingTables(tableNames) || shardingRule.isAllInDefaultDataSource(tableNames)) {
            routingEngine = new StandardRoutingEngine(shardingRule, tableNames.iterator().next(), shardingConditions);
        } else {
            // TODO config for cartesian set
            routingEngine = new ComplexRoutingEngine(shardingRule, parameters, tableNames, shardingConditions);
        }
        /**
         * =========》 StandardRoutingEngine.route
         * 
         * 根据上边类型匹配对应的路由
         */
        return routingEngine.route();
    }
    /**
     * StandardRoutingEngine.route
     * 
     */
    public RoutingResult route() {
        // 表分片规则
        TableRule tableRule = shardingRule.getTableRule(logicTableName);
        // 数据库分片列名，可多列分片
        Collection<String> databaseShardingColumns = shardingRule.getDatabaseShardingStrategy(tableRule).getShardingColumns();
        // 表分片列名，可多列分片
        Collection<String> tableShardingColumns = shardingRule.getTableShardingStrategy(tableRule).getShardingColumns();
        // 路由的真实数据源和真实表
        Collection<DataNode> routedDataNodes = new LinkedHashSet<>();
        // hint分片
        if (HintManagerHolder.isUseShardingHint()) {
            List<ShardingValue> databaseShardingValues = getDatabaseShardingValuesFromHint(databaseShardingColumns);
            List<ShardingValue> tableShardingValues = getTableShardingValuesFromHint(tableShardingColumns);
            Collection<DataNode> dataNodes = route(tableRule, databaseShardingValues, tableShardingValues);
            for (ShardingCondition each : shardingConditions.getShardingConditions()) {
                if (each instanceof InsertShardingCondition) {
                    ((InsertShardingCondition) each).getDataNodes().addAll(dataNodes);
                }
            }
            routedDataNodes.addAll(dataNodes);
        // 正常配置分片
        } else {
            // 无分片条件则添加所有数据库节点
            if (shardingConditions.getShardingConditions().isEmpty()) {
                routedDataNodes.addAll(route(tableRule, Collections.<ShardingValue>emptyList(), Collections.<ShardingValue>emptyList()));
            } else {
                for (ShardingCondition each : shardingConditions.getShardingConditions()) {
                    List<ShardingValue> databaseShardingValues = getShardingValues(databaseShardingColumns, each);
                    List<ShardingValue> tableShardingValues = getShardingValues(tableShardingColumns, each);
                    // 根据分库分表条件和规则配置来确定物理数据库和物理表 这个源码后边再看
                    Collection<DataNode> dataNodes = route(tableRule, databaseShardingValues, tableShardingValues);
                    routedDataNodes.addAll(dataNodes);
                    if (each instanceof InsertShardingCondition) {
                        ((InsertShardingCondition) each).getDataNodes().addAll(dataNodes);
                    }
                }
            }
        }
        return generateRoutingResult(routedDataNodes);
    }
    /**
     * 拼装数据
     * TableUnits(tableUnits=[TableUnit(dataSourceName=demo_ds_1, routingTables=[RoutingTable(logicTableName=t_order, 
     * actualTableName=t_order_0)])])
     */
    private RoutingResult generateRoutingResult(final Collection<DataNode> routedDataNodes) {
        RoutingResult result = new RoutingResult();
        for (DataNode each : routedDataNodes) {
            TableUnit tableUnit = new TableUnit(each.getDataSourceName());
            tableUnit.getRoutingTables().add(new RoutingTable(logicTableName, each.getTableName()));
            result.getTableUnits().getTableUnits().add(tableUnit);
        }
        return result;
    }
```

至此` RoutingResult routingResult = route(parameters, sqlStatement, shardingConditions); `结束

剩下的流程在上边看也比较简单了，`Collection<PreparedStatementUnit> preparedStatementUnits = route()`执行的过程还是比较复杂的