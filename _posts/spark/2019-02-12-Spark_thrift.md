---
title: Spark Thrift 原理
tagline: "spark"
category : spark
layout: post
tags : [Spark thrift]
---
hive jdbc client


客户端执行sql入口statement.execute(sql);
```
HiveStatement.java
public boolean execute(String sql) throws SQLException {
  runAsyncOnServer(sql);
  TGetOperationStatusResp status = waitForOperationToComplete();

  // The query should be completed by now
  if (!status.isHasResultSet() && !stmtHandle.isHasResultSet()) {
    return false;
  }
  resultSet = new HiveQueryResultSet.Builder(this).setClient(client)
      .setStmtHandle(stmtHandle).setMaxRows(maxRows).setFetchSize(fetchSize)
      .setScrollable(isScrollableResultset)
      .build();
  return true;
}

private void runAsyncOnServer(String sql) throws SQLException {
  checkConnection("execute");

  reInitState();

  TExecuteStatementReq execReq = new TExecuteStatementReq(sessHandle, sql);
  /**
   * Run asynchronously whenever possible
   * Currently only a SQLOperation can be run asynchronously,
   * in a background operation thread
   * Compilation can run asynchronously or synchronously and execution run asynchronously
   */
  execReq.setRunAsync(true);
  execReq.setConfOverlay(sessConf);
  execReq.setQueryTimeout(queryTimeout);
  try {
    TExecuteStatementResp execResp = client.ExecuteStatement(execReq);
    Utils.verifySuccessWithInfo(execResp.getStatus());
    stmtHandle = execResp.getOperationHandle();
    isExecuteStatementFailed = false;
  } catch (SQLException eS) {
    isExecuteStatementFailed = true;
    isLogBeingGenerated = false;
    throw eS;
  } catch (Exception ex) {
    isExecuteStatementFailed = true;
    isLogBeingGenerated = false;
    throw new SQLException(ex.toString(), "08S01", ex);
  }
}
```
thrift client侧通过sendBase发送ExecuteStatement给服务端
```
//TCLIService.Iface
public TExecuteStatementResp ExecuteStatement(TExecuteStatementReq req) throws org.apache.thrift.TException
{
  send_ExecuteStatement(req);
  return recv_ExecuteStatement();
}

public void send_ExecuteStatement(TExecuteStatementReq req) throws org.apache.thrift.TException
{
  ExecuteStatement_args args = new ExecuteStatement_args();
  args.setReq(req);
  sendBase("ExecuteStatement", args);
}
```

服务端相关启动的类是CLASS="org.apache.spark.sql.hive.thriftserver.HiveThriftServer2"，入口函数main

```
  val server = new HiveThriftServer2(SparkSQLEnv.sqlContext)
  server.init(executionHive.conf)
  server.start()
```

init()会添加两种service,cliService，还有个thriftCLIService

```
override def init(hiveConf: HiveConf) {
  val sparkSqlCliService = new SparkSQLCLIService(this, sqlContext)
  setSuperField(this, "cliService", sparkSqlCliService)
  addService(sparkSqlCliService)

  val thriftCliService = if (isHTTPTransportMode(hiveConf)) {
    new ThriftHttpCLIService(sparkSqlCliService)
  } else {
    new ThriftBinaryCLIService(sparkSqlCliService)
  }

  setSuperField(this, "thriftCLIService", thriftCliService)
  addService(thriftCliService)
  initCompositeService(hiveConf)
}
```
cliService就是SparkSQLCLIService,thriftCLIService这里会有两种可选择,ThriftBinaryCLIService以及tcp模式的ThriftHttpCLIService

这两个都是封装的thrift相关的，按理thrift server服务直接看processor就好了，

