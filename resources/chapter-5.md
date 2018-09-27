# sharding-jdbc-learning


## sharding-jdbc分库分表处理流程（三）

上节看完route()路由的过程，这节继续主流程的执行

```
List<ResultSet> resultSets = new PreparedStatementExecutor(
        getConnection().getShardingContext().getExecutorEngine(), routeResult.getSqlStatement().getType(), preparedStatementUnits).executeQuery();
```

这里创建预处理sql执行器，然后`executeQuery()`执行查询

```java
    /**
     * PreparedStatementExecutor.executeQuery
     */
    public List<ResultSet> executeQuery() throws SQLException {
        return executorEngine.execute(sqlType, preparedStatementUnits, new ExecuteCallback<ResultSet>() {
            
            @Override
            public ResultSet execute(final BaseStatementUnit baseStatementUnit) throws Exception {
                return ((PreparedStatement) baseStatementUnit.getStatement()).executeQuery();
            }
        });
    }
```

```java
    /**
     * ExecutorEngine.execute
     * 
     */
    public <T> List<T> execute(
            final SQLType sqlType, final Collection<? extends BaseStatementUnit> baseStatementUnits, final ExecuteCallback<T> executeCallback) throws SQLException {
        // 判断空
        if (baseStatementUnits.isEmpty()) {
            return Collections.emptyList();
        }
        // 生成event
        OverallExecutionEvent event = new OverallExecutionEvent(sqlType, baseStatementUnits.size());
        // guava eventbus事件处理机制 EventBus 发布 EventExecutionType.BEFORE_EXECUTE
        EventBusInstance.getInstance().post(event);
        // sql执行迭代器
        Iterator<? extends BaseStatementUnit> iterator = baseStatementUnits.iterator();
        // 第一个sql执行任务
        BaseStatementUnit firstInput = iterator.next();
        /**
         * 除上边的任务之外其他任务都异步执行
         * 至于原因，来源网上资料：
         * 优化那些只会路由到一张实际表的SQL为同步执行，减少线程开销
         * —-因为分库分表后，承担绝大部分流量的API底层的SQL条件都会有sharding column，
         * 只会路由到一张实际表，这类SQL同步执行即可，不需要异步执行
         * 
         * AAA部分
         * 
         */        
        ListenableFuture<List<T>> restFutures = asyncExecute(sqlType, Lists.newArrayList(iterator), executeCallback);
        T firstOutput;
        List<T> restOutputs;
        try {
            // 第一个任务同步执行结果   AAB部分
            firstOutput = syncExecute(sqlType, firstInput, executeCallback);
            // 其余任务异步执行结果
            restOutputs = restFutures.get();
            // CHECKSTYLE:OFF
        } catch (final Exception ex) {
            // CHECKSTYLE:ON
            event.setException(ex);
            event.setEventExecutionType(EventExecutionType.EXECUTE_FAILURE);
            EventBusInstance.getInstance().post(event);
            ExecutorExceptionHandler.handleException(ex);
            return null;
        }
        // EventBus 发布 EventExecutionType.EXECUTE_SUCCESS
        event.setEventExecutionType(EventExecutionType.EXECUTE_SUCCESS);
        EventBusInstance.getInstance().post(event);
        // 合并结果
        List<T> result = Lists.newLinkedList(restOutputs);
        result.add(0, firstOutput);
        return result;
    }
```

### AAA部分：

```java
    private <T> ListenableFuture<List<T>> asyncExecute(
            final SQLType sqlType, final Collection<BaseStatementUnit> baseStatementUnits, final ExecuteCallback<T> executeCallback) {
        // 构造存放异步结果的list
        List<ListenableFuture<T>> result = new ArrayList<>(baseStatementUnits.size());
        final boolean isExceptionThrown = ExecutorExceptionHandler.isExceptionThrown();
        final Map<String, Object> dataMap = ExecutorDataMap.getDataMap();
        // 线程池异步执行sql任务
        for (final BaseStatementUnit each : baseStatementUnits) {
            result.add(executorService.submit(new Callable<T>() {
                
                @Override
                public T call() throws Exception {
                    return executeInternal(sqlType, each, executeCallback, isExceptionThrown, dataMap);
                }
            }));
        }
        return Futures.allAsList(result);
    }
```

### AAB部分:

```java
    private <T> T syncExecute(final SQLType sqlType, final BaseStatementUnit baseStatementUnit, final ExecuteCallback<T> executeCallback) throws Exception {
        return executeInternal(sqlType, baseStatementUnit, executeCallback, ExecutorExceptionHandler.isExceptionThrown(), ExecutorDataMap.getDataMap());
    }
```

不论是同步执行还是异步执行，最终都是执行`executeInternal`

