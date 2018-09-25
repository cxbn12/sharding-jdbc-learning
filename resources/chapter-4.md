# sharding-jdbc-learning


## sharding-jdbc使用问题-绑定表bug

在使用sharding-sphere的3.0.0.M1版本中，在绑定表设置中有个bug,在新版本中已经修复，在这里记录下

## 版本号

sharding-sphere：3.0.0.M1

### 问题

绑定表设置之后会出现一些问题，问题表现：

在设置绑定表之后使用`where`条件，在条件中对`from`后的主表按分片键设置分片条件（比如id=1），分片路由无效，使用无条件的绑定表路由

### 示例

为了演示，使用了3个表，对3个表进行绑定(t_user,t_user_info,t_user_info_other)

分片规则设置：

```properties
sharding.jdbc.config.sharding.default-database-strategy.inline.sharding-column=type_id
sharding.jdbc.config.sharding.default-database-strategy.inline.algorithm-expression=ds-$->{type_id % 2}

sharding.jdbc.config.sharding.tables.t_user.actual-data-nodes=ds-$->{0..1}.t_user_$->{0..1}
sharding.jdbc.config.sharding.tables.t_user.table-strategy.inline.sharding-column=user_id
sharding.jdbc.config.sharding.tables.t_user.table-strategy.inline.algorithm-expression=t_user_$->{user_id % 2}
#sharding.jdbc.config.sharding.tables.t_user.key-generator-column-name=user_id

sharding.jdbc.config.sharding.tables.t_user_info_other.actual-data-nodes=ds-$->{0..1}.t_user_info_other_$->{0..1}
sharding.jdbc.config.sharding.tables.t_user_info_other.table-strategy.inline.sharding-column=user_id
sharding.jdbc.config.sharding.tables.t_user_info_other.table-strategy.inline.algorithm-expression=t_user_info_other_$->{user_id % 2}
sharding.jdbc.config.sharding.tables.t_user_info_other.key-generator-column-name=id

sharding.jdbc.config.sharding.tables.t_user_info.actual-data-nodes=ds-$->{0..1}.t_user_info_$->{0..1}
sharding.jdbc.config.sharding.tables.t_user_info.table-strategy.inline.sharding-column=user_id
sharding.jdbc.config.sharding.tables.t_user_info.table-strategy.inline.algorithm-expression=t_user_info_$->{user_id % 2}
sharding.jdbc.config.sharding.tables.t_user_info.key-generator-column-name=id

sharding.jdbc.config.sharding.binding-tables[0]=t_user,t_user_info,t_user_info_other
```

mybatis：

```xml
    <select id="getUserByBindTable" resultMap="userInfoVO" >
		SELECT o.*
        FROM t_user_info i
        JOIN t_user_info_other t ON i.user_id=t.user_id
        JOIN t_user o ON i.user_id=o.user_id
        WHERE i.user_id= #{user_id}
        AND i.type_id= #{type_id}
    </select>
```

页面请求：

http://localhost:8080/getUserByBindTable.do?user_id=1&type_id=1

执行日志：

```youtrack
2018-09-25 18:14:16.714  INFO 8016 --- [nio-8080-exec-3] Sharding-Sphere-SQL                      : Logic SQL: SELECT o.*
        FROM t_user_info i
        JOIN t_user_info_other t ON i.user_id=t.user_id
        JOIN t_user o ON i.user_id=o.user_id
        WHERE i.user_id= ?
        AND i.type_id= ?
2018-09-25 18:14:16.714  INFO 8016 --- [nio-8080-exec-3] Sharding-Sphere-SQL                      : SQLStatement: SelectStatement(super=DQLStatement(super=AbstractSQLStatement(type=DQL, tables=Tables(tables=[Table(name=t_user_info, alias=Optional.of(i)), Table(name=t_user_info_other, alias=Optional.of(t)), Table(name=t_user, alias=Optional.of(o))]), conditions=Conditions(orCondition=OrCondition(andConditions=[AndCondition(conditions=[Condition(column=Column(name=user_id, tableName=t_user_info), operator=EQUAL, positionValueMap={}, positionIndexMap={0=0}), Condition(column=Column(name=type_id, tableName=t_user_info), operator=EQUAL, positionValueMap={}, positionIndexMap={0=1})])])), sqlTokens=[TableToken(beginPosition=24, skippedSchemaNameLength=0, originalLiterals=t_user_info), TableToken(beginPosition=51, skippedSchemaNameLength=0, originalLiterals=t_user_info_other), TableToken(beginPosition=107, skippedSchemaNameLength=0, originalLiterals=t_user)], parametersIndex=2)), containStar=false, selectListLastPosition=19, groupByLastPosition=0, items=[StarSelectItem(owner=Optional.of(o))], groupByItems=[], orderByItems=[], limit=null, subQueryStatement=null)
2018-09-25 18:14:16.714  INFO 8016 --- [nio-8080-exec-3] Sharding-Sphere-SQL                      : Actual SQL: ds-0 ::: SELECT o.*
        FROM t_user_info_0 i
        JOIN t_user_info_other_0 t ON i.user_id=t.user_id
        JOIN t_user_0 o ON i.user_id=o.user_id
        WHERE i.user_id= ?
        AND i.type_id= ? ::: [[1, 1]]
2018-09-25 18:14:16.714  INFO 8016 --- [nio-8080-exec-3] Sharding-Sphere-SQL                      : Actual SQL: ds-0 ::: SELECT o.*
        FROM t_user_info_1 i
        JOIN t_user_info_other_1 t ON i.user_id=t.user_id
        JOIN t_user_1 o ON i.user_id=o.user_id
        WHERE i.user_id= ?
        AND i.type_id= ? ::: [[1, 1]]
2018-09-25 18:14:16.714  INFO 8016 --- [nio-8080-exec-3] Sharding-Sphere-SQL                      : Actual SQL: ds-1 ::: SELECT o.*
        FROM t_user_info_0 i
        JOIN t_user_info_other_0 t ON i.user_id=t.user_id
        JOIN t_user_0 o ON i.user_id=o.user_id
        WHERE i.user_id= ?
        AND i.type_id= ? ::: [[1, 1]]
2018-09-25 18:14:16.714  INFO 8016 --- [nio-8080-exec-3] Sharding-Sphere-SQL                      : Actual SQL: ds-1 ::: SELECT o.*
        FROM t_user_info_1 i
        JOIN t_user_info_other_1 t ON i.user_id=t.user_id
        JOIN t_user_1 o ON i.user_id=o.user_id
        WHERE i.user_id= ?
        AND i.type_id= ? ::: [[1, 1]]
```