```
at org.apache.hive.service.cli.session.SessionManager.submitBackgroundOperation(SessionManager.java:354)
at org.apache.spark.sql.hive.thriftserver.SparkExecuteStatementOperation.runInternal(SparkExecuteStatementOperation.scala:196)
at org.apache.hive.service.cli.operation.Operation.run(Operation.java:257)
at org.apache.hive.service.cli.session.HiveSessionImpl.executeStatementInternal(HiveSessionImpl.java:388)
at org.apache.hive.service.cli.session.HiveSessionImpl.executeStatementAsync(HiveSessionImpl.java:375)
at org.apache.hive.service.cli.CLIService.executeStatementAsync(CLIService.java:275)
at org.apache.hive.service.cli.thrift.ThriftCLIService.ExecuteStatement(ThriftCLIService.java:436)
at org.apache.hive.service.cli.thrift.TCLIService$Processor$ExecuteStatement.getResult(TCLIService.java:1313)
at org.apache.hive.service.cli.thrift.TCLIService$Processor$ExecuteStatement.getResult(TCLIService.java:1298)
at org.apache.thrift.ProcessFunction.process(ProcessFunction.java:39)
at org.apache.thrift.TBaseProcessor.process(TBaseProcessor.java:39)
at org.apache.hive.service.auth.TSetIpAddressProcessor.process(TSetIpAddressProcessor.java:53)
at org.apache.thrift.server.TThreadPoolServer$WorkerProcess.run(TThreadPoolServer.java:286)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
```

先看ThriftHttpCLIService，在其run函数里边启动jettyserver,processor是TCLIService.Processor

```
 TProcessor processor = new TCLIService.Processor<Iface>(this);
 TServlet thriftHttpServlet = new ThriftHttpServlet(processor, protocolFactory, authType,
    serviceUGI, httpUGI);
// Context handler
final ServletContextHandler context = new ServletContextHandler(
    ServletContextHandler.SESSIONS);
context.setContextPath("/");
String httpPath = getHttpPath(hiveConf
    .getVar(HiveConf.ConfVars.HIVE_SERVER2_THRIFT_HTTP_PATH));
httpServer.setHandler(context);
context.addServlet(new ServletHolder(thriftHttpServlet), httpPath);
```
在TServlet的post看到processor的处理逻辑

```
protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    TTransport inTransport = null;
    Object var4 = null;

    try {
        response.setContentType("application/x-thrift");
        if (null != this.customHeaders) {
            Iterator i$ = this.customHeaders.iterator();

            while(i$.hasNext()) {
                Entry<String, String> header = (Entry)i$.next();
                response.addHeader((String)header.getKey(), (String)header.getValue());
            }
        }

        InputStream in = request.getInputStream();
        OutputStream out = response.getOutputStream();
        TTransport transport = new TIOStreamTransport(in, out);
        TProtocol inProtocol = this.inProtocolFactory.getProtocol(transport);
        TProtocol outProtocol = this.outProtocolFactory.getProtocol(transport);
        this.processor.process(inProtocol, outProtocol);
        out.flush();
    } catch (TException var10) {
        throw new ServletException(var10);
    }
}
```
ThriftBinaryCLIService里边的不是直接使用的processor 而是processorFactory

```
TTransportFactory transportFactory = hiveAuthFactory.getAuthTransFactory();
TProcessorFactory processorFactory = hiveAuthFactory.getAuthProcFactory(this);
TThreadPoolServer.Args sargs = new TThreadPoolServer.Args(serverSocket)
    .processorFactory(processorFactory).transportFactory(transportFactory)
    .protocolFactory(new TBinaryProtocol.Factory())
    .inputProtocolFactory(new TBinaryProtocol.Factory(true, true, maxMessageSize, maxMessageSize))
    .requestTimeout(requestTimeout).requestTimeoutUnit(TimeUnit.SECONDS)
    .beBackoffSlotLength(beBackoffSlotLength).beBackoffSlotLengthUnit(TimeUnit.MILLISECONDS)
    .executorService(executorService);

// TCP Server
server = new TThreadPoolServer(sargs);
server.setServerEventHandler(serverEventHandler);
String msg = "Starting " + ThriftBinaryCLIService.class.getSimpleName() + " on port "
    + portNum + " with " + minWorkerThreads + "..." + maxWorkerThreads + " worker threads";
LOG.info(msg);
server.serve();
```

这个factory也份两种形式 

