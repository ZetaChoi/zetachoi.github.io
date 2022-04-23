---
title: 从jdbc到Spring-tx，分析事务如何产生效果。（施工中）
date: 2022-04-22 02:55:15 +0800
categories: [JAVA,源码阅读]
tags: [Java, Spring, AOP, Transation, Jdbc, Mysql, CGLib]
---


## 前言

今天同事提出一个观点：Spring Transation、Mybatis、jdbc中各有一个autoCommit配置项，代码中添加@Transation注解时Spring会设置自己的AutoCommit为false(关闭)，但由于Druid(连接池会创建jdbc connnect)默认为true(开启)，每一条sql查询还是会直接提交（相当于调用connection.comit()），导致事务失效。    
那么我们就具体问题具体分析，通过自己阅读源码证实下这个上述结论的正确与否。

## 源码分析
### JDBC的autoCommit
 
我们一般会这么使用JDBC：
```java
// 未开启事务
Connection conn = DriverManager.getConnection("url", "user", "paasword")
try{
    Statement stmt = conn.createStatement();
    stmt.execute("insert into xxx values ()");
}catch(Exception e){
    e.printStackTrace();
}finally {
    conn.close();
}

// 开启事务
Connection conn = DriverManager.getConnection("url", "user", "paasword")
try{
    conn.setAutoCommit(false);

    Statement stmt = conn.createStatement();
    stmt.execute("insert into xxx values ()");

    conn.commit();
}catch(Exception e){
    conn.rollback();
}finally {
    conn.close();
}
```

要知道，数据库开启事务的方式是使用命令`begin`，JDBC没有通过API体现出来，说明他通过控制autoCommit的开关，影响了execute方法的执行逻辑。JDBC是一个通用协议，具体的执行由JdbcDriver提供，其实每种驱动实现方式都大同小异，这里展示比较简洁易懂的pgsql驱动代码。
```java
private void executeInternal(CachedQuery cachedQuery, @Nullable ParameterList queryParameters, int flags) throws SQLException {
    this.closeForNextExecution();
    if (this.fetchSize > 0 && !this.wantsScrollableResultSet() && !this.connection.getAutoCommit() && !this.wantsHoldableResultSet()) {
        flags |= 8;
    }

    if (this.connection.getAutoCommit()) {
        flags |= 16; // 0x0000 | 0x0010 = 0x0010
    }

    ... // 中间是一些对flags的赋值和特殊条件下的sql执行，暂且忽略

    PgStatement.StatementResultHandler handler = new PgStatement.StatementResultHandler();
    synchronized(this) {
        this.result = null;
    }

    try {
        this.startTimer();
        this.connection.getQueryExecutor().execute(queryToExecute, queryParameters, handler, this.maxrows, this.fetchSize, flags);
    } finally {
        this.killTimerTask();
    }

    ... // 一些连接关闭操作，暂且忽略
}
```
一路跟踪execute方法的调用链，我们找到了```org.postgresql.jdbc.PgStatement#executeInternal```，此时虽然不知道flags是什么，但我们看见了关键代码，开启autoCommit时，flags的第5位被设置为1，然后调用Executor的execute方法执行真正的查询操作。那么接下来只要找到类似判断`flags & 16 == 1`的地方即可。


## 思考

