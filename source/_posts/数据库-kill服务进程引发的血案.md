title: 数据库-kill服务进程引发的血案
author: Xiang Chuang
tags:
  - db
categories:
  - 这些年，那些坑
date: 2018-07-30 11:01:00
---
1、问题描述：
18.07.27， 10：50分dba进行mysql数据库碎片整理，11：04分，发现数据库连接爆满，此时应用已处于down掉的状态，11：22数据库强行kill，11：40分之前发现应用并没有进行新的连接，此时停止应用发现停不下来，不得不kill掉应用进程，重启后恢复。但此时发现，之前入库的记录并没有成功，而mq由于没有收到应用的ack确认消息，消息重新进入队列并重发，加上数据没入库，交易进行了重复的操作！由于是全名抢车业务，导致重复上账，重复充退。

2、问题分析
由于没有及时的dump保护现场，加上应用down掉后，日志少的可怜，只有一些Broken pipe，以及该getOuptStream()异常，后期只能从代码层面分析。

3、分析
jdbc sockeTimeout时间并没有设置，所以默认使用的mysql的或者操作系统本身的socketTimeout 该问题会引发类似问题

相关文档:
http://www.sohu.com/a/142039227_505885
http://www.sohu.com/a/141155649_539864?qq-pf-to=pcqq.c2c 
https://blog.csdn.net/lx348321409/article/details/76095751 
https://segmentfault.com/a/1190000012944562
https://www.cubrid.org/blog/understanding-jdbc-internals-and-timeout-configuration

=====================================================
jdbc timeout类别

主要有如下几个类别

![upload successful](\images\pasted-16.png)
transaction timeout
设置的是一个事务的执行时间，里头可能包含多个statement
statement timeout(也相当于result set fetch timeout)
设置的是一个statement的执行超时时间，即driver等待statement执行完成，接收到数据的超时时间(注意statement的timeout不是整个查询的timeout，只是statement执行完成并拉取fetchSize数据返回的超时，之后resultSet的next在必要的时候还会触发fetch数据，每次fetch的超时时间是单独算的，默认也是以statement设置的timeout为准)
jdbc socket timeout
设置的是jdbc I/O socket read and write operations的超时时间，防止因网络问题或数据库问题，导致driver一直阻塞等待。(建议比statement timeout的时间长)
os socket timeout
这个是操作系统级别的socket设置(如果jdbc socket timeout没有设置，而os级别的socket timeout有设置，则使用系统的socket timeout值)。
上面的不同级别的timeout越往下优先级越高，也就是说如果下面的配置比上面的配置值小的话，则会优先触发timeout，那么相当于上面的配置值就"失效"了。
jdbc socket timeout

这个不同数据的jdbc driver实现不一样
mysql

jdbc:mysql://localhost:3306/ag_admin?useUnicode=true&amp;characterEncoding=UTF8&connectTimeout=60000&socketTimeout=60000
通过url参数传递即可
pg

jdbc:postgresql://localhost/test?user=fred&password=secret&&connectTimeout=60&socketTimeout=60
pg也是通过url传递，不过它的单位与mysql不同，mysql是毫秒，而pg是秒
oracle

oracle需要通过oracle.jdbc.ReadTimeout参数来设置，连接超时参数是oracle.net.CONNECT_TIMEOUT
通过properties设置
            Class.forName("oracle.jdbc.driver.OracleDriver");
            Properties props = new Properties() ;
            props.put( "user" , "test_schema") ;
            props.put( "password" , "pwd") ;
            props.put( "oracle.net.CONNECT_TIMEOUT" , "10000000") ;
            props.put( "oracle.jdbc.ReadTimeout" , "2000" ) ;
            Connection conn = DriverManager.getConnection( "jdbc:oracle:thin:@10.0.1.9:1521:orcl" , props ) ;