```
public TProcessorFactory getAuthProcFactory(ThriftCLIService service) throws LoginException {
  if (authTypeStr.equalsIgnoreCase(AuthTypes.KERBEROS.getAuthName())) {
    return KerberosSaslHelper.getKerberosProcessorFactory(saslServer, service);
  } else {
    return PlainSaslHelper.getPlainProcessorFactory(service);
  }
}
//KerberosSaslHelper
public static TProcessorFactory getKerberosProcessorFactory(Server saslServer,
  ThriftCLIService service) {
  return new CLIServiceProcessorFactory(saslServer, service);
}
//PlainSaslHelper
public static TProcessorFactory getPlainProcessorFactory(ThriftCLIService service) {
  return new SQLPlainProcessorFactory(service);
}
```
CLIServiceProcessorFactory与SQLPlainProcessorFactory都继承自TProcessorFactory

```
private KerberosSaslHelper() {
  throw new UnsupportedOperationException("Can't initialize class");
}

private static class CLIServiceProcessorFactory extends TProcessorFactory {

  private final ThriftCLIService service;
  private final Server saslServer;

  CLIServiceProcessorFactory(Server saslServer, ThriftCLIService service) {
    super(null);
    this.service = service;
    this.saslServer = saslServer;
  }

  @Override
  public TProcessor getProcessor(TTransport trans) {
    TProcessor sqlProcessor = new TCLIService.Processor<Iface>(service);
    return saslServer.wrapNonAssumingProcessor(sqlProcessor);
  }
}
private static final class SQLPlainProcessorFactory extends TProcessorFactory {

  private final ThriftCLIService service;

  SQLPlainProcessorFactory(ThriftCLIService service) {
    super(null);
    this.service = service;
  }

  @Override
  public TProcessor getProcessor(TTransport trans) {
    return new TSetIpAddressProcessor<Iface>(service);
  }
}
```
TSetIpAddressProcessor也是继承自TCLIService.Processor，这里讲了这么多Processor，看下里边具体的定义

```
private static <I extends Iface> Map<String,  org.apache.thrift.ProcessFunction<I, ? extends  org.apache.thrift.TBase>> getProcessMap(Map<String,  org.apache.thrift.ProcessFunction<I, ? extends  org.apache.thrift.TBase>> processMap) {
  processMap.put("OpenSession", new OpenSession());
  processMap.put("CloseSession", new CloseSession());
  processMap.put("GetInfo", new GetInfo());
  processMap.put("ExecuteStatement", new ExecuteStatement());
  processMap.put("GetTypeInfo", new GetTypeInfo());
  processMap.put("GetCatalogs", new GetCatalogs());
  processMap.put("GetSchemas", new GetSchemas());
  processMap.put("GetTables", new GetTables());
  processMap.put("GetTableTypes", new GetTableTypes());
  processMap.put("GetColumns", new GetColumns());
  processMap.put("GetFunctions", new GetFunctions());
  processMap.put("GetOperationStatus", new GetOperationStatus());
  processMap.put("CancelOperation", new CancelOperation());
  processMap.put("CloseOperation", new CloseOperation());
  processMap.put("GetResultSetMetadata", new GetResultSetMetadata());
  processMap.put("FetchResults", new FetchResults());
  processMap.put("GetDelegationToken", new GetDelegationToken());
  processMap.put("CancelDelegationToken", new CancelDelegationToken());
  processMap.put("RenewDelegationToken", new RenewDelegationToken());
  return processMap;
}
```
ExecuteStatement应该不陌生，就是前面client提交过来的method了，在里还有这么一段代码