从日志可以看出来，`where`条件没有进行对根据分片键进行分库分表

### 原因

通过跟踪发现问题在下面这个地方：

ParsingSQLRouter的route方法

```java
routingEngine = new StandardRoutingEngine(shardingRule, tableNames.iterator().next(), shardingConditions);

List<ShardingValue> databaseShardingValues = getShardingValues(databaseShardingColumns, each);
List<ShardingValue> tableShardingValues = getShardingValues(tableShardingColumns, each);
```

`tableNames.iterator().next()`这里是获取到对应迭代器的第一个，但是代码里使用的是HashSet，而HashSet底层上使用的还是HashMap,
利用HashMap key的唯一性来作为HashSet的属性，而这里HashSet迭代器获取的next值时无序的，因为HashMap本身在保存键值对时进行了位置上的无序
存放操作（通过hash值来计算位置，有兴趣去看源码，本身位置还是固定的），导致上边代码取出来的不是存放的顺序，并且Sharding-Sphere以此为主表，
其他表都依据这个表进行操作，`where`条件上的表和这里获取的表不一致，错误的认为没有分片条件，进而执行绑定表的全路由操作

可以用下列代码测试下，顺便看下源码实现就明白了

```java
		Collection<String> result = new HashSet<>(3, 1);

		result.add("t_user_info");
		result.add("t_user");
        	result.add("t_user_info_other");

		Collection<String> tableNames = result;
		for (Iterator<String> iterator = tableNames.iterator(); iterator.hasNext(); ) {
			String next =  iterator.next();
			System.out.println(next);
		}
```

### 修复

为了获取正确的表名（from后的第一个表名），需要使用有顺序的HashSet,即LinkedHashSet，底层使用LinkedHashMap，多了2个成员变量，指向首尾节点
来保证顺序

Tables类中：

3.0.0.M1版本：

```java
    public Collection<String> getTableNames() {
        Collection<String> result = new HashSet<>(tables.size(), 1);
        for (Table each : tables) {
            result.add(each.getName());
        }
        return result;
    }
```

3.0.0.M2版本中已经修复:

```java
    public Collection<String> getTableNames() {
        Collection<String> result = new LinkedHashSet<>(tables.size(), 1);
        for (Table each : tables) {
            result.add(each.getName());
        }
        return result;
    }
```

使用M2版本之后打印日志如下：

```youtrack
2018-09-25 18:46:45.993  INFO 9704 --- [nio-8080-exec-1] Sharding-Sphere-SQL                      : Logic SQL: SELECT o.*
        FROM t_user_info i
        JOIN t_user_info_other t ON i.user_id=t.user_id
        JOIN t_user o ON i.user_id=o.user_id
        WHERE i.user_id= ?
        AND i.type_id= ?
2018-09-25 18:46:45.994  INFO 9704 --- [nio-8080-exec-1] Sharding-Sphere-SQL                      : SQLStatement: SelectStatement(super=DQLStatement(super=AbstractSQLStatement(type=DQL, tables=Tables(tables=[Table(name=t_user_info, alias=Optional.of(i)), Table(name=t_user_info_other, alias=Optional.of(t)), Table(name=t_user, alias=Optional.of(o))]), conditions=Conditions(orCondition=OrCondition(andConditions=[AndCondition(conditions=[Condition(column=Column(name=user_id, tableName=t_user_info), operator=EQUAL, positionValueMap={}, positionIndexMap={0=0}), Condition(column=Column(name=type_id, tableName=t_user_info), operator=EQUAL, positionValueMap={}, positionIndexMap={0=1})])])), sqlTokens=[TableToken(beginPosition=24, skippedSchemaNameLength=0, originalLiterals=t_user_info), TableToken(beginPosition=51, skippedSchemaNameLength=0, originalLiterals=t_user_info_other), TableToken(beginPosition=107, skippedSchemaNameLength=0, originalLiterals=t_user)], parametersIndex=2)), containStar=false, selectListLastPosition=19, groupByLastPosition=0, items=[StarSelectItem(owner=Optional.of(o))], groupByItems=[], orderByItems=[], limit=null, subQueryStatement=null)
2018-09-25 18:46:45.994  INFO 9704 --- [nio-8080-exec-1] Sharding-Sphere-SQL                      : Actual SQL: ds-1 ::: SELECT o.*
        FROM t_user_info_1 i
        JOIN t_user_info_other_1 t ON i.user_id=t.user_id
        JOIN t_user_1 o ON i.user_id=o.user_id
        WHERE i.user_id= ?
        AND i.type_id= ? ::: [[1, 1]]
```

可以看出是正常的了