```java
    /**
     * 
     */
    private <T> T executeInternal(final SQLType sqlType, final BaseStatementUnit baseStatementUnit, final ExecuteCallback<T> executeCallback,
                                  final boolean isExceptionThrown, final Map<String, Object> dataMap) throws Exception {
        /**
         * 关于synchronized (baseStatementUnit.getStatement().getConnection()) 
         * MySQL、Oracle 的 Connection 实现是线程安全的。数据库连接池实现的 
         * Connection 不一定是线程安全，例如 Druid 的线程池 Connection 非线程安全。    
         * 考虑到mysql驱动在执行statement时对同一个connection是线程安全的。也就是说同一个数据库链接的会话是串行执行的。    
         * 故在shardingjdbc的executor对于多线程执行的情况也进行了针对数据库链接级别的同步。故该方案不会降低shardingjdbc的性能
         */
        synchronized (baseStatementUnit.getStatement().getConnection()) {
            T result;
            ExecutorExceptionHandler.setExceptionThrown(isExceptionThrown);
            ExecutorDataMap.setDataMap(dataMap);
            List<AbstractExecutionEvent> events = new LinkedList<>();
            for (List<Object> each : baseStatementUnit.getSqlExecutionUnit().getSqlUnit().getParameterSets()) {
                events.add(getExecutionEvent(sqlType, baseStatementUnit, each));
            }
            // EventBus 发布 EventExecutionType.BEFORE_EXECUTE
            for (AbstractExecutionEvent event : events) {
                EventBusInstance.getInstance().post(event);
            }
            try {
                // 执行回调函数 获取正常的执行sql结果
                result = executeCallback.execute(baseStatementUnit);
            } catch (final SQLException ex) {
                for (AbstractExecutionEvent each : events) {
                    each.setEventExecutionType(EventExecutionType.EXECUTE_FAILURE);
                    each.setException(ex);
                    EventBusInstance.getInstance().post(each);
                    ExecutorExceptionHandler.handleException(ex);
                }
                return null;
            }
            for (AbstractExecutionEvent each : events) {
                // EventBus 发布 EventExecutionType.EXECUTE_SUCCESS
                each.setEventExecutionType(EventExecutionType.EXECUTE_SUCCESS);
                EventBusInstance.getInstance().post(each);
            }
            return result;
        }
    }
```

继续，看到

```java
    MergeEngine mergeEngine = MergeEngineFactory.newInstance(connection.getShardingContext().getShardingRule(), queryResults, routeResult.getSqlStatement());
    result = new ShardingResultSet(resultSets, mergeEngine.merge(), this);
```

结果集归并 `mergeEngine.merge()`

```java
    public MergedResult merge() throws SQLException {
        // 设置表列名位置映射关系，未获得到查询列位置，该方法会自行进行初始化
        selectStatement.setIndexForItems(columnLabelIndexMap);
        return decorate(build());
    }
    
    private MergedResult build() throws SQLException {
        // 分组或者聚合
        if (!selectStatement.getGroupByItems().isEmpty() || !selectStatement.getAggregationSelectItems().isEmpty()) {
            if (selectStatement.isSameGroupByAndOrderByItems()) {
                return new GroupByStreamMergedResult(columnLabelIndexMap, queryResults, selectStatement);
            } else {
                return new GroupByMemoryMergedResult(columnLabelIndexMap, queryResults, selectStatement);
            }
        }
        // order by
        if (!selectStatement.getOrderByItems().isEmpty()) {
            return new OrderByStreamMergedResult(queryResults, selectStatement.getOrderByItems());
        }
        // 普通方式,流式归并结果集
        return new IteratorStreamMergedResult(queryResults);
    }
    /**
     * limit 语句处理
     */
    private MergedResult decorate(final MergedResult mergedResult) throws SQLException {
        Limit limit = selectStatement.getLimit();
        if (null == limit) {
            return mergedResult;
        }
        if (DatabaseType.MySQL == limit.getDatabaseType() || DatabaseType.PostgreSQL == limit.getDatabaseType() || DatabaseType.H2 == limit.getDatabaseType()) {
            return new LimitDecoratorMergedResult(mergedResult, selectStatement.getLimit());
        }
        if (DatabaseType.Oracle == limit.getDatabaseType()) {
            return new RowNumberDecoratorMergedResult(mergedResult, selectStatement.getLimit());
        }
        if (DatabaseType.SQLServer == limit.getDatabaseType()) {
            return new TopAndRowNumberDecoratorMergedResult(mergedResult, selectStatement.getLimit());
        }
        return mergedResult;
    }
```

最终返回实现ResultSet接口的ShardingResultSet对象，像之前代码使用一样即可