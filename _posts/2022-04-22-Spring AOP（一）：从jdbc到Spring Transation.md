---
title: Spring AOP（一）：从jdbc到Spring Transation
date: 2022-04-22 02:55:15 +0800
categories: [源码阅读]
tags: [Java, Spring, AOP, Transation, Jdbc]
---


## 前言

今天同事提出一套观点：
1. Transation、Durid、jdbc中各有一个autoCommit配置项，相互不影响。
2. Druid默认设置autoCommit为true。
3. Spring开启事务时，Spring的autoCommit被设置为false。
4. 由于Spring没有修改Durid的autoCommit状态，JDBC执行方式不受影响，相当于每次查询都是直接提交。
5. JDBC未开启事务，每次查询会调用connection.commit()。

并得出结论：
- 要将Durid启动参数中的autoCommit设置为flase，spring事务才会生效。

## 同事的分析过程

首先，Durid在初始化时可以配置autoCommit，并且在初始化连接时会配置JDBC的autoCommit。
![2-1 Druid配置](/assets/img/20200422/Druid_AutoCommitConfig.png)_2-1 Druid配置_
![2-2 Druid初始化连接](/assets/img/20200422/Druid_init.png)_2-2 Druid初始化连接_

接着，Spring在commit时判断自身状态，仅autoCommit开启时有效。
![2-3 Spirng commit](/assets/img/20200422/Druid_AutoCommitConfig.png)_2-3 Spirng commit_

这样看起来，观点1、观点2都是对的。  
即使这个结论给的很粗糙，但结合当时在处理问题的情景，对autoCommit的概念又不熟悉，确实很难反驳。但已经隐隐感觉到这个结论有点问题。于是就有了以下的分析。

## 我的分析

因为不熟，所以先从AutoCommit的使用开始了解整个过程。

### JDBC的事务原理
 
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
        // 调用Executor方法
        this.connection.getQueryExecutor().execute(queryToExecute, queryParameters, handler, this.maxrows, this.fetchSize, flags);
    } finally {
        this.killTimerTask();
    }

    ... // 一些连接关闭操作，暂且忽略
}
```
从`java.sql.Statement#execute()`一路跟踪调用链，我们找到了`org.postgresql.jdbc.PgStatement#executeInternal`，虽然不知道flags是什么，但可以看见一段关键代码：开启autoCommit时，flags的第5位标志位被设置为1，然后调用Executor的execute方法执行真正的查询操作。那么接下来只要找到类似判断`flags & 16 == 1`的地方即可。

追踪`this.connection.getQueryExecutor().execute`不难看出这里就是具体发送sql命令的地方，限于篇幅问题就不再展开讲解了，这里可以进debug看看各个参数的值都是什么，我把关键点在注释里标注出来。
```java
public synchronized void execute(Query query, @Nullable ParameterList parameters, ResultHandler handler, int maxRows, int fetchSize, int flags) throws SQLException {
    this.waitOnLock();
    if (LOGGER.isLoggable(Level.FINEST)) {
        LOGGER.log(Level.FINEST, "  simple execute, handler={0}, maxRows={1}, fetchSize={2}, flags={3}", new Object[]{handler, maxRows, fetchSize, flags});
    }

    if (parameters == null) {
        parameters = SimpleQuery.NO_PARAMETERS;
    }

    flags = this.updateQueryMode(flags);
    boolean describeOnly = (32 & flags) != 0;
    ((V3ParameterList)parameters).convertFunctionOutParameters();
    if (!describeOnly) {
        ((V3ParameterList)parameters).checkAllParametersSet();
    }

    boolean autosave = false;

    try {
        try {
            // autoCommit=ture，并且事务处于预备状态（也就是当前connet还没有执行过），发送begin
            handler = this.sendQueryPreamble(handler, flags);
            // autoCommit=ture时，发送savePoint "SAVEPOINT PGJDBC_AUTOSAVE"
            autosave = this.sendAutomaticSavepoint(query, flags);
            // 执行具体Query
            this.sendQuery(query, (V3ParameterList)parameters, maxRows, fetchSize, flags, handler, (BatchResultHandler)null);
            if ((flags & 1024) == 0) {
                this.sendSync();
            }

            this.processResults(handler, flags);
            this.estimatedReceiveBufferBytes = 0;
        } catch (PGBindException var11) {
            this.sendSync();
            this.processResults(handler, flags);
            this.estimatedReceiveBufferBytes = 0;
            handler.handleError(new PSQLException(GT.tr("Unable to bind parameter values for statement.", new Object[0]), PSQLState.INVALID_PARAMETER_VALUE, var11.getIOException()));
        }
    } catch (IOException var12) {
        this.abort();
        handler.handleError(new PSQLException(GT.tr("An I/O error occurred while sending to the backend.", new Object[0]), PSQLState.CONNECTION_FAILURE, var12));
    }

    try {
        // 
        handler.handleCompletion();
        if (this.cleanupSavePoints) {
            this.releaseSavePoint(autosave, flags);
        }
    } catch (SQLException var10) {
        // autoCommit=ture时发生异常，回滚到savePoint "ROLLBACK TO SAVEPOINT PGJDBC_AUTOSAVE"，并抛出异常
        this.rollbackIfRequired(autosave, var10);
    }

}
```
到这一步就很清晰了，JDBC在每次调用execute方法时都去判断是否需要添加`begin`和`savePoint`，没有添加`commit`命令的原因也很简单，假如没有开启事务，数据库只要Query语句就够了；开启事务时，业务需要手动调用commit。