```
public boolean process(TProtocol in, TProtocol out) throws TException {
    TMessage msg = in.readMessageBegin();
    ProcessFunction fn = (ProcessFunction)this.processMap.get(msg.name);
    if (fn == null) {
        TProtocolUtil.skip(in, (byte)12);
        in.readMessageEnd();
        TApplicationException x = new TApplicationException(1, "Invalid method name: '" + msg.name + "'");
        out.writeMessageBegin(new TMessage(msg.name, (byte)3, msg.seqid));
        x.write(out);
        out.writeMessageEnd();
        out.getTransport().flush();
        return true;
    } else {
        fn.process(msg.seqid, in, out, this.iface);
        return true;
    }
}
//ProcessFunction
public final void process(int seqid, TProtocol iprot, TProtocol oprot, I iface) throws TException {
    TBase args = this.getEmptyArgsInstance();

    try {
        args.read(iprot);
    } catch (TProtocolException var10) {
        iprot.readMessageEnd();
        TApplicationException x = new TApplicationException(7, var10.getMessage());
        oprot.writeMessageBegin(new TMessage(this.getMethodName(), (byte)3, seqid));
        x.write(oprot);
        oprot.writeMessageEnd();
        oprot.getTransport().flush();
        return;
    }

    iprot.readMessageEnd();
    TBase result = null;

    try {
        result = this.getResult(iface, args);
    } catch (TException var9) {
        LOGGER.error("Internal error processing " + this.getMethodName(), var9);
        TApplicationException x = new TApplicationException(6, "Internal error processing " + this.getMethodName());
        oprot.writeMessageBegin(new TMessage(this.getMethodName(), (byte)3, seqid));
        x.write(oprot);
        oprot.writeMessageEnd();
        oprot.getTransport().flush();
        return;
    }

    if (!this.isOneway()) {
        oprot.writeMessageBegin(new TMessage(this.getMethodName(), (byte)2, seqid));
        result.write(oprot);
        oprot.writeMessageEnd();
        oprot.getTransport().flush();
    }

}
```
这个getResult是关键,就是map对应的，这里以上面的ExecuteStatement为例
```
  public ExecuteStatement_result getResult(I iface, ExecuteStatement_args args) throws org.apache.thrift.TException {
      ExecuteStatement_result result = new ExecuteStatement_result();
      result.success = iface.ExecuteStatement(args.req);
      return result;
    }
    }
```
iface就是服务端实现了

```
//ThriftCLIService
public TExecuteStatementResp ExecuteStatement(TExecuteStatementReq req) throws TException {
  TExecuteStatementResp resp = new TExecuteStatementResp();
  try {
    SessionHandle sessionHandle = new SessionHandle(req.getSessionHandle());
    String statement = req.getStatement();
    Map<String, String> confOverlay = req.getConfOverlay();
    Boolean runAsync = req.isRunAsync();
    OperationHandle operationHandle = runAsync ?
        cliService.executeStatementAsync(sessionHandle, statement, confOverlay)
        : cliService.executeStatement(sessionHandle, statement, confOverlay);
        resp.setOperationHandle(operationHandle.toTOperationHandle());
        resp.setStatus(OK_STATUS);
  } catch (Exception e) {
    LOG.warn("Error executing statement: ", e);
    resp.setStatus(HiveSQLException.toTStatus(e));
  }
  return resp;
}
//CLIService
public OperationHandle executeStatementAsync(SessionHandle sessionHandle, String statement,
    Map<String, String> confOverlay) throws HiveSQLException {
  OperationHandle opHandle = sessionManager.getSession(sessionHandle)
      .executeStatementAsync(statement, confOverlay);
  LOG.debug(sessionHandle + ": executeStatementAsync()");
  return opHandle;
}
//SessionManager
public HiveSession getSession(SessionHandle sessionHandle) throws HiveSQLException {
  HiveSession session = handleToSession.get(sessionHandle);
  if (session == null) {
    throw new HiveSQLException("Invalid SessionHandle: " + sessionHandle);
  }
  return session;
}
```

这个cliService不就是init里边放进去的SparkSQLCLIService吗，其父类是CLIService，注意区分前面的TCLIService

这里需要关注下HiveThriftServer2初始化方法init()里的一段代码

```
override def init(hiveConf: HiveConf) {
  val sparkSqlCliService = new SparkSQLCLIService(this, sqlContext)
  setSuperField(this, "cliService", sparkSqlCliService)
  addService(sparkSqlCliService)

  val thriftCliService = if (isHTTPTransportMode(hiveConf)) {
    new ThriftHttpCLIService(sparkSqlCliService)
  } else {
    new ThriftBinaryCLIService(sparkSqlCliService)
  }

  setSuperField(this, "thriftCLIService", thriftCliService)
  addService(thriftCliService)
  initCompositeService(hiveConf)
}
```
就是最后这个 initCompositeService(hiveConf)