通过环境变量设置
String readTimeout = "10000"; // ms
System.setProperty("oracle.jdbc.ReadTimeout", readTimeout);
Class.forName("oracle.jdbc.OracleDriver");
Connection conn = DriverManager.getConnection(jdbcUrl, user, pwd);
注意需要在connection连接之前设置环境变量
tomcat jdbc pool
一般我们不直接使用jdbc connection，而是使用连接池。由于tomcat jdbc pool是springboot默认使用的数据库连接池，这里就讲述一下如何在tomcat jdbc pool下设置。
spring.datasource.tomcat.connectionProperties=oracle.net.CONNECT_TIMEOUT=10000;oracle.jdbc.ReadTimeout=60000
注意，这里是分号分隔，单位是毫秒，这里可以根据各自的情况配置前缀(tomcat jdbc连接池的话，默认是spring.datasource.tomcat)，可以自定义，比如
@Bean@Qualifier("writeDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.write")
    public DataSource writeDataSource() {
        returnDataSourceBuilder.create().build();
    }
假设你这里是自定义了prefix为spring.datasource.write，那么上述配置就变为
spring.datasource.write.connectionProperties=oracle.net.CONNECT_TIMEOUT=10000;oracle.jdbc.ReadTimeout=60000
oracle.jdbc.ReadTimeout如果没有设置的话，driver里头默认是0
oracle.jdbc.ReadTimeout

driver内部将该值设置到oracle.net.READ_TIMEOUT变量上
oracle.net.nt.TcpNTAdapter
    @Override
    publicvoid setReadTimeoutIfRequired(final Properties properties) throws IOException, NetException {
        String s = ((Hashtable<K, String>)properties).get("oracle.net.READ_TIMEOUT");
        if (s == null) {
            s = "0";
        }
        this.setOption(3, s);
    }
    
    publicvoid setOption(int var1, Object var2) throws IOException, NetException {
        String var3;
        switch(var1) {
        case0:
            var3 = (String)var2;
            this.socket.setTcpNoDelay(var3.equals("YES"));
            break;
        case1:
            var3 = (String)var2;
            if(var3.equals("YES")) {
                this.socket.setKeepAlive(true);
            }
        case2:
        default:
            break;
        case3:
            this.sockTimeout = Integer.parseInt((String)var2);
            this.socket.setSoTimeout(this.sockTimeout);
        }

    }
可用看到最后设置的是socket的soTimeout
实例

    @Test
    publicvoid testReadTimeout() throws SQLException {
        Connection connection = dataSource.getConnection();
        String sql = "select * from demo_table";
        PreparedStatement pstmt;
        try {
            pstmt = (PreparedStatement)connection.prepareStatement(sql);
            ResultSet rs = pstmt.executeQuery();
            int col = rs.getMetaData().getColumnCount();
            System.out.println("============================");
            while (rs.next()) {
                for (int i = 1; i <= col; i++) {
                    System.out.print(rs.getObject(i));
                }
                System.out.println("");
            }
            System.out.println("============================");
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            //close resources
        }
    }
超时错误输出
//部分数据输出......
java.sql.SQLRecoverableException: IO 错误: Socket read timed out
    at oracle.jdbc.driver.T4CPreparedStatement.fetch(T4CPreparedStatement.java:1128)
    at oracle.jdbc.driver.OracleResultSetImpl.close_or_fetch_from_next(OracleResultSetImpl.java:373)
    at oracle.jdbc.driver.OracleResultSetImpl.next(OracleResultSetImpl.java:277)
    at com.example.demo.DemoApplicationTests.testReadTimeout(DemoApplicationTests.java:68)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:497)
    at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
    at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
    at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
    at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
    at org.springframework.test.context.junit4.statements.RunBeforeTestMethodCallbacks.evaluate(RunBeforeTestMethodCallbacks.java:75)
    at org.springframework.test.context.junit4.statements.RunAfterTestMethodCallbacks.evaluate(RunAfterTestMethodCallbacks.java:86)
    at org.springframework.test.context.junit4.statements.SpringRepeat.evaluate(SpringRepeat.java:84)
    at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:325)
    at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:252)
    at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:94)
    at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
    at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
    at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
    at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
    at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
    at org.springframework.test.context.junit4.statements.RunBeforeTestClassCallbacks.evaluate(RunBeforeTestClassCallbacks.java:61)
    at org.springframework.test.context.junit4.statements.RunAfterTestClassCallbacks.evaluate(RunAfterTestClassCallbacks.java:70)
    at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
    at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.run(SpringJUnit4ClassRunner.java:191)
    at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
    at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:69)
    at com.intellij.rt.execution.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:234)
    at com.intellij.rt.execution.junit.JUnitStarter.main(JUnitStarter.java:74)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:497)
    at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