### SpringManagedTransaction

同事没有告诉我他是怎么找到这个类的，代码也跟踪不到这里，尝试使用IDE find usages的功能，也没找到调用端。于是只能先百度下，结果发现，这个类是Mybatis用来实现在Spring中管理多数据源的功能的。所属的包也是`org.mybatis.spring.transaction.SpringManagedTransaction`。

摆了个乌龙，感觉也没必要深入看了。

## 初步结论

经过上面分析，可以得出结论*观点3、观点5是错误的*。    

反观同事的结论，我们应该配置Durid的AutoCommit状态为false吗？其实思考下就明白不行，这样相当于所有查询都自动加上了`conn.setAutoCommit(false)`，那一些简单的select方法由于没有手动调用commit，会直接执行异常。*观点6也是错误的。*

但观点4我们还没搞明白：Spring有没有修改Durid的autoCommit状态？是不是每次查询都相当于直接提交？这个问题应该可以在Spring的事务逻辑中找到答案。

## Spring Transation

要跟踪Spring的事务其实很简单，只要写一段demo，debug调试下就能从堆栈中看到了。

```java
@RestController
public class SimpleController {

    @Autowired
    private SimpleService service;
    
    @RequestMapping("/tran")
    public void tran(){
        service.serve();
    }
    
}
```
```java
@Service
public class SimpleService {

    @Transactional
    public void serve(){
        System.out.printf("hello");
    }
}
```
我们在`System.out.printf("hello")`打个断点观察堆栈。
![5-1 debug](/assets/img/20200422/transaction_debug.png)_5-1 transaction debug_
从SimpleController开始，经过Spring创建的代理类，进入了事务切面方法。
```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    @Nullable
    protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
            final InvocationCallback invocation) throws Throwable {
        
        ... 

        if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
            // 创建了一个标准事务
            TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

            Object retVal;
            try {
                // invoke执行业务方法
                retVal = invocation.proceedWithInvocation();
            }
            catch (Throwable ex) {
                // 异常rollback
                completeTransactionAfterThrowing(txInfo, ex);
                throw ex;
            }
            finally {
                cleanupTransactionInfo(txInfo);
            }

            if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
                // Set rollback-only in case of Vavr failure matching our rollback rules...
                TransactionStatus status = txInfo.getTransactionStatus();
                if (status != null && txAttr != null) {
                    retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
                }
            }

            // 正常commit
            commitTransactionAfterReturning(txInfo);
            return retVal;
        }
    }
}
```
跟踪createTransactionIfNecessary方法。
```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
            @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

        ...

        TransactionStatus status = null;
        if (txAttr != null) {
            if (tm != null) {
                // 获取事务
                status = tm.getTransaction(txAttr);
        
        ...
    }
}
```
```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    @Override
    public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
            throws TransactionException {

        ...

            try {
                // 开启事务
                return startTransaction(def, transaction, debugEnabled, suspendedResources);
            }
            catch (RuntimeException | Error ex) {
                resume(null, suspendedResources);
                throw ex;
            }
        
        ...
    }
}
```
```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    /**
        * Start a new transaction.
        */
    private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
            boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {

        boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
        DefaultTransactionStatus status = newTransactionStatus(
                definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
        // 找到begin了
        doBegin(transaction, definition);
        prepareSynchronization(status, definition);
        return status;
    }
}
```
```java
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager implements ResourceTransactionManager, InitializingBean {
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        ...

            if (con.getAutoCommit()) {
                txObject.setMustRestoreAutoCommit(true);
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
                }

                con.setAutoCommit(false);
            }

        ...
    }
}
```

最终，我们在这里找到了doBegin方法。Spring调用了connect.setAutoCommit(false)，接着就是前面分析过的，JDBC会在第一次执行时发送begin命令。

## 结论

Spring没有修改Durid的autoCommit状态，并且跟其他任何的autoCommit配置都无关，因为他会在执行前判断连接的autoCommit状态，如果开启会强制关闭。

## 后续思考

至此，我们从JDBC开始到Spring梳理了一遍事务的执行过程，这部分的逻辑还是比较清晰的，关键是要熟练掌握源码阅读的方法和技巧。

然而，我们还可以把问题继续问下去：为什么Spring可以这么暴力，直接将autoCommit设置成false，不会有不需要使用事务的场景吗？这个问题涉及到了整个AOP的加载和执行过程，我会在下一篇文章中展开讨论。