```
private[thriftserver] trait ReflectedCompositeService { this: AbstractService =>
  def initCompositeService(hiveConf: HiveConf) {
    // Emulating `CompositeService.init(hiveConf)`
    val serviceList = getAncestorField[JList[Service]](this, 2, "serviceList")
    serviceList.asScala.foreach(_.init(hiveConf))

    // Emulating `AbstractService.init(hiveConf)`
    invoke(classOf[AbstractService], this, "ensureCurrentState", classOf[STATE] -> STATE.NOTINITED)
    setAncestorField(this, 3, "hiveConf", hiveConf)
    invoke(classOf[AbstractService], this, "changeState", classOf[STATE] -> STATE.INITED)
    getAncestorField[Log](this, 3, "LOG").info(s"Service: $getName is inited.")
  }
}
```
会通过反射调用serviceList的init(),一开始添加的两个service都会在这里被init，

看下SparkSQLCLIService的init过程

```
override def init(hiveConf: HiveConf) {
  setSuperField(this, "hiveConf", hiveConf)

  val sparkSqlSessionManager = new SparkSQLSessionManager(hiveServer, sqlContext)
  setSuperField(this, "sessionManager", sparkSqlSessionManager)
  addService(sparkSqlSessionManager)
  var sparkServiceUGI: UserGroupInformation = null

  if (UserGroupInformation.isSecurityEnabled) {
    try {
      HiveAuthFactory.loginFromKeytab(hiveConf)
      sparkServiceUGI = Utils.getUGI()
      setSuperField(this, "serviceUGI", sparkServiceUGI)
    } catch {
      case e @ (_: IOException | _: LoginException) =>
        throw new ServiceException("Unable to login to kerberos with given principal/keytab", e)
    }
  }

  initCompositeService(hiveConf)
}
```
这里同样也会调用SparkSQLSessionManager的init吧，

```
//SparkSQLSessionManager
override def init(hiveConf: HiveConf) {
  setSuperField(this, "hiveConf", hiveConf)

  // Create operation log root directory, if operation logging is enabled
  if (hiveConf.getBoolVar(ConfVars.HIVE_SERVER2_LOGGING_OPERATION_ENABLED)) {
    invoke(classOf[SessionManager], this, "initOperationLogRootDir")
  }

  val backgroundPoolSize = hiveConf.getIntVar(ConfVars.HIVE_SERVER2_ASYNC_EXEC_THREADS)
  setSuperField(this, "backgroundOperationPool", Executors.newFixedThreadPool(backgroundPoolSize))
  getAncestorField[Log](this, 3, "LOG").info(
    s"HiveServer2: Async execution pool size $backgroundPoolSize")

  setSuperField(this, "operationManager", sparkSqlOperationManager)
  addService(sparkSqlOperationManager)

  initCompositeService(hiveConf)
}
```

这个初始化里添加的service是SparkSQLOperationManager,但是SparkSQLOperationManager本身没有init，只有父类有了

```
private[thriftserver] class SparkSQLOperationManager()
  extends OperationManager with Logging {

  val handleToOperation = ReflectionUtils
    .getSuperField[JMap[OperationHandle, Operation]](this, "handleToOperation")

  val sessionToActivePool = new ConcurrentHashMap[SessionHandle, String]()
  val sessionToContexts = new ConcurrentHashMap[SessionHandle, SQLContext]()

  override def newExecuteStatementOperation(
      parentSession: HiveSession,
      statement: String,
      confOverlay: JMap[String, String],
      async: Boolean): ExecuteStatementOperation = synchronized {
    val sqlContext = sessionToContexts.get(parentSession.getSessionHandle)
    require(sqlContext != null, s"Session handle: ${parentSession.getSessionHandle} has not been" +
      s" initialized or had already closed.")
    val conf = sqlContext.sessionState.conf
    val runInBackground = async && conf.getConf(HiveUtils.HIVE_THRIFT_SERVER_ASYNC)
    val operation = new SparkExecuteStatementOperation(parentSession, statement, confOverlay,
      runInBackground)(sqlContext, sessionToActivePool)
    handleToOperation.put(operation.getHandle, operation)
    logDebug(s"Created Operation for $statement with session=$parentSession, " +
      s"runInBackground=$runInBackground")
    operation
  }
```