Caused by: oracle.net.ns.NetException: Socket read timed out
    at oracle.net.ns.Packet.receive(Packet.java:339)
    at oracle.net.ns.DataPacket.receive(DataPacket.java:106)
    at oracle.net.ns.NetInputStream.getNextPacket(NetInputStream.java:315)
    at oracle.net.ns.NetInputStream.read(NetInputStream.java:260)
    at oracle.net.ns.NetInputStream.read(NetInputStream.java:185)
    at oracle.net.ns.NetInputStream.read(NetInputStream.java:102)
    at oracle.jdbc.driver.T4CSocketInputStreamWrapper.readNextPacket(T4CSocketInputStreamWrapper.java:124)
    at oracle.jdbc.driver.T4CSocketInputStreamWrapper.read(T4CSocketInputStreamWrapper.java:80)
    at oracle.jdbc.driver.T4CMAREngine.unmarshalUB1(T4CMAREngine.java:1137)
    at oracle.jdbc.driver.T4CTTIfun.receive(T4CTTIfun.java:290)
    at oracle.jdbc.driver.T4CTTIfun.doRPC(T4CTTIfun.java:192)
    at oracle.jdbc.driver.T4C8Oall.doOALL(T4C8Oall.java:531)
    at oracle.jdbc.driver.T4CPreparedStatement.doOall8(T4CPreparedStatement.java:207)
    at oracle.jdbc.driver.T4CPreparedStatement.fetch(T4CPreparedStatement.java:1119)
    ... 35 more
刚开始会有数据输出，但是到了某个resultSet的next的时候，报了超时(close_or_fetch_from_next)，这个超时指定的是当result.next方法触发新的一批数据的拉取(当一个fetchSize的数据消费完之后，接下来的next会触发新一批数据的fetch)之后在timeout时间返回内没有收到数据库返回的数据。
oracle的jdbc默认的fetchSize为10，也就是每个fetch，如果超过指定时间没接收到数据，则抛出timeout异常。
小结

jdbc的socketTimeout值的设置要非常小心，不同数据库的jdbc driver设置不一样，特别是使用不同连接池的话，设置也可能不尽相同。对于严重依赖数据库操作的服务来说，非常有必要设置这个值，否则万一网络或数据库异常，会导致服务线程一直阻塞在java.net.SocketInputStream.socketRead0。
如果查询数据多，则会导致该线程持有的data list不能释放，相当于内存泄露，最后导致OOM
如果请求数据库操作很多且阻塞住了，会导致服务器可用的woker线程变少，严重则会导致服务不可用，nginx报504 Gateway Timeout
=====================================================

附：错误日志
org.apache.catalina.connector.ClientAbortException: java.io.IOException: Broken pipe
        at org.apache.catalina.connector.OutputBuffer.realWriteBytes(OutputBuffer.java:393)
        at org.apache.tomcat.util.buf.ByteChunk.flushBuffer(ByteChunk.java:426)
        at org.apache.catalina.connector.OutputBuffer.doFlush(OutputBuffer.java:342)
        at org.apache.catalina.connector.OutputBuffer.flush(OutputBuffer.java:317)
        at org.apache.catalina.connector.CoyoteOutputStream.flush(CoyoteOutputStream.java:110)
        at org.springframework.session.web.http.OnCommittedResponseWrapper$SaveContextServletOutputStream.flush(OnCommittedResponseWrapper.java:437)
        at com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter.write(FastJsonHttpMessageConverter.java:216)
        at org.springframework.web.servlet.mvc.method.annotation.HttpEntityMethodProcessor.handleReturnValue(HttpEntityMethodProcessor.java:183)
        at org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite.handleReturnValue(HandlerMethodReturnValueHandlerComposite.java:81)
        at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:126)
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:832)
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:743)
        at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)
        at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:961)
        at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:895)
        at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:967)
        at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:858)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:687)
        at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:843)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:790)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:292)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at com.yjf.common.web.CrossScriptingFilter.doFilter(CrossScriptingFilter.java:120)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at com.yiji.boot.web.common.ResponseHeaderFilter.doFilterInternal(ResponseHeaderFilter.java:62)
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at com.yiji.boot.actuator.acl.ActuatorACLFilter.doFilter(ActuatorACLFilter.java:36)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:121)
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:212)
        at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:106)
        at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:502)
        at org.apache.catalina.valves.RemoteIpValve.invoke(RemoteIpValve.java:676)
        at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:141)
        at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:79)
        at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:88)
        at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:522)
        at org.apache.coyote.http11.AbstractHttp11Processor.process(AbstractHttp11Processor.java:1095)
        at org.apache.coyote.AbstractProtocol$AbstractConnectionHandler.process(AbstractProtocol.java:672)
        at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1502)
        at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.run(NioEndpoint.java:1458)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)
Caused by: java.io.IOException: Broken pipe
        at sun.nio.ch.FileDispatcherImpl.write0(Native Method)
        at sun.nio.ch.SocketDispatcher.write(SocketDispatcher.java:47)
        at sun.nio.ch.IOUtil.writeFromNativeBuffer(IOUtil.java:93)
        at sun.nio.ch.IOUtil.write(IOUtil.java:65)
        at sun.nio.ch.SocketChannelImpl.write(SocketChannelImpl.java:471)
        at org.apache.tomcat.util.net.NioChannel.write(NioChannel.java:124)
        at org.apache.tomcat.util.net.NioBlockingSelector.write(NioBlockingSelector.java:101)
        at org.apache.tomcat.util.net.NioSelectorPool.write(NioSelectorPool.java:172)
        at org.apache.coyote.http11.InternalNioOutputBuffer.writeToSocket(InternalNioOutputBuffer.java:139)
        at org.apache.coyote.http11.InternalNioOutputBuffer.addToBB(InternalNioOutputBuffer.java:197)
        at org.apache.coyote.http11.InternalNioOutputBuffer.access$000(InternalNioOutputBuffer.java:41)
        at org.apache.coyote.http11.InternalNioOutputBuffer$SocketOutputBuffer.doWrite(InternalNioOutputBuffer.java:320)
        at org.apache.coyote.http11.filters.ChunkedOutputFilter.doWrite(ChunkedOutputFilter.java:116)
        at org.apache.coyote.http11.AbstractOutputBuffer.doWrite(AbstractOutputBuffer.java:256)
        at org.apache.coyote.Response.doWrite(Response.java:501)
        at org.apache.catalina.connector.OutputBuffer.realWriteBytes(OutputBuffer.java:388)
        ... 63 common frames omitted



2018-07-27 11:04:24.377 ERROR [http-nio-8313-exec-12] [dispatcherServlet]-- Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.IllegalStateException: getOutputStream() has already been called for this response] with root cause
java.lang.IllegalStateException: getOutputStream() has already been called for this response
        at org.apache.catalina.connector.Response.getWriter(Response.java:564)
        at org.apache.catalina.connector.ResponseFacade.getWriter(ResponseFacade.java:212)
        at javax.servlet.ServletResponseWrapper.getWriter(ServletResponseWrapper.java:152)
        at javax.servlet.ServletResponseWrapper.getWriter(ServletResponseWrapper.java:152)
        at org.springframework.session.web.http.OnCommittedResponseWrapper.getWriter(OnCommittedResponseWrapper.java:133)
        at javax.servlet.ServletResponseWrapper.getWriter(ServletResponseWrapper.java:152)
        at org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration$SpelView.render(ErrorMvcAutoConfiguration.java:195)
        at org.springframework.web.servlet.DispatcherServlet.render(DispatcherServlet.java:1246)
        at org.springframework.web.servlet.DispatcherServlet.processDispatchResult(DispatcherServlet.java:1029)
        at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:973)
        at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:895)
        at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:967)
        at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:858)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:687)
        at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:843)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:790)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:292)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at com.yjf.common.web.CrossScriptingFilter.doFilter(CrossScriptingFilter.java:92)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:101)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:240)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:207)
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:101)