上边的sessionManager应该是SparkSQLSessionManager，这里里边除了init只有两个方法了openSession与closeSession，其实还是调用的父类方法，
只是封装了下，那获取session就不难找了

```
  public SessionHandle openSession(TProtocolVersion protocol, String username, String password, String ipAddress,
      Map<String, String> sessionConf, boolean withImpersonation, String delegationToken)
          throws HiveSQLException {
    HiveSession session;
    // If doAs is set to true for HiveServer2, we will create a proxy object for the session impl.
    // Within the proxy object, we wrap the method call in a UserGroupInformation#doAs
    if (withImpersonation) {
      HiveSessionImplwithUGI sessionWithUGI = new HiveSessionImplwithUGI(protocol, username, password,
          hiveConf, ipAddress, delegationToken);
      session = HiveSessionProxy.getProxy(sessionWithUGI, sessionWithUGI.getSessionUgi());
      sessionWithUGI.setProxySession(session);
    } else {
      session = new HiveSessionImpl(protocol, username, password, hiveConf, ipAddress);
    }
    session.setSessionManager(this);
    session.setOperationManager(operationManager);
    try {
      session.open(sessionConf);
    } catch (Exception e) {
      try {
        session.close();
      } catch (Throwable t) {
        LOG.warn("Error closing session", t);
      }
      session = null;
      throw new HiveSQLException("Failed to open new session: " + e, e);
    }
    if (isOperationLogEnabled) {
      session.setOperationLogSessionDir(operationLogRootDir);
    }
    handleToSession.put(session.getSessionHandle(), session);
    return session.getSessionHandle();
  }
```
回到之前调用的代码

```
public OperationHandle executeStatementAsync(SessionHandle sessionHandle, String statement,
    Map<String, String> confOverlay) throws HiveSQLException {
  OperationHandle opHandle = sessionManager.getSession(sessionHandle)
      .executeStatementAsync(statement, confOverlay);
  LOG.debug(sessionHandle + ": executeStatementAsync()");
  return opHandle;
}
```
这里就是HiveSessionImpl里的调用了

```
public OperationHandle executeStatementAsync(String statement, Map<String, String> confOverlay)
    throws HiveSQLException {
  return executeStatementInternal(statement, confOverlay, true);
}
  private OperationHandle executeStatementInternal(String statement, Map<String, String> confOverlay,
      boolean runAsync)
          throws HiveSQLException {
    acquire(true);

    OperationManager operationManager = getOperationManager();
    ExecuteStatementOperation operation = operationManager
        .newExecuteStatementOperation(getSession(), statement, confOverlay, runAsync);
    OperationHandle opHandle = operation.getHandle();
    try {
      operation.run();
      opHandleSet.add(opHandle);
      return opHandle;
    } catch (HiveSQLException e) {
      // Refering to SQLOperation.java,there is no chance that a HiveSQLException throws and the asyn
      // background operation submits to thread pool successfully at the same time. So, Cleanup
      // opHandle directly when got HiveSQLException
      operationManager.closeOperation(opHandle);
      throw e;
    } finally {
      release(true);
    }
  }
//Operation
public void run() throws HiveSQLException {
  beforeRun();
  try {
    runInternal();
  } finally {
    afterRun();
  }
}
```

这个Operation是啥呢，就是SparkExecuteStatementOperation,绕了一圈这里才是熟悉的sqlContext.sql()

```
override def runInternal(): Unit = {
 //...
  if (!runInBackground) {
    execute()
  } else {
  //...
 }
 
  private def execute(): Unit = {
  //...
  result = sqlContext.sql(statement)
  //...
  }
```